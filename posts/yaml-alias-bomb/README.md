---
title: um alias de 447 bytes que travou meu parser
description: como uma regra YAML minúscula levou 34 s e 13 GB pra decodar no sentinel-forge, e como o fuzz achou um panic de brinde
date: 2026-10-03
---

# um alias de 447 bytes que travou meu parser

## tl;dr

no sentinel-forge as regras de detecção são arquivos YAML. uma regra de **447 bytes**, bem abaixo do limite de tamanho que eu tinha, levou **uns 34 segundos e 13 GB de alocação** pra passar em `rule.Parse`. é o clássico "billion laughs", só que em YAML: alias que referencia alias que referencia alias. o `yaml.Strict()` não segura isso. a correção foi olhar o AST **antes** de decodar e recusar qualquer nó de alias. de quebra, o `FuzzParse` achou um panic dentro do go-yaml, que agora vira erro normal.

## o problema

o sentinel-forge é um motor de detecção como código em Go. a regra é YAML versionado, e eu trato arquivo de regra como entrada não confiável: o formato é uma DSL restrita, sem expressão nem template, campo desconhecido dá erro, tem limite de tamanho (64 KiB) e janela máxima de 24h.

eu achava que isso bastava. não bastava.

o que eu tinha esquecido: YAML tem **âncora e alias**. dá pra marcar um valor com `&nome` e reusar com `*nome`:

```yaml
cores: &base [azul, verde]
tema_claro: *base
tema_escuro: *base
```

útil pra não repetir config. o problema é que o decoder, quando resolve o alias, **expande** o valor. e nada impede um alias de apontar pra uma lista cheia de outros aliases. a forma da coisa é essa:

```yaml
a: &a [lol, lol, lol]
b: &b [*a, *a, *a]
c: &c [*b, *b, *b]
d: &d [*c, *c, *c]
# ...e assim por diante
```

cada nível multiplica o anterior. o texto cresce linear, o resultado decodado cresce exponencial. esse é o "billion laughs": poucas linhas, bilhões de "lol" na memória.

## por que limite de tamanho não ajuda

meu limite era de 64 KiB por arquivo. a regra problemática tinha 447 bytes, menos de 1% do limite. o tamanho do **arquivo** não diz nada sobre o tamanho do que sai da decodificação. limitar o texto de entrada é necessário, mas aqui é o lugar errado.

e o `yaml.Strict()` também não ajuda. ele serve pra rejeitar campo desconhecido e chave duplicada. não tem nada a ver com quanto o decoder vai expandir.

o teste que eu escrevi gera nove níveis de nove referências, e o último alias entra num campo da regra (`tags: *i`). só pra ter noção: 9 elevado a 9 dá 387.420.489 itens.

## o que eu fiz

a ideia é simples: se a regra não precisa de alias, **não deixo o decoder chegar neles**. o parser do go-yaml monta um AST antes de decodar, e nesse AST o alias é só um nó (`*ast.AliasNode`) apontando pra âncora, sem expandir nada. então eu ando pelo AST, e se achar um nó de alias, rejeito:

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

e o `Parse` chama isso antes do `yaml.UnmarshalWithOptions`:

```go
if err := checkYAML(data); err != nil {
	return Rule{}, err
}
if err := yaml.UnmarshalWithOptions(data, &r, yaml.Strict()); err != nil {
	return Rule{}, err
}
```

aproveitei a mesma checagem pra exigir **um documento só** por arquivo (`---` separando vários documentos também passa a dar erro). regra não tem motivo pra usar nenhuma das duas coisas, então a política fica simples: não pode.

## como testar sem travar a CI

o teste de regressão monta a bomba, chama `Parse` e checa duas coisas: que veio erro de alias **e** que foi rápido. se alguém um dia tirar o `checkYAML`, o teste não trava a máquina por 34 segundos; ele falha em 2:

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

o assert de tempo é o que importa aqui. só checar "deu erro" não pega a regressão, porque o decoder pode muito bem dar erro **depois** de gastar 13 GB.

também tem um teste de aninhamento profundo (30 mil `[` seguidos de 30 mil `]`) que só espera um erro, e casos na tabela `TestParseRejects` pra alias, segundo documento e chave duplicada.

## o fuzz que achou outro bug

junto com a correção eu escrevi um `FuzzParse`. fuzzing em Go, em poucas linhas:

```go
func FuzzParse(f *testing.F) {
	f.Add(valid) // sementes: entradas boas e ruins
	f.Fuzz(func(t *testing.T, src string) {
		r, err := Parse([]byte(src))
		if err != nil {
			return
		}
		// o que for aceito tem que ser uma regra completa e válida
		if err := r.Validate(); err != nil {
			t.Fatal(err)
		}
	})
}
```

roda com `go test -fuzz=FuzzParse ./internal/rule`. o Go gera entradas mutando as sementes até quebrar a propriedade ou dar crash.

e deu crash: o go-yaml (v1.19.2) **dá panic** em tag malformada. um arquivo de regra ruim derrubava o processo inteiro. a entrada que achou o problema tem uma `!` solta no meio do YAML:

```text
go test fuzz v1
string("id: AAAA-000\nversion: 1\nname: 0000000000000000000000000\nseverity: high\nwhen:\n type: 0000000000000000000000\nthreshold:\n  count: 10\n  window: 10s\ngroup_by:\n  ! 00000000000000000000000000000000000000000")
```

o Go salva esse arquivo em `testdata/fuzz/FuzzParse/` e, daí em diante, um `go test` normal já roda ele como regressão. eu mantive o arquivo no repo como semente.

a correção foi um `recover` no `Parse`, que transforma o panic em erro:

```go
defer func() {
	if p := recover(); p != nil {
		r, err = Rule{}, fmt.Errorf("invalid YAML (decoder panic: %v)", p)
	}
}()
```

não é bonito confiar em `recover` pra lidar com bug de biblioteca, mas pra uma ferramenta que lê arquivo de terceiros a regra é: arquivo ruim é erro, nunca crash.

## o que eu aprendi

- **limite de tamanho de entrada não é limite de custo.** o que importa é quanto o programa gasta processando, e em formatos com referência (como o alias do YAML) isso não tem relação com o tamanho do texto.
- **recusar o recurso é melhor que domá-lo.** se a regra não precisa de alias, a defesa mais barata é não aceitar.
- **teste de DoS precisa de relógio.** um assert de tempo transforma "travou a CI" em "falhou rápido".
- **fuzz acha o que eu não penso em testar.** o panic do go-yaml eu jamais teria escrito como caso de teste.
- o commit com tudo isso (`fix(security): reject YAML alias bombs, bound state and input, sanitize output`) também limitou o estado do motor e a leitura do replay, mas isso fica pra outro post.

o código está em [di0rio/sentinel-forge](https://github.com/di0rio/sentinel-forge), no branch `feat/log-parsers`.
