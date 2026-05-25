# 01 — Dev environment: surviving `EMFILE` and the `useWatchman: false` trap

## Symptom

You install [Watchman](https://facebook.github.io/watchman/) following standard React Native advice, run `granite dev`, and Metro crashes with:

```
Error: EMFILE: too many open files, watch '/.../node_modules/...'
```

Raising `ulimit -n` to something huge (`65536`, `1048576`) helps for a few minutes, then the same error returns. Installing Watchman appears to do nothing.

## Root cause

Granite (`@granite-js/native`) **explicitly disables Watchman** in its Metro config — it forces `useWatchman: false`. Metro then falls back to `NodeWatcher`, which opens a file descriptor per watched file. On a mid-sized mini-app, that quickly exceeds the default macOS FD limit.

Watchman being installed on your machine is irrelevant because Granite isn't using it.

The path forward is to give Metro a watcher that **doesn't** need one FD per file: `fsevents`.

## Fix

Install `fsevents` as a dev dependency, **skipping its postinstall** (the native build fails on Node 25, and you don't need it — the prebuilt `.node` ships in the package):

```bash
npm install --save-dev --ignore-scripts fsevents@^2.3.3
```

Pin Node to 22.x for the project (the prebuilt `fsevents` binary is happy here):

```bash
# .nvmrc
22
```

And in `package.json`, force Node 22 in your dev script:

```jsonc
{
  "scripts": {
    "dev": "PATH=/opt/homebrew/opt/node@22/bin:$PATH granite dev --host 0.0.0.0 --port 8083"
  }
}
```

You can drop the `ulimit` workaround entirely once `fsevents` is on the path.

## Why it works

`fsevents` uses macOS's kernel-level FSEvents API. One subscription covers a whole directory tree; no per-file FDs. Metro autodetects it when present and routes file watching through it instead of `NodeWatcher`. With Watchman disabled by Granite, this is the only viable path on macOS.

## Gotchas

- **Node 25 will silently break this.** `fsevents`' native build script fails on Node 25.x. Use `--ignore-scripts` on install and pin to Node 22 in dev. If you don't, you'll see the EMFILE error come back with no obvious explanation.
- **Don't try to convince Granite to use Watchman.** Even if you locate the Metro config and flip `useWatchman: true`, downstream Granite plugins assume the file is what they shipped. You'll lose more time than you save.
- **`patch-package` your fixes.** If you end up editing anything inside `node_modules`, freeze it with [patch-package](https://github.com/ds300/patch-package) before your next `npm install` wipes it.

## Related

- [02 — Granite bootstrap](./02-granite-bootstrap.md) covers another `node_modules` patch you'll likely need at the same time (the `plugin-compat` `conditionNames` fix).
