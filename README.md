# pod-strata

The Strata **pod**: [Qwen3.8-Flash-Next](https://github.com/Niko1221/Strata) — a 125B-parameter
MoE — served from one consumer NVIDIA card plus system RAM, with OpenAI-, Responses- and
Anthropic-compatible APIs on port 8080.

This repo holds the image (`box/strata`) and the disposable bed that proves it
(`check-strata-gpu-pod`). The engine layer it composes lives in
[opencharly/layer-strata](https://github.com/opencharly/layer-strata); the host-side management
CLI lives in [opencharly/plugin-strata](https://github.com/opencharly/plugin-strata).

## What the image is

`box/strata` bases on **[opencharly/pod-cdi](https://github.com/opencharly/pod-cdi)** — the CDI
pod contract — and composes the `strata` engine layer.

A GPU **pod** does not own its driver. The kernel module is host-global, so the driver
user-space is injected per container by the runtime, from the NVIDIA container toolkit's CDI
spec. This image therefore contains **no** driver user-space, no `nvidia-ctk`, and no
`LD_LIBRARY_PATH` pointing at a baked driver; `layer-cdi`'s own build steps fail the build if any
of them appears. The engine is upstream's **published** Terminal and CUDA-13 engine component,
installed at a pinned release — never compiled on your machine, never compiled on the first
start.

The model weights are **not** in the image: they are 37-188 GB, licensed separately from the
engine, and live in a store you mount.

## Deploy it

Requirements on the host: an NVIDIA card the release's component supports (driver 580 or newer
for the CUDA-13 package), the NVIDIA container toolkit, and a CDI spec:

```bash
nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

Then, from a checkout of this repo:

```bash
charly box build strata                 # the image, tagged ghcr.io/opencharly/strata:<CalVer>
charly config strata                    # records the deploy (auto-allocates a loopback host port)
charly start strata
charly status strata                    # the mapped host port, the store, the secret
charly logs strata
```

Deploy entry, for a store you already populate (see below):

```yaml
strata:
  pod:
    image: strata
    env:
      FAMILY: qwen
      MODEL: IQ2_XS
      MODEL_PATH: /models                 # shards you already staged
      CONTEXT: "65536"
    volume:
      - {name: strata-data, type: bind, host: ~/.local/share/strata, path: /data}
      - {name: gguf,        type: bind, host: ~/.local/share/strata/models/IQ2_XS, path: /models}
```

No `secret:` line: the image's layer declares `API_KEY` as a **required** secret, so `charly config`
mints one for you (or takes yours if you stored it first — see below).

With no `MODEL_PATH`, the first start downloads and prepares the model instead (tens of GB;
watch it with `charly logs strata`). Either way the prepared pack is built once into the store.

## The model store

`/data` is **your** directory, bind-mounted — `~/.local/share/strata` by default, because that is
`$XDG_DATA_HOME`'s default. Updating the image never touches it.

| Path | What it holds |
|---|---|
| `Strata-data/models/<model>/` | the GGUF shards |
| `Strata-data/packs/<model>/` | the prepared pack (tens of GB, built once) |
| `Strata-data/mtp/` | the drafter layer (~6 GB) |
| `state/` | the preset, the tuning result, and the API key (`state/api-key`) |
| `install/` | the run settings and the logs |
| `home/` | `HOME` inside the container |

The pre-0.2.16 layout (`/data/models`, `/data/packs`, `/data/mtp`) is migrated by upstream's own
entrypoint on the first start — a rename inside the volume, nothing downloaded again, and a
folder that is its own bind mount is left alone.

Budget ~80 GB for a 60 GB-class model, ~0.9 GB more if you serve pictures. `charly volume list` /
`charly volume reset` apply only to named volumes; a bind-mounted store is reset with the
filesystem.

## The environment

Every setting is an environment variable of the deployment, and the names are upstream's own.
The layer's `var:` block carries the defaults below.

| Variable | Default (engine) | What it does |
|---|---|---|
| `FAMILY` | `qwen` | model family (also selects the picture reader) |
| `MODEL` | `IQ2_XS` | the size to install on a fresh store (`IQ2_XS`, `IQ3_XXS`, `IQ3_S`, `Q2_0`, ...) |
| `MODEL_PATH` | empty | GGUF files you already have: a folder or one `.gguf`, inside the container |
| `CONTEXT` | `65536` | context window: `32k`/`64k`/`128k`/`256k` or a number. Upstream recommends less on a 16 GB card |
| `PARALLEL` | fitted to the card | requests answered at once, 1-8 |
| `VISION` | `cpu` | pictures: `off`, `on`, `gpu`, `cpu`. Upstream defaults to `off`; this family keeps them **on**, read on the CPU |
| `KV` | `int8` | conversation-memory format: `int8`, `q4_0` (half the memory), `k8v4`, `fp16` |
| `DRAFTING` | `auto` | which drafter guesses ahead: `auto`, `mtp`, `dflash2`, `off` |
| `DRAFT_VOCAB` | from the container's languages | which tokens the drafter may guess: `cjk`, `en`, `cyrillic`, `fr`, `latin` |
| `GPU` / `GPUS` | all that fit | card numbers as the monitor lists them (`0`, `0,1`); empty or `all` = the engine's choice |
| `GPU_ROLES` | none | a role per card (`1=main,0=helper`); not together with `GPU`/`GPUS` |
| `TUNE` | `skip` | `quick`/`thorough` tune ONCE per preset (up to ~20/120 minutes) and keep the result; `RETUNE=1` tunes again |
| `PORT` | `8080` | the port inside the container |
| `PRESET` | `docker` | the preset name in the store; your settings are saved into it |
| `STRATA_SET` | none | advanced engine settings, space separated `KEY=VALUE` |
| `API_KEY` / `API_KEY_FILE` | minted | the key clients must send (see below) |
| `ALLOW_DOWNLOAD` | off | `1`: go on and download although model files of the OLDER layout sit in `/data/models` |

Two different retirements, not one: `STRATA_RUN_ARGS` and `STRATA_OVERLAY_JSON` were this family's
OWN surface before the cutover (a second implementation of upstream's config, vision pass
included) and are deleted with it — their settings are now **setup** choices (`CONTEXT`,
`STRATA_SET`, `TUNE`). `HOST`, `LOW_RAM`, `LAYER_SPLIT`, `REINSTALL`, `KV_STREAMING`,
`RESIDENT_BUDGET_GIB` and `CONFIG` are upstream's older-image names, which the 0.2.x entrypoint
warns about and ignores.

### After changing the box's `security:` block

The image always carries the current security set (it is baked into the
`ai.opencharly.security` label). A **deployment already created** does not: its resolved security
set is captured when you run `charly deploy add`. So after an edit to `security:` in
`box/strata/charly.yml`:

```bash
charly box build strata
charly deploy del strata
charly deploy add strata
charly start strata
```

Nothing warns you if you skip this — the unit simply keeps the args it was created with.

## Talking to it

| Client | Base URL | Endpoint |
|---|---|---|
| OpenAI Chat Completions | `http://127.0.0.1:<host-port>/v1` | `POST /v1/chat/completions` |
| OpenAI Responses | `http://127.0.0.1:<host-port>/v1` | `POST /v1/responses` |
| Anthropic Messages | `http://127.0.0.1:<host-port>` | `POST /v1/messages` |
| health (no key) | `http://127.0.0.1:<host-port>/health` | `GET /health` |
| models (keyed) | `http://127.0.0.1:<host-port>/v1` | `GET /v1/models` |

`charly status strata` prints the auto-allocated host port. From inside the pod the port is
always 8080.

## The API key

Strata does not serve beyond loopback without a key, so a key is always on. The image's layer
declares `API_KEY` as a **required** secret (`secret_require`), which means:

- **`charly config strata` mints one** if you have not stored your own — a 32-byte token — and the
  generated unit carries it as `Secret=<name>-api-key,type=env,target=API_KEY`. The engine never
  starts with a blank key.
- **To use your own**, store it first and the mint leaves it alone:

  ```bash
  charly secrets set charly/secret/API_KEY "$(openssl rand -hex 24)"
  charly config strata
  ```

- **To read it**: `charly secrets list`, or `charly status strata` (the `Secrets:` line).

You do not need a `secret:` entry on the deploy — a deploy-level name that has no stored value
injects nothing at all (no secret, no environment variable, no warning), so it would look like the
key is under test while the keyless path runs. Every `/v1/*` request carries
`Authorization: Bearer <key>`; `GET /health` is answered **without** the key by design, so a
liveness probe needs no secret. Upstream's own mint-a-key-on-first-start path still exists for a
deployment that bypasses charly entirely, and the key then lands in `/data/state/api-key`:

```bash
charly shell strata -c 'strata launch --show-key --preset docker'
```

## Proving it

`check-strata-gpu-pod` is a `disposable: true` pod bed — the family's only witness, because the
engine needs the actual card. It is **not** in the umbrella's default roster (it needs a named
venue and pre-staged weights), so run it on demand:

```bash
charly check run check-strata-gpu-pod
```

It asserts four things, in order: `/health` publishes the context this deployment asked for (65536 —
a literal above what upstream recommends for the card, so the assertion can fail) **and**
`images: true` with no vision variable in the bed's `env:`, which is what makes pictures the
LAYER's default rather than a value the bed imposed; the engine is **loaded**, not merely
listening, which separates "the HTTP server answers" from "the weights are in memory"; a real PNG
round-trips through the encoder and the answer names the colour; and the Terminal reports the build
the image pins (`strata 0.2.16`, `build 0.1.62`), executing as **container uid 0 — the mapped host
user**, since a CDI-device unit runs with `userns=host` / `User=0:0` and there uid 0 is the invoking
host user, not a root escalation.

`charly box build strata` proves the other half: the image composes, and it carries no driver
user-space — `layer-cdi`'s own steps check that in the build.

## Layout

| Path | What it is |
|---|---|
| `box/strata/charly.yml` | the `strata` image: the base, the engine candy pin, the build-scope checks, the box skill's routing |
| `charly.yml` | `defaults:`/`import:`/`discover:` plus the `check-strata-gpu-pod` bed |
| `.agents/skills/strata/SKILL.md` | the deployment skill (what it serves, the env surface, the store, the key, the bed) |
| `AGENTS.md` | the repo's own rules: the pod boundary, the bed as the only witness, PR-only landing |
| `.github/workflows/tag-on-merge.yml` | the org-wide CalVer tag + `CHANGELOG/` dispatcher every repo installs |
