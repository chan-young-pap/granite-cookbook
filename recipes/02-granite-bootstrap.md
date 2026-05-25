# 02 — Granite bootstrap: `index.tsx` shim, `require.context`, and the `node` condition patch

Granite's dev server expects a few things from your project that aren't loudly documented. Miss any of them and the app either fails to start, silently pulls Node-only Firebase code into the bundle, or won't resolve your pages directory.

This recipe covers three small, tightly-coupled fixes you almost certainly need.

---

## A. Root `index.tsx` shim

### Symptom

`granite dev` boots fine, but launching from the Toss console / sandbox just shows the loading state forever. Or the bundle build prints `Unable to resolve module ./src/_app` and the entry never registers.

### Fix

Drop a one-line shim at the project root:

```ts
// index.tsx
export { default } from './src/_app';
```

Where `src/_app.tsx` is your actual `AppsInToss.registerApp(...)` call:

```tsx
// src/_app.tsx
import type { InitialProps } from '@granite-js/react-native';
import { AppsInToss } from '@apps-in-toss/framework';
import type { PropsWithChildren } from 'react';
import { context } from '../require.context';

function AppContainer({ children }: PropsWithChildren<InitialProps>) {
  return <>{children}</>;
}

export default AppsInToss.registerApp(AppContainer, { context });
```

### Why

Granite's Metro setup follows React Native's convention of looking for a root `index.tsx` to be the entry. If your real entry lives at `src/_app.tsx`, the shim is just the redirect Metro is looking for. Builds are also noticeably faster because Metro can short-circuit on the root resolution instead of walking your `src` tree to find an entry.

---

## B. `require.context` polyfill for pages

### Symptom

You write `require.context('./pages', true, /\.tsx$/)` to autoload your routes — works in Webpack/Vite, blows up in Metro:

```
TypeError: require.context is not a function
```

### Fix

Add a typed declaration so TypeScript doesn't yell, then import the helper Granite provides:

```ts
// require.context.ts
interface RequireContext {
  keys(): string[];
  (id: string): any;
  <T>(id: string): T;
  resolve(id: string): string;
  id: string;
}

declare const require: {
  context(path: string, deep?: boolean, filter?: RegExp): RequireContext;
};

export const context = require.context('./pages', true, /\.(tsx|ts)$/);
```

Then pass `context` into `AppsInToss.registerApp({ context })` (see snippet above).

### Why

Granite's Metro transformer recognizes `require.context` calls at build time and expands them into a static import map. The runtime `require.context` doesn't exist — the transform replaces it. The shape above just satisfies the TS checker; the actual value is rewritten before bundling.

---

## C. `plugin-compat` `conditionNames` patch (the Firebase / undici trap)

### Symptom

You add `firebase` to your mini-app and the bundle blows up with:

```
Unable to resolve module undici from .../node_modules/firebase/...
```

Or you get a runtime crash deep inside Firebase that mentions Node-only APIs (`stream`, `Buffer`, `process`).

### Root cause

`@apps-in-toss/plugin-compat` configures Metro's resolver with these condition names:

```js
["react-native", "require", "node", "default"]
```

The `node` condition tells the resolver to pick the **Node.js** entry of any package that ships multi-export `package.json`. Firebase ships a Node entry that depends on `undici`. `undici` is Node-only. Boom.

### Fix

Patch `plugin-compat` to swap `node` for `browser`:

```diff
- ["react-native", "require", "node", "default"]
+ ["react-native", "browser", "require", "default"]
```

The files you need to edit are roughly:

- `node_modules/@apps-in-toss/plugin-compat/dist/index.js` (around line 141)
- `node_modules/@apps-in-toss/plugin-compat/dist/index.cjs` (around line 170)

Line numbers drift between versions — search the file for the literal array.

**Freeze the change with `patch-package`** so it survives the next `npm install`:

```bash
npx patch-package @apps-in-toss/plugin-compat
```

Commit the resulting `patches/@apps-in-toss+plugin-compat+<version>.patch`.

### Why it works

Firebase ships separate entries for `browser`, `react-native`, and `node`. The RN entry is what you want; if that's missing for a sub-module, `browser` is the safe fallback (web crypto, fetch, etc. — all available in the Toss WebView). `node` should never be in the list for a WebView-hosted app.

### Gotcha

You'll regret this if you skip the `patch-package` step. The patched files in `node_modules` are not tracked by git, and CI / fresh clones will fail in the exact same way you just spent two hours debugging.

---

## Related

- [01 — Dev environment](./01-dev-environment.md) covers another patch (`fsevents`) you'll likely apply in the same session.
- A future recipe will cover detecting the `node`-condition trap *before* you ship — by grepping the production bundle for `undici` / `process.binding` strings.
