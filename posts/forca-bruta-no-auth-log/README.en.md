---
title: how sentinel-forge finds brute force in a real auth.log
description: from the sshd log to an explainable detection - parser, normalized event, the AUTH-001 rule and a sliding window per IP
date: 2026-10-03
---

# how sentinel-forge finds brute force in a real auth.log

## tl;dr

sentinel-forge reads an sshd `auth.log`, turns each relevant line into a **normalized event**, runs the events through the `AUTH-001` rule (10 failures in 60 s, per source IP) and prints a detection that **explains where it came from**. along the way there's a classic trap: sshd logs the same attempt on two lines, and counting both would inflate the alert. that's why the `Invalid user` line became its own event type, `invalid_user`.

## the problem

sshd logs are loose text. a failed login attempt looks like this:

```text
Oct  3 14:32:14 bastion01 sshd[1303]: Failed password for root from 203.0.113.45 port 51240 ssh2
```

to detect brute force I need three things the text doesn't hand over: **what happened** (an authentication failure), **from where** (source IP) and **when** (and the classic log doesn't even have a year). and the rule shouldn't know anything about log formats: if tomorrow I read nginx or another log, the same rule has to keep working.

## the full path

```text
sshd line → parser → normalized event → AUTH-001 rule → per-IP window → detection
```

### 1. the parser: line becomes event

the sshd parser (`internal/parser/sshd.go`) matches each message with an anchored regex. the failure one, for example:

```go
sshdFailed = regexp.MustCompile(`^Failed (\S+) for (invalid user )?(.+) from (\S+) port ([0-9]{1,5})(?: ssh2)?$`)
```

