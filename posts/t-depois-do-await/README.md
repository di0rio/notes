---
title: o t do better-intl quebrava depois de um await
description: o erro que derrubou meu build, por que o use() do React some depois de um await, e o PR que mandei pro better-intl e foi aceito
date: 2026-10-09
---

# o t do better-intl quebrava depois de um await

## tl;dr

o [better-intl](https://github.com/luannzin/better-intl) é a biblioteca de tradução do Luan que eu uso no portfólio. no servidor, ler o `t` depois de um `await` num Server Component async quebrava com `Cannot read properties of null (reading 'use')`. o motivo é que o `t` pega o idioma com o `use()` do React, e o `use()` não funciona mais depois que o componente volta de um `await`. mandei um PR que guarda o idioma já resolvido e deixa o `t` ler ele direto, sem `use()`. o Luan aceitou no dia 8.

## o erro

uma página async do portfólio buscava dados e depois lia o `t`. o `next build` quebrou com isso:

```text
TypeError: Cannot read properties of null (reading 'use')
```

reproduzi num app mínimo, com Next 16.3.8 e React 19.3, só com isso:

```tsx
export default async function Page() {
  await getData()
  return <p>{t.hello}</p> // quebra aqui
}
```

sem o `await`, funciona. com o `await` antes, quebra.

## por que quebra

o `t` do servidor é síncrono: você escreve `t.home.title` sem `await`. pra isso, por dentro ele pega o idioma de uma promise que lê o cookie, e desembrulha essa promise com o `use()` do React.

o `use()` só funciona enquanto o React tá renderizando o componente. num componente async, depois de um `await`, o código continua rodando, mas fora daquele momento, e o React não tem mais o "contexto" que o `use()` precisa. aí ele quebra com aquele `null`.

## a correção

a ideia foi não depender do `use()` depois que o idioma já foi descoberto. o cache por requisição, que antes guardava só a promise, agora guarda também o valor:

```ts
const localeOnce = cache(() => {
  const entry: { promise: Promise<Locale>; value?: Locale } = {
    promise: resolveServerLocale(translations, config).then((locale) => {
      entry.value = locale;
      return locale;
    }),
  };
  return entry;
});
```

quando o `t` vai ler o idioma, ele olha primeiro o `value`. se já tem, devolve direto, sem `use()`. se ainda não tem, usa o `use()` como antes.

do lado de quem usa a biblioteca, a regra ficou simples: num componente async ou no `generateMetadata`, chama `await setLocale()` antes. depois disso o `t` funciona em qualquer lugar da função:

```tsx
export default async function Page() {
  await setLocale()
  const posts = await getPosts()

  return <h1>{t.blog.title}</h1>
}
```

e se alguém ler o `t` cedo demais, agora aparece uma mensagem do better-intl explicando isso, em vez do `TypeError` do React.

## por que desse jeito

- o `t` continua síncrono. componente síncrono e o lado do cliente funcionam igual antes, então nada quebra pra quem já usava.
- não tem API nova. o `setLocale()` já existia, e o layout raiz já chamava ele.
- a mudança ficou dentro de uma função só, a `createServerT`, mais uma seção no README explicando o caso.

testei no app mínimo: o build passa, a página async e o `generateMetadata` seguem o cookie de idioma (`pt`, `en` e sem cookie), e ler o `t` cedo demais mostra a mensagem nova.

## o que eu aprendi

- o app mínimo foi o que fez o PR andar. com ele dava pra mostrar o erro em cinco linhas, sem precisar explicar meu portfólio inteiro.
- o `use()` parece um `await` que dá pra usar em qualquer lugar, mas não é. ele depende do React estar no meio da renderização.
- num PR pra projeto dos outros, mudar pouco ajuda: sem API nova e sem quebrar quem já usa, fica fácil de revisar.

o PR é o [luannzin/better-intl#1](https://github.com/luannzin/better-intl/pull/1).
