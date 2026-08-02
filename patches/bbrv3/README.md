# TCP BBR v3 (ports from github.com/google/bbr)

This folder ports **Google's TCP BBR v3** into the android12‑5.10 GKI kernel.

## Source

The patch is generated from the upstream **BBRv3 release branch**:

* repo:  https://github.com/google/bbr
* branch: `v3` (tag `bbrv3-2025-03-18`), which is a full Linux tree based on
  **6.13.7** (VERSION=6, PATCHLEVEL=13, SUBLEVEL=7) carrying the BBR v3 work.

The bbr repo's `v3` branch changes these files vs. mainline 6.13.7:

| File                                   | Change |
|----------------------------------------|--------|
| `net/ipv4/tcp_bbr.c`                   | BBR v1+v2+v3 all‑in‑one congestion control (`BBR_VERSION=3`) |
| `net/ipv4/tcp_bbr1.c` (new)            | BBR v1 standalone module, "for testing" |
| `net/ipv4/tcp_rate.c`                  | `tcp_set_tx_in_flight()`, per‑skb `tx.lost`/`tx.in_flight`/`tx.delivered_ce`, 32‑bit tx timestamps |
| `net/ipv4/tcp_input.c`                 | `CA_EVENT_TLP_RECOVERY`, `skb_marked_lost` hook, TLP ack tagging, `tcp_rate_check_app_limited()`, `fast_ack_mode`, `tcp_ca_wants_ce_events()` |
| `net/ipv4/tcp_output.c`                | `tso_segs` ca‑op (replaces `min_tso_segs`), `tcp_set_tx_in_flight()`, ECN‑LOW / ECT‑permanent, TLP app‑limited tracking |
| `net/ipv4/tcp_timer.c`                 | call `tcp_rate_check_app_limited()` on retransmit |
| `net/ipv4/tcp_cong.c`                  | reset `fast_ack_mode` |
| `net/ipv4/tcp.c` / `tcp_minisocks.c`   | `fast_ack_mode`, `TCPI_OPT_ECN_LOW` stats, ECN‑LOW from dst |
| `net/ipv4/bpf_tcp_ca.c`                | `tso_segs` op rename |
| `include/net/*`, `include/linux/tcp.h` | new BBR fields/ops, `tcp_skb_cb.tx` layout, rate_sample fields |
| `include/uapi/.../*`                    | `bbr3` diag info, `RTAX_FEATURE_ECN_LOW`, `TCPI_OPT_ECN_LOW`, `TCP_PLB` MIB |
| `net/ipv4/Kconfig`, `Makefile`          | `TCP_CONG_BBR1` + `tcp_bbr1.o` |
| `arch/arm64/configs/gki_defconfig`     | enable `TCP_CONG_ADVANCED` + `TCP_CONG_BBR` (below) |

## Porting notes (6.13.7 → android12‑5.10)

5.10 predates several things the 6.13 BBRv3 code depends on; this patch adapts them:

- **`BTF kfunc` registration dropped.** 5.10 has no `BTF_KFUNCS_*` /
  `register_btf_kfunc_id_set()`. The `_kfunc_ids`/`__bpf_kfunc` exports were
  stripped; the module still registers normally and BPF‑style tracing of the
  bbr callbacks is simply not exposed (a kernel‑internal behavior difference,
  no effect on the CC itself).
- **`cong_control` signature.** 6.13's `tcp_congestion_ops.cong_control`
  gained an `int flag` argument; 5.10 has `cong_control(sk, rs)`. `bbr_main()`
  and `bbr1_main()` did not use `ack`/`flag`, so the extra parameter is
  dropped to match 5.10.
- **PLB backport.** BBRv3 uses TCP PLB (Protective Load Balancing, upstream
  5.16). This patch includes the PLB subsystem backport (`net/ipv4/tcp_plb.c`,
  5 `tcp_plb_*` sysctls, `struct tcp_plb_state`, `TCP_PLB_SCALE`,
  `LINUX_MIB_TCPPLBREHASH`, `tp->plb_rehash`). One adaptation:
  `get_random_u32_below()` (6.11+) → `prandom_u32_max()`.
- **`tcp_snd_cwnd()` / `tcp_snd_cwnd_set()`** (6.4 helpers) are added inline.
- **`GSO_LEGACY_MAX_SIZE`** (6.11) → 5.10's `GSO_MAX_SIZE` (same value).
- **`tcp_mstamp`/tx timestamps:** 5.10's `struct tcp_sock`/`tcp_skb_cb`
  already use µs u64 mstamps; the bbr rate path uses the new 32‑bit
  `tcp_stamp32_us_delta()` plus the rewritten `tcp_skb_cb.tx` layout.

### GKI integration

- **`ICSK_CA_PRIV_SIZE`** 104 → **144** so `struct bbr` fits in `icsk_ca_priv`.
- **Defconfig:** android12‑5.10's `gki_defconfig` explicitly sets
  `# CONFIG_TCP_CONG_ADVANCED is not set`, which hides every advanced CC
  including BBR. The patch edits `arch/arm64/configs/gki_defconfig` to set
  `CONFIG_TCP_CONG_ADVANCED=y` and `CONFIG_TCP_CONG_BBR=y`, and to keep the GKI
  loadable‑module list empty it also sets `# CONFIG_TCP_CONG_BIC/HTCP/
  WESTWOOD is not set` (those three default to `=m` with ADVANCED enabled and
  would otherwise trip the build's "modules list out of date" check). All
  lines round‑trip through `scripts/savedefconfig`, so Android's
  `check_defconfig` still passes.
- **Default CC stays CUBIC** (`CONFIG_DEFAULT_TCP_CONG="cubic"`). BBRv3 becomes
  *selectable*, not the default.
- **KMI:** new `EXPORT_SYMBOL_GPL`s (the 3 `tcp_plb_*`) are trimmed by the
  GKI build's `TRIM_NONLISTED_KMI=1` (they are not in the KMI symbol list and
  only used by the built‑in BBR module), so the 1‑1 KMI check is unaffected.

## Using BBRv3

The `setup-bbrv3` action applies this patch in `common/` right after
`setup-susfs`. Once the kernel is built and booted:

```sh
# list available congestion controls (should include bbr)
cat /proc/sys/net/ipv4/tcp_available_congestion_control

# switch the default congestion control to BBRv3
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

You can also enable it per‑route: `ip route ... congestion_control bbr`.

BBRv3 diag (`ss -tin`) uses the extended `struct tcp_bbr_info` added to
`include/uapi/linux/inet_diag.h` (bw_hi/bw_lo, inflight_hi/inflight_lo, mode,
phase, version).

## Files

- `50_add_bbrv3_in_gki-android12-5.10.patch` — the port (23 files,
  applies cleanly with `patch -p1 -F2` against a pristine android12‑5.10
  `kernel/common` checkout).