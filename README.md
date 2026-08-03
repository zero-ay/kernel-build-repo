# Auto Kernel Builder (android12-5.10 + KernelSU-Next)

Builds the Android Common Kernel via GitHub Actions, integrating KernelSU-Next,
and packages the result into a **flashable AnyKernel3 zip**. Kernel source,
build scripts, and toolchain are all cloned fresh on the runner each run — no
`repo init`/`repo sync`, and the only things committed to this repo are the
workflow, its composite actions, the SUSFS patchset (`patches/susfs/`), and the
AnyKernel3 template.

## Structure

```
.
├── .github/
│   ├── workflows/
│   │   └── build-kernel.yml           # orchestrates the whole build
│   └── actions/
│       ├── install-deps/              # host packages (apt)
│       ├── clone-kernel-source/       # kernel/common (kernel_branch input) + kernel/build @ master-kernel-build-2021 (the branch that ships build.sh)
│       ├── setup-kernelsu/            # KernelSU-Next integration (CONFIG_KSU via its Kconfig default)
│       ├── setup-susfs/               # applies the SUSFS patchset (patches/susfs/) to kernel + KernelSU-Next
│       ├── setup-bbrv3/               # applies the TCP BBR v3 port (patches/bbrv3/) to common/
│       ├── setup-toolchain/           # toolchain mirroring the official android12-5.10 manifest: shallow clones @ master-kernel-build-2021, clang-r416183b
│       └── build-kernel/              # strips -dirty, runs the pinned build.sh command, locates dist/
├── patches/
│   ├── susfs/                         # SUSFS patchset adapted for KernelSU-Next v3.3.0: 10_enable + 50_add patches, fs/susfs.c, include/linux/susfs*.h
│   └── bbrv3/                         # BBRv3 backport patchset (fatalcoder524) for android12-5.10 / 13-5.15 / 14-6.1 / 15-6.6
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
Google repos, mirroring the official `kernel/manifest`
(`common-android12-5.10`) exactly: same paths and same
`master-kernel-build-2021` revision (the manifest's `<default revision=...>`).
That is the only `kernel/build` branch that still ships the legacy `build.sh`
entry point — newer branches (`main-kernel`, `main`) dropped it in favor of
Kleaf/Bazel. `common/build.config.common` pins `CLANG_PREBUILT_BIN` to
`clang-r416183b`; `setup-toolchain` verifies that folder exists before
building (fail-fast instead of a cryptic `build.sh: No such file or
directory`).

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
  the built vmlinux). The clang comes from the manifest's
  `master-kernel-build-2021` branch (`clang-r416183b`), which passes this
  check; some much newer clangs (e.g. `clang-r547379`) drop `__stack_chk_guard`
  from ksymtab and fail it, so keep the toolchain on the manifest revision.

## Trigger

The workflow only runs when you trigger it manually: **Actions** →
**Build Android Kernel (android12-5.10)** → **Run workflow**. There are no
automatic triggers (no push/schedule), so pushes to `main` won't start a build.

## Modifying the kernel further

- **Small changes:** keep a `patches/*.patch` folder in this repo and add a
  step/composite action that applies them after cloning `common/`.
- **Heavy/ongoing changes:** fork `kernel/common` to your own GitHub repo and
  point `clone-kernel-source` at it instead of Google's source.

## TCP BBR v3

The build integrates **TCP BBR v3** into the common kernel via the
fatalcoder524 backport patchset in `patches/bbrv3/` (one patch per
android12-5.10 / android13-5.15 / android14-6.1 / android15-6.6, plus two
optional sysctl prerequisites). It adds a separate `net/ipv4/tcp_bbr3.c`
module (`CONFIG_TCP_CONG_BBR3`).

- **Applied by:** the `setup-bbrv3` action, immediately after SUSFS. It picks
  `0001-net-tcp-backport-BBRv3-to-<branch>.patch` matching the workflow's
  `kernel_branch`, applies sysctl prerequisites best-effort (they don't apply
  to 5.10+, which already have `proc_dou8vec_minmax`), then applies the patch
  with `patch -p1 -F2`.
- **Enablement:** the action edits `arch/arm64/configs/gki_defconfig` to set
  `CONFIG_TCP_CONG_ADVANCED=y` and `CONFIG_TCP_CONG_BBR3=y`, plus
  `# CONFIG_TCP_CONG_BIC/HTCP/WESTWOOD is not set` to keep GKI's
  loadable-module list empty. The lines round-trip through `savedefconfig`, so
  `check_defconfig` is unaffected. The default CC stays CUBIC; BBRv3 is
  *selectable*, not the default.
- **Use after flashing:** `sysctl -w net.ipv4.tcp_congestion_control=bbr3`
  (or `ip route ... congestion_control bbr3`).

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
