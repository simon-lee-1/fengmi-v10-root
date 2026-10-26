# 3. Read stock partitions and establish the AVB trust chain

[Guide index](../README.md) · [Previous](02-memory-and-root.md) · [Next: Magisk BOOT](04-magisk-boot.md)

## Add only the private identity needed for reads

Use the validated PID 1 task to obtain the active **init SELinux SID**. Check its name, group leader, credential pointers and security record twice, and establish that PID 1 corresponds to `u:r:init:s0`. Resolve the numeric SID at runtime.

The inspected policy grants init the relevant block-file access. In the temporary child, change only its own private security object's active-SID word at `+4`, alongside the already qualified UID/capability scalars. Preserve all other security fields. The parent continues to observe without writing the child.

For each target partition:

1. Allocate buffers before elevation.
2. Enter the validated private root/init identity.
3. Open the fixed block device with `O_RDONLY`.
4. Check native `fstat`: block-device type and the expected major/minor.
5. Query size with read-only `BLKGETSIZE64`.
6. Read 4096 bytes and independently read the first 64 bytes again; require equal headers.
7. Close the file and restore the SID and credential scalars before any output.
8. Allow deferred file cleanup to complete, then require the full original objects again.

A file opened by the child adds one real credential reference. Account for the exact transition **2 → 3 → 2**. Use a bounded wait for the final RCU release rather than writing or adjusting the usage counter. The demonstrated implementation bounded the drain at two seconds and checked the entire original objects afterward.

## Capture complete, independently compared backups

After the header read is qualified, allocate enough private userspace memory for a complete partition. Read the entire image and compare it byte-for-byte with an independent second read in bounded chunks. Restore the helper's original identity before streaming binary data over non-PTY ADB; keep binary output separate from status output.

The following complete backups were obtained and independently compared:

| Partition | Observed node | Exact size |
|---|---|---:|
| BOOT | `/dev/block/mmcblk0p11` | 32 MiB |
| recovery | `/dev/block/mmcblk0p10` | 32 MiB |
| MBOOT | `/dev/block/mmcblk0p1` | 5 MiB |
| MBOOTBAK | `/dev/block/mmcblk0p3` | 5 MiB |
| vbmeta | `/dev/block/mmcblk0p4` | 64 KiB |
| vbmeta_a | `/dev/block/mmcblk0p22` | 1 MiB |
| dtb | `/dev/block/mmcblk0p15` | 1 MiB |
| dtbo | `/dev/block/mmcblk0p16` | 8 MiB |

Verify each `by-name` mapping and device identity on the target before using these node names. BOOT and recovery have block major 179, minors 11 and 10 respectively.

Keep backups in mode-600 files beneath mode-700 directories, created with `umask 077`. Calculate and retain local checksums. A captured partition is a backup artifact; a restore procedure requires its own verification.

## Parse the actual boot format

The tested stock layout has:

- Android boot-image header **version 1** and 2048-byte pages.
- A legacy **uImage** wrapper around the kernel payload, with header and data CRC32 values.
- Appended device-tree material that must remain intact.
- A zero-length BOOT ramdisk and a populated recovery ramdisk.
- Legacy system-as-root with the Android root supplied by the existing verified system path.

Use the full stock images and MBOOT to establish loader behavior. Preserve the original load addresses and inspect the exact section boundaries, alignment, footer and metadata extents before constructing another image.

## Validate the complete trust chain

The decisive signing observation was a **byte-for-byte public-key identity match**, not a display property:

1. Extract the complete serialized AVB RSA-4096 public key from BOOT and external vbmeta.
2. Parse external chain descriptors and compare their complete authorized keys.
3. Locate the actual trusted-key storage and the active trust callback in **both MBOOT copies**.
4. Compare all 1032 serialized public-key bytes with the AOSP AVB RSA-4096 test identity.
5. Independently verify the stock RSA signatures, authenticated header/auxiliary data and salted partition hashes.

These comparisons matched on the tested firmware. The associated public AOSP test signing identity could therefore sign a replacement BOOT while retaining the existing chain authorization. This repository contains no private signing-key material; derive and validate signing inputs locally from their upstream provenance.

The verified chain is:

| Metadata relationship | Observed behavior |
|---|---|
| External vbmeta → BOOT | Delegates the BOOT signing key at rollback-index location 1 |
| External vbmeta → vendor | Delegates its key at rollback-index location 3 |
| External vbmeta → recovery / dtbo | Authenticates partition hashes |
| External vbmeta → system | Authenticates the verity mapping named `vroot` |
| vbmeta_a | Entire captured partition is zero-filled |

Trace the exact MBOOT AVB load path as well as parsing metadata. Preserve the distinction between a chain descriptor's rollback-index location and the child image header's own reserved field.

For the tested BOOT rebuild, keep the same authorized key, salt, hash algorithm, partition name, rollback index and zero flags. Recompute its payload hash and signature correctly. The external vbmeta images can remain byte-identical.

**Attribution:** Android's `avbtool` and `libavb` are the authoritative format and signing references. The public MStar source and **dipcore/mstar-bin-tool** supplied valuable bootloader and signing research context; the authorization conclusion above came from comparisons against the exact captured V10 images. See [the source map](../CREDITS.md).
