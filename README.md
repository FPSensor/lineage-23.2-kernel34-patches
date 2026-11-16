# LineageOS 23.2 (Android 16) on Linux 3.4 kernels

Patches to boot LineageOS 23.2 on devices still running a 3.4 kernel, with
no eBPF and none of the syscalls Android 16 now takes for granted. Brought up
on the Moto G 2013 (falcon, msm8226), 32-bit userspace.

These go **on top of**
[legacy_support_patches](https://github.com/Ultra-Legacy-Hippeastrum/legacy_support_patches)
(lineage-23.2), which they don't duplicate.

## Usage

```sh
# from the root of a lineage-23.2 tree
./legacy_support_patches/apply.sh
./lineage-23.2-kernel34-patches/apply.sh kernel/<vendor>/<device>
```

The kernel path is optional; without it only the platform patches are
applied. The non-A/B OTA patches also need
[LineageOS/android_system_tools_dtbtool](https://github.com/LineageOS/android_system_tools_dtbtool)
(`lineage-18.1`) at `system/tools/dtbtool` for devices with
`BOARD_KERNEL_SEPARATED_DT`.

## What they fix

### Missing syscalls and kernel features (platform)

| Repo | Problem on 3.4 | Fix |
|---|---|---|
| bionic | `arc4random` aborts: `MADV_WIPEONFORK` needs 4.14 (kills every process, even the recovery updater) | pid based fork detection, like OpenBSD's Linux port |
| bionic | `fstat()` on `O_PATH` fds needs 3.6: `realpath()` fails, first stage init can't mount `/system` | emulate it through `/proc/self/fd` |
| bionic | `rename()`/`renameat()` use `renameat2()` (3.15): every rename fails (derive_classpath → zygote, installd, vold) | fall back to `renameat()` |
| system/core | tombstoned uses `O_TMPFILE` (3.11) and aborts on every crash | named temporary file + rename |
| frameworks/native, system/hwservicemanager | servicemanagers abort without binder security contexts (`TXN_SECURITY_CTX`, 4.20) | fall back to `getpidcon()` |
| packages/modules/Connectivity | `netbpfload` fails with no `bpf()` syscall at all (3.18) and init reboots | mark the programs loaded and exit |

### Building and packaging

| Repo | Fix |
|---|---|
| build/make | Non-A/B OTAs for devices without a recovery partition (fstab and install helpers in the target files, `cache_size` without a cache image) |
| build/make | Pass the QCDT `dt.img` (`BOARD_KERNEL_SEPARATED_DT`) to the boot image, also when releasetools rebuilds it |
| build/soong | Keep dex compressed when dexpreopt is disabled globally (space) |

### Kernel (`kernel-3.4/`, generic 3.4 patches)

| Patch | Fix |
|---|---|
| kconfig, dtc, xz_wrap, kbuild | Build with current host tools |
| selinux | Android 15+ policies carry `nlmsg` extended permissions; the Android M compatibility code took them for an M policy and failed to load it |
| usb: gadget: f_fs | Android 16 adbd only uses AIO on FunctionFS (3.15); without it the device stays `offline` |

### Third party patches (authors kept)

Picked from [MisterZtr/LineageOS_gsi](https://github.com/MisterZtr/LineageOS_gsi)
`patches/trebledroid` (TrebleDroid), for BPF-less kernels and old encryption
setups:

- **Pierre-Hugues Husson (phhusson)**: BPF-less support in Connectivity,
  DnsResolver, system/bpf, netd; gpuservice off without BPF; FDE fallback,
  quota handling, fscrypt errors in system/core; `O_DIRECT` off in apexd.
- **Alberto Ponces**: encryptable fstab flag in fs_mgr; netd init abort revert.
- **Michael Bestas**: netd kernel version restrictions.
- **ChonDoit**: no-bpf use case in netd.
- **Luca Stefani**: libfscrypt directory bail out.
- **danielzhang130**: sepolicy without the bpf file context.

The FunctionFS AIO support in `kernel-3.4/` is taken from **acroreiser**'s
[lineage-23.2 hammerhead kernel](https://github.com/acroreiser/android_kernel_lge_hammerhead),
who also maintains legacy_support_patches.

Everything else: FPSensor.

## Device side notes

Not patches, but needed in the device tree on these kernels:

- `ro.hardware.egl=<name>` (libEGL no longer scans `/vendor/lib/egl`).
- `PRODUCT_COMPRESSED_APEX := false` if `androidboot.apex.early_adbd=true`
  (compressed APEXes can't be bootstrap ones).
- `AB_OTA_UPDATER := false` (Android 16 defaults it to true).
- No early mount fstab in the device tree if the ramdisk fstab differs.
- IRadio < 1.4 is rejected by telephony: wrap the RIL (see LineageOS'
  hammerhead radio 1.4 service).
