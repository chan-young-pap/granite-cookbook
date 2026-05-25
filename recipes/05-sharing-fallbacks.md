# 05 — Share fallback chain inside a Toss WebView (WIP)

> **Status:** Stub. The shape of the fix is captured below; code samples will be filled in from the receipt-sharing flow of one of my mini-apps.

## Symptom (preview)

You call `navigator.share({ files: [...] })` from inside the Apps in Toss WebView and one of:

- The share sheet opens but the file attachment is dropped (text/URL only)
- `navigator.share` is undefined entirely (older iOS WKWebView)
- The share sheet opens, you cancel, and the page hangs in a "share-in-progress" state on subsequent calls

## Chain to implement (preview)

The fallback ladder I've landed on:

1. `navigator.share({ files })` — modern path
2. `navigator.share({ url, text })` — drop the file, share a link
3. `navigator.clipboard.write(...)` of the image blob — copy-to-clipboard with a toast
4. Trigger an `<a download>` — last resort, lets the user save and share manually
5. `window.open(url)` — for share-as-URL flows

Each step needs a feature-detect that's stricter than "the API exists" — on Toss WebView the API is sometimes present but no-ops.

---

*Filling in with real code from production share flow. Open a PR if you've solved this differently.*
