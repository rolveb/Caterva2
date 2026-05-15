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

## Deferred work — prompts for the next contributor

These are designed-out but not yet implemented. Each is a self-contained
prompt: drop it into Claude / a contributor and they can pick it up.

### Prompt A — Serve non-`.b2nd` files

> Today `/api/list/<root>/<path>/` filters listings to `.b2nd` and
> `/api/datasets/.../<file>.json` returns 404 even when the file exists
> on disk. Producers writing `recipe.yaml` / `dataset.json` next to their
> arrays cannot retrieve them through Caterva2.
>
> Add a second endpoint `/api/files/<root>/<path>` that streams any file
> in the published root (with the same auth as `/api/datasets/`), and
> extend `walk_files` in `srv_utils.py` to optionally include non-`.b2nd`
> entries when a `include_all=true` query param is set. Keep the default
> listing behaviour unchanged for backwards compatibility.
>
> Acceptance: `curl http://host:8081/api/files/@public/path/recipe.yaml`
> returns the raw YAML; `curl '.../api/list/@public/path/?include_all=true'`
> returns a list including `recipe.yaml` alongside the `.b2nd` entries.
> Unit test exercises both endpoints against a temp root containing one
> `.b2nd` and one `.yaml`.

### Prompt B — Pluggable viewer registry + JupyterLite kernel

> The current web viewer in `info_view.html` (1-D SVG line chart, 2-D
> canvas heatmap — see commit 634e521) hard-codes two render strategies.
> Real datasets need many more: ISO-8601 time-series scrubber, folium /
> deck.gl maps when `attrs.crs` is set, UGRID mesh viewer, multi-channel
> uncertainty plots, etc.
>
> Build a plug-in surface where each viewer declares a predicate over
> `vlmeta` / dataset attrs and renders into the existing info page slot.
> Two layers:
>
> 1. **Server-side discovery**
>    - `caterva2/services/viewers.py`: load viewers from
>      `importlib.metadata.entry_points(group="caterva2.viewers")` plus
>      a static `caterva2-viewers.toml` mapping name → HTML/JS template.
>    - New endpoint `GET /api/viewers/<root>/<path>` returns the list of
>      matching viewers for that dataset as
>      `[{name, kind, predicate_match, url}]`, in priority order. `kind`
>      is `"html"` (static template) or `"jupyterlite"` (Pyodide kernel).
>
> 2. **Client-side rendering**
>    - `info_view.html` fetches `/api/viewers/...` first; if a match is
>      returned, embeds it (iframe for jupyterlite, inline for html).
>    - Falls back to the current built-in SVG / canvas when no plug-in
>      matches — no regression for the default case.
>
> 3. **JupyterLite asset bundle** (optional, behind a `[jupyterlite]`
>    extra in `pyproject.toml`): ship a minimal JupyterLite distribution
>    under `caterva2/services/static/jupyterlite/` so a viewer of kind
>    `"jupyterlite"` can load Python code in the browser, `import
>    xarray_blosc2`, fetch the dataset via the C2Array HTTP client, and
>    render with whatever the user already uses in notebooks.
>
> Reference implementations to wire up:
> - `xarray_blosc2` exposes a `register_viewer(name, predicate, render)`
>   API. Caterva2 plug-ins can re-use those Python renderers verbatim
>   through the JupyterLite kernel.
> - `iso8601-intervals` ships an ISO-8601 time-series scrubber that
>   would register as the first concrete plug-in.
>
> Acceptance: with no plug-ins installed, the info page behaves exactly
> as today. With `pip install iso8601-intervals[viewer]`, opening a
> dataset whose `vlmeta["dims"]` contains `"time"` renders the
> interval scrubber instead of the plain SVG. Toggleable per-dataset
> via a `?viewer=default` query string for debugging.

Both prompts are independent — A can land before B or vice versa.

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
