# 06 — Loading `@apps-in-toss/web-framework` without breaking your boot

## Symptom

You add `@apps-in-toss/web-framework` to a mini-app and `import` it like any other module:

```ts
import { share, getAnonymousKey, loadFullScreenAd, showFullScreenAd } from '@apps-in-toss/web-framework';
```

Then one of these happens:

- The app hangs on a blank screen in the Toss sandbox
- `vite dev` fails to start, or your build produces a bundle that throws on first eval in a browser preview
- It works inside the Toss host but the moment someone opens your dev URL in regular Safari, the page is dead

## Root cause

Top-level imports of `@apps-in-toss/web-framework` cause **Toss bridge constants to be evaluated at module-load time**. If your runtime isn't a real Toss host:

- The bridge constants reach into native APIs that don't exist
- The module's top-level code throws (or hangs awaiting something that never resolves)
- Vite / Metro can't recover — your whole bundle is dead before your code ever runs

The framework was designed to be loaded inside the Toss host, not in regular browser contexts. Anything that runs outside the host — local dev preview, web build, certain sandbox states — will trip this.

## Fix: lazy dynamic import with a cached promise

Centralize **all** access to the framework behind a single getter. Every Toss API call goes through this gate.

```ts
// src/tossBridge.ts
type TossFramework = typeof import('@apps-in-toss/web-framework');

let frameworkPromise: Promise<TossFramework | null> | null = null;

async function getTossFramework() {
  if (!frameworkPromise) {
    frameworkPromise = import('@apps-in-toss/web-framework').catch(() => null);
  }
  return frameworkPromise;
}
```

Three load-bearing details:

1. **`import(...)` is a dynamic import.** Vite / Metro emit it as a separate chunk; the top-level code of `@apps-in-toss/web-framework` runs only when something actually awaits this promise.
2. **`.catch(() => null)`** swallows any module-eval failure. In a browser-preview context where the framework can't even load, you get `null` instead of an unhandled rejection that kills your render.
3. **`frameworkPromise` is cached.** The dynamic chunk loads exactly once across the entire app lifetime, even with dozens of call sites.

## Use it: capability-checked, every time

Every framework API call goes through the gate and feature-detects the specific function it needs:

```ts
export async function shareMessage(message: string) {
  const framework = await getTossFramework();
  if (!framework?.share) return false;
  try {
    await framework.share({ message });
    return true;
  } catch {
    return false;
  }
}

export async function getTossAnonymousKey() {
  const framework = await getTossFramework();
  if (!framework?.getAnonymousKey) return null;
  try {
    return await framework.getAnonymousKey();
  } catch {
    return null;
  }
}

export async function loadNativeRecord(key: string) {
  const framework = await getTossFramework();
  if (!framework?.Storage?.getItem) return null;
  try {
    return await framework.Storage.getItem(key);
  } catch {
    return null;
  }
}
```

The pattern is identical every time:

1. `await getTossFramework()` — get the module or `null`
2. `if (!framework?.<api>) return <fallback>` — feature-detect; outside a Toss host the function won't exist
3. `try { await ... } catch { return <fallback> }` — even with the function present, individual calls can throw

Every call site of every Toss API has the right answer for "what if this isn't running in Toss?" wired in.

## Reward ads: a state machine, not a one-shot call

The pattern extends to ads, which are the most subtle Toss APIs. They run load → show → resolve over several event callbacks, and any step can hang, fail, or be cancelled by the user.

```ts
type AdResult = {
  ok: boolean;
  reason?: 'unsupported' | 'missing-id' | 'load-timeout' | 'show-timeout' | 'dismissed' | 'failed';
};

export async function showRewardedAd(placement: 'energy' | 'bonus' = 'energy'): Promise<AdResult> {
  const adGroupId = getRewardedAdGroupId(placement);
  const framework = await getTossFramework();

  if (!adGroupId) return { ok: false, reason: 'missing-id' };
  if (!framework?.loadFullScreenAd || !framework?.showFullScreenAd) {
    return { ok: false, reason: 'unsupported' };
  }

  const loadAd = framework.loadFullScreenAd;
  const showAd = framework.showFullScreenAd;

  // The host can also self-report unsupported via .isSupported().
  // Skip this check in local/sandbox so dev iteration stays usable.
  if (!isLocalOrSandboxRuntime() && (!loadAd.isSupported?.() || !showAd.isSupported?.())) {
    return { ok: false, reason: 'unsupported' };
  }

  return new Promise<AdResult>((resolve) => {
    let settled = false;
    let loaded = false;
    let unregisterLoad: (() => void) | undefined;
    let unregisterShow: (() => void) | undefined;

    const finish = (result: AdResult) => {
      if (settled) return;
      settled = true;
      unregisterLoad?.();
      unregisterShow?.();
      resolve(result);
    };

    unregisterLoad = loadAd({
      options: { adGroupId },
      onEvent: (event) => {
        if (event.type === 'loaded') {
          loaded = true;
          unregisterShow = showAd({
            options: { adGroupId },
            onEvent: (showEvent) => {
              if (showEvent.type === 'userEarnedReward') finish({ ok: true });
              if (showEvent.type === 'dismissed') finish({ ok: false, reason: 'dismissed' });
              if (showEvent.type === 'failedToShow') finish({ ok: false, reason: 'failed' });
            },
            onError: () => finish({ ok: false, reason: 'failed' }),
          });
        }
        if (event.type === 'failedToLoad' || event.type === 'failedToShow') {
          finish({ ok: false, reason: 'failed' });
        }
      },
      onError: () => finish({ ok: false, reason: 'failed' }),
    });

    window.setTimeout(() => {
      if (!loaded) finish({ ok: false, reason: 'load-timeout' });
    }, 10000);
    window.setTimeout(() => finish({ ok: false, reason: 'show-timeout' }), 45000);
  });
}
```

Patterns worth lifting verbatim:

- **`settled` guard.** Multiple event paths can fire after a result is final (timeout firing after a real success, etc.). The first call to `finish` wins; the rest are no-ops.
- **`unregister*` cleanup.** Toss event handlers don't auto-unregister on resolution. Holding onto them leaks listeners across ad sessions.
- **Two timeouts: load (10s), show (45s).** A user who never interacts with the ad isn't an error — it's a `show-timeout`. Different from `load-timeout`, which usually means SDK or network trouble.
- **`isSupported()` skipped in local/sandbox.** During development the runtime self-reports unsupported, even though you can still test the rendering. Honoring the check in dev would block iteration on the surrounding UX.

```ts
function isLocalOrSandboxRuntime() {
  const host = window.location.hostname;
  return host === 'localhost' || host === '127.0.0.1' || /^192\.168\./.test(host);
}
```

## Why centralize all this in one file

When every framework call goes through `src/tossBridge.ts`, three things become easy:

1. **Stubbing for tests.** Mock the bridge module; you don't have to mock `@apps-in-toss/web-framework` itself.
2. **Swapping the framework version.** When `@apps-in-toss/web-framework` releases a breaking change, you have one file to update.
3. **Auditing for "what does this app need from Toss?"** — the bridge is the literal API surface, in <200 lines.

Anything outside `tossBridge.ts` that reaches directly into `@apps-in-toss/web-framework` is a bug waiting to be found by a user opening a dev URL.

## Related

- [02 — Granite bootstrap](./02-granite-bootstrap.md) covers the same family of "framework bites you at module-load time" problems, on the Granite-RN side.
- [05 — Supabase leaderboard](./05-supabase-leaderboard.md) uses `getTossAnonymousKey()` from the bridge to derive user identity.
