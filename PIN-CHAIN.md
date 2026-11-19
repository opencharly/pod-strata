# PIN-CHAIN.md — the proven pin chain for `pod-strata`

Every claim below is a pasted command and its output. Commands were run from the umbrella checkout
`/home/atrawog/Sync/Atrapub/a/dsh/opencharly` on **2026-11-19**; `gh api` reads are pinned by
`?ref=<tag>`, so each line is the file AT THAT TAG and not at whatever the local checkout happens
to hold (the local `pod-cdi/` checkout was behind its remote when this leg started).

## The chain, in one table

| Hop | Pin | Read at | Names next |
|---|---|---|---|
| 1 | `opencharly/pod-cdi` **`v2026.323.1302`** (newest merged) | that tag | `layer-cdi:v2026.323.0302`, `layer-agent-forwarding:v2026.274.0608`, namespace `distro-arch:v2026.274.1256` |
| 2 | `opencharly/layer-cdi` **`v2026.323.0302`** | that tag | `layer-cuda:v2026.323.0252` — and nothing else |
| 3 | `opencharly/layer-cuda` **`v2026.323.0252`** (newest tag) | that tag | `layer-ffmpeg:v2026.272.0056` — **no `layer-nvidia`**, deliberately |
| 4 | `opencharly/layer-strata`'s `candy/strata` **`v2026.323.1751`** (`T1`; peels to `b61201d052cdc41690c377b78dea6a90874594e0`, merged 2026-11-19T17:51:55Z) | that tag | `layer-cdi:v2026.323.1259` — **no `layer-nvidia`, no `layer-cuda`** |

A POD image therefore gets its driver from the host (CDI) and its toolkit from `layer-cuda`, and
no hop in the chain names the VM contract's driver user-space.

## 1 — `pod-cdi`'s newest MERGED release

```
$ git -C pod-cdi fetch origin --tags
 * [new tag]         v2026.323.1302 -> v2026.323.1302
$ git -C pod-cdi for-each-ref --sort=-creatordate \
    --format='%(refname:short) %(creatordate:iso8601)' refs/tags | head -3
v2026.323.1302 2026-11-19 13:02:12 +0000
v2026.322.1717 2026-11-18 17:17:23 +0000
v2026.322.1325 2026-11-18 13:25:32 +0000
$ git -C pod-cdi merge-base --is-ancestor v2026.323.1302 origin/main; echo "ancestor_rc=$?"
ancestor_rc=0
```

