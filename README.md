# granite-cookbook

A practical playbook for shipping **Apps in Toss** mini-apps with the **Granite** framework, gathered from three production mini-apps built and operated by [@chan-young-pap](https://github.com/chan-young-pap).

> Status: **Active WIP.** This repo collects field notes from real deployments. Some chapters are polished; others are stubs being filled in as I revisit the source projects. Issues and PRs welcome.

---

## Why this exists

The Apps in Toss platform has a sizable Korean developer community, but public English / Korean documentation for the *day-2 problems* — sandbox quirks, AIT packaging gotchas, native module bootstrapping, store-review rejections — is almost nonexistent.

I solved most of these the hard way across three shipped mini-apps. This repo is an attempt to compress that pain into recipes the next indie dev doesn't have to rediscover.

The intended reader: a developer who has read the official Apps in Toss / Granite docs, run `granite init`, and immediately hit something the docs don't cover.

---

## Source projects

Recipes here are distilled from three apps I built and operate:

| App | Stack | Notable mechanics |
|---|---|---|
| **Coin Mining** | Granite + Supabase | 100-floor progression loop, global ranking, reward-ad integration, AIT packaging for review |
| **Malrang Receipt** | Granite + Vite + Canvas | 100-sticker collection dex, Canvas-rendered shareable PNG receipts, daily capsule loop |
| **Poco** (in development) | Granite + Firebase RTDB | Solo / small-group (≤8) daily mood check-in, time-capsule write/open, real-time room sync |

---

## Recipes

### Environment & bootstrap

- [01 — Dev environment: surviving `EMFILE` and the `useWatchman: false` trap](./recipes/01-dev-environment.md) ✅
- [02 — Granite bootstrap: `index.tsx` shim, `require.context`, and the `node` condition patch](./recipes/02-granite-bootstrap.md) ✅

### Backend patterns

- [03 — Firebase RTDB rules: room-scoped access with an immutable `authorId`](./recipes/03-firebase-rtdb-rules.md) ✅
- [05 — Supabase as your backend: an anon-RLS leaderboard for a no-server mini-app](./recipes/05-supabase-leaderboard.md) ✅

### Platform integration

- [06 — Loading `@apps-in-toss/web-framework` without breaking your boot (dynamic import, capability checks, reward-ad state machine)](./recipes/06-toss-framework-dynamic-import.md) ✅

### Store review

- [04 — The icon-rejection saga: when four SHA-identical PNGs still aren't enough](./recipes/04-icon-rejection.md) ✅

### Coming next (WIP)

- 07 — `.ait` packaging: what's actually inside, and how to inspect it
- 08 — `localStorage` quota guards & graceful degradation
- 09 — `ErrorBoundary` patterns for a Toss WebView host
- 10 — Sandbox testing: the dark-screen, app-scheme, and SDK-loading race-condition gallery

---

## Conventions used in recipes

Each recipe follows the same shape:

1. **Symptom** — what you see (error, rejection, blank screen, etc.)
2. **Root cause** — what's actually happening underneath
3. **Fix** — the smallest change that gets you unstuck
4. **Why it works** — so you can adapt it to your case
5. **Related** — links to other recipes touching the same machinery

Code samples are real, not pseudo-code. They've been sanitized of project-specific identifiers (Firebase project IDs, deployment IDs, mini-app IDs, local IPs) but otherwise match what's running in production.

---

## License

MIT. Use freely.

If a recipe saved you time, a star or a PR with your own war story is appreciated.

---

## Maintainer

**Chanyoung Park** ([@chan-young-pap](https://github.com/chan-young-pap)) — solo AI product builder shipping B2C mini-apps on Apps in Toss. Heavy [Claude Code](https://claude.com/claude-code) user; this entire repo was extracted and edited with it.
