---
title: como o sentinel-forge acha força bruta num auth.log de verdade
description: do log do sshd até uma detecção que se explica - parser, evento normalizado, regra AUTH-001 e janela deslizante por IP
date: 2026-10-03
---

# como o sentinel-forge acha força bruta num auth.log de verdade

## tl;dr

o sentinel-forge lê um `auth.log` do sshd, transforma cada linha relevante num **evento normalizado**, passa os eventos pela regra `AUTH-001` (10 falhas em 60 s, por IP de origem) e imprime uma detecção que **explica de onde veio**. no caminho tem uma armadilha clássica: o sshd loga a mesma tentativa em duas linhas, e contar as duas inflaria o alerta. por isso a linha `Invalid user` virou um tipo de evento próprio, o `invalid_user`.

## o problema

log de sshd é texto solto. uma tentativa de login falha parece com isso:

```text
Oct  3 14:32:14 bastion01 sshd[1303]: Failed password for root from 203.0.113.45 port 51240 ssh2
```

pra detectar força bruta eu preciso de três coisas que o texto não entrega de bandeja: **o que aconteceu** (falha de autenticação), **de onde** (IP de origem) e **quando** (e o log clássico nem tem ano). e a regra não deveria saber nada de formato de log: se amanhã eu ler nginx ou outro log, a mesma regra tem que continuar valendo.

## o caminho completo

```text
linha do sshd → parser → evento normalizado → regra AUTH-001 → janela por IP → detecção
```

### 1. o parser: linha vira evento

o parser do sshd (`internal/parser/sshd.go`) casa cada mensagem com um regex ancorado. a de falha, por exemplo:

```go
sshdFailed = regexp.MustCompile(`^Failed (\S+) for (invalid user )?(.+) from (\S+) port ([0-9]{1,5})(?: ssh2)?$`)
```

o nome de usuário pode ter espaço (é controlado por quem ataca), então ele é casado de forma gulosa: o último ` from <ip> port <n>` da linha é o que o sshd acrescentou. o IP passa por `netip` pra validar e normalizar, então um host é sempre um grupo só.

o resultado é um evento com campos fixos, sem nada de texto de log:

```go
event.Event{
	ID:        "sshd-<n da linha>",
	Timestamp: ts,
	Source:    "sshd",
	Type:      "authentication_failure",
	Actor:     &event.Actor{Username: user},
	Network:   &event.Network{SourceIP: ip},
	Target:    &event.Target{Host: host},
	Metadata:  md, // method, port, reason
}
```

duas decisões que importam:

- **o ID vem da linha.** `sshd-14`, `sshd-15`... ler o mesmo arquivo duas vezes dá os mesmos eventos.
- **o ano é parâmetro.** o syslog clássico (`Oct  3 14:32:11`) não tem ano, então o replay recebe `--year 2026`. sem isso o resultado mudaria de um ano pro outro.

o que não é evento de interesse (`Server listening`, `pam_unix(...)`, `Connection closed`) é contado como linha pulada. nunca é erro.

### 2. a regra

essa é a `AUTH-001`, em `rules/authentication/AUTH-001.yml`, do jeito que está no repo:

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

lendo de cima pra baixo: **10 eventos** `authentication_failure` **em 60 s**, contados **separadamente por IP de origem**. é a técnica T1110 do MITRE ATT&CK. a regra só fala de `type` e de `network.sourceIp`, que são campos do evento normalizado. ela não sabe que existe sshd.

### 3. o motor: janela deslizante no tempo do evento

pra cada evento que bate com o `when`, o motor monta uma chave (regra + IP) e guarda os horários dos acertos daquela chave. a cada evento novo, ele **descarta o que saiu da janela** e checa se bateu o limiar:

```go
hits := append(prune(e.windows[key], ev.Timestamp.Add(-window)), hit{ev.ID, ev.Timestamp})
if len(hits) < r.Threshold.Count {
	e.windows[key] = hits
	continue
}
```

o ponto central: a janela usa `ev.Timestamp`, o horário **que está no log**, e nunca o relógio da máquina. rodar o mesmo log de novo dá sempre a mesma detecção, hoje ou daqui a um ano. por isso o replay ordena os eventos por timestamp antes de avaliar.

