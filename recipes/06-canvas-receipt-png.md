# 06 — Canvas 2D → 1080×1920 PNG inside a Toss WebView (WIP)

> **Status:** Stub. The receipt-PNG generator in one of my shipped mini-apps renders a 1080×1920 image from layered Canvas 2D draws (background, stickers, dynamic text, dotted divider, serial number). Recipe to follow.

## Outline

- Why HTML-to-image libraries (`html2canvas`, `dom-to-image`) are unreliable inside Toss WebView — and what to use instead
- Canvas sizing: render at 1080×1920 regardless of device DPR, then scale down for preview
- Font loading: webfont readiness before first draw, otherwise letters fall back mid-render
- Sticker layering: pre-decode `Image` objects, draw to a single canvas, export as `toBlob('image/png')`
- Hand-off into the share fallback chain (see [05 — Share fallbacks](./05-sharing-fallbacks.md))

## Known pitfalls

- `OffscreenCanvas` is unreliable in WKWebView; stick to a hidden on-DOM `<canvas>`
- `toBlob` callback can run on a microtask boundary that loses your share-sheet user-gesture window — generate the blob *before* the user taps share, not after
- Memory: a 1080×1920 ARGB canvas is ~8 MB raw; don't keep three of them around

---

*Concrete code samples to follow.*
