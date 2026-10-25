# pixel8-custom-kernel

This overlay repo builds custom Google Pixel GKI kernels (Linux 6.1, Android 16) with KernelSU-Next for root access and configuration options required by Droidspaces (Linux containers on Android). The repo contains no kernel source; a GitHub Actions workflow synchronises the official AOSP Pixel kernel source, applies patches, adds KernelSU-Next, and appends the required config before building and uploading the final images.

## Supported devices

- `shusky` - Pixel 8 / Pixel 8 Pro (zuma SoC)
- `pantah` - Pixel 7 / Pixel 7 Pro (gs201 SoC)

## How it works / Repo layout

The repository consists of:

- `.github/workflows/build-kernel.yml` - the complete build pipeline (repo sync -> patch apply -> KernelSU-Next add -> config append -> Kleaf/Bazel build -> upload images).
- `configs/droidspaces.config` - Kconfig lines that are appended to the default `gki_defconfig` (namespaces, `CONFIG_SYSVIPC`, cgroup device, devtmpfs, seccomp, overlayfs, veth, bridge, user namespaces), plus Droidspaces' optional features (macvlan, IPv6 NAT/masquerade, CFS bandwidth, PID cgroup).
- `patches/` - kernel patches; only files ending in `.patch` are applied. Currently a SYSVIPC kABI fix that lets `CONFIG_SYSVIPC` be enabled without breaking the GKI kernel ABI.

## Building

The workflow runs only on GitHub Actions; no local build setup is required. It is triggered manually (`workflow_dispatch`) with the following inputs:

| Input             | Description                                                  | Default   |
|-------------------|--------------------------------------------------------------|-----------|
| `device`          | Device to build for: `shusky` or `pantah`.                   | `shusky`  |
| `kernel_source`   | `google` (AOSP Pixel kernel, stock Android 16) or `lineage` (LineageOS 24.0 kernel, for phones running LineageOS). For `lineage`, take stock images for rollback from the matching LineageOS build. | `google` |
| `manifest_branch` | Kernel manifest branch; must match the Android release on the phone. Blank = `android-gs-<device>-6.1-android16`. | Blank |
| `ksu_ref`         | KernelSU-Next version; blank = latest release, or `dev` / `stable` / `vX.Y.Z`. | Blank |
| `strict_kmi`      | Boolean; default `true`. Untick if the build fails on KMI/ABI symbol-list checks. | `true`   |

Builds can take several hours (job timeout 6 h). The workflow artifact is named `<device>-kernel-<run number>` and contains `boot.img`, `dtbo.img`, `vendor_kernel_boot.img`, `vendor_dlkm.img`, and `system_dlkm.img`.

Trigger the workflow from the GitHub web UI (Actions tab -> "Build Pixel kernel (KernelSU-Next + Droidspaces)" -> Run workflow), or via the CLI:

```sh
# Trigger the workflow:
gh workflow run build-kernel.yml -f device=pantah

# Monitor the run:
gh run list
gh run watch

# After completion, download the artifact:
gh run download <run-id>
```

## Installing

Requirements/warnings:

- Unlocked bootloader.
- Phone must be running the Android release matching the manifest branch (Android 16 by default).
- `platform-tools` (`adb`/`fastboot`) installed.
- Back up data and keep the stock images from the matching Google factory image so you can roll back.
- Flash all five images from the same build together.
- Disabling verified boot via `vbmeta` (required because `vendor_dlkm`/`system_dlkm` are modified) requires a factory reset (data wipe) the first time.

Steps:

```sh
# Reboot to bootloader:
adb reboot bootloader

# First time only, disable verified boot using vbmeta.img from the matching stock factory image (wipes data):
fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img
fastboot -w

# Flash bootloader-mode partitions:
fastboot flash boot boot.img
fastboot flash dtbo dtbo.img
fastboot flash vendor_kernel_boot vendor_kernel_boot.img

# Switch to fastbootd for the dynamic partitions:
fastboot reboot fastboot
fastboot flash vendor_dlkm vendor_dlkm.img
fastboot flash system_dlkm system_dlkm.img

# Reboot:
fastboot reboot
```

Install the KernelSU-Next manager app to manage root, and the Droidspaces app to run containers.

## Rolling back

To rollback, flash the same five partitions from the stock factory image the same way:

```sh
adb reboot bootloader
fastboot flash boot boot.img      # stock
fastboot flash dtbo dtbo.img      # stock
fastboot flash vendor_kernel_boot vendor_kernel_boot.img  # stock
fastboot reboot fastboot
fastboot flash vendor_dlkm vendor_dlkm.img  # stock
fastboot flash system_dlkm system_dlkm.img  # stock
fastboot reboot
```

Note that the stock images must come from the factory image matching the installed Android build.
