---
title: my docs were all dynamic because of a cookie
description: how a locale cookie in the root layout made every cd/ui page dynamic and uncached, and what changed when the locale became part of the route
date: 2026-10-06
---

# my docs were all dynamic because of a cookie

## tl;dr

clicking between pages in the [cd/ui](https://cd-ui.vercel.app) docs felt slow to respond. on my machine everything was fast, so I didn't see it. what showed the problem was `next build` itself: **every route was dynamic** and went to the browser with `no-store`. the cause was one line in the root layout that read the locale from a **cookie**. once the locale became a **route segment** (`/[locale]/...`), every page turned static and CDN-cached. lazy-loading the search, which I thought would help, saved only about 4 KB.

## the symptom

on the live site, clicking a sidebar link had a lag. nothing crazy, but you could feel it: you click and nothing happens for a moment.

on `localhost` it didn't show. once warm, the local server answered in 10 to 55 ms. the lag only existed in production, so guesses like "must be the bundle" or "must be an animation" weren't going to get me anywhere.

## what the build showed

`next build` prints a route table with a symbol for each route: `●` is static (generated at build time), `ƒ` is dynamic (rendered on every request). mine looked like this:

```text
ƒ /
ƒ /blocks
ƒ /blocks/[name]
ƒ /docs
ƒ /docs/components/[name]
ƒ /docs/instalacao
...
```

all `ƒ`. and every page response came with:

```text
Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate
```

so every click went all the way to the function on Vercel, rendered the whole page again and kept nothing on the CDN. `<Link>` prefetch barely helps here either, because on a dynamic route with no `loading.tsx` it only fetches the layout, not the page.

the detail that got me: I had `generateStaticParams` and `dynamicParams = false` on the pages. I thought that guaranteed a static page. it guaranteed nothing, because reading a cookie anywhere in the tree makes everything dynamic.

## the cause: one line in the root layout

the site is bilingual. `proxy.ts` took `/pt/docs`, rewrote it to `/docs` and stored the locale in a cookie. the root layout read that cookie to know which language to render:

```ts
// src/app/layout.tsx (before)
export default async function RootLayout({ children }: LayoutProps<"/">) {
  const locale = (await setLocale()) as Locale; // reads the "locale" cookie
  // ...
}
```

reading a cookie is a dynamic API: Next only knows the answer at request time. since it lived in the **root** layout, it tainted every page on the site.

## the fix: the locale becomes part of the route

instead of hiding the locale in a cookie, it became a URL segment that Next knows at build time. the pages moved to `src/app/[locale]/` and the layout generates both languages:

```ts
// src/app/[locale]/layout.tsx (after)
export const dynamicParams = false;
export const generateStaticParams = () => locales.map((locale) => ({ locale }));
```

and the locale comes from the route params, not the cookie:

```ts
// src/i18n/server.ts
import { locale as localeParam } from "next/root-params";

export async function getT() {
  const locale = (await localeParam()) as Locale;
  return { locale, tr: translations[locale], href: (path: string) => withLocale(locale, path) };
}
```

`proxy.ts` got much smaller: now it only redirects visitors who arrive without a prefix (`/docs`) to `/pt/docs` or `/en/docs`, checking the cookie and then `Accept-Language`. it doesn't rewrite anything anymore.

public URLs stayed the same. the build table after:

```text
● /en
● /pt
● /en/docs/components/button
● /pt/blocks/login-01
...
```

and the response now comes with `Cache-Control: s-maxage=31536000`. the page ships ready from the CDN, and `<Link>` prefetch can fetch the whole page before the click.

## what I thought would help and didn't

along with this I started loading the search dialog (⌘K) only when it opens, with `lazy` and `Suspense`. the idea was to take Base UI out of the first load.

I measured before and after: first load dropped about **4 KB**, from 1,292,020 to 1,287,574 bytes. gzipped, the total actually went up a little, because Turbopack reshuffled the shared chunks. the Base UI code the dialog uses was already in chunks the page loads anyway.

I kept the lazy load (it costs nothing), but the real win was the static page. if I had stopped at "must be the bundle", I'd have optimized the wrong thing.

## what I take from this

- **lag only in production is a clue.** if `localhost` is fast and the live site is slow, the problem is probably caching or the network, not the component code.
- **the build table is the first thing I check now.** a `ƒ` where I expected `●` explains a lot of slowness.
- **cookies, headers and search params in the root layout make the whole site dynamic.** if the information can live in the URL, put it in the URL.
- **measure before celebrating.** lazy loading looked like the obvious optimization and gave me 4 KB.

the code is in the [cd/ui repository](https://github.com/di0rio/cd-ui), and the whole change shipped in [v0.2.0](https://github.com/di0rio/cd-ui/blob/master/CHANGELOG.md).
