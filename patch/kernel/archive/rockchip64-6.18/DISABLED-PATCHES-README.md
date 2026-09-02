# Disabled Armbian core patches for this fork (kernel 6.18, rockchip64)

## rk3308-vop-output.patch.disabled-conflicts-with-rk3308bs-0003

Renamed (extension changed from .patch so Armbian's *.patch glob skips it) on
2026-08-20. This board (Artillery M1 Pro S1-SOC / RK3308BS) ships its own
self-contained VOP driver patch,
userpatches/kernel/archive/rockchip64-6.18/0003-drm-rockchip-rk3308-vop-driver.patch
(mirrored from M1-SOC-Armbian-Build-scripts-private/patches/0003-*), which adds
the identical `rk3308_vop_lit_win_data` / `rk3308_vop` struct plus register
defines. Confirmed empirically: userpatches apply strictly before this fork's
own catalog patches (see `advanced_patch()` in
lib/functions/compilation/patch/patching.sh + build log "Using kernel patch
dir: archive/rockchip64-6.18" showing our userpatches at position ~5/240 vs
this patch at position ~158/240). If both are active, this patch's context-
fuzzy matching finds a DIFFERENT insertion point nearby and adds a SECOND,
duplicate `static const struct vop_data rk3308_vop` definition -> C
redefinition compile error.

This is a board-specific fork customization, not a statement that Armbian's
patch is wrong. To reconcile upstream later (treating this board as a new
variant), the two implementations should be diffed and merged into one, then
this disable removed.

## Update 2026-08-20 (later same day): trimmed, not fully disabled

Fully disabling rk3308-vop-output.patch broke the UNRELATED sakurapi-rk3308b
board: its DTS references `&vop` / `&display_subsystem` labels that only this
patch's `rk3308.dtsi` hunk provides (confirmed via build error: "Label or path
vop not found", "Label or path display_subsystem not found" while compiling
rk3308-sakurapi-rk3308b.dtb).

Fix: `rk3308-vop-output.patch` now contains ONLY the original `rk3308.dtsi`
hunk (shared infra other boards need). The `rockchip_vop_reg.c`/`.h` hunks
(which duplicated our own userpatches 0003) were removed. Original full patch
kept for reference as
`rk3308-vop-output.patch.disabled-conflicts-with-rk3308bs-0003`.
