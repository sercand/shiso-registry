---
name: add-model-recipe
description: Add or update a model recipe in the shiso registry and publish it, using the shiso-registry tool (editor UI, inspect, validate, build). Use when asked to add a model, add a variant, pin a Hugging Face repo, change server defaults, or rebuild/publish registry/v1.
---

# Adding a model recipe to the shiso registry

This repository is the shiso model registry. `shiso pull <name>` reads the
compiled index and manifests under `registry/v1/`. `shiso serve <name>` turns
the installed recipe into server flags.

## Repository layout

```
recipes/<name>.json          authored recipes, one per model (edit these)
registry/v1/index.json       compiled index         (generated, committed)
registry/v1/models/<name>.json  compiled manifests  (generated, committed)
index.html, error.html       static landing pages for the host
```

Never edit `registry/v1/` by hand. It is the output of `shiso-registry build`,
and each manifest's digest is pinned in `index.json`.

Clients take the registry URL **including** `/registry/v1/`. The default is
`https://models.shiso.run/registry/v1/`, and `shiso` appends `index.json` and
`models/<name>.json` to it.

## The tool

`shiso-registry` is built from the shiso-engine repository:

```sh
cd ~/Developer/src/github.com/sercand/shiso-engine
cargo build --release --locked -p shiso-registry
# binary: target/release/shiso-registry
```

| Command | What it does |
|---|---|
| `shiso-registry serve --repo <this repo> [--port 8095] [--open] [--require-token]` | Web editor on 127.0.0.1. No token unless `--require-token`. |
| `shiso-registry inspect <owner/name> --revision <branch\|sha>` | Prints the resolved commit and the file list (names, sizes, LFS). It does **not** print digests. |
| `shiso-registry validate --repo <this repo>` | Strictly validates every recipe in `recipes/`. |
| `shiso-registry build --repo <this repo> [--out DIR] [--allow-dirty]` | Compiles `recipes/` into `registry/v1/`. |

