# Auto Kernel Builder (android12-5.10 + KernelSU-Next)

Builds the Android Common Kernel via GitHub Actions, integrating KernelSU-Next,
and packages the result into a **flashable AnyKernel3 zip**. Kernel source,
build scripts, and toolchain are all cloned fresh on the runner each run — no
`repo init`/`repo sync`, and the only things committed to this repo are the
workflow, its composite actions, and the AnyKernel3 template.

## Structure

```
.
├── .github/
│   ├── workflows/
│   │   └── build-kernel.yml           # orchestrates the whole build
│   └── actions/
│       ├── install-deps/              # host packages (apt)
│       ├── clone-kernel-source/       # kernel/common (version-specific) + kernel/build
│       ├── setup-kernelsu/            # KernelSU-Next integration (CONFIG_KSU via its Kconfig default)
│       ├── setup-susfs/               # SUSFS root-hiding patchset (patches/susfs/), adapted for KernelSU-Next
│       ├── setup-toolchain/           # prebuilts: build-tools, mkbootimg, gcc hermetic sysroot, Clang
│       └── build-kernel/              # strips -dirty, runs build.sh, locates dist/
└── kernel-zipping/                    # AnyKernel3 flashable zip template
```

Each folder under `.github/actions/` is a **composite action** — a
self-contained, reusable unit with its own inputs. The main workflow
(`build-kernel.yml`) just calls them in order.

## Setup

1. Push this repo to GitHub (or use it directly).
2. Go to **Actions** → **Build Android Kernel (android12-5.10)** → **Run workflow**.
3. Download the `kernel-<branch>-<run_number>` artifact and flash it from a
   custom recovery (or via Magisk — `do.systemless=1`).

## Inputs (workflow_dispatch)

| Input | Default | Meaning |
|---|---|---|
| `kernel_branch` | `android12-5.10` | Branch/tag of `kernel/common` to build |
| `ksu_ref` | `v3.3.0` | KernelSU-Next tag/commit to check out. Must match the version the SUSFS patch in `patches/susfs/` was adapted for; bumping it requires re-basing that patch. |

Toolchain (`kernel/build`, `prebuilts/build-tools`, the AOSP clang repo, the gcc
hermetic sysroot, mkbootimg) is cloned shallow (`--depth=1`) from the official
Google repos on their default branches — see `setup-toolchain` and
`clone-kernel-source`. The newest `clang-r*` folder in the clang clone is
auto-linked to the path `build.config.common` expects.

## Output

The artifact is the **flashable AnyKernel3 zip** itself (not a zip inside a
zip): `META-INF/`, `anykernel.sh`, `Image` and `tools/` sit at its root, built
from the `kernel-zipping/` template with the freshly compiled kernel image.
Artifacts are kept for 14 days.

## Operational notes

- **Runner:** pinned to `ubuntu-22.04`, the best-tested host for the
  prebuilt toolchain (gcc-4.8 hermetic sysroot).
- **Hermetic build:** android12-5.10 builds with `HERMETIC_TOOLCHAIN=1`, so
  host tools compile against the gcc host prebuilt's sysroot — `setup-toolchain`
  clones it (`prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.17-4.8`).
- **Release string:** the `-dirty` suffix is stripped from `setlocalversion`
  (the tree has uncommitted KernelSU-Next changes), e.g.
  `5.10.223-android12-9-g<sha>`.
- **KMI/ABI check:** enabled. `build.config.gki.aarch64` defaults
  `KMI_SYMBOL_LIST_STRICT_MODE` to 1 (1-1 match of the KMI symbol list against
  the built vmlinux). Note: some newer clangs drop `__stack_chk_guard` from
  ksymtab, which fails this check; if the newest clang in the shallow clone
  breaks it, point the toolchain at an older clang branch/folder.

## Trigger

The workflow only runs when you trigger it manually: **Actions** →
**Build Android Kernel (android12-5.10)** → **Run workflow**. There are no
automatic triggers (no push/schedule), so pushes to `main` won't start a build.

## Modifying the kernel further

- **Small changes:** keep a `patches/*.patch` folder in this repo and add a
  step/composite action that applies them after cloning `common/`.
- **Heavy/ongoing changes:** fork `kernel/common` to your own GitHub repo and
  point `clone-kernel-source` at it instead of Google's source.

## SUSFS (root hiding)

The build integrates **SUSFS** (`susfs4ksu`, branch `gki-android12-5.10`) on
top of KernelSU-Next — see `patches/susfs/README.md` for details.

- `patches/susfs/10_enable_susfs_for_ksu.patch` is **adapted for KernelSU-Next
  v3.3.0** (upstream's patch targets official KernelSU and does not apply to
  KernelSU-Next). It is applied inside the `KernelSU-Next` checkout by
  `setup-susfs`.
- `patches/susfs/50_add_susfs_in_gki-android12-5.10.patch` plus
  `fs/susfs.c` and `include/linux/susfs*.h` are applied to `common/`.
- `CONFIG_KSU_SUSFS` and all `CONFIG_KSU_SUSFS_*` options are enabled via their
  Kconfig `default y` (no defconfig edits — adding explicit lines would break
  `check_defconfig`).
- After flashing, install the SUSFS userspace module (e.g.
  [sidex15/susfs4ksu-module](https://github.com/sidex15/susfs4ksu-module))
  through the KernelSU-Next manager to use `ksu_susfs`.
