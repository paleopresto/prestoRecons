# PReSto Reconstruction Input Standards

This document is the authoritative contract for adding a reconstruction method to
PReSto (the Paleoclimate Reconstruction Storehouse). It covers the **repository
layout**, the **inputs PReSto provides at run time** (data query + parameters),
the **parameter schema** that drives the auto-generated GUI form, and the
**outputs** your container must produce.

For the step-by-step registration walk-through (registry entry, form folder, pull
request, promotion), see the companion guide in the server repo:
[`prestoServer/docs/adding-a-reconstruction.md`](https://github.com/DaveEdge1/prestoServer/blob/master/docs/adding-a-reconstruction.md).

---

## Architecture in one paragraph

A reconstruction runs as a **GitHub Actions workflow inside a per-run repository**
that PReSto creates from your **template repo**. When a user submits, the PReSto
server commits the user's **data query** and **parameters** into that repo; the
push triggers the workflow, which pulls the proxy data, runs your **container**,
and publishes results (CSV/NetCDF + figures, optionally to GitHub Pages). The
server itself does **not** execute the reconstruction — it only gathers inputs and
dispatches. A method therefore consists of three things:

1. a **template repo** (start from [`DaveEdge1/presto-template`](https://github.com/DaveEdge1/presto-template)),
2. a **parameter schema + registry entry** in `prestoServer`, and
3. a **container** that reads the inputs below and writes the outputs below.

---

## 1. Repository layout

Create your method's repo with **"Use this template"** from
[`DaveEdge1/presto-template`](https://github.com/DaveEdge1/presto-template) and
follow its `ADAPTING.md`. The template already ships the PReSto wiring:

| Path | Purpose |
|------|---------|
| `.github/workflows/reconstruct.yml` | **Push-triggered on `query_params.json`** — the main run |
| `.github/workflows/visualize.yml` | Builds/publishes the interactive visualization |
| `.github/workflows/update-readme.yml` | Regenerates `README.md` from the run |
| `.github/workflows/release-recon.yml` | Publishes large outputs (e.g. NetCDF) as a Release |
| `config/user_config.yml` | **Runtime parameters** (see §3) — overwritten by the server on submit |
| `scripts/lipd_to_input.py` | Converts the LiPD data PReSto provides into your container's input (see §2) |
| `scripts/reconstruct.py` | Your reconstruction entry point |
| `scripts/outputs_to_netcdf.py`, `make_figures.py`, `generate_readme.py` | Post-processing |
| `Dockerfile`, `environment.yml` | Your container |
| `results/` | Output layout (see §5) |

> **Do not commit a `query_params.json` to your template repo.** The workflow is
> push-triggered on that path and the server commits the real one at submission.
> If the template ships a placeholder, *creating* a repo from the template fires
> an extra reconstruction with those default params before the user's query lands.
> (`presto-template` and the live method templates intentionally omit it.)

---

## 2. Inputs PReSto commits at submission

On submit, the server commits the following into the run repo in a **single
commit** (`query_params.json` is included, which triggers the workflow):

| File | Always? | Contents |
|------|---------|----------|
| `config/user_config.yml` | yes (unless the method declares no config) | The user's runtime parameters (§3) |
| `query_params.json` | yes | The data query (§2.1) — the trigger file |
| `cleaning_report.json` | only if the user ran data cleaning | Record of duplicate/removed proxies |
| `variable_filter.yaml` | method-dependent | Which `variableName`s the user included/excluded |
| `README.md` | yes | Updated with the run ID |

### 2.1 The data query — `query_params.json`

Describes **which proxy records** to reconstruct from. Two modes:

**Archived** — a published compilation:
```json
{ "mode": "archived", "compilation": "Pages2kTemperature", "version": "2_2_0" }
```

**Filtered** — a custom LiPDverse query, optionally narrowed by the cleaning step:
```json
{
  "mode": "filtered",
  "archiveType": ["LakeSediment", "GlacierIce"],
  "variableName": ["temperature"],
  "compilation": "Temp12k",
  "...": "other query fields",
  "tsids": ["...selected TSIDs..."],
  "removedTsids": ["...explicitly/implicitly removed TSIDs..."]
}
```

Your workflow reads `query_params.json` and turns it into the proxy data your
container needs (see §2.2). `tsids`/`removedTsids` are present only on the
filtered path after cleaning.

### 2.2 Proxy data into your container

Proxy data is **pulled from a curated repository (lipdverse.org) at run time**, not
baked into the container — so newly published data is picked up and images stay
small. The template's workflow resolves `query_params.json` into LiPD data (via the
shared `lipdGenerator`), then `scripts/lipd_to_input.py` converts that into the
matrix/format your model expects (e.g. a proxy matrix + metadata). Adapt
`lipd_to_input.py` to your container; keep the conversion in the template repo, not
the server.

---

## 3. Parameters

There are **two representations** of a method's parameters. Keep them distinct:

- **(a) Editor schema** — the rich YAML that PReSto turns into the "Reconstruction
  Parameters" GUI form. It lives in the **server** repo at
  `prestoForm/<handle>/configs.yml` (and `querypathconfigs.yml`), **not** in your
  template. This is what §3.1–§3.4 below specify.
- **(b) Runtime config** — `config/user_config.yml` in your template/run repo. Plain
  key→value (no GUI metadata); **flat or nested** as your container prefers. This is
  what your container actually reads.

PReSto bridges (a)→(b) on submit. If your runtime keys match the form keys, no
mapping is needed (`configStrategy: passthrough`). Otherwise provide a
`prestoForm/<handle>/lookup.json` that maps each form key to its runtime key/path
and pick the matching `configStrategy` (`nested` for nested configs,
`holocene_da`-style for a flat lookup, or a custom strategy). See the server guide
for details.

### 3.1 Editor schema file

YAML (`.yml`). Comments are preserved but not shown to GUI users. A complete, live
example is
[`prestoForm/holocene_da/configs.yml`](https://github.com/DaveEdge1/prestoServer/blob/master/prestoForm/holocene_da/configs.yml);
the rendered form is served at `/editor/querypath?recon=holocene_da`.

### 3.2 Parameter keys

Keys follow `group_specific`: underscore-separated lowercase, the first token a
**grouping term** and the rest a specific term, e.g. `time_range_to_reconstruct`.

Common grouping terms (reused across methods — prefer these where they fit):
`recon`, `time`, `prior`, `proxy`, `psm`, `geo`, `model`, `uncertainty`.

- `recon` — the reconstruction broadly (e.g. the climate variable)
- `time` — temporal resolution and interval
- `prior` — Bayesian priors
- `proxy` — proxy selection (e.g. seasonality, archive types)
- `psm` — proxy system models (e.g. calibration period)
- `geo` — geospatial bounds
- `model` — climate-model choices
- `uncertainty` — error / confidence-interval settings

**New grouping terms are allowed** when a method needs them (e.g. BayGMST adds
`stan`, `partition`, `co2`, `vol`). Each grouping term used must have a display
title in the server's `jsonEditor/headings.json`, or its section header renders as
"undefined".

### 3.3 Parameter values

Each parameter is a map of up to 10 keys. **Bold** keys are required for all data
types; others are required only for the data types noted.

- **`value`** — current input (the adjustable one)
- **`default`** — default input (never changes)
- `limits` — inclusive `[min, max]`; required for `numeric` and `range`
- `options` — list of valid choices; required for `character` and `list`
- `precision` — minimum increment; required for `numeric` and `range`
- **`long_name`** — label shown in the GUI
- **`description`** — help text for the user
- **`data_type`** — drives the form element (§3.4)
- **`complexity`** — `standard` | `advanced` | `experimental` (§3.5)
- `URL` — optional link to more info

### 3.4 `data_type`

| data_type | extra keys required | form element |
|-----------|---------------------|--------------|
| `free-form` | — | text box (use only as a last resort) |
| `boolean` | — | true/false radio |
| `character` | `options` | single-select radios |
| `list` | `options` | multi-select checkboxes |
| `numeric` | `limits`, `precision` | range slider + numeric input |
| `range` | `limits`, `precision` | dual range slider + min/max inputs |

> **Note:** the current query-path form generator renders the types above. The
> older `lat-lon` map widget is **not** rendered on the query path; express
> geographic bounds with `numeric`/`range` parameters (e.g. `geo_lat_min`,
> `geo_lat_max`) until map support is reinstated.

Example (`numeric`):
```yaml
proxy_min_resolution:
  value: 200
  default: 200
  limits: [10, 1000]
  precision: 10
  long_name: minimum resolution of proxies
  description: Drop records coarser than this (years).
  data_type: numeric
  complexity: advanced
  URL: null
```

### 3.5 `complexity`

- `standard` — shown in the form by default
- `advanced` — shown when "show advanced parameters" is enabled
- `experimental` — **not adjustable in the GUI** (hidden); deep understanding
  required and/or may break the run. (If you want users to set it, use `advanced`.)

---

## 4. Container contract

- Read `config/user_config.yml` for parameters (mounted by the workflow).
- Read the proxy data prepared from `query_params.json` (§2.2).
- Pull curated data at run time; do not bake datasets into the image.
- Write outputs to the `results/` layout (§5).
- Be deterministic given identical inputs where feasible.

---

## 5. Outputs — the `results/` layout

The visualization / readme / release workflows expect a predictable `results/`
tree. Mirror the template's layout:

| Path | Purpose |
|------|---------|
| `results/reconstruction.csv` | The reconstruction time series (tabular) |
| `results/reconstruction.nc` | The reconstruction as NetCDF (gridded / ensemble) |
| `results/proxy_matrix.csv` | Proxy matrix actually used |
| `results/proxy_metadata.csv` | Metadata for those proxies |
| `results/configs.yml` | The resolved config used for the run (provenance) |
| `results/figures/` | Generated figures (e.g. `reconstruction_ts.png`) |

Large NetCDFs that exceed GitHub's file limits are published via
`release-recon.yml` rather than committed.

---

## 6. Registering with PReSto

Adding the method to the platform (so it appears in the picker and query flow) is a
pull request against `prestoServer`:

1. Add `prestoForm/<handle>/` (form intro + editor schema + any `lookup.json`).
2. Add one entry to `presto/reconRegistry.json` with `ui.category:
   "New methods, in testing"`.
3. Generate the editor form and regenerate the derived artifacts.
4. Open the PR using the repository's pull-request template.

New methods ship under **"New methods, in testing"**; promotion to the main
**Reconstructions** group is a separate, maintainer-approved PR after testing.
Full instructions:
[`prestoServer/docs/adding-a-reconstruction.md`](https://github.com/DaveEdge1/prestoServer/blob/master/docs/adding-a-reconstruction.md).
