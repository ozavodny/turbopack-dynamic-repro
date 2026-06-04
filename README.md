# Turbopack `'use cache'` + `next/dynamic` factory-not-available repro

Minimal reproduction of a Next.js 16.2.4 turbopack bug where the SSR page entry
fails to reference the chunks containing factories for `next/dynamic` children.
At prerender the worker hits **"Module factory is not available."**

## Trigger

All four conditions must be present:

1. `experimental.cacheComponents` (`next.config.ts`).
2. A dynamic route segment with `generateStaticParams` (`app/[locale]/page.tsx`).
3. A server component carrying the `'use cache'` directive (`components/BlockList.tsx`).
4. That component renders a `next/dynamic()` import of a `'use client'` module.

Removing any one of them makes the build succeed.

## Reproduce

```bash
npm install
npm run build           # next build --turbopack -> FAILS
npm run build:webpack   # next build --webpack   -> succeeds
```

Expected turbopack output:

```
Error: Module 3088 was instantiated because it was required from module 74794,
but the module factory is not available.
Export encountered an error on /[locale]/page: /[locale], exiting the build.
```

## Root cause

The page entry `.next/server/app/[locale]/page.js` emits ~40 `R.c(chunkPath)`
calls to pre-register sibling SSR chunks. The chunk that contains the
factory for the dynamic-imported `'use client'` child is **never referenced**
from that list. When `'use cache'`'s renderer synchronously instantiates the
child during prerender, the factory map doesn't have the id and the runtime
throws from `instantiateModule()` in
`.next/server/chunks/ssr/[turbopack]_runtime.js`.

Webpack emits a complete chunk graph for the same source and does not have
this bug.
