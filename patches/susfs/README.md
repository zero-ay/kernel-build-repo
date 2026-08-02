# SUSFS patches (for KernelSU-Next)

SUSFS (`susfs4ksu`, branch `gki-android12-5.10`) is an addon root-hiding
patchset for KernelSU. Upstream's patches are written against **official
KernelSU**; this repo builds **KernelSU-Next**, so the KernelSU-side patch is
re-adapted here.

| File | What it is |
|---|---|
| `10_enable_susfs_for_ksu.patch` | **Adapted** for KernelSU-Next `v3.3.0`'s `kernel/` tree (applied inside the `KernelSU-Next` repo with `patch -p1`). Upstream's version applies cleanly only to official KernelSU `main`. |
| `50_add_susfs_in_gki-android12-5.10.patch` | Upstream kernel-side patch (applied in `common/` with `patch -p1`). Verified to apply cleanly to the latest `android12-5.10` HEAD. |
| `fs/susfs.c`, `include/linux/susfs.h`, `include/linux/susfs_def.h` | SUSFS kernel-side sources copied into `common/fs/` and `common/include/linux/`. |

## Pinning

- `ksu_ref` (workflow input) defaults to **`v3.3.0`** — the KernelSU-Next
  version the adapted `10_enable_susfs_for_ksu.patch` targets. Keep
  `setup-kernelsu` and `setup-susfs` on the same ref.
- If you bump `ksu_ref` to a newer KernelSU-Next, the patch may fail to apply;
  re-base it (same procedure as below) before building.

## How the adaptation was produced (for re-basing)

1. Clone `KernelSU-Next` at the target tag and official `tiann/KernelSU` `main`.
2. Apply upstream `10_enable_susfs_for_ksu.patch` to the official tree.
3. 3-way merge each touched file (`git merge-file` with the official file as
   base) onto the KernelSU-Next file, resolving conflicts so KernelSU-Next's own
   features are kept (e.g. `extras.o` in Kbuild, the legacy reboot-supercall
   commands in `supercall/supercall.c`, the SULog compat bridge in
   `sulog/event.c`).
4. `git diff` the KernelSU-Next tree and verify `git apply --check` passes on a
   fresh checkout of the tag.

Note: KernelSU-Next ≥ v3.3.0 already carries partial SUSFS glue (e.g.
`supercall/dispatch.c` dispatches `SUSFS_MAGIC` commands, `selinux/` and
`feature/adb_root.c` use susfs helpers, `Makefile` prints `SUSFS_VERSION`).
The patch completes that integration — including defining
`ksu_supercall_reboot_handler()`, which v3.3.0 calls but does not define.