the username can contain spaces (it's attacker-controlled), so it's matched greedily: the last ` from <ip> port <n>` on the line is the one sshd appended. the IP goes through `netip` to be validated and normalized, so one host is always one group.

the result is an event with fixed fields and no log text in it:

```go
event.Event{
	ID:        "sshd-<line number>",
	Timestamp: ts,
	Source:    "sshd",
	Type:      "authentication_failure",
	Actor:     &event.Actor{Username: user},
	Network:   &event.Network{SourceIP: ip},
	Target:    &event.Target{Host: host},
	Metadata:  md, // method, port, reason
}
```

two decisions that matter:

- **the ID comes from the line.** `sshd-14`, `sshd-15`... reading the same file twice gives the same events.
- **the year is a parameter.** classic syslog (`Oct  3 14:32:11`) has no year, so replay takes `--year 2026`. without it the result would change from one year to the next.

anything that isn't an event of interest (`Server listening`, `pam_unix(...)`, `Connection closed`) is counted as a skipped line. never an error.

### 2. the rule

this is `AUTH-001`, in `rules/authentication/AUTH-001.yml`, exactly as it is in the repo:

```yaml
id: AUTH-001
version: 1

name: Brute Force Authentication

description: >
  Detects repeated authentication failures
  from the same source IP.

severity: high

tags:
  - authentication
  - brute-force

when:
  type: authentication_failure

threshold:
  count: 10
  window: 60s

group_by:
  - network.sourceIp

attack:
  tactic: credential-access
  technique: T1110

references:
  - https://attack.mitre.org/techniques/T1110/
```

reading top to bottom: **10** `authentication_failure` **events in 60 s**, counted **separately per source IP**. that's MITRE ATT&CK technique T1110. the rule only talks about `type` and `network.sourceIp`, which are fields of the normalized event. it doesn't know sshd exists.

### 3. the engine: a sliding window on event time

for each event that matches `when`, the engine builds a key (rule + IP) and keeps the timestamps of that key's hits. on every new event it **drops whatever left the window** and checks whether the threshold was reached:

```go
hits := append(prune(e.windows[key], ev.Timestamp.Add(-window)), hit{ev.ID, ev.Timestamp})
if len(hits) < r.Threshold.Count {
	e.windows[key] = hits
	continue
}
```

the key point: the window uses `ev.Timestamp`, the time **that's in the log**, never the machine's clock. replaying the same log again always gives the same detection, today or a year from now. that's also why replay sorts events by timestamp before evaluating.

when the threshold is reached, the detection opens. if hits keep arriving within the window, they **extend** that same detection instead of opening another. one attack is one alert, not one alert per attempt.

## the output: a detection that explains itself

this is the real output of `go run ./cmd/sentinelforge replay --format sshd --year 2026 fixtures/authentication/auth.log`, run on the `feat/log-parsers` branch:

```text
SentinelForge Detection Engine

✓ 25 events processed
· 10 lines skipped (not security events)
✓ 2 rules evaluated

────────────────────────────────────────

🚨 Detection triggered

AUTH-001 v1
Brute Force Authentication

Severity:      HIGH
Group:         network.sourceIp=203.0.113.45
Matched:       14 events
Window:        42s (14:32:12 → 14:32:54)
Threshold:     10 events / 60s
MITRE ATT&CK:  T1110 (credential-access)

Reason:
14 events matched type=authentication_failure, reaching the threshold of 10 within 60s.

────────────────────────────────────────

1 detections in 8ms
exit 0
```

the test file has 35 lines: regular people logging in, `alice` getting the password wrong twice and then right, `bob` with one failure, and an attacker at `203.0.113.45` trying `admin`, `root`, `test`, `oracle`, `ubuntu` and `postgres`. the IPs are documentation ranges (`192.0.2.0/24`, `198.51.100.0/24` and `203.0.113.0/24`), nobody's real addresses.

and what the output shows: **only the attacker's IP** fired. alice and bob have few failures, each in a separate group, so they never get near 10. every field of the detection answers a question: which rule and version, which group, how many events, in what window, what threshold, which technique. you can check it against the log by hand.

also notice the window: it starts at `14:32:12`, not `14:32:11`. the `14:32:11` line is an `Invalid user admin`, which doesn't count. that brings me to the trap.

## the "Invalid user" double-counting trap

when someone tries a user that doesn't exist, sshd writes **two lines** for the same attempt:

```text
Oct  3 14:32:11 bastion01 sshd[1301]: Invalid user admin from 203.0.113.45 port 51234
Oct  3 14:32:12 bastion01 sshd[1301]: Failed password for invalid user admin from 203.0.113.45 port 51234 ssh2
```

if I treated both as `authentication_failure`, each attempt with a nonexistent user would be worth **two** hits in the window. the test log has 5 attempts like that (`admin`, `test`, `oracle`, `ubuntu`, `postgres`), so 5 real attempts would become 10 hits. an attacker guessing only nonexistent users would reach the threshold of 10 with half the attempts, and the alert's count (`Matched`) would lie.

I caught this while writing the parser: its commit already records that "Invalid user" is its own type so `AUTH-001` counts each attempt once. the line got the `invalid_user` type, and the rule only counts `authentication_failure`:

```go
case sshdInvalid.MatchString(msg):
	// sshd logs "Invalid user" before the "Failed password for invalid user" of the
	// same attempt; a separate type keeps AUTH-001 from counting one attempt twice.
	typ, user, ip = "invalid_user", m[1], m[2]
	md["reason"] = "invalid_user"
```

the `Failed password for invalid user ...` line is still an `authentication_failure`, just with `reason: invalid_user` in its metadata. the information isn't lost: the `invalid_user` event stays available for a future rule that wants to look at user enumeration, say, without polluting the failure count.

that's why the replay above has 25 events but only 14 count toward the detection: 17 `authentication_failure` in the whole log (14 from the attacker, 2 from alice and 1 from bob), 3 `authentication_success` and 5 `invalid_user`.

## what I learned

- **normalize before detecting.** the rule talks about events, not logs. changing the input format doesn't touch the detection.
- **one attempt is one attempt.** when the log has two lines for the same fact, modeling them as different types is more honest than filtering later.
- **event time, not the clock.** that's what makes the result reproducible and testable with a 35-line file.
- **a detection without an explanation is just a number.** showing rule, group, window and threshold makes the alert checkable.
- **real logs have traps.** the `Invalid user` / `Failed password for invalid user` pair only shows up when you look at a real `auth.log`, not an invented event.

the code is at [di0rio/sentinel-forge](https://github.com/di0rio/sentinel-forge), on the `feat/log-parsers` branch.
