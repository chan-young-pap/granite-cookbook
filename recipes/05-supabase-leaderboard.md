# 05 — Supabase as your backend: an anon-RLS leaderboard for a no-server mini-app

A pattern for adding a global ranking (or any small shared dataset) to a mini-app **without operating a server**. Anyone — including unauthenticated users in the Toss WebView — can read the leaderboard and write their own row, and Postgres-level row-level security keeps the surface honest.

This is the leaderboard I shipped in **Coin Mining** (a 100-floor progression mini-game). Top-10 ranking, anonymous identity, ~10 LOC of network code on the client, no Edge Functions, no Express server.

## Symptom / motivation

You want a global leaderboard in a mini-app. Options:

- **Realtime DB (Firebase RTDB)** — fast but writing ranking math in JSON rules is painful, and pagination is awkward.
- **Cloud Functions / Edge Functions** — works but adds a service to operate, observe, and bill.
- **Your own backend** — too much for a 100-floor mini-game.

What you actually want: "I'd like to `SELECT TOP 10 ORDER BY score` from anywhere in the world, and have my users UPSERT their own row when they beat their best, without me running anything."

Supabase + RLS + anon key gives you exactly that.

## Schema

The full table for Coin Mining's leaderboard:

```sql
create table if not exists public.coin_mine_leaderboard (
  user_id          text       primary key,
  nickname         text       not null,
  best_depth       integer    not null default 1   check (best_depth between 1 and 100),
  daily_mines      integer    not null default 0   check (daily_mines >= 0),
  attendance_streak integer   not null default 0   check (attendance_streak >= 0),
  badge_count      integer    not null default 0   check (badge_count >= 0),
  updated_at       timestamptz not null default now()
);
```

A few things to notice:

- **`user_id text primary key`** — not a UUID, not an auth user. The client picks the ID (we'll get to identity in a moment) and upserts on this key.
- **`check` constraints** — the cheapest defense-in-depth you'll ever write. Even if a misbehaving client sends `best_depth = 9999`, the database rejects it.
- **No FK to `auth.users`** — we're not using Supabase Auth. Anonymous all the way.

## RLS policies: anyone can read, anyone can write their row

```sql
alter table public.coin_mine_leaderboard enable row level security;
grant select, insert, update on public.coin_mine_leaderboard to anon;

create policy "coin mine leaderboard public read"
on public.coin_mine_leaderboard
for select to anon using (true);

create policy "coin mine leaderboard public insert"
on public.coin_mine_leaderboard
for insert to anon with check (true);

create policy "coin mine leaderboard public update"
on public.coin_mine_leaderboard
for update to anon using (true) with check (true);
```

The `to anon` clause is the load-bearing part. Supabase's `anon` role is what you get with the anon API key — exactly what the client ships with. We're not granting `delete`, so even a hostile client can't nuke other rows.

This is permissive by design — `with check (true)` lets any row through. That's fine for an MVP leaderboard where the cost of cheating is "someone has fake top score" and the upside is no auth, no server, no friction. The path to harden is documented at the bottom of this recipe.

## Index for ranking

```sql
create index if not exists coin_mine_leaderboard_rank_idx
on public.coin_mine_leaderboard (
  best_depth desc,
  badge_count desc,
  daily_mines desc,
  updated_at asc
);
```

Tie-breaking goes: depth → badges → mines today → earliest update. The `desc/desc/desc/asc` ordering matches the exact `ORDER BY` clause the client uses, so the planner uses the index directly.

## Client: identity

The client needs a stable `user_id` to upsert against. The pattern:

```ts
// src/supabaseLeaderboard.ts (excerpt)
import { getTossAnonymousKey } from './tossBridge';

const LOCAL_USER_KEY = 'coin-mine-leaderboard-user-id';

function makeLocalUserId() {
  if (crypto.randomUUID) return crypto.randomUUID();
  return `local-${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 10)}`;
}

export async function getLeaderboardUserId() {
  const tossKey = await getTossAnonymousKey();
  if (tossKey) return `toss-${tossKey}`;

  const saved = localStorage.getItem(LOCAL_USER_KEY);
  if (saved) return saved;

  const nextId = makeLocalUserId();
  localStorage.setItem(LOCAL_USER_KEY, nextId);
  return nextId;
}
```

Three layers, in order of preference:

