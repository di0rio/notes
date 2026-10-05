---
title: a skill que escrevi pra IA parar de complicar meu código
description: um SKILL.md com prioridade de regras, escada de solução, escopo mínimo (sem global à toa) e quality gate, pra agente de código entregar a menor mudança correta
date: 2026-10-04
---

# a skill que escrevi pra IA parar de complicar meu código

atualizado em 2026-10-05: reescrevi a skill (de 44 seções pra uma versão bem menor) depois de compará-la com o ponytail. veja "o que mudou / o que aprendi com o ponytail" mais abaixo.

atualizado em 2026-10-05 (de novo): a skill virou **clean-code-ai**: juntei com um rascunho meu mais antigo, também chamado clean-code-ai, e ela substitui a pragmatic-code. veja "virou clean-code-ai" mais abaixo.

## tl;dr

agente de código tem um vício: pedir uma coisa pequena e receber de volta uma camada nova, uma interface com uma implementação só, uma dependência e um `catch {}` pra "garantir". escrevi um `SKILL.md` chamado **clean-code-ai** (antes **pragmatic-code**) que diz pro agente como analisar, gerar, corrigir, refatorar e revisar código com um objetivo só: **entregar a menor solução correta e segura, no menor escopo que funciona, seguindo as convenções do projeto.** o arquivo completo está em [`SKILL.md`](./SKILL.md). aqui eu resumo as partes que mais importam. é uma versão em andamento: eu continuo mexendo nela.

## o problema

código gerado por IA costuma funcionar e ainda assim ser ruim de manter. os sintomas que mais me incomodaram:

- abstração sem necessidade: repository, factory, service, tudo passando chamada pra frente sem fazer nada;
- dependência nova pra coisa que a stdlib ou o framework já resolvem;
- lint e TypeScript silenciados (`@ts-ignore`, `eslint-disable`) só pra passar;
- `try/catch` vazio escondendo erro;
- código comentado e TODO inútil largados no diff;
- mudança que quebra contrato de outra parte do sistema sem ninguém avisar;
- estilo e estado indo parar no escopo global só pra "reaproveitar" (esse eu só percebi usando o ponytail, mais abaixo).

como sou dev front-end júnior e uso IA pra aprender, isso pesa em dobro: se eu não percebo o excesso, ele vira o "jeito certo" na minha cabeça. então escrevi as regras que eu queria que o agente seguisse, e que eu mesmo quero aprender a seguir.

## a ideia central

antes de escrever código, o agente precisa entender o problema, ler os arquivos relevantes, achar quem consome aquilo e checar contratos. só depois escolhe a menor solução. e explica de forma proporcional: pra mudança trivial, uma frase basta.

> **Objetivo:** a menor solução correta, segura e compatível com o projeto, no menor escopo que funciona e seguindo as convenções que já existem.

repara que "menor" vem junto de "correta" e "segura". o menor diff só vale depois de entender o fluxo inteiro.

## o que eu acho mais importante

### 1. prioridade de regras

quando duas regras brigam, tem ordem pra decidir: segurança, correção e requisitos explícitos, contratos e integridade de dados, convenções do projeto, simplicidade, legibilidade, performance (só com necessidade concreta), e por último abstração e gosto. simplicidade fica no meio de propósito. e tem uma frase que existe pra evitar o pior efeito colateral de "faça simples":

> Simplicidade **nunca** justifica remover autenticação, autorização, validação, integridade, tratamento de erro, acessibilidade básica ou requisito explícito.

sem isso, "simplifica" vira "tira a validação".

### 2. escada de solução

antes de adicionar código, o agente desce essa escada e para no primeiro degrau que resolve direito:

```text
precisa existir? → reusar → stdlib → API nativa → dependência já instalada → poucas linhas → abstração → dependência nova
```

"uma dependência instalada não é uma dependência obrigatória". estar no `package.json` não é motivo pra usar.

### 3. escopo e localidade (minimal não é centralizado)

esse é o degrau novo, e o mais importante da versão 2. regra: reusar não significa jogar no global. coloca no menor escopo que funciona e segue como o projeto já faz.

