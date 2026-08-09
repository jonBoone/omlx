# oMLX macOS App Packaging

Produces the venvstacks Python layers that the Swift macOS bundle
embeds. Building the user-facing `.app` itself is owned by
[`apps/omlx-mac/Scripts/build.sh`](../apps/omlx-mac/Scripts/build.sh);
this directory only hands it a `_export/` tree of Python layers.

> **PyObjC menubar retired.** The earlier Python / PyObjC menubar
> (`packaging/omlx_app/`) and the `packaging/build.py` `.app` + DMG
> pipeline that wrapped it have been removed. The Swift app under
> [`apps/omlx-mac/`](../apps/omlx-mac/) is now the only macOS bundle.

## Requirements

- macOS 15.0+ (Sequoia) — required by MLX ≥ 0.29.2
- Apple Silicon (M1/M2/M3/M4)
- Python 3.11+ on the host
- venvstacks (installed via `pip install -e ".[dev]"` from the repo
  root, or any of `uv`, `pipx run`)

## Build

```bash
# Re-export the venvstacks layers (cold ~10-20 min, warm ~4 min)
python packaging/build.py --venvstacks-only

# Stable fingerprint of the inputs that drive the export shape — used
# by build.sh to decide whether to re-export
python packaging/build.py --print-fingerprint
```

Then the Swift bundle:

```bash
apps/omlx-mac/Scripts/build.sh release             # full bundle
apps/omlx-mac/Scripts/build.sh release --no-rebuild-donor   # reuse _export/
apps/omlx-mac/Scripts/build.sh release --with-custom-kernel  # bundle GLM-5.2 / MiniMax M3 native kernels
```

## Output

```
packaging/
├── _build/         # venvstacks intermediate layers
├── _export/        # venvstacks export — embedded into the .app
└── _wheels/        # cached local wheels (e.g. mlx + mlx-metal pins)
```

## Layer Configuration

| Layer | Contents |
|-------|----------|
| Runtime (`cpython-3.13`) | Python 3.13 |
| Framework (`mlx-base`) | MLX, mlx-lm, mlx-vlm, FastAPI, transformers, mlx-audio, paroquant, spaCy |

No application layer — the Swift app is the application surface.

## Changing the Bundled Python Version

The bundled Python version is declared in a single place:
`packaging/venvstacks.toml` `[[runtimes]]` block:

```toml
[[runtimes]]
name = "cpython-3.13"
python_implementation = "cpython@3.13"
```

The build pipeline (`build.py`, `build.sh`, and the CLI wrapper test)
dynamically parses `cpython@X.Y` from this file, so changing it here
propagates to all derived paths.

**When bumping the version, also update:**

1. **`Formula/omlx.rb`** — `depends_on "python@X.Y"` and `system "pythonX.Y"`
2. **`pyproject.toml`** — if the minimum supported version changes,
   update `requires-python` and the classifiers
3. **`packaging/venvstacks.toml`** — ensure `name` matches
   `cpython-{version}` (used as the layer directory name)

## Installation

The Swift build (`build.sh release`) produces
`apps/omlx-mac/build/Stage/oMLX.app` directly — no DMG step. To install:

1. Drag `apps/omlx-mac/build/Stage/oMLX.app` to `/Applications`, or
   `open` it in-place to launch from `apps/omlx-mac/build/Stage/`.
2. Launch the app (appears in the menubar).
3. Walk through the first-run wizard (Storage + API key), then Start
   Server.

> The DMGs on the [Releases](https://github.com/jundot/omlx/releases)
> page are produced by an off-tree maintainer pipeline, not by anything
> in this repo. End users follow the Releases install path; this
> section is for developers building from source.