1. **Toss anonymous key.** When the app runs inside the Toss host, `framework.getAnonymousKey()` returns a stable identifier tied to the user across reinstalls (within the Toss app). Best possible identity without auth.
2. **`localStorage` UUID.** Survives across sessions on the same device, doesn't survive reinstall. Good enough for browser preview / dev.
3. **Generated UUID, persisted.** Last resort if `crypto.randomUUID` isn't available.

Prefixing (`toss-…` vs `local-…`) keeps the namespaces from colliding if the same person sometimes opens the app in Toss and sometimes via a direct browser URL.

See [recipe 06](./06-toss-framework-dynamic-import.md) for why `getTossAnonymousKey` has to live behind a dynamic import.

## Client: upsert

```ts
export async function upsertLeaderboardRecord(record: {
  userId: string;
  nickname: string;
  bestDepth: number;
  dailyMines: number;
  attendanceStreak: number;
  badgeCount: number;
}) {
  const response = await fetch(
    `${SUPABASE_URL}/rest/v1/coin_mine_leaderboard?on_conflict=user_id`,
    {
      method: 'POST',
      headers: {
        apikey: SUPABASE_ANON_KEY,
        Authorization: `Bearer ${SUPABASE_ANON_KEY}`,
        'Content-Type': 'application/json',
        Prefer: 'resolution=merge-duplicates,return=minimal',
      },
      body: JSON.stringify({
        user_id: record.userId,
        nickname: record.nickname,
        best_depth: Math.max(1, Math.min(100, record.bestDepth)),
        daily_mines: Math.max(0, record.dailyMines),
        attendance_streak: Math.max(0, record.attendanceStreak),
        badge_count: Math.max(0, record.badgeCount),
        updated_at: new Date().toISOString(),
      }),
    },
  );
  return response.ok;
}
```

Three lines do all the work:

- **`?on_conflict=user_id`** — tells PostgREST to treat this POST as an upsert keyed on `user_id`.
- **`Prefer: resolution=merge-duplicates`** — when there's a conflict, merge instead of erroring.
- **`Prefer: return=minimal`** — don't ship the new row back; we don't need it.

The client-side `Math.max/min` clamps mirror the database `check` constraints. Server-side wins on conflict, but failing client-side first avoids a round trip on obviously-bad input.

## Client: fetch top 10

```ts
export async function fetchLeaderboard(myUserId: string) {
  const query = new URLSearchParams({
    select: 'user_id,nickname,best_depth,daily_mines,attendance_streak,badge_count,updated_at',
    order: 'best_depth.desc,badge_count.desc,daily_mines.desc,updated_at.asc',
    limit: '10',
  });
  const response = await fetch(
    `${SUPABASE_URL}/rest/v1/coin_mine_leaderboard?${query.toString()}`,
    {
      headers: {
        apikey: SUPABASE_ANON_KEY,
        Authorization: `Bearer ${SUPABASE_ANON_KEY}`,
      },
    },
  );
  if (!response.ok) return [];
  const rows = await response.json();
  return rows.map((row, i) => ({
    rank: i + 1,
    ...row,
    isMine: row.user_id === myUserId,
  }));
}
```

The `order=` value matches the index exactly — so the planner uses it, and the query is O(log n) instead of a sort.

## What this trades off

- **Anyone can write any row.** A determined cheater can POST `{user_id: 'someoneelse', best_depth: 100}` and overwrite a real user's score. For a small mini-game this is fine; the cost of "fake top scorer" is low and the design avoids auth friction.
- **Nicknames can be anything.** Add a `check (length(nickname) between 1 and 16)` if you care.
- **No deletes.** We didn't grant `delete` to anon, so rows stick around forever.

## Hardening path, when you outgrow this

1. **Move writes behind an Edge Function.** Client posts `{score, signature}`; function validates the signature against a session token issued at app start. RLS becomes `to authenticated`.
2. **Replace `with check (true)` with score-progression checks.** `with check (new.best_depth >= old.best_depth or new.best_depth = old.best_depth + 1)` — a row can only advance.
3. **Add abuse heuristics.** Rate-limit updates per user_id via a separate table + trigger.

For an MVP shipped to a few hundred users, none of this is worth doing upfront. Ship the simple version; harden when someone actually starts cheating.

## Related

- [06 — Loading the Toss web framework without bootstrap hang](./06-toss-framework-dynamic-import.md) — `getTossAnonymousKey` is part of the framework that can't be top-level imported.