`ancestor_rc=0` is what makes it a PIN and not a tag that exists beside `main`: the commit is on
the served line (AGENTS.md Part II #3, "pin only MERGED refs"). The tag is not the local checkout's
HEAD — the checkout was at `v2026.322.1717`'s CHANGELOG commit when this leg started, which is why
every figure below is read at an explicit `?ref=`.

## 2 — what `pod-cdi@v2026.323.1302` composes

```
$ gh api repos/opencharly/pod-cdi/contents/charly.yml?ref=v2026.323.1302 --jq .content \
    | base64 -d | grep -n "@github.com/opencharly"
30:    - arch: '@github.com/opencharly/distro-arch:v2026.274.1256'
84:            - '@github.com/opencharly/layer-agent-forwarding:v2026.274.0608'
93:            - '@github.com/opencharly/layer-cdi:v2026.323.0302'
```

Its own header states the contract the tag exists to carry: *"The tag is the CONTRACT here, not a
version bump: v2026.323.0302 is the release whose `require:` names `layer-cuda:v2026.323.0252`, the
toolkit release that dropped `require: layer-nvidia`."*

## 3 — what `layer-cdi@v2026.323.0302` requires

```
$ gh api repos/opencharly/layer-cdi/contents/charly.yml?ref=v2026.323.0302 --jq .content \
    | base64 -d | grep -n "@github.com/opencharly"
65:            - '@github.com/opencharly/layer-cuda:v2026.323.0252'
```

Exactly one `@github` ref. `layer-cuda` **is** the whole GPU stack of this chain — no
`layer-nvidia`, and the layer's own comment says the ordering is the contract ("this layer's
assertions must run AFTER every layer that can install host-coupled driver user-space").

## 4 — what `layer-cuda@v2026.323.0252` requires

```
$ gh api repos/opencharly/layer-cuda/contents/charly.yml?ref=v2026.323.0252 --jq .content \
    | base64 -d | grep -n '^        require:'
12:        require:
$ gh api repos/opencharly/layer-cuda/contents/charly.yml?ref=v2026.323.0252 --jq .content \
    | base64 -d | grep -n "@github.com/opencharly"
37:            - '@github.com/opencharly/layer-ffmpeg:v2026.272.0056'
```

The `require:` block carries a long comment — *"NO `layer-nvidia` HERE, DELIBERATELY … A
`require:` is TARGET-BLIND: it pushed the driver into EVERY pod that composes this toolkit"* — and
one pin: `layer-ffmpeg`. So the toolkit no longer drags the driver user-space into a pod, and
`layer-cdi`'s strip (`drv=`) removes what a distro package manager installs behind it
(`pacman -S cuda` DEPENDS on `nvidia-utils`).

`v2026.323.0252` is also the newest `layer-cuda` tag, so this hop is unambiguous in both
directions:

```
$ git ls-remote --tags https://github.com/opencharly/layer-cuda \
    | sed 's#.*refs/tags/##' | grep -v '\^{}' | sort -V | tail -5
v2026.320.2103
v2026.321.1520
v2026.323.0124
v2026.323.0138
v2026.323.0252
```

## 5 — the strata candy's own `layer-cuda` pin

### 5a. BEFORE the upgrade (`v2026.320.2149`, the tag the family published before `T1`) — it disagreed, twice

```
$ gh api repos/opencharly/layer-strata/contents/candy/strata/charly.yml?ref=v2026.320.2149 \
    --jq .content | base64 -d | grep -n "@github.com/opencharly"
55:            - '@github.com/opencharly/layer-supervisord:v2026.314.0710'
65:            - '@github.com/opencharly/layer-nvidia:v2026.320.1938'
76:            - '@github.com/opencharly/layer-cuda:v2026.320.2103'
84:            - '@github.com/opencharly/layer-build-toolchain:v2026.314.0704'
```

Two disagreements with the chain of §1–§4, and only one of them is about a tag:

- `layer-cuda:v2026.320.2103` ≠ the chain's `v2026.323.0252` — **a second pin of the same candy at
  a different tag in one closure**. That is the case this file exists to name.
- `layer-nvidia:v2026.320.1938` is composed at all. That is not a skew, it is the VM contract
  entering a pod, and it is what `T1` deleted (ledger D-2).

### 5b. What the resolution actually does with a skew — MEASURED, not assumed

The brief for this leg described a skew as *"a hard refusal"*. **It is not a refusal.** The engine
is `sdk/loaderkit/candy_version.go` (`PickCandyVersion`), whose own doc-comment fixes the severity:
*"a resolvable skew is INFO, never WARNING … The arbiter always has a deterministic winner (the
newest referenced tag), so resolution SUCCEEDS"*. When the skew sits inside ONE box scope it prints
one line and carries on:

```
$ # a probe box that pins layer-cdi v2026.322.1100 while its base's own chain pins v2026.323.0302
$ charly box generate
Notice: candy github.com/opencharly/layer-cuda resolved to multiple git tags with differing content
within one scope (box=strata); using newest referenced v2026.323.0252 (from
github.com/opencharly/layer-cuda@v2026.323.0252); also referenced v2026.321.1520 (from
github.com/opencharly/layer-cuda@v2026.321.1520; scopes: box=strata); also referenced
v2026.320.2103 (from github.com/opencharly/layer-cuda@v2026.320.2103; scopes: box=strata)
```

Three candidates, newest wins, `rc=0`. And when the two references sit in **different** box scopes —
which is the shape of the real composition, where the base is one box and the consumer another —
the line is not printed at all (`scopeConflicts` is a within-one-box filter, and a version
difference across boxes is "the normal shape of a multi-box closure and is silent by construction"):

```
$ charly box generate          # pod-cdi base (layer-cdi 0302) + the strata candy (layer-cuda 2103)
$ echo "rc=$?"
rc=0
$ ls .build/_candy/ | grep -E 'cuda|nvidia'
cuda.v2026.323.0252
nvidia.v2026.320.1938
```

So the skew does not stop the build, and the SILENT version is the dangerous one: the winner is
whoever's tag sorts newest, and `charly#1179` records the case where a stale SHA won the
comparison outright. What a pin chain must therefore prove is not "the loader will refuse" but
"no second pin exists", which is what the pinned tag `v2026.323.1751` delivers (§5c).

### 5c. AS PINNED by `v2026.323.1751` — the candy names `layer-cdi` and nothing else GPU

The tag exists, and it is MERGED — the two facts AGENTS.md Part II #3 requires of a pin, read from
the remote and from a clone of the repo:

```
$ git ls-remote https://github.com/opencharly/layer-strata.git refs/tags/v2026.323.1751
b61201d052cdc41690c377b78dea6a90874594e0	refs/tags/v2026.323.1751
$ git -C <layer-strata clone> fetch origin main
$ git -C <layer-strata clone> merge-base --is-ancestor b61201d052cdc41690c377b78dea6a90874594e0 origin/main
$ echo "ancestor_rc=$?"
ancestor_rc=0
$ git -C <layer-strata clone> show v2026.323.1751:candy/strata/charly.yml \
    | grep -n "@github.com/opencharly\|STRATA_VERSION\|STRATA_ENGINE_VERSION"
72:            - '@github.com/opencharly/layer-supervisord:v2026.314.0710'
83:            - '@github.com/opencharly/layer-cdi:v2026.323.1259'
89:            STRATA_VERSION: v0.2.16
92:            STRATA_ENGINE_VERSION: 0.1.62
```

