# oracle.exe

Static deployment of the self-contained **oracle.exe** app (Y2K magic 8-ball interview + oracle).

## Run locally

Requires a local HTTP server (the bundle unpacks gzip assets with `DecompressionStream`; `file://` is unreliable).

```bash
npm run dev
```

Then open [http://localhost:8080](http://localhost:8080).

Or without npm:

```bash
python3 -m http.server 8080
```

## Deploy (Option A — static)

This repo is a single `index.html` (~4 MB) with all assets embedded. Deploy the project root to any static host:

| Host | How |
|------|-----|
| **Netlify** | Drag the folder to [app.netlify.com/drop](https://app.netlify.com/drop), or connect this repo (`netlify.toml` included). |
| **Vercel** | `npx vercel` from this directory. |
| **GitHub Pages** | Push to `gh-pages` or enable Pages on `main`; publish `/` (root). |
| **Cloudflare Pages** | Build command: *(none)* · Output directory: `/` |

Hash routes (`#/ball/<uuid>`) work on static hosts without extra rewrite rules.

## Files

- `index.html` — full app (from `oracle.exe _standalone_.html`)
- `docs/oracle.css` — extracted stylesheet (reference; styles are inlined in the bundle)
- `docs/HANDOFF.md` — engineering spec for a future Vite rebuild

## Notes

- **LLM**: The reference build calls `window.claude.complete`. Outside Cursor, answers fall back to canned lines unless you host your own API and patch the bundle.
- **Storage**: Profiles use `localStorage` key `oracle.exe.v1` in the browser.
