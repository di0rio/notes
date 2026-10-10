---
title: a skill que eu fiz pra IA escrever do meu jeito
description: a primeira versão da humanizar deixava passar texto com cara de IA. o que eu fui apontando na bio do portfólio, rodada por rodada, virou a versão nova
date: 2026-10-09
---

# a skill que eu fiz pra IA escrever do meu jeito

## tl;dr

a **humanizar** é uma skill pra agente de IA (Claude Code, Codex, Cursor) que reescreve texto com cara de IA e escreve texto novo na voz de quem assina. a primeira versão tinha a lista de tique de sempre ("alavancar", "jornada", travessão pra todo lado) e mesmo assim deixava passar muita coisa. usei ela pra arrumar os textos do meu portfólio e fui apontando o que tava errado, rodada por rodada. cada ponto virou uma regra. a versão nova tá no meu repositório de skills, em português e inglês.

## o problema

tirar "alavancar" de um texto é fácil. o difícil é o que sobra depois: um texto limpo, sem nenhuma palavra proibida, que ainda soa como IA. ele é vago e fala de um jeito que eu nunca falaria.

a bio da home do portfólio foi o teste. levou umas seis rodadas.

## rodada por rodada

**1. o ponto de partida.** a bio dizia:

> Trabalho na Loopscape fazendo produto de ponta a ponta. Meu forte é o front, mas pego do banco e da API até o último estado da tela.

e, um pouco abaixo, "nada entra no projeto sem eu entender o que o código tá fazendo". não tem palavra inflada nenhuma ali, e mesmo assim eu achei cru, com cara de IA. o problema era outro: "do banco até a tela" soa abrangente e não mostra nada, e a última frase é um lema, daqueles que cabem numa caneca.

**2. a referência.** mandei uma bio em inglês de outra pessoa e falei que a ideia era aquela. a IA copiou a estrutura dos parágrafos, traduziu uma expressão quase literal ("software people spend their whole workday in" virou "pra quem passa o expediente nela") e quase trouxe junto fato que não é meu, tipo ERP e perfil no X. não era pra copiar. era pra pegar o espírito.

**3. a imagem esperta.** a versão seguinte tinha "gosto da parte que não aparece no print: o carregamento, a mensagem de erro, a tela que abre rápido". eu não entendi o que era "a parte que não aparece no print". se nem eu entendi, o leitor também não ia entender. e tinha mais dois problemas: dois-pontos seguido de uma lista de três, e "no Loopvet", quando o certo é "na Loopvet".

**4. o jargão.** vieram "estado de carregamento e de erro", "rota nova na API", "mudança no banco". é tudo verdade, mas não é assim que eu me apresento. o que eu curto é motion e design, o resto é resto.

**5. a palavra.** "o lab aqui do site é onde eu brinco com isso" virou "onde eu demonstro isso". detalhe pequeno, mas a palavra tem que ser a que eu usaria.

**6. a frase da IA.** pra falar de como eu uso IA, a skill me deu três opções e eu não quis nenhuma. escrevi eu mesmo, e ficou essa.

a bio que ficou:

> Sou dev front-end júnior na Loopscape e trabalho na Loopvet, um sistema pra clínica veterinária. O que eu mais curto é motion e design, e o lab aqui do site é onde eu demonstro isso.
>
> No tempo livre eu toco o cd/ui e o converter-hub e estudo segurança no Sentinel Forge. Uso IA todo dia pra aprender mais rápido, mas tento entender ao máximo o que acontece no código e reviso bastante. Ainda tô aprendendo o que é o ideal.

## o que mudou na skill

cada rodada virou uma regra:

- **ache a voz antes de escrever.** a skill lê de 2 a 4 textos que a pessoa já publicou e anota traços concretos, tipo se a pessoa escreve "pra" ou "para" e que palavra ela nunca usaria. o chat é um degrau mais solto que o site: eu escrevo "tlgd" no chat, mas não no portfólio.
- **modo inspirar.** quando você manda o texto de outra pessoa, ela diz em uma linha o que a referência faz bem e escreve isso com os seus fatos. tem um teste no fim: lendo as duas lado a lado, dá pra perceber que uma foi feita em cima da outra? se dá, reescreve.
- **fato só com fonte.** detalhe concreto só entra se veio do texto original, dos arquivos do projeto ou do que a pessoa disse. sem fonte, ela pergunta. foi o que faltou com "no Loopvet".
- **palavra que a pessoa usaria.** em bio e texto de site, nada de termo técnico que ela não falaria numa conversa. o detalhe técnico fica pro estudo de caso.
- **texto curto ganha 2 ou 3 versões**, cada uma com um ângulo, pra eu escolher em vez de ficar no vai e volta.
- **uma checagem no fim**, rodada no próprio texto antes de entregar: frase de efeito no fim do parágrafo, dois-pontos com três itens, "do X até Y", fato sem fonte.

o catálogo de tiques ganhou uma seção nova, a dos que sobrevivem à primeira revisão. quase todos os exemplos dela saíram dessa bio.

## o que eu aprendi

- o "antes" não tinha nenhuma palavra proibida. o problema era ser vago, e lista de palavra proibida não pega isso.
- a IA acerta o tom genérico de dev, não o meu. ela só chega perto quando lê o que eu já escrevi.
- a frase sobre IA na bio quem escreveu fui eu, depois de recusar três opções. a skill ajudou a limpar o resto e a entender por que as versões dela não soavam como eu.

a skill tá no meu repositório de skills: [github.com/di0rio/cd-skills](https://github.com/di0rio/cd-skills/tree/main/skills/humanizar), e na página [/skills](https://cauadiorio.vercel.app/skills) do portfólio.
