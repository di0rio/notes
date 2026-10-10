---
title: better-intl's t broke after an await
description: the error that took down my build, why React's use() goes away after an await, and the PR I sent to better-intl that got merged
date: 2026-10-09
---

# better-intl's t broke after an await

## tl;dr

[better-intl](https://github.com/luannzin/better-intl) is Luan's translation library, and I use it on my portfolio. on the server, reading `t` after an `await` in an async Server Component crashed with `Cannot read properties of null (reading 'use')`. the reason is that `t` gets the locale with React's `use()`, and `use()` stops working once the component resumes from an `await`. I sent a PR that stores the resolved locale and lets `t` read it directly, without `use()`. Luan merged it on the 8th.

## the error

an async page on my portfolio fetched data and then read `t`. `next build` crashed with this:

```text
TypeError: Cannot read properties of null (reading 'use')
```

I reproduced it in a minimal app, with Next 16.3.8 and React 19.3, with just this:

```tsx
export default async function Page() {
  await getData()
  return <p>{t.hello}</p> // crashes here
}
```

without the `await`, it works. with the `await` before it, it crashes.

## why it breaks

the server `t` is synchronous: you write `t.home.title` without `await`. to make that work, it gets the locale from a promise that reads the cookie, and unwraps that promise with React's `use()`.

`use()` only works while React is rendering the component. in an async component, after an `await`, the code keeps running but outside that moment, and React no longer has the "context" `use()` needs. so it crashes with that `null`.

## the fix

the idea was to stop depending on `use()` once the locale is known. the per-request cache, which used to store only the promise, now stores the value too:

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

when `t` reads the locale, it checks `value` first. if it's there, it returns it directly, without `use()`. if not, it uses `use()` as before.

for people using the library, the rule is simple: in an async component or in `generateMetadata`, call `await setLocale()` first. after that, `t` works anywhere in the function:

```tsx
export default async function Page() {
  await setLocale()
  const posts = await getPosts()

  return <h1>{t.blog.title}</h1>
}
```

and if someone reads `t` too early, they now get a better-intl message explaining this, instead of React's `TypeError`.

## why this way

- `t` stays synchronous. sync components and the client work exactly as before, so nothing breaks for existing users.
- no new API. `setLocale()` already existed, and the root layout already called it.
- the change lives in a single function, `createServerT`, plus a README section explaining the case.

I tested it in the minimal app: the build passes, the async page and `generateMetadata` follow the locale cookie (`pt`, `en` and no cookie), and reading `t` too early shows the new message.

## what I learned

- the minimal app is what moved the PR forward. it showed the error in five lines, without having to explain my whole portfolio.
- `use()` looks like an `await` you can use anywhere, but it isn't. it depends on React being in the middle of rendering.
- in a PR to someone else's project, changing little helps: no new API and nothing breaking for existing users makes it easy to review.

the PR is [luannzin/better-intl#1](https://github.com/luannzin/better-intl/pull/1).