Paths always resolve against `--repo`, never against the working directory.
An empty or relative `--out` (and the editor's Output Directory field) lands
inside the repository, and `published_at`/`updated_at` come from this
repository's git history. Run the tool from anywhere, but point `--repo` here.

## Workflow

1. **Pick the weights.** Find the Hugging Face repo and branch for each
   container you want to ship (GGUF, MLX, EXL3 directory, ...). Check the
   backend table below to see which shiso backend can run it.
2. **Get the pinned file list** (commit SHA, every file's size and SHA-256).
   Either:
   - **Editor (easiest):** `shiso-registry serve --repo . --open`, open or
     create the recipe, add a variant, enter the repo and revision, click
     **Inspect HF Files**, then tick files and set **Entry Point** / **Binds**.
     It computes digests for non-LFS files by downloading them.
   - **Headless:** with the editor running, call its API (no token needed by
     default):
     ```sh
     curl -s -X POST http://127.0.0.1:8095/api/hf/inspect \
       -H 'content-type: application/json' \
       -d '{"repo":"owner/name","revision":"main"}'
     # -> {"repo","revision","commit":"<40 hex>","files":[{"name","size","sha256","lfs"}]}
     ```
   Use the returned `commit` as `revision`. Never pin a branch name.
3. **Write `recipes/<name>.json`** following the schema below. Copy the
   structure of an existing recipe (`recipes/qwen3.8-flash.json`).
4. **Validate:** `shiso-registry validate --repo .` must print `All recipes strictly valid.`
5. **Commit the recipe.** `build` refuses uncommitted changes under
   `recipes/`, because the manifest dates come from git.
6. **Build:** `shiso-registry build --repo .`, which writes `registry/v1/`.
7. **Commit `registry/v1/`** together with nothing else, then push.
8. **Test before announcing** (see "Testing a recipe").

## Recipe schema (schema_version 1)

```jsonc
{
  "schema_version": 1,
  "name": "qwen3.8-flash",            // [a-z0-9][a-z0-9._-]*, max 64; the name users pull/serve
  "aliases": ["qwen3.8-flash-next"],   // must not repeat the name
  "family": "qwen4exp",               // shiso-models family, see list below
  "min_shiso_version": "0.1.0",
  "description": "One line, non-empty",
  "tags": ["vision", "tools"],
  "tasks": ["chat"],                   // chat | transcription | diarization | image_generation | embedding
  "modalities": { "inputs": ["text", "image", "video"], "outputs": ["text"] },   // text | image | video | audio
  "features": {
    "tools": true, "reasoning": true, "structured_output": false,
    "reasoning_efforts": ["none", "low", "medium", "high"],
    "default_reasoning_effort": "medium"          // must be one of reasoning_efforts
  },
  "variants": [ /* author preference order; the first that fits the host wins */ ]
}
```

Families this build of shiso knows: `qwen35`, `qwen35moe`, `qwen4exp`,
`gemma4`, `nemotron_h`, `nemotron_asr`, `nemotron_diar`, `qwen3_asr`, `flux2`.
The editor lists them from `/api/families`.

### A variant

```jsonc
{
  "key": "cuda-exl3-405",             // [a-z0-9_-]+, unique in the recipe; `shiso pull name:key`
  "backends": ["cuda"],               // non-empty, no repeats; "metal" and/or "cuda" ONLY if the
                                      // files really run there (see backend table)
  "quant": "exl3-4.05bpw",            // free-form label shown to users
  "requires": {                       // optional hard host constraints
    "min_cuda_sm": 80,                // CUDA compute capability (80 = Ampere)
    "min_metal_family": 7             // Apple GPU family: 7=M1, 8=M2, 9=M3/M4, 10=M5
  },
  "requirements": {
    "min_ram_bytes": 120259084288,    // > 0; below it the variant is skipped
    "recommended_ram_bytes": 137438953472,   // optional; preferred when the host has it
    "min_vram_bytes": null,           // optional; CUDA VRAM, or Metal recommended working set
    "disk_bytes": 107366392363        // MUST equal the sum of every file size (strict check)
  },
  "components": { "<role>": { /* component */ } },
  "server_defaults": { /* optional, see below */ }
}
```

Selection: `shiso pull name` takes the first variant in order whose backend is
compiled in, whose hard requirements hold, and whose recommended (else
minimum) RAM fits. `name:key` picks a variant explicitly and warns instead of
refusing on RAM.

### A component, roles and flags

```jsonc
"base": {
  "provider": "huggingface",
  "repo": "owner/name",
  "revision": "<40-hex commit sha>",
  "entry": "model.gguf",              // optional: the file the role's flag points at;
                                      // absent = the component directory itself
  "files": [
    { "name": "model.gguf", "size": 123, "sha256": "<64 hex>" },
    { "name": "vision.safetensors", "size": 456, "sha256": "...", "binds": "vision" }
  ]
}
```

Each role becomes one `shiso serve` flag pointing into the installed variant
(`~/.shiso/models/<name>/<key>/<role>/...`):

| Role | Flag | Notes |
|---|---|---|
| `base` | `--model` | Required unless the variant has `asr` or `image`. A directory checkpoint omits `entry`. |
| `draft` | `--draft-model` | Requires `server_defaults.spec_k > 0`. |
| `draft_vocab` | `--draft-vocab` | Requires `draft`. |
| `vision` | `--vision` | Vision tower file. |
| `ngram` | `--ngram` | Separate n-gram table file (the EXL3 qwen4exp export). Must name a file. |
| `config` | `--config` | `config.json` when the weights ship without one. The tokenizer and chat template are read beside it. Must name a file. |
| `asr` | `--asr-model` | |
| `asr_mmproj` | `--asr-mmproj` | Requires `asr`. |
| `diarization` | `--diarization-model` | Requires `asr`. |
| `image` | (reserved) | `shiso serve` refuses it today. |

**`binds`** hands one file to *another* role's flag, so a single repository
can supply several roles. For example, the EXL3 repo ships the shards, the
vision tower and the n-gram table together. Its `base` component has no
`entry` (the directory is `--model`), and:

```json
{ "name": "vision_k6.safetensors", "binds": "vision", ... },
{ "name": "ngram_embedding.safetensors", "binds": "ngram", ... }
```

Rules: a role comes from exactly one place, either its own component or one
bound file, never both. A file cannot bind its own component's role (use
`entry` for that). In the editor, **Binds** is a dropdown on each ticked file,
both in the inspector's file table and in the Selected Files table.

Only list files the loader needs. Skip READMEs, `.gitattributes`, licenses,
benchmark dumps and large unused sidecars such as `quantization_config.json`.
Every listed file is downloaded and verified, and `disk_bytes` must still
equal their sum.

### server_defaults (allowlist; unknown keys are rejected)

`ctx`, `max_num_seqs`, `spec_k`, `spec_lookup`, `turn_replay_tokens`,
`vision_max_pixels`, `video_fps`, `video_max_frames`, `asr_lookahead`,
`asr_batch`, `expert_residency` (`auto` | `resident` | `stream`),
`expert_reserve_gb`, `expert_slots`, `capture_verify`, `capture_batch`,
`gpu_memory_utilization` (0..1), `solo_prefill_chunk`.

Precedence when serving: explicit CLI flag > `SHISO_*` env var > recipe
default > built-in default.

## Backend compatibility (check before listing a backend)

| Container | Metal | CUDA | Notes |
|---|---|---|---|
| GGUF (K-quants, Q8, F16/BF16) | yes | yes | One file. List both backends when the family supports both. |
| MLX 4-bit directory, qwen4exp | yes | no | Streams experts from SSD when RAM is short (`expert_residency`). The sparse CUDA kernels have no affine tier. |
| MLX 4-bit directory, dense families (gemma4, qwen35) | yes | verify first | CUDA has dense affine kernels; confirm with a real load before listing `cuda`. |
| FP8 reference export directory, qwen4exp | yes | verify first | Experts always go through the expert stream. |
| EXL3 directory (qwen4exp) | **no** | yes | Trellis kernels are CUDA-only (min sm80). Needs `ngram` and, for vision, `vision` bindings. |

When unsure, check `crates/shiso-server/src/model_factory.rs` in
shiso-engine. Listing a backend whose kernels don't exist makes `shiso pull`
download ~100 GB that then fails at load.

Note: the CUDA host probe currently reports sm89 / 16 GB for every device, so
`min_cuda_sm` above 89 and any `min_vram_bytes` on CUDA variants would wrongly
reject real hosts. Keep `min_cuda_sm` at or below 89 and leave CUDA
`min_vram_bytes` out until that is fixed.

## Testing a recipe

From this repository, after `build`:

```sh
python3 -m http.server 8766 --bind 127.0.0.1      # serves registry/v1 locally
R=http://127.0.0.1:8766/registry/v1/
shiso info <name> --registry-url $R                  # shows variants; marks the one for this host
shiso pull <name> --registry-url $R                  # downloads + installs (blobs are reused if present)
shiso verify <name>                                  # re-hashes every installed file
shiso serve <name> --port 8899                       # flags come from the recipe
```

Avoid a 100 GB re-download when the files are already on disk: hard-link each
file into `~/.shiso/blobs/sha256/<sha256>` (same filesystem) before
`shiso pull`. Pull skips any blob that exists with the right size, and
`shiso verify` checks the digests.

## Pitfalls

- `build` fails on a dirty `recipes/`: commit first. `--allow-dirty` stamps
  uncommitted recipes with the current time.
- `inspect` and the editor cannot read gated/private repos (no HF token is
  forwarded).
- In the editor, **Clear Selection** removes the files' bindings, and
  unticking a file drops its binding. **Select All** keeps existing bindings.
- Changing a recipe changes its manifest digest. Hosts that installed the old
  one need `shiso pull <name>` again to pick it up; blobs are reused, so
  nothing re-downloads. The running server must be stopped first.
- A role that needs a file (`config`, `ngram`) must name one with `entry` or
  `binds`, or validation fails.
