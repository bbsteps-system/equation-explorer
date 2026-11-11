# Web app (legacy)

This directory holds the original browser app, `web/index.html`. It is the accepted
parity reference for the React Native migration and remains live and maintained
until a separate retirement decision.

## Run

```sh
# From the repository root
python3 -m http.server 8000 --directory web
```

Open <http://localhost:8000/>. The app also opens directly from
`web/index.html` in a modern browser.

## Constraints

- Single self-contained file: inline CSS and vanilla JavaScript.
- No build step, package manager, CDN, backend, account, persistence, or
  third-party dependency.
- No `eval()` or `Function()`; equation math uses exact rational arithmetic.
- Pointer Events provide mouse and touch interaction.
- Behavior and content are unchanged by the RN-M0 move; `web/index.html` is
  byte-identical to the tagged baseline (`web-baseline-2026-11-11`).

## Relationship to the native app

The React Native app will live in `mobile/` and is specified under
`docs/specs/react-native/`. It is a native rebuild, not a WebView wrapper of
this app. The web app stays at `web/` so the two codebases cannot be confused.