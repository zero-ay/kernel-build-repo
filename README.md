# Auto Kernel Builder (android12-5.10 + KernelSU-Next)

Builds the Android Common Kernel via GitHub Actions, integrating KernelSU-Next.
Source, build scripts, and toolchain are all cloned fresh on the runner each
run — no `repo init`/`repo sync`, and nothing is committed to this repo except
the workflow itself.

## Structure

```
.github/
├── workflows/
│   └── build-kernel.yml           # orchestrates the whole build
└── actions/
    ├── install-deps/              # host packages (apt)
    ├── clone-kernel-source/       # kernel/common + kernel/build
    ├── setup-kernelsu/            # KernelSU-Next integration + CONFIG_KSU
    ├── setup-toolchain/           # build-tools + Clang (cached)
    └── build-kernel/              # runs build.sh, locates dist/
```

Each folder under `.github/actions/` is a **composite action** — a
self-contained, reusable unit with its own inputs. The main workflow
(`build-kernel.yml`) just calls them in order. This keeps each concern
isolated and easy to modify without touching the rest of the pipeline.

## Setup

1. Push this whole `.github/` folder to a new GitHub repo.
2. Go to **Actions** → **Build Android Kernel (android12-5.10)** → **Run workflow**.

## Inputs (workflow_dispatch)

| Input | Default | Meaning |
|---|---|---|
| `kernel_branch` | `android12-5.10` | Branch/tag of `kernel/common` and `kernel/build` |
| `build_config` | `common/build.config.gki.aarch64` | Build config to build |
| `lto` | `thin` | LTO mode: `thin` \| `full` \| `none` |
| `clang_version` | `clang-r416183b` | Clang prebuilt folder name |
| `clang_ref` | `master-kernel-build-2021` | Git ref on the clang prebuilts repo |
| `ksu_ref` | *(empty)* | KernelSU-Next tag/commit to pin; empty = latest tag |

If you switch `kernel_branch` to a different Android/kernel version, check
that branch's `build.config.common` for the correct `CLANG_VERSION` and
update `clang_version`/`clang_ref` to match. Also confirm the defconfig path
used in `setup-kernelsu` (`common/arch/arm64/configs/gki_defconfig` by
default) matches that branch's layout.

## Output

Finished kernel images are uploaded as a downloadable **Actions artifact**
named `kernel-<branch>-<run_number>` (kept 14 days).

## Automatic triggers

- Every push to `main`
- Weekly, Monday 00:00 UTC (remove the `schedule:` block if unwanted)

## Modifying the kernel further

- **Small changes:** keep a `patches/*.patch` folder in this repo and add a
  step/composite action that applies them after cloning `common/`.
- **Heavy/ongoing changes:** fork `kernel/common` to your own GitHub repo and
  point `clone-kernel-source` at it instead of Google's source.