quando o limiar é atingido, a detecção abre. se os acertos continuam chegando dentro da janela, eles **estendem** a mesma detecção em vez de abrir outra. um ataque vira um alerta, não um alerta por tentativa.

## a saída: detecção que se explica

esse é o resultado real de `go run ./cmd/sentinelforge replay --format sshd --year 2026 fixtures/authentication/auth.log`, rodado no branch `feat/log-parsers`:

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

o arquivo de teste tem 35 linhas: gente normal entrando, a `alice` errando a senha duas vezes e acertando, o `bob` com uma falha, e um atacante em `203.0.113.45` tentando `admin`, `root`, `test`, `oracle`, `ubuntu` e `postgres`. os IPs são de documentação (`192.0.2.0/24`, `198.51.100.0/24` e `203.0.113.0/24`), não de ninguém.

e o que a saída mostra: **só o IP do atacante** disparou. a alice e o bob têm poucas falhas, cada um num grupo separado, então nem chegam perto de 10. cada campo da detecção responde uma pergunta: qual regra e versão, qual grupo, quantos eventos, em que janela, qual o limiar, qual a técnica. dá pra conferir contra o log na mão.

repara também na janela: ela começa em `14:32:12`, não em `14:32:11`. a linha de `14:32:11` é um `Invalid user admin`, que não conta. isso me leva à armadilha.

## a armadilha do "Invalid user" contando em dobro

quando alguém tenta um usuário que não existe, o sshd escreve **duas linhas** pra mesma tentativa:

```text
Oct  3 14:32:11 bastion01 sshd[1301]: Invalid user admin from 203.0.113.45 port 51234
Oct  3 14:32:12 bastion01 sshd[1301]: Failed password for invalid user admin from 203.0.113.45 port 51234 ssh2
```

se eu tratasse as duas como `authentication_failure`, cada tentativa com usuário inexistente valeria **dois** acertos na janela. no log de teste são 5 tentativas assim (`admin`, `test`, `oracle`, `ubuntu`, `postgres`), então 5 tentativas reais virariam 10 acertos. um atacante chutando só usuários inexistentes bateria o limiar de 10 com a metade das tentativas, e a contagem do alerta (`Matched`) mentiria.

eu peguei isso na hora de escrever o parser: o commit dele já registra que "Invalid user" é um tipo próprio pra `AUTH-001` contar cada tentativa uma vez. a linha ganhou o tipo `invalid_user`, e a regra só conta `authentication_failure`:

```go
case sshdInvalid.MatchString(msg):
	// sshd logs "Invalid user" before the "Failed password for invalid user" of the
	// same attempt; a separate type keeps AUTH-001 from counting one attempt twice.
	typ, user, ip = "invalid_user", m[1], m[2]
	md["reason"] = "invalid_user"
```

a linha `Failed password for invalid user ...` continua sendo `authentication_failure`, só que com `reason: invalid_user` no metadata. a informação não se perde: o evento `invalid_user` fica disponível pra uma regra futura que queira, por exemplo, olhar enumeração de usuários, sem poluir a contagem de falhas.

é por isso que no replay acima são 25 eventos, mas só 14 contam na detecção: 17 `authentication_failure` no log todo (14 do atacante, 2 da alice e 1 do bob), 3 `authentication_success` e 5 `invalid_user`.

## o que eu aprendi

- **normalizar antes de detectar.** a regra fala de evento, não de log. trocar o formato de entrada não mexe na detecção.
- **uma tentativa é uma tentativa.** quando o log tem duas linhas pro mesmo fato, modelar como tipos diferentes é mais honesto do que filtrar depois.
- **tempo do evento, não do relógio.** é o que deixa o resultado reproduzível e testável com um arquivo de 35 linhas.
- **detecção sem explicação é só um número.** mostrar regra, grupo, janela e limiar faz o alerta ser conferível.
- **log real tem armadilha.** o par `Invalid user` / `Failed password for invalid user` só aparece quando você olha um `auth.log` de verdade, não um evento inventado.

o código está em [di0rio/sentinel-forge](https://github.com/di0rio/sentinel-forge), no branch `feat/log-parsers`.
