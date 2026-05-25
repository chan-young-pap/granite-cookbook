# 04 — The icon-rejection saga: when four SHA-identical PNGs still aren't enough

A field report rather than a clean recipe. The fix at the bottom is real; the value is in the debugging path, because the reviewer's rejection message points you somewhere it isn't.

## Symptom

You upload a `.ait` build of your mini-app to the Apps in Toss console for review. The reviewer comes back with:

> **기타** — 브랜드 아이콘 설정이 정상적으로 되어 있는지 확인해주세요.
> `granite.config.ts`에 콘솔에 등록한 아이콘과 동일한 아이콘을 등록해야 해요.
>
> *("Other — please verify your brand icon settings. The icon registered in `granite.config.ts` must match the one registered in the console.")*

You check. The icons match. You re-upload. Rejected. Same message.

In my case this loop ran **four times** before I figured out what was happening.

## What I verified (and what didn't help)

By the fourth round I had:

- The console-registered **light** icon
- The console-registered **dark** icon
- The local `assets/logo.png` referenced by `granite.config.ts > brand.icon`
- The local `assets/logo_dark.png`

…all four with **byte-identical SHA-1 hashes**. Literally the same file four times. The rejection still came back, same wording.

Things I tried that didn't move the needle:

- Re-downloading the console-registered icons and re-uploading them locally
- Setting `logo_dark.png` to be a literal copy of `logo.png`
- Adding unofficial `brand.darkIcon` / `brand.iconDark` keys to `granite.config.ts` (silently ignored; SDK schema doesn't define them)
- New `.ait` build, new deployment ID

## What's actually going on

The rejection message is a **catch-all** that the reviewer (or an automated pre-screen) returns when *anything* in the brand-related console setup is missing. The icon mismatch is one possible cause — but not the only one. Other things that surface the same message in practice:

- Mismatch between `primaryColor` in `granite.config.ts` and the color registered in the console
- `displayName` differing from the console-registered app name by a stray space or character
- Required brand fields (dark icon, splash configuration) missing on the **console** side, not the code side
- Stale review cache: a re-uploaded build can inherit the previous review verdict if the upstream queue hasn't cycled

The message says "your code is wrong." Often it means "your console form has a gap."

## What actually got me unstuck

After exhausting the code-side fixes, I escalated through the only path Anthropic / the SDK can't help with: **channel-talk to a human reviewer.** Template that worked for me:

```
안녕하세요. <앱이름> 앱이 N회 반려되었습니다.

반려 사유: "<문구 그대로 붙여넣기>"

확인된 사항:
- 콘솔 라이트 아이콘 SHA: <hex>
- 콘솔 다크 아이콘 SHA: <hex>
- granite.config.ts의 icon(<경로>) SHA: <hex>
- assets/logo_dark.png SHA: <hex>

네 파일이 픽셀 단위로 동일합니다. 같은 사유로 반복 반려되고 있어, 실제 사유 또는 우회 경로를 알려주실 수 있을지 문의드립니다.

최신 deploymentId: <id>
```

The SHA-hash table is the unlock. It moves the conversation from "the user didn't follow instructions" to "the user has verified the literal claim and needs the actual reason," and a human picks it up.

## Practical debugging order, next time

If you hit this same rejection:

1. **Verify the literal claim first** (SHAs match). This eliminates ~50% of cases and gives you ammunition for step 4.
2. **Check non-icon brand fields**: `primaryColor` exact value, `displayName` exact characters (no trailing space), `appName` matches console slug.
3. **Try a new app entry** if you suspect review-queue caching: register the same code as a new app slug (e.g. `myapp-v2`) and re-submit. Confirms or rules out cache.
4. **Open channel-talk with the SHA evidence table.** Don't iterate on guesses past round two; the human review path is faster than the n-th re-upload.

## What I'd want from the platform

A reviewer-side checkbox for *which* brand field actually failed, surfaced in the rejection message. The current single-message-for-many-causes design wastes both sides' time. If anyone from the Apps in Toss team reads this — that's the request.

## Related

- A future recipe will cover what's actually inside a `.ait` archive and how to grep its manifest, so you can verify the *built* artifact contains what you think it does.
