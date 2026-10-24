# pixel8-custom-kernel

Overlay repo that builds Pixel GKI kernels with KernelSU-Next and the config Droidspaces needs:
Pixel 8 (`shusky`, zuma SoC) and Pixel 7 (`pantah`, gs201 SoC). There is no kernel source here:
CI syncs AOSP, then applies this repo on top.

Both devices' `android16` 6.1 branches build from the same `kernel/common` commit, so `configs/`
and `patches/` are shared. Only the values set in the workflow's "Resolve device settings" step
differ: Kleaf `--config`, dist target, and dist dir.

## Layout
- `.github/workflows/build-kernel.yml` is the whole build pipeline (repo sync → patches → KernelSU-Next → defconfig → Kleaf → upload images).
- `configs/droidspaces.config` holds Kconfig lines appended verbatim to `kernel/aosp/arch/arm64/configs/gki_defconfig`.
- `patches/` holds patches applied with `patch -p1 --forward` inside `kernel/aosp`.

## Building
- Builds run only on GitHub Actions; there is no local build setup. Trigger with
  `gh workflow run build-kernel.yml -f device=<shusky|pantah> [-f manifest_branch=...] [-f ksu_ref=...] [-f strict_kmi=false]`,
  and check runs with `gh run list` / `gh run view --log-failed`.
- `manifest_branch` must match the Android release on the phone. If it's blank, it defaults to
  `android-gs-<device>-6.1-android16`. Pre-Android-16 pantah branches use 5.10, which the 6.1 patches don't fit.
- Outputs are uploaded as artifacts: boot, dtbo, vendor_kernel_boot, vendor_dlkm, system_dlkm images.

## Gotchas
- Only `patches/*.patch` is applied. The three `fix_sysvipc_kabi_*` files are alternative versions
  of the same SYSVIPC kABI fix, each using different `ANDROID_KABI_RESERVE` slots in `task_struct`.
  Exactly one should carry the `.patch` extension (currently `6_7_8`). Applying two will conflict.
- Enabling `CONFIG_SYSVIPC` changes the size of `task_struct` and breaks the GKI kABI unless one of those patches is applied.
- `check_defconfig` is removed from `build.config.gki` because appended options are never
  savedefconfig-canonical. Don't add it back.
- New Kconfig options can fail KMI symbol-list checks. Run with `strict_kmi=false` before changing anything else.
- The "Maximize build space" step has to run before checkout, because it remounts the workspace.
