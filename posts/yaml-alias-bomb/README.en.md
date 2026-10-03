---
title: a 447-byte alias that froze my parser
description: how a tiny YAML rule took 34 s and 13 GB to decode in sentinel-forge, and how fuzzing found a panic along the way
date: 2026-10-03
---

# a 447-byte alias that froze my parser

## tl;dr

in sentinel-forge, detection rules are YAML files. one **447-byte** rule, far below my file size limit, took **about 34 seconds and 13 GB of allocation** to get through `rule.Parse`. it's the classic "billion laughs", but in YAML: an alias that references an alias that references an alias. `yaml.Strict()` doesn't bound that. the fix was to look at the AST **before** decoding and reject any alias node. as a bonus, `FuzzParse` found a panic inside go-yaml, which now becomes a regular error.

## the problem

sentinel-forge is a detection-as-code engine in Go. a rule is versioned YAML, and I treat rule files as untrusted input: the format is a restricted DSL with no expressions or templates, unknown fields are errors, there's a size limit (64 KiB) and windows are capped at 24h.

I thought that was enough. it wasn't.

what I had forgotten: YAML has **anchors and aliases**. you can tag a value with `&name` and reuse it with `*name`:

```yaml
colors: &base [blue, green]
light_theme: *base
dark_theme: *base
```

handy for not repeating config. the catch is that the decoder **expands** the value when it resolves the alias. and nothing stops an alias from pointing at a list full of other aliases. the shape of it is this:

```yaml
a: &a [lol, lol, lol]
b: &b [*a, *a, *a]
c: &c [*b, *b, *b]
d: &d [*c, *c, *c]
# ...and so on
```

each level multiplies the previous one. the text grows linearly, the decoded result grows exponentially. that's the "billion laughs": a few lines, billions of "lol" in memory.

## why a size limit doesn't help

my limit was 64 KiB per file. the problem rule was 447 bytes, less than 1% of the limit. the size of the **file** says nothing about the size of what comes out of decoding. limiting the input text is necessary, but it's the wrong place here.

and `yaml.Strict()` doesn't help either. it rejects unknown fields and duplicate keys. it has nothing to do with how much the decoder will expand.

the test I wrote generates nine levels of nine references, and the last alias goes into a rule field (`tags: *i`). for scale: 9 to the power of 9 is 387,420,489 items.

## what I did

the idea is simple: if a rule doesn't need aliases, **don't let the decoder get to them**. go-yaml's parser builds an AST before decoding, and in that AST an alias is just a node (`*ast.AliasNode`) pointing at the anchor, with nothing expanded. so I walk the AST, and if I find an alias node, I reject:

```go
// checkYAML requires exactly one document without aliases.
func checkYAML(data []byte) error {
	file, err := parser.ParseBytes(data, 0)
	if err != nil {
		return err
	}
	if len(file.Docs) != 1 {
		return fmt.Errorf("expected exactly one YAML document, found %d", len(file.Docs))
	}
	var f aliasFinder
	ast.Walk(&f, file.Docs[0])
	if f.found {
		return errors.New("YAML aliases are not allowed")
	}
	return nil
}

type aliasFinder struct{ found bool }

func (f *aliasFinder) Visit(n ast.Node) ast.Visitor {
	if _, ok := n.(*ast.AliasNode); ok {
		f.found = true
	}
	return f
}
```

and `Parse` calls it before `yaml.UnmarshalWithOptions`:

```go
if err := checkYAML(data); err != nil {
	return Rule{}, err
}
if err := yaml.UnmarshalWithOptions(data, &r, yaml.Strict()); err != nil {
	return Rule{}, err
}
```

I used the same check to require **a single document** per file (a `---` separating several documents is now an error too). a rule has no reason to use either feature, so the policy is simple: not allowed.

## how to test it without freezing CI

the regression test builds the bomb, calls `Parse` and checks two things: that an alias error came back **and** that it was fast. if someone removes `checkYAML` someday, the test doesn't lock up the machine for 34 seconds; it fails at 2:

```go
func TestParseRejectsAliasBomb(t *testing.T) {
	var b strings.Builder
	b.WriteString("a: &a [lol, lol, lol, lol, lol, lol, lol, lol, lol]\n")
	prev := "a"
	for _, name := range []string{"b", "c", "d", "e", "f", "g", "h", "i"} {
		b.WriteString(name + ": &" + name + " [" + strings.TrimSuffix(strings.Repeat("*"+prev+", ", 9), ", ") + "]\n")
		prev = name
	}
	src := b.String() + strings.Replace(valid, "group_by:", "tags: *i\ngroup_by:", 1)

	start := time.Now()
	_, err := Parse([]byte(src))
	if err == nil || !strings.Contains(err.Error(), "aliases") {
		t.Fatalf("err = %v, want alias rejection", err)
	}
	if d := time.Since(start); d > 2*time.Second {
		t.Fatalf("took %s, the document was probably expanded", d)
	}
}
```

the time assert is what matters here. only checking "it returned an error" doesn't catch the regression, because the decoder can happily return an error **after** burning 13 GB.

there's also a deep-nesting test (30 thousand `[` followed by 30 thousand `]`) that only expects an error, and cases in the `TestParseRejects` table for alias, second document and duplicate key.

## the fuzz that found another bug

along with the fix I wrote a `FuzzParse`. fuzzing in Go, in a few lines:

```go
func FuzzParse(f *testing.F) {
	f.Add(valid) // seeds: good and bad inputs
	f.Fuzz(func(t *testing.T, src string) {
		r, err := Parse([]byte(src))
		if err != nil {
			return
		}
		// whatever is accepted must be a complete, valid rule
		if err := r.Validate(); err != nil {
			t.Fatal(err)
		}
	})
}
```

run it with `go test -fuzz=FuzzParse ./internal/rule`. Go generates inputs by mutating the seeds until the property breaks or something crashes.

and it crashed: go-yaml (v1.19.2) **panics** on a malformed tag. one bad rule file took the whole process down. the input that found it has a stray `!` in the middle of the YAML:

```text
go test fuzz v1
string("id: AAAA-000\nversion: 1\nname: 0000000000000000000000000\nseverity: high\nwhen:\n type: 0000000000000000000000\nthreshold:\n  count: 10\n  window: 10s\ngroup_by:\n  ! 00000000000000000000000000000000000000000")
```

Go saves that file under `testdata/fuzz/FuzzParse/` and from then on a plain `go test` already runs it as a regression. I kept the file in the repo as a seed.

the fix was a `recover` in `Parse` that turns the panic into an error:

```go
defer func() {
	if p := recover(); p != nil {
		r, err = Rule{}, fmt.Errorf("invalid YAML (decoder panic: %v)", p)
	}
}()
```

relying on `recover` to handle a library bug isn't pretty, but for a tool that reads other people's files the rule is: a bad file is an error, never a crash.

## what I learned

- **an input size limit is not a cost limit.** what matters is how much the program spends processing, and in formats with references (like YAML aliases) that has no relation to the size of the text.
- **refusing the feature beats taming it.** if the rule doesn't need aliases, the cheapest defense is not accepting them.
- **DoS tests need a clock.** a time assert turns "froze CI" into "failed fast".
- **fuzzing finds what I don't think to test.** I would never have written the go-yaml panic as a test case.
- the commit with all of this (`fix(security): reject YAML alias bombs, bound state and input, sanitize output`) also capped the engine's state and the replay input, but that's for another post.

the code is at [di0rio/sentinel-forge](https://github.com/di0rio/sentinel-forge), on the `feat/log-parsers` branch.
