# custom-proton

A thin overlay for [CachyOS/proton-cachyos](https://github.com/CachyOS/proton-cachyos)
and Valve's Proton Experimental.

This repo holds **no Proton source** — only patches and GitHub Actions workflows. Each
workflow clones upstream at a branch you choose, applies the patches on top, builds it
with the CPU optimisation you pick, and hands you the result as a downloadable artifact.

```
proton-cachyos/                            CachyOS overlay patches
valve-experimental/                        Proton Experimental patch set
.github/workflows/build.yml                CachyOS build workflow
.github/workflows/build-experimental.yml   Proton Experimental build workflow
```

## Building Proton-CachyOS

1. Go to the **Actions** tab → **Build Proton-CachyOS** → **Run workflow**.
2. Fill in the inputs (defaults are fine) and hit the green button.
3. When it finishes, download the `.tar.xz` from the run's **Artifacts** section.

### Inputs

| Input | Default | What it does |
|---|---|---|
| `branch` | `cachyos-11.0-20260702-slr` | Branch **or tag** of `CachyOS/proton-cachyos` to build from. Prefer an `-slr` **tag** — see below |
| `march` | `zen4` | CPU target. One of `zen4`, `zen3`, `zen2`, `x86-64-v4`, `x86-64-v3`, `nocona` |
| `patch_001` | `true` | Apply the CPUID, KUSER_SHARED_DATA, and faketime hardware patch. Untick it for a stock upstream build with optimisations only |
| `dry_run` | `false` | Validate only — checkout, patch and configure, then stop. Takes ~5 min instead of hours. Use it to check that a new upstream branch still applies cleanly before committing to a full build |

`march` selects `CFLAGS` only (`nocona` is upstream's stock setting):

| `march` | `CFLAGS` |
|---|---|
| `zen4` | `-O3 -march=znver4 -mtune=znver4` |
| `zen3` | `-O3 -march=znver3 -mtune=znver3` |
| `zen2` | `-O3 -march=znver2 -mtune=znver2` |
| `x86-64-v4` | `-O3 -march=x86-64-v4 -mtune=core-avx2` |
| `x86-64-v3` | `-O3 -march=x86-64-v3 -mtune=core-avx2` |
| `nocona` | `-O3 -march=nocona -mtune=core-avx2` |

`RUSTFLAGS` is always `-Copt-level=3 -Ctarget-cpu=nocona`, regardless of `march`. That is
deliberate and matches upstream, which pairs even its `x86-64-v3` build with nocona Rust
flags. The SteamRT SDK's bundled rustc does not recognise newer CPU names — an
unrecognised `-Ctarget-cpu` is silently ignored and LLVM falls back to a subtarget that
cannot emit 64-bit code, so `gst-plugins-rs` fails with *"LLVM ERROR: 64-bit code
requested on a subtarget that doesn't support it"* about 90 minutes into the build.
The Rust components are a small part of the tree; the C/C++ bulk still gets your `march`.

## Building Proton Experimental

Use **Actions → Build Proton Experimental → Run workflow**. It checks out
`ValveSoftware/Proton` directly, applies the hardware patch to Valve's Wine submodule,
applies the extracted OptiScaler series to a pinned `umu-protonfixes` checkout, and
packages the result as an artifact. The default `ref` tracks Valve's
`experimental_11.0` branch and the default CPU target is `zen4`.

The Experimental workflow has the same `march` choices as the CachyOS workflow. It
prompts separately for the hardware patch and the OptiScaler patch set, both enabled by
default. The OptiScaler option includes all eight dependent upscaler patches. Its
`dry_run` input stops after checkout, patch validation, and configuration. Enable
OptiScaler at runtime with:

```sh
PROTON_USE_OPTISCALER=1 %command%
```

### How long it takes

A cold first run is the slow one — expect **1–3 hours**. Subsequent runs with the same
`branch` + `march` reuse a ccache and typically land around **35–45 minutes**. The job
cap is 350 minutes. Runners are free and unlimited because this repo is public.

## Which upstream branch should I pick?

Upstream tags and branches carry suffixes that are easy to misread:

- **`slr`** — *Steam Linux Runtime*. Runs inside Valve's pinned SLR container, exactly
  like official Proton (`require_tool_appid 4183110`). This is what CachyOS maintainers
  recommend "for maximum compatibility", and the only variant their own CI publishes.
  **This is the one you want**, and it's what the default branch produces.
- **`native`** — links against your host's system libraries instead, with no container.
  Avoids the SLR layer but depends on your distro's `lib32-*` packages (which Arch is
  progressively dropping) and is more likely to trip over anti-cheat.
- **`base`** — *not an end-user build.* It's an internal rebase anchor: upstream Proton
  at that date, before CachyOS's wine fork and patch stack are reapplied. It ships with
  roughly half the submodules. Don't build it.
- **`experimental-11.0-<date>`** / **`experimental-bleeding-edge-…`** — mirrors of
  **Valve's** upstream tags recording which Proton Experimental commit a CachyOS release
  was rebased onto. Tracking markers, not CachyOS flavours.
- **`_sauce` / `_action` / `_umu` / `_native`** — rolling feature branches that get merged
  into each dated release. `_sauce` is the feature grab-bag, `_action` is CI, `_umu` is
  umu-launcher integration.

There is **no `-c` variant**. A trailing letter such as `20260724c` is Valve's own
same-day disambiguator on their upstream tags.

### Prefer an `-slr` tag over a dated branch

**Dated `cachyos_11.0_YYYYMMDD/main` branches are deleted once superseded**, and not every
dated line ever gets released. The `20260713` line, for instance, built out to a full
stack and was then dropped without ever receiving an `-slr` tag; its branch no longer
exists. Release tags are permanent, so build from those. List them:

```sh
git ls-remote --tags https://github.com/CachyOS/proton-cachyos 'refs/tags/cachyos-11.0-*-slr' \
  | sed 's|.*/tags/||' | grep -v '\^{}' | sort | tail
```

The newest `-slr` tag is the newest finished release. A date that has only a `-base` tag
and no `-slr` is **not** finished.

**Why a `-base`-only ref cannot be built:** CachyOS pushes each dated line at its `-base`
rebase anchor first and applies the wine fork and feature stack afterwards. In that
intermediate state it carries roughly half the submodules and still points wine at
upstream's relative `../wine`, which resolves to a repository that does not exist — the
submodule fetch then fails with a confusing credential prompt. To check any ref:

```sh
git describe --long --tags        # in a checkout of that ref
```

`cachyos-11.0-20260724-base-0-g…` — offset `0` from a `-base` tag means **not ready**.
`cachyos-11.0-20260702-slr-0-g…` — a `-slr` tag means a finished release, good to build.
The workflow enforces this and stops early with a clear message rather than failing deep
in the submodule fetch.

## Naming

Builds are named `proton-zi0p4tch0-<version>-<postfix>`, where the version comes from
`git describe --tags` on the ref you built and the postfix ends in your `march` target:

```
proton-zi0p4tch0-11.0-20260702-slr-zen4
└──────────── version ───────────┘└──┘ postfix
```

Building an exact tag yields the bare tag name; a ref that sits past a tag keeps its
`-N-g<sha>` offset so the build stays identifiable. Names are kept short deliberately —
Steam's Gamemode UI truncates long compatibility-tool names in the dropdown.

An unpatched build and a patched build of the same ref get different postfixes, so both can
sit in `compatibilitytools.d` at once and be compared from Steam's dropdown.

That exact string is what Steam shows in its compatibility-tool dropdown. It reaches
there via a single knob — `configure.sh --build-name=` → the `BUILD_NAME` make variable →
`sed` substitution into `compatibilitytool.vdf`'s `display_name` field
(`Makefile.in:1862`). The same variable also names the tarball and the
`compatibilitytools.d/` install directory, so everything stays consistent.

Note that `toolmanifest_x86_64.vdf` carries no name field at all — only `commandline`,
`require_tool_appid` and friends. The user-visible name lives solely in
`compatibilitytool.vdf`.

## Installing the result

The run's **Artifacts** section gives you a single `<build-name>.zip`. GitHub always wraps
artifacts in a zip — that cannot be turned off — so unwrap it once, then extract the
tarball inside:

```sh
unzip proton-zi0p4tch0-….zip                  # yields the .tar.xz and its .sha512sum
sha512sum -c proton-zi0p4tch0-….sha512sum     # optional
mkdir -p ~/.local/share/Steam/compatibilitytools.d
tar -xf proton-zi0p4tch0-….tar.xz -C ~/.local/share/Steam/compatibilitytools.d/
```

The tarball contains a single top-level directory named after the build, so it lands as
`~/.local/share/Steam/compatibilitytools.d/<build-name>/` with no extra nesting. Check it:

```sh
ls ~/.local/share/Steam/compatibilitytools.d/*/compatibilitytool.vdf
```

If you run Steam via Flatpak the path is
`~/.var/app/com.valvesoftware.Steam/data/Steam/compatibilitytools.d` instead.

Restart Steam, then pick the build under
*Properties → Compatibility → Force the use of a specific Steam Play compatibility tool*.

## Caveat: AVX and `-march=znver4`

Upstream's `Makefile.in` unconditionally appends `-mno-avx -mno-avx2 -mno-avx512f` after
computing `x86_64_CFLAGS` (lines 113–114 for GCC, 121–122 for Clang). Since last-flag-wins,
`-march=znver4` still gives you znver4 scheduling plus BMI1/BMI2/LZCNT/MOVBE/ADX, but
**not** AVX2/AVX-512 codegen. This is intentional and matches what upstream's own
`x86-64-v3` build does. The workflow does not fight it.

## Patches

The CachyOS workflow applies the selected patches from `proton-cachyos/` with `git apply`
from the root of its upstream checkout. The Experimental workflow uses the separate
`valve-experimental/` patch set. A patch that fails to apply stops the run immediately,
so after an upstream bump use `dry_run` before starting a full build.
