# TCP BBR v3 (fatalcoder524 backport patchset)

This folder replaces the earlier single-patch BBRv3 port with the
`fatalcoder524` android-common backport of the modularized **BBRv3**
(`net/ipv4/tcp_bbr3.c`, `CONFIG_TCP_CONG_BBR3`), shipped with Android KABI
padding.

## Files

| Patch | Applies to |
|-------|-----------|
| `0001-net-tcp-backport-BBRv3-to-android12-5.10.patch` | kernel/common `android12-5.10` |
| `0001-net-tcp-backport-BBRv3-to-android13-5.15.patch` | `android13-5.15` |
| `0001-net-tcp-backport-BBRv3-to-android14-6.1.patch`  | `android14-6.1` |
| `0001-net-tcp-backport-BBRv3-to-android15-6.6.patch`  | `android15-6.6` |
| `sysctl_add_proc_dou8vec_minmax.patch` | optional, older bases only |
| `sysctl_fix_data-races_in_proc_dou8vec_minmax.patch`  | optional, older bases only |

The two sysctl patches are upstream prerequisites for `proc_dou8vec_minmax()`
(then a data-race fix). The android 5.10–6.6 GKI branches already contain that
handler, so `setup-bbrv3` skips them there; they are applied best-effort only
for bases/forks that lack it.

## Workflow integration

`.github/actions/setup-bbrv3` chooses the patch matching the workflow's
`kernel_branch` input, runs the two sysctl patches best-effort, then applies
the BBRv3 patch with `patch -p1 -F2` inside `common/`. Finally it enables the
config in `arch/arm64/configs/gki_defconfig`:

```kconfig
CONFIG_TCP_CONG_ADVANCED=y
# CONFIG_TCP_CONG_BIC is not set
# CONFIG_TCP_CONG_WESTWOOD is not set
# CONFIG_TCP_CONG_HTCP is not set
CONFIG_TCP_CONG_BBR3=y
```

Keeping `CONFIG_TCP_CONG_BBR3=y` (built-in) and disabling the three
`=m`-by-default CCs keeps the GKI loadable-module list empty (so the build's
"modules list out of date" check no-ops). All lines round-trip through
`scripts/savedefconfig`, so `check_defconfig` still passes. The default CC
remains CUBIC.

## Use after flashing

```sh
sysctl -w net.ipv4.tcp_congestion_control=bbr3
# or per-route:
ip route ... congestion_control bbr3
```

