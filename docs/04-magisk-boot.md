# 4. Construct and boot the compatible Magisk image

[Guide index](../README.md) · [Previous](03-backups-and-avb.md) · [Next: verification](05-verification.md)

## Use the correct official payloads

Use **topjohnwu/Magisk v30.7**, verified against the official release and pinned source. Select the **ARM32 / armeabi-v7a** payloads for the Android userspace, despite the 64-bit kernel. The preparation and signing method below is specific to the V10's boot container and loader.

Keep these effective build settings:

```text
KEEPVERITY=true
KEEPFORCEENCRYPT=true
PATCHVBMETAFLAG=false
RECOVERYMODE=false
LEGACYSAR=true
VENDORBOOT=false
```

Construct the new ramdisk as **raw newc CPIO** using the official missing-ramdisk layout and tools:

| CPIO path | Contents / mode |
|---|---|
| `init` | Official ARM32 `magiskinit`, 0750 |
| `overlay.d` | Directory, 0750 |
| `overlay.d/sbin` | Directory, 0750 |
| `overlay.d/sbin/magisk.xz` | XZ-compressed official Magisk payload, 0644 |
| `overlay.d/sbin/stub.xz` | XZ-compressed official stub APK, 0644 |
| `overlay.d/sbin/init-ld.xz` | XZ-compressed official init-ld payload, 0644 |
| `.backup` | Directory, 000 |
| `.backup/.magisk` | Config file, 000 |

Use the following bootstrap configuration to establish Magisk root. After the official runtime detector identifies the pre-init partition, add its measured value as described in the completion step below:

```text
KEEPVERITY=true
KEEPFORCEENCRYPT=true
RECOVERYMODE=false
VENDORBOOT=false
SHA1=<ORIGINAL_STOCK_BOOT_SHA1>
```

Compute the SHA1 from the complete original stock BOOT image. Use root ownership for CPIO entries and preserve the exact official file modes. Validate each compressed member by decompression and byte comparison with its authenticated upstream input.

## Coordinate the kernel and Magisk's legacy-SAR selection

The stock loader can supply a new BOOT ramdisk. The successful arrangement also lets Magisk recognize the legacy system-as-root path while allowing the kernel to unpack that ramdisk:

1. In the exact stock kernel payload, replace the single audited, NUL-terminated option string `skip_initramfs` with the equal-length `want_initramfs`.
2. Recalculate the uImage data CRC32 and header CRC32.
3. Put a flags-zero **kernel-command-line descriptor** containing `skip_initramfs` in the child BOOT AVB metadata.

The kernel's option handler no longer consumes that command-line token to skip the new initramfs. Magisk still sees the token and selects its appropriate legacy-SAR startup path.

The exact MBOOT collects authenticated child command-line descriptors into `bootargs_ext` and the FDT boot arguments. Preserve the stock Android-header command line: this loader's active behavior is determined by the authenticated descriptor path. Follow the exact source and disassembly rather than changing arbitrary boot arguments.

This coordinated behavior follows Magisk's official initialization and boot-patching logic. See the [pinned Magisk references](../CREDITS.md#magisk-and-android-verified-boot).

## Preserve the complete vendor container

Rebuild the Android v1 image while preserving its kernel load information, original header fields and stock device trees. Update only the necessary ramdisk size, aligned section offsets and Android image ID, plus the qualified kernel option/CRC changes.

Keep the appended DTB, recovery-DTBO content, second stage and device-tree partitions byte-identical. Add the raw CPIO and reconstruct the exact Android section padding. Then create the BOOT hash descriptor and the authenticated command-line descriptor, sign with the locally validated matching identity using `SHA256_RSA4096`, and restore the AVB footer and complete partition extent.

Preserve:

- Original BOOT salt and partition name.
- Child flags 0 and rollback index 0.
- Child header reserved/location field 0.
- The unchanged external BOOT chain at rollback-index location 1.
- External vbmeta and vbmeta_a bytes.

The working bootstrap and completed-preinit images satisfy these format and loader bounds:

| Check | Demonstrated value / bound |
|---|---|
| Full BOOT partition | 33,554,432 bytes |
| Raw CPIO | Bootstrap: 376,964 bytes; completed pre-init: **376,988 bytes** |
| Page-rounded initrd span | 378,880 bytes |
| Parsed Android image span | 16,480,256 bytes, within `0x1900000` |
| Initrd staging start | `0x24000000` |
| Initrd end | Bootstrap: `0x2405c084`; completed pre-init: **`0x2405c09c`**; both below `0x24fff800` |
| Kernel load/image range | `0x20280000` through `0x2140e000` |
| Resulting command-line length | Within the 2048-byte loader buffer |

Independently parse the candidate and compare every allowed and preserved field against stock. The bootstrap candidate passed 39 structural, payload, cryptographic and loader checks; the final pre-init candidate passed 56, preserving the earlier conditions and verifying its exact configuration addition.

## Perform a complete, verified BOOT write

Use the qualified temporary-root helper for the initial installation. Before the first partition write, check the exact fixed BOOT device identity and size and compare **all current BOOT bytes** with the expected stock image. The candidate must already have passed independent validation.

The initial write used a bounded temporary permissive interval. The qualified implementation profiled the exact kernel enforcing word and its surrounding bytes, changed only that word for the interval, and verified the externally observable mode. It subsequently restored Enforcing through the normal SELinuxfs interface, including the kernel's normal enforcement-change handling. Treat this as a distinct global change with an explicit restoration requirement.

Write only the complete validated BOOT image, handle partial I/O, call `fsync`, and compare the **entire** partition readback with the candidate. Keep the known original available for verified rollback on a returned I/O or comparison error. A process-level rollback routine cannot execute during power loss, so a software backup and a tested recovery route remain distinct protections.

The demonstrated initial write transferred and verified all 32 MiB. It preserved the external metadata, restored private credentials and security records, drained references, removed temporary helpers and restored global SELinux Enforcing before the normal boot test.

## Verify Magisk through actual normal boots

Install the official matching Magisk manager and start the projector normally. **On this Chinese firmware, `adb reboot` powers the unit off; use the physical power button to start it again.** Confirm a new boot, completed startup, the installed BOOT image identity, and actual superuser execution:

```sh
adb -s <PROJECTOR_IP>:5555 shell su -c id
adb -s <PROJECTOR_IP>:5555 shell su -c getenforce
adb -s <PROJECTOR_IP>:5555 shell su -c 'magisk -v'
```

The recorded successful output included:

```text
uid=0(root) gid=0(root) groups=0(root) context=u:r:magisk:s0
Enforcing
30.7:MAGISK:R
```

Root was verified across three consecutive successful normal boots spanning bootstrap, official support-file setup and final pre-init completion, alongside `boot_completed=1`, locked/green state and the expected BOOT content for each stage.

The official manager's files-only environment setup populated `/data/adb/magisk` with `busybox`, `magiskboot`, `magiskinit`, `magiskpolicy`, `util_functions.sh` and `boot_patch.sh`. That action schedules a reboot after completion; power the projector on again as above. Verify the installed files and repeat the actual root check.

## Complete and verify the pre-init environment

With Magisk root working and its official support files installed, use the official runtime detector to identify the appropriate pre-init partition. The measured result on this Chinese V10 was `userdata`. Run the detector without `MAGISKTMP` or `MAKEDEV` environment variables when only measuring its selection; those variables enable additional setup actions in the official code.

Rebuild the same proven BOOT image with exactly this one additional line in `.backup/.magisk`:

```text
PREINITDEVICE=userdata
```

Preserve the original config bytes, official payloads, entry order and modes, kernel/CRCs, device trees, command-line descriptor, signing identity, salt, rollback fields and external metadata. Recompute the config's CPIO size/padding and the necessary image ID, payload hash and signature. The config grows from 120 to 143 bytes and raw CPIO from 376,964 to 376,988 bytes; its page-rounded and Android-image spans stay unchanged.

From working Magisk root with SELinux Enforcing, require a complete byte comparison against the known-booting bootstrap BOOT before writing the final image. Write and synchronize the complete final partition, then verify all 32 MiB against the validated candidate. Retain the known-booting image for a verified rollback on a returned write/readback error. This update's only intentional payload change is the measured pre-init configuration.

Start the projector again and require all of the following:

- New normal boot and completed Android startup.
- Actual Magisk UID 0 and version 30.7.
- Full BOOT identity equal to the validated final image.
- `/sbin/.magisk/config` contains exactly `PREINITDEVICE=userdata`.
- `/sbin/.magisk/device/preinit` is an actual block device matching `userdata`: major 179, minor 30 on the tested layout.
- The official `env_check 30.7 30700`, using `/data/adb/magisk` and `/sbin` as its known paths, returns **exit 0**.
- SELinux Enforcing, verified boot locked/green, and the existing Home helper retained.

These final-runtime conditions were verified. The complete procedure produced three consecutive successful boots with working root. Future Magisk upgrades require renewed validation of this device's container, loader and signing constraints.
