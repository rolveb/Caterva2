# Caterva2 — `rolveb` fork

This is a fork of [`ironArray/Caterva2`](https://github.com/ironArray/Caterva2)
maintained by Rolv Erlend Bredesen for the Tangenvika bridge instrumentation
project and the `xarray-blosc2` ecosystem.

The fork tracks `main` of upstream and adds small, focused features on the
`feature/b2z-support` branch. Each feature is one commit so it can be
cherry-picked upstream cleanly when ready.

## Differences from upstream `main`

### 1. `.b2z` archive support (commit `1c445a4`)

Caterva2 now serves `.b2z` files transparently. A `.b2z` is a zip archive
of `.b2nd` files created by `xarray_blosc2.freeze_dataset()` — a portable
single-file form of a blosc2 dataset directory.

Changes in `caterva2/services/`:

- `srv_utils.py` — adds `expand_b2z()`: extracts `.b2z` to a sibling
  directory only if missing or stale (mtime-based). Avoids re-extraction
  on every access.
- `srv_utils.py::walk_files` — when discovering a `.b2z`, auto-expand and
  walk the resulting directory; skip the `.b2z` itself in listings.
- `server.py` — `/api/unfold/` endpoint accepts `.b2z` alongside the
  existing `.h5`/`.hdf5` handling.

Effect: drop a `.b2z` into the published root and Caterva2 serves the
contained `.b2nd` files via the normal list/info/data API. No manual
extraction step.

### 2. vlmeta-aware web viewer (commit `634e521`)

The HTML info page renders blosc2 NDArray metadata more usefully:

- Reads `vlmeta["dims"]` for dimension names (so the page shows
  `time × position` instead of `dim 0 × dim 1`).
- Reads `vlmeta["units"]` and surfaces it on the info page.
- Renders a 1-D SVG line chart when viewing a single row/column slice.
- Renders a 2-D canvas heatmap with viridis colormap for 2-D slices.
- Falls back to the plain-text view when `vlmeta` is missing.

Changes:

- `caterva2/services/server.py` — passes `vlmeta` into the info templates.
- `caterva2/services/templates/info.html` — dim-name lookup.
- `caterva2/services/templates/info_view.html` — SVG / canvas plot blocks.

This matches the `dims`/`units`/`long_name` conventions that
`xarray_blosc2.save_dataset()` writes into each `.b2nd`.

## Planned (not yet on this branch)

- **Serve non-`.b2nd` files** so a `recipe.yaml` next to the arrays can be
  fetched directly. Today the API filters listings to `.b2nd`.
- **JupyterLite viewer plug-in slot** so the web viewer can defer to
  custom renderers based on dataset metadata (e.g. ISO-8601 time-series,
  geographic CRS, mesh data).

## Branches

| Branch | Purpose |
|---|---|
| `main` | Tracks `ironArray/Caterva2:main` — no local changes |
| `feature/b2z-support` | The two commits above, applied on top of `main` |

To pull upstream:

```bash
git fetch upstream
git checkout main && git merge --ff-only upstream/main
git checkout feature/b2z-support && git rebase main
```

## Upstreaming

Both commits are designed to be cherry-pickable into `ironArray/Caterva2`
without follow-on changes. When the time comes, open one PR per commit
with the corresponding subset of the descriptions above.

## License

Upstream Caterva2 is BSD-3-Clause; this fork keeps that license. See
`LICENSE.txt`.