- **estilo:** usa a convenção que já existe (Tailwind, CSS Modules, estilo colocalizado). valor de uso único fica local. só vira token global quando é uma decisão de design compartilhada, usada em 2+ lugares, e o projeto já tem sistema de tokens.
- **estado:** fica no componente, sobe só até o pai comum mais próximo. context, store ou singleton só quando o estado é de fato compartilhado.
- **helpers e constantes:** do lado de quem usa. vão pra `utils/` quando aparece o segundo consumidor de verdade.

```diff
- /* globals.css */
- :root { --contact-card-gap: 14px; }
- .contact-card { gap: var(--contact-card-gap); }
+ <div className="flex gap-3.5">   {/* um uso, projeto usa Tailwind */}
```

### 4. não silenciar ferramenta nem erro

`@ts-ignore`, `eslint-disable` e parecidos não entram só pra fazer a implementação passar. o certo é corrigir a causa. suspensão legítima existe, mas tem que ser mínima e explicada. mesma lógica pra `catch {}`: um erro é tratado, convertido, propagado ou ignorado de propósito, com comentário dizendo o porquê.

### 5. breaking change tem que ser dito

se a única solução correta quebra um contrato existente (função exportada, endpoint, schema, props, tipo), o agente não aplica em silêncio. ele aponta o contrato afetado, explica por que é necessário, avalia quem consome e, se der, propõe um caminho de transição. quebrar pode ser a decisão certa, mas tem que ser explícita.

### 6. quando minimal está errado

seção nova, nascida das falhas de skills "minimalistas": apagar código que parece morto mas é usado por string ou config; pular validação, erro ou acessibilidade pra encurtar diff; recusar um teste que a lógica precisa; espremer código legível num one-liner esperto; editar arquivo gerado; apagar comentário que explica o porquê; ignorar i18n. a skill manda fazer mais, não menos, nesses casos.

### 7. quality gate

uma checklist curta antes de entregar: requisito atendido sem extra, causa raiz e contratos, escopo mínimo (estilo, estado, helpers), abstração e global realmente necessários, fronteiras de confiança e erros intactos, nada silenciado, acessibilidade e i18n sem regressão, e os comandos reais do projeto rodados (sem inventar script).

## o que mudou / o que aprendi com o ponytail

