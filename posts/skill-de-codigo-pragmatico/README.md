---
title: a skill que escrevi pra IA parar de complicar meu código
description: um SKILL.md com prioridade de regras, hierarquia de solução e quality gate, pra agente de código entregar a menor solução correta, segura e legível
date: 2026-10-04
---

# a skill que escrevi pra IA parar de complicar meu código

## tl;dr

agente de código tem um vício: pedir uma coisa pequena e receber de volta uma camada nova, uma interface com uma implementação só, uma dependência e um `catch {}` pra "garantir". escrevi um `SKILL.md` chamado **Self-documenting Code and Pragmatic Engineering for AI** que diz pro agente como analisar, gerar, corrigir, refatorar e revisar código com um objetivo só: **entregar a menor solução correta, segura e legível, compatível com o projeto atual.** o arquivo completo está em [`SKILL.md`](./SKILL.md). aqui eu resumo as partes que mais importam. é uma versão em andamento: eu continuo mexendo nela.

## o problema

código gerado por IA costuma funcionar e ainda assim ser ruim de manter. os sintomas que mais me incomodaram:

- abstração sem necessidade: repository, factory, service, tudo passando chamada pra frente sem fazer nada;
- dependência nova pra coisa que a stdlib ou o framework já resolvem;
- lint e TypeScript silenciados (`@ts-ignore`, `eslint-disable`) só pra passar;
- `try/catch` vazio escondendo erro;
- código comentado e TODO inútil largados no diff;
- mudança que quebra contrato de outra parte do sistema sem ninguém avisar.

como sou dev front-end júnior e uso IA pra aprender, isso pesa em dobro: se eu não percebo o excesso, ele vira o "jeito certo" na minha cabeça. então escrevi as regras que eu queria que o agente seguisse, e que eu mesmo quero aprender a seguir.

## a ideia central

antes de escrever código, o agente precisa entender o problema, ler os arquivos relevantes, achar quem consome aquilo e checar contratos. só depois escolhe a menor solução. e explica a estratégia de forma proporcional: pra mudança trivial, uma frase basta.

> **Goal: deliver the smallest correct, safe, readable solution compatible with the current project.**

repara que "menor" vem junto de "correta" e "segura". o menor diff só vale depois de entender o fluxo inteiro.

## o que eu acho mais importante

### 1. prioridade de regras

quando duas regras brigam, tem ordem pra decidir:

1. segurança
2. correção funcional
3. requisitos explícitos
4. integridade de contrato e compatibilidade
5. integridade de dados
6. simplicidade (KISS/YAGNI)
7. legibilidade
8. manutenibilidade
9. performance, quando há necessidade concreta
10. abstração
11. preferência de estilo

simplicidade fica no meio de propósito. e tem uma frase que existe pra evitar o pior efeito colateral de "faça simples":

> Simplicity never justifies removing authorization, validation, integrity, error handling, or explicit requirements.

sem isso, "simplifica" vira "tira a validação".

### 2. hierarquia de solução

antes de adicionar código, o agente desce essa escada e para no primeiro degrau que resolve direito:

```text
remover → reusar → stdlib → API nativa do runtime/framework → dependência já instalada → código simples → abstração → dependência nova → infraestrutura
```

e tem um detalhe que eu gosto: "uma dependência instalada não é uma dependência obrigatória". estar no `package.json` não é motivo pra usar.

### 3. anti-overengineering

a skill lista o que **não** pode entrar no automático: Clean Architecture, DDD, Repository, Unit of Work, Factory, Adapter, Strategy, CQRS, Event Bus, container de injeção de dependência, microsserviços. a frase que resume: padrões são ferramentas, não requisitos. e camada vazia também não vale:

```text
Route → Controller → Service → Repository → Database
```

se nenhuma dessas camadas tem responsabilidade de verdade, é só cerimônia.

### 4. não silenciar ferramenta

`@ts-ignore`, `eslint-disable` e parecidos não entram só pra fazer a implementação passar. o certo é corrigir a causa. suspensão legítima existe, mas tem que ser mínima e justificável. mesma lógica pra erro: nada de

```ts
try {
  await operation();
} catch {}
```

um erro é tratado, convertido, propagado ou ignorado de propósito, quando isso é parte do contrato.

### 5. breaking change tem que ser dito

se a única solução correta quebra um contrato existente (função exportada, endpoint, schema, tipo), o agente não aplica em silêncio. ele aponta o contrato afetado, explica por que é necessário, avalia quem consome e, se der, propõe um caminho compatível ou de transição. quebrar pode ser a decisão certa, mas tem que ser uma decisão explícita, não efeito colateral.

### 6. quality gate

uma checklist antes de entregar. algumas das perguntas:

- requisito atendido, sem feature extra?
- menor diff correto, causa raiz corrigida?
- consumidores e contratos verificados?
- abstração e dependência realmente necessárias?
- sem comentário redundante, código comentado, TODO trivial ou log temporário?
- fronteiras de confiança, autorização e integridade de dados preservadas?
- erros tratados? typecheck, lint, testes e build rodados quando aplicável?
- se tem breaking change, foi sinalizada?

e uma regra pra não inventar: usar os comandos reais do projeto, sem criar script.

### 7. a regra de ouro

são perguntas em cascata, na ordem:

> Can I remove it? → Does it already exist? → Does the platform already solve it? → Does an existing dependency solve it? → Do a few simple lines solve it? → Is there a real need for abstraction?

só depois disso se considera dependência nova ou infraestrutura. e fecha com: entenda o problema por completo antes de tentar resolver do jeito mínimo.

## como usar

duas formas, as que eu uso:

1. **Claude Code:** salva o arquivo como `SKILL.md` numa pasta em `~/.claude/skills/<nome>/` (por exemplo `~/.claude/skills/pragmatic-code/SKILL.md`). o arquivo precisa de frontmatter com `name` e `description`:

   ```yaml
   ---
   name: pragmatic-code
   description: Self-documenting code and pragmatic engineering rules for AI coding agents - deliver the smallest correct, safe, readable solution compatible with the current project.
   ---
   ```

   o `description` é o que o agente lê pra decidir quando carregar a skill, então vale ajustar pro seu caso.

2. **outros agentes:** cola o conteúdo no arquivo de regras do projeto, tipo `AGENTS.md`, `CLAUDE.md` ou as regras do Cursor.

o arquivo completo, com todas as seções (TypeScript, banco, upload, API, estado global, testes e o resto), está em [`./SKILL.md`](./SKILL.md).

## o que ainda falta

é uma versão em andamento. algumas seções ainda estão grandes demais pra um prompt, e eu estou testando o que o agente realmente segue e o que ele ignora. as regras nascem de coisa que me irritou no código gerado, então a lista cresce (e às vezes encolhe) conforme eu uso. se eu descobrir que alguma regra não muda o comportamento, ela sai: a skill também precisa seguir a própria regra de deletar primeiro.
