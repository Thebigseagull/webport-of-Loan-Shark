# Loan Shark — HTML5 / browser port

Playable web build of Loan Shark. Nothing to install: it is plain static files, no
build step, no server-side code.

## Publish it on GitHub Pages

1. Put these files in the root of a repository (or in `docs/`).
2. Repository → **Settings → Pages → Build and deployment → Deploy from a branch**,
   pick the branch and the folder (`/` or `/docs`).
3. Open `https://<user>.github.io/<repo>/`.

`.nojekyll` is included, so Pages serves every file as-is.

## What is in here

| File | Size | What it is |
| --- | --- | --- |
| `index.html` | 10 KB | The page: engine config, credit intro, loading meter. |
| `index.js` | 317 KB | Godot's engine glue (the loader). |
| `index.wasm` | 42 MB | The engine itself; the host gzips/brotlis it on the wire. |
| `index.pck.part1`, `index.pck.part2`, `index.pck.part3` | ≈90 MB each | The game data, split into parts under the limit (see below). |
| `index.pck.head` | 64 KB | The first 64 KB of the pck, for the loader's header probe. |
| `sw.js` | 6 KB | Service worker: stitches the parts, caches the pck. |
| `index.png`, `index.icon.png`, … | 45 KB | Icons and worklets. |

Total ≈ 223 MB, and **no file is over 100 MiB**, which is the hard limit GitHub
enforces per file (`remote: error: File ... exceeds GitHub's file size limit of
100.00 MB`). Git LFS does not help, because GitHub Pages does not serve LFS objects.

## Why the pck is split

GitHub rejects any file larger than 100 MiB, and the whole pck is 192 MB. Splitting it
is transparent to the engine: `sw.js` intercepts the request for `index.pck`, streams
`part1` then `part2` back as one body, and answers the loader's 2-byte `Range` probe
from `index.pck.head`. The engine only ever sees a single, normal 192 MB pck.

Two consequences worth knowing:

* **Do not delete or rename `sw.js`** (or the parts) — without the worker the pck
  cannot be reassembled and the game will not start. It is not optional here.
* Re-deploys are picked up on the next reload: the worker's cache is keyed by a build
  id, so a new build installs a new worker, which drops the old cache.
* Service workers need `https://` or `localhost`; GitHub Pages is https, so this works
  as-is. Opening `index.html` straight from disk (`file://`) cannot work for any Godot
  web build, because the engine needs `fetch`.

## Loading behaviour

* A black intro fades **"Ported by Thebigseagull"** in and out, then a loading screen
  shows the same credit with a live meter — how many MB have been downloaded out of the
  total, e.g. `192.4 MB / 234.0 MB (82%)`.
* The intro and the credit disappear completely once the game is running.
* The first visit downloads everything; every later visit starts the pck from the
  browser's own cache (`Cache Storage`, via the worker) instead of the network.

## Rebuilding the build

The build is exported from a Godot 4.4.1 project with
`--export-release Web`, then finished by `patch_shell.py`, which adds the credit
intro, the loading meter, the worker and the pck split:

```bash
python D:/loan_shark_web_tools/patch_shell.py <build_dir> --copy-to <repo_dir>
```

The full port notes — texture policy, sizes, hosting requirements, what was changed
and why — live in `WEB_PORT_NOTES.md` next to the tooling.