`ancestor_rc=0` is what makes it a PIN and not a tag standing beside `main`. The box that tag carries
is the one this repo mirrors — read at the same tag:

```
$ git -C <layer-strata clone> show v2026.323.1751:box/strata/charly.yml | grep -n 'base:\|security_opt'
21:        base: pod-cdi.pod-cdi
49:            security_opt:
```

Its own comment states the reason for the tag it names: *"v2026.323.1259 is a merged release whose
`require:` names `layer-cuda:v2026.323.0252`, the toolkit release that dropped
`require: layer-nvidia` … Naming `layer-nvidia` here would bake the VM contract's driver
user-space."* Verified:

```
$ gh api repos/opencharly/layer-cdi/contents/charly.yml?ref=v2026.323.1259 --jq .content \
    | base64 -d | grep -n "@github.com/opencharly"
69:            - '@github.com/opencharly/layer-cuda:v2026.323.0252'
$ git ls-remote --tags https://github.com/opencharly/layer-cdi \
    | sed 's#.*refs/tags/##' | grep -v '\^{}' | sort -V | tail -5
v2026.321.1745
v2026.322.1100
v2026.323.0156
v2026.323.0302
v2026.323.1259
```

So, as pinned: the candy names `layer-cdi:v2026.323.1259`; both `layer-cdi` tags in the closure
(§2's `0302` from the base, `1259` from the candy) resolve `layer-cuda:v2026.323.0252`; and
**`layer-cuda` is pinned once, at one tag, by a single layer**. Measured proxy for exactly that
closure (a box on the `pod-cdi` base that additionally names the candy's `layer-cdi`):

```
$ charly box generate
$ echo "rc=$?"
rc=0
$ ls .build/_candy/ | grep -E 'cdi|cuda|nvidia'
cdi.v2026.323.1259
cuda.v2026.323.0252
```

Newest wins for `layer-cdi` (`1259` > `0302`), no notice printed, no `nvidia` stage at all.

### 5d. The one skew the pinned closure carries — NAMED, MEASURED, and owned elsewhere

The base pins `layer-cdi:v2026.323.0302` and the candy requires `layer-cdi:v2026.323.1259`: the same
candy at two tags, in two different box scopes. It resolves to `1259` (measured in §5c) and both
tags name the SAME `layer-cuda:v2026.323.0252`, so the image that gets built is coherent either way.

**Measured harmless, not argued.** The composition's own bed ran on exactly this closure — twice:

- `check-strata-gpu-pod`, calver **`2026.323.1728`** (the producer's run): PASS, `steps=16`,
  `skipped_checks=12`, diagnostics `{errors 0, warnings 0, advisories 0}`, all 15 strata probes PASS,
  including `strata-engine-component` (upstream's own verifier: usable, signed, `0.1.62`, cuda13)
  and `strata-engine-usable` against the CDI-injected driver;
- the SAME bed in THIS repo, calver **`2026.323.1759`**: PASS, `steps=16`, `bed_rc=0`, with all four
  of this repo's own assertions PASS (`strata-gpu-projection`, `strata-gpu-loaded`,
  `strata-gpu-vision-roundtrip`, `strata-gpu-terminal`) — pasted in the PR body's `## How tested`.

The resolver settles the skew on the direct candy list; the bed is what proves which `layer-cdi`
actually landed.

The fix is **not** here. It belongs to `pod-cdi` — filed as
[`opencharly/pod-cdi#7`](https://github.com/opencharly/pod-cdi/issues/7) ("the base's own
`layer-cdi` pin and a consumer's can disagree, and the resolver resolves it silently") — never to a
hand-pin in this repo and never to a branch checkout.

## 6 — the staged repo proves its own composition

Run on the staged tree (`.dsh/strata-pod-staging`, `charly` = `./charly/bin/charly` at
`2026.321.2122`). No image is built and no GPU is touched by either command.

```
$ charly box validate
charly box validate: OK — checked 136 candies, 13 boxes, 1 deploy, 6 distros, 7 builders;
0 warnings, 0 errors
$ echo "validate_rc=$?"
validate_rc=0

$ charly box generate
$ echo "generate_rc=$?"
generate_rc=0

$ charly box inspect strata
{
  "name": "strata",
  "status": "testing",
  "info": "The Strata POD — Qwen3.8-Flash-Next (a 125B-parameter MoE) served from ONE consumer",
  "check_level": "noagent",
  "base": "pod-cdi.pod-cdi",
  "platforms": [
    "linux/amd64"
  ],
  "tag": "2026.323.1454",
  "registry": "ghcr.io/opencharly",
  "pkg": "pac",
  "distro": [
    "cachyos",
    "arch"
  ],
  "build_formats": [
    "pac"
  ],
  "tags": [
    "all",
    "cachyos",
    "arch",
    "pac"
  ],
  "candy": [
    "github.com/opencharly/layer-strata/candy/strata"
  ],
  "user": "user",
  "uid": 1000,
  "gid": 1000,
  "home": "/home/user",
  "full_tag": "ghcr.io/opencharly/strata:2026.323.1454",
  "ports": [
    "8080"
  ],
  "volumes": [
    {
      "volume_name": "charly-strata-strata-data",
      "container_path": "/data"
    }
  ],
  "description": "The Strata POD — Qwen3.8-Flash-Next (a 125B-parameter MoE) served from ONE consumer\n
  … (the box `description:`, elided here; the remainder of the JSON is unchanged) …"

**`validate` alone is not the gate for this file — `generate` is.** The `pod-cdi` import must sit at
the PROJECT root. Moving the same import into `box/strata/charly.yml`, beside the `base:` that uses
it, still passes `box validate` and silently loses the base:

```
$ charly box validate          # import in box/strata/charly.yml, nothing else changed
charly box validate: OK — checked 12 candies, 1 box, 1 deploy, 6 distros, 7 builders;
0 warnings, 0 errors
$ charly box generate
charly: error: command "generate": rendering Containerfiles: generating Containerfile for strata:
candy github.com/opencharly/layer-build-toolchain: resolved 12 package(s) [autoconf automake binutils
bison ccache cmake flex gdb git libtool nasm patch] but box strata has no distro definition to
install them with
$ echo "generate_rc=$?"
generate_rc=1
```

`12 candies, 1 box` against the root placement's `136 candies, 13 boxes` is the diagnostic: a
namespace mounted by a discovered document is visible to that document's own lookups and not to the
project's entity set, so the base's `distro:`/`build:` chain never arrives — and `validate` stays
green because every entity it did load is individually consistent. Run both verbs after any
manifest edit.

`0 warnings` is part of the pass, and the `distro`/`build_formats`/`tags` triple is the measured
answer to the base change: the distro CHAIN is inherited across the namespace, the build FORMAT
LIST is not (`box/strata/charly.yml` declares `build: [pac]` for that reason, and the reason is
pasted there).

The materialized candy set **as pinned** — the tree this repo ships. No `nvidia` stage appears
anywhere: the VM contract's driver user-space is not in the closure.

```
$ ls .build/_candy/ | grep -E 'strata|cdi|cuda|nvidia'
cdi.v2026.323.1259
cuda.v2026.323.0252
strata.v2026.323.1751
```

For contrast, the PRE-upgrade pin (`v2026.320.2149`) composed the driver user-space — which is
exactly what `T1` was required to delete, and why no build of this box was attempted at that tag:

```
$ ls .build/_candy/ | grep -E 'cuda|nvidia'          # at the pre-upgrade pin, measured earlier
cuda.v2026.323.0252
nvidia.v2026.320.1938
```

`nvidia.v2026.320.1938` is exactly the stage `layer-cdi`'s `cdi-no-baked-driver-libs` /
`cdi-no-baked-driver-binaries` steps fail the build on. That refusal is the contract working; it
is not run here (this host has one GPU and the `strata-layer` teammate holds it).

## 7 — the exact pins `task-3` writes

Two pins, both in this repo, both `@github` refs, both MERGED TAGS:

| File | Line | Value |
|---|---|---|
| `charly.yml` (root) | `import:` → `pod-cdi:` | `'@github.com/opencharly/pod-cdi:v2026.323.1302'` |
| `box/strata/charly.yml` | `candy:` | `'@github.com/opencharly/layer-strata/candy/strata:v2026.323.1751'` |

`v2026.323.1751` is the merge tag of `layer-strata`'s v0.2.16 PR (ledger plan step 1,
`opencharly/layer-strata#24`, merged 2026-11-19T17:51:55Z) — the tag whose candy names
`layer-cdi:v2026.323.1259` and no `layer-nvidia`, proven at the tag in §5c. When it is advanced,
re-read `pod-cdi`'s newest merged tag too (§1's command): a newer `pod-cdi` may name a different
`layer-cdi` and therefore a different `layer-cuda`, and this file's §2–§4 must be re-run against it.

Nothing else in this repo is pinned by hand. The bed's `image: strata` is a local box name, and its
assertions carry literals (`65536`, `strata 0.2.16`, `build 0.1.62`, container `uid 0`) that belong
to the candy's `env:`/`var:` and to the substrate's mapping — they move with the candy's tag, not
with a pin here.