o [ponytail](https://ponytail.dev/) é uma skill popular com a mesma ideia (código mínimo). instalei, usei e li o `SKILL.md` inteiro. ele me ensinou bastante sobre como escrever uma skill que o agente segue de verdade:

**o que eu peguei dele**

- é curto: dá pra ler em 2 minutos, e o agente de fato segue. a minha tinha 44 seções que se repetiam;
- `description` com gatilhos claros ("be lazy", "yagni", reclamação de over-engineering) e com o "quando não usar";
- formato de resposta fechado: código primeiro, no máximo três linhas, `skipped: X, add when Y`;
- níveis de intensidade (lite, full, ultra) e exemplos curtos de cada um;
- comentário `ponytail:` marcando o teto de uma simplificação, e a regra de deixar um check executável pra lógica não trivial.

**o que eu quis manter da minha**

- prioridade explícita: segurança, correção e contratos acima de simplicidade;
- fronteiras de confiança, autorização, integridade e concorrência de verdade (o ponytail só cita validação de passagem);
- breaking change tratado como decisão explícita, com migração;
- não silenciar ferramenta, quality gate, arquivo gerado.

**onde eu senti falta de algo**

- usando o ponytail no meu dia a dia, senti o "reutilizar" puxar pro global: o agente cria variável CSS em `:root`, token, classe em stylesheet global, store ou singleton pra valor de uso único, porque "centralizar" parece menos código. na prática vira acoplamento e diff maior. também não diz nada sobre acessibilidade de verdade, i18n, estilo do projeto ou apagar comentário que explica o porquê, e "uma linha antes de cinquenta" incentiva one-liner ilegível;
- na minha versão antiga, o problema era outro: longa demais, seções repetidas, pouco exemplo, `description` genérica que dispara pouco, e nenhuma palavra sobre escopo.

**o que eu fiz**

- cortei de ~2500 pra ~1400 palavras consolidando as seções repetidas;
- `description` nova no estilo ponytail (gatilhos, quando usar, quando não usar);
- adicionei "escopo e localidade" e "quando minimal está errado" (precauções), exemplos curtos com antes/depois, seção de frontend (React, Next.js, Tailwind, i18n, acessibilidade) e formato de resposta curto;
- **não** copiei os níveis de intensidade. pra quem usa no dia a dia eu não vi ganho real: pedir "mais enxuto" já resolve, e três modos viram mais uma coisa pro agente decidir. se eu sentir falta, volto.

a lição que eu levo: minimal não é sinônimo de centralizado nem de "o menor número de caracteres". é o menor mudança correta, no menor escopo, no estilo que o projeto já usa.

## virou clean-code-ai

atualização de 2026-10-05: juntei a skill com um rascunho meu mais antigo, também chamado clean-code-ai, e o resultado se chama **clean-code-ai** e substitui a pragmatic-code. o arquivo [`SKILL.md`](./SKILL.md) agora é a versão em português; em inglês fica em [`SKILL.en.md`](./SKILL.en.md). o que a versão nova tem a mais que a pragmatic-code:

- **os níveis voltaram:** lite, full e ultra, com `/clean-code-ai off` pra desligar. antes eu tinha escrito que não copiei os níveis; com essa junção eles voltaram, já que o rascunho antigo tinha;
- **a escada ganhou "remover código"** como degrau, e abstração só com 2+ usos reais hoje;
- **prioridade de regras:** contratos existentes e integridade dos dados viraram itens separados (eram um só), e "gosto pessoal de estilo e abstração" fica por último;
- **exemplo de estado no lugar errado:** `useState` + `useEffect` pra um valor que dá pra calcular durante o render;
- **critério de parada:** se o diff passa de ~50 linhas, cria arquivo novo, abstração, global ou dependência, a skill pede uma frase de justificativa concreta. se eu pedi a feature explicitamente, o arquivo novo já está justificado e ela segue sem travar;
- **arquivo regenerado por ferramenta** (dicionário de i18n, bloco que o `next dev` escreve): não reverte à mão, commita junto ou deixa a ferramenta cuidar;
- **comentário de limite** com prefixo `clean-code-ai:`, pra simplificação com teto conhecido.

os números do teste rápido (5 tarefas) foram medidos com a versão anterior, a pragmatic-code, e ainda não rodei de novo com a clean-code-ai.

## como usar

a skill agora tem repo próprio, com instalação e um teste rápido (5 tarefas pequenas, com e sem regras, avaliadas às cegas por outro modelo): [github.com/di0rio/clean-code-ai](https://github.com/di0rio/clean-code-ai).

duas formas, as que eu uso:

1. **Claude Code:** salva o arquivo como `SKILL.md` numa pasta em `~/.claude/skills/<nome>/` (por exemplo `~/.claude/skills/clean-code-ai/SKILL.md`). o arquivo precisa de frontmatter com `name` e `description`. o `description` é o que o agente lê pra decidir quando carregar a skill, então vale ajustar pro seu caso.

2. **outros agentes:** cola o conteúdo no arquivo de regras do projeto, tipo `AGENTS.md`, `CLAUDE.md` ou as regras do Cursor.

o arquivo completo está em [`./SKILL.md`](./SKILL.md).

## o que ainda falta

continua sendo uma versão em andamento. já rodei um teste rápido com 5 tarefas (está no repo), mas foi uma rodada só e com tarefas que eu mesmo escrevi. falta usar em projeto real por mais tempo pra saber se o agente segue de verdade, principalmente a parte de escopo. as regras nascem de coisa que me irritou no código gerado, então a lista cresce (e às vezes encolhe). se alguma regra não mudar o comportamento, ela sai: a skill também precisa seguir a própria regra de deletar primeiro.
