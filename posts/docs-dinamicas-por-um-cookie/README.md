---
title: minhas docs eram todas dinâmicas por causa de um cookie
description: como um cookie de idioma no layout raiz deixou todas as páginas do cd/ui dinâmicas e sem cache, e o que mudou quando o idioma virou parte da rota
date: 2026-10-06
---

# minhas docs eram todas dinâmicas por causa de um cookie

## tl;dr

clicar entre as páginas das docs do [cd/ui](https://cd-ui.vercel.app) demorava pra responder. no meu computador tava tudo rápido, então eu não via. quem mostrou o problema foi o próprio `next build`: **todas as rotas eram dinâmicas** e iam pro navegador com `no-store`. a causa era uma linha no layout raiz, que lia o idioma de um **cookie**. quando o idioma virou um **segmento da rota** (`/[locale]/...`), todas as páginas viraram estáticas, com cache de CDN. o lazy load da busca, que eu achei que ia ajudar, economizou só uns 4 KB.

## o sintoma

no site publicado, clicar num link da sidebar tinha um atraso. nada absurdo, mas dava pra sentir: você clica e nada acontece por um instante.

no `localhost` não aparecia. depois de quente, o servidor local respondia em 10 a 55 ms. o atraso só existia em produção, então chute de "deve ser o bundle" ou "deve ser animação" não ia me levar a lugar nenhum.

## o que o build mostrou

o `next build` imprime uma tabela de rotas com um símbolo pra cada uma: `●` é estática (gerada no build), `ƒ` é dinâmica (renderizada a cada requisição). a minha tabela era assim:

```text
ƒ /
ƒ /blocks
ƒ /blocks/[name]
ƒ /docs
ƒ /docs/components/[name]
ƒ /docs/instalacao
...
```

tudo `ƒ`. e a resposta de cada página vinha com:

```text
Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate
```

ou seja: todo clique ia até a função na Vercel, renderizava a página inteira de novo e não guardava nada no CDN. o prefetch do `<Link>` também quase não ajuda nesse caso, porque em rota dinâmica sem `loading.tsx` ele só busca o layout, não a página.

detalhe que me pegou: eu tinha `generateStaticParams` e `dynamicParams = false` nas páginas. achei que isso garantia página estática. não garantia nada, porque uma leitura de cookie em qualquer lugar da árvore deixa tudo dinâmico.

## a causa: uma linha no layout raiz

o site é bilíngue. o `proxy.ts` pegava `/pt/docs`, reescrevia pra `/docs` e guardava o idioma num cookie. o layout raiz lia esse cookie pra saber em que língua renderizar:

```ts
// src/app/layout.tsx (antes)
export default async function RootLayout({ children }: LayoutProps<"/">) {
  const locale = (await setLocale()) as Locale; // lê o cookie "locale"
  // ...
}
```

ler cookie é uma API dinâmica: o Next só sabe a resposta na hora da requisição. como isso tava no layout **raiz**, contaminava todas as páginas do site.

## a correção: o idioma vira parte da rota

em vez de esconder o idioma num cookie, ele passou a ser um segmento da URL que o Next conhece no build. as páginas foram pra `src/app/[locale]/` e o layout gera as duas línguas:

```ts
// src/app/[locale]/layout.tsx (depois)
export const dynamicParams = false;
export const generateStaticParams = () => locales.map((locale) => ({ locale }));
```

e o idioma sai dos params da rota, não do cookie:

```ts
// src/i18n/server.ts
import { locale as localeParam } from "next/root-params";

export async function getT() {
  const locale = (await localeParam()) as Locale;
  return { locale, tr: translations[locale], href: (path: string) => withLocale(locale, path) };
}
```

o `proxy.ts` ficou bem menor: agora ele só redireciona quem chega sem prefixo (`/docs`) pra `/pt/docs` ou `/en/docs`, olhando o cookie e depois o `Accept-Language`. não reescreve mais nada.

as URLs públicas continuaram iguais. a tabela do build depois:

```text
● /en
● /pt
● /en/docs/components/button
● /pt/blocks/login-01
...
```

e a resposta agora vem com `Cache-Control: s-maxage=31536000`. a página sai pronta do CDN, e o prefetch do `<Link>` consegue buscar a página inteira antes do clique.

## o que eu achava que ia ajudar e não ajudou

junto com isso eu passei a carregar o diálogo da busca (⌘K) só quando abre, com `lazy` e `Suspense`. a ideia era tirar o Base UI do primeiro carregamento.

medi antes e depois: o first load caiu uns **4 KB**, de 1.292.020 pra 1.287.574 bytes. em gzip, o total até subiu um pouco, porque o Turbopack reorganizou os chunks compartilhados. o código do Base UI que o diálogo usa já vinha em chunks que a página carrega de qualquer jeito.

deixei o lazy load (não custa nada), mas o ganho de verdade foi a página estática. se eu tivesse parado no "deve ser o bundle", tinha otimizado a coisa errada.

## o que eu levo disso

- **o atraso só em produção é uma pista.** se no `localhost` é rápido e no ar é lento, o problema provavelmente é cache ou rede, não o código do componente.
- **a tabela do build é a primeira coisa que eu olho agora.** um `ƒ` onde eu esperava `●` explica muita lentidão.
- **cookie, headers e search params no layout raiz deixam o site inteiro dinâmico.** se a informação dá pra colocar na URL, coloca na URL.
- **medir antes de comemorar.** o lazy load parecia a otimização óbvia e deu 4 KB.

o código tá no [repositório do cd/ui](https://github.com/di0rio/cd-ui), e a mudança inteira entrou na [v0.2.0](https://github.com/di0rio/cd-ui/blob/master/CHANGELOG.md).
