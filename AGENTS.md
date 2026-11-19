# AGENTS.md — pod-strata

The org rulebook at the umbrella (`opencharly/opencharly` `AGENTS.md`) applies in full. This
file adds only what is specific to this repo.

## This repo is the POD side of a two-repo boundary

`opencharly/layer-strata` owns the ENGINE LAYER (`candy/strata`) and the skills that describe it.
This repo owns the POD: the `strata` image in `box/strata` and the bed that proves it. The split
is ledger D-1 in `opencharly/opencharly`'s `plan/strata-pod-and-engine-v0216-ledger.md`.

- **`pod-strata` (here)** — a pod. The driver comes from the HOST through the NVIDIA CDI spec.
  `box/strata` bases on `pod-cdi.pod-cdi` and composes **nothing** from `layer-nvidia`.
- **`vm-nvidia`** — the VM contract: the guest owns its kernel and bakes the driver user-space,
  so its images compose `layer-nvidia`.

`layer-cdi`'s own plan steps enforce the boundary from the artifact's side:
`cdi-no-baked-driver-libs` and `cdi-no-baked-driver-binaries` FAIL THE BUILD when a host-coupled
driver library or an `nvidia*` binary is baked in. **The box IS the pod contract** — it does not
restate it, and it must never override it.

### What may not be composed here

- `@github.com/opencharly/layer-nvidia:*` — the VM driver user-space. An image that bakes it
  shadows the host's injected NVML, and `nvidia-smi` dies with *Driver/library version mismatch*.
- `nvidia-utils`, `nvidia-ctk`, or an `LD_LIBRARY_PATH` export that points at a baked driver —
  `nvidia-ctk` is a HOST tool that never executes inside a container.
- A second copy of the CDI rule. `layer-cdi` carries the predicate; re-deriving it in this repo's
  box would be two implementations of one rule (R2).

If a change needs any of these, it belongs in `opencharly/vm-nvidia` (or, for the engine itself,
in `opencharly/layer-strata`), not here.

## The bed is the only witness

`charly check run check-strata-gpu-pod` is this repo's acceptance gate: the REAL engine, on the
GPU venue, loading a provisioned model store on a card that arrived through CDI. There is no
second bed and no GPU-free substitute — a mock proves the harness, not the engine, and the family
withdrew its mock bed for exactly that reason.

- It is `disposable: true`, `lifecycle: dev`, `requires_shared: [nvidia-gpu]`. The `disposable`
  flag is what authorizes the check runner to destroy and rebuild it unattended; `lifecycle` is
  an informational tag and authorizes nothing (`/charly-internals:disposable`).
- It is **excluded from the umbrella's default roster** (`strata.check-strata-gpu-pod`): it needs
  a named venue and pre-staged weights, so it runs on demand and its evidence is pasted.
- **Budget the time, and never abort a quiet run.** The bed takes about **7 minutes of live time
  when the store and the image layers are warm**, and **15-20 minutes on a cold store** (upstream's
  first-start migration of an older-layout store, the picture reader's one-time fetch, then the
  model load). Its `eventually:` windows are therefore 1800 s or more — do not add a step with a
  shorter one, and do not kill a run that has gone quiet: silence during a multi-GB load is the
  engine working, and the poll-until-pass matcher reports its attempt count instead of timing out
  on the first pass.
- Never weaken a step to make it pass. A step that cannot FAIL proves nothing (R7): the
  pictures step sets no vision variable for exactly that reason, and the context step declares a
  context ABOVE what the card recommends so the projection can disagree.

## Load these skills first (R0)

- `/charly-image:image` — box composition (`base:`, `candy:`, `build:`, `platform:`, build-scope
  checks). Load before editing `box/strata/charly.yml`.
- `/charly-strata:strata-layer` — the engine layer this box composes: the artifact pins, the
  install, the service, the env defaults. Load before reasoning about anything the box inherits.
- `/charly-pod:pod` and `/charly-core:deploy` — the `kind: pod` entity and the deploy verbs the
  bed uses.
- `/charly-check:check` — `charly check run`, bed authoring, the R10 gate by change class.
- `/charly-internals:disposable` — `disposable:` / `requires_shared:` semantics and what an
  unattended rebuild authorizes.
- `/charly-internals:git-workflow` — before any git/PR action.

## Changing a security posture means re-adding the deploy

`box/strata/charly.yml`'s `security:` block (today: `security_opt: [no-new-privileges]`) is baked
into the image's `ai.opencharly.security` label at build time, so the IMAGE always carries the
current set. The DEPLOY does not: its resolved security set is captured into the deploy state at
`charly deploy add` time. So a box-level `security:` that lands AFTER a deployment already exists
does not reach `charly config` — the unit keeps the old args and nothing says so.

After editing `security:` in the box, rebuild and re-add the deploy:

```bash
charly box build strata
charly deploy del strata
charly deploy add strata        # re-capture the resolved security set
charly start strata
```

This is a property of the deploy state, not of this repo's manifest — the same edit through
`charly deploy add --add-candy` (an overlay) re-captures it too.

## Build / validate / test

- `charly box validate` at the repo root — the structural gate: the manifests parse, the pins
  resolve, the box composes. Run it after every manifest edit.
- `charly box build strata` — the image. A build that composes `layer-nvidia` is REFUSED by
  `layer-cdi`'s build steps; that refusal is the contract working.
- `charly check run check-strata-gpu-pod` — the live gate (see above).
- The merge gate is the **org-wide** `charly/pr-validator` (required check `validate / validate`,
  defined in `opencharly/.github`); this repo has no per-repo candy gate.

## Modify this repo

- Edit the bed in `charly.yml` or the box in `box/strata/charly.yml`; `discover:` loads `box/`.
  Keep the bed's `image:` in step with the box name.
- The box's `build: [pac]` is load-bearing: it does not cross the namespace boundary from
  `pod-cdi`, and a consumer that omits it dies with *image strata: build: field required*. The
  `distro:` chain, by contrast, IS inherited — do not restate it.
- Keep the `@github.com/opencharly/pod-cdi:` and `@github.com/opencharly/layer-strata/candy/strata:`
  pins at MERGED TAGS. A branch or SHA does not resolve for an `@github` candy ref
  (opencharly/charly#1201), and only a merged ref is a pin (AGENTS.md Part II #3).
- The skill at `.agents/skills/strata/SKILL.md` is this repo's prose surface: its routing lives in
  `box/strata/charly.yml`'s `strata-skill:` node, and the node carries **no** prose — an in-line
  description that disagrees with the file is a hard generator error.

## Landing

- PR-only. Every change lands through a pull request; the org-required `charly/pr-validator`
  validates the diff and body and arms native auto-merge on PASS. Direct pushes to `main` are
  blocked by the org ruleset.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR body IS the
  changelog — stage no `CHANGELOG/` file by hand.
- The authoritative rulebook is the umbrella `AGENTS.md` in `opencharly/opencharly`. Do not
  restate its rules here.
