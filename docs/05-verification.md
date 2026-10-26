# 5. Acceptance criteria and demonstrated result

[Guide index](../README.md) · [Previous](04-magisk-boot.md)

Use direct resulting-state evidence at each boundary. A process exit status or a healthy-looking service alone is insufficient.

## Memory access

- Exact fingerprint, kernel profile and Mali module identity match.
- Mali version 11.13 and GPU-property parsing agree across the qualified callers.
- The initial Image header is stable and valid before capturing its declared span.
- Packed-bank translation is checked through real task identities and list links.
- All requested records remain within known RAM ranges and do not cross holes.
- Discovery mappings are read-only and all exports are unmapped and closed.

## Temporary credential root

- Parent and child have independently verified task identities.
- Child credential and security allocations are private, with equal real/effective credential pointers.
- Three complete baseline snapshots agree, and usage is exactly 2.
- Descriptor passing retains the observer's file credentials.
- The child is the only writer of its own live allocation.
- Eight elevated ID fields are zero and the exact valid capability mask is observed both in memory and through syscalls.
- Original bounding set `0xc0` remains unchanged.
- All 48 selected scalar bytes are restored; the complete credential and security records equal the originals.
- The parent independently checks restoration before the child exits and is reaped.

## Privileged reads and backups

- PID 1's init SID is resolved and checked dynamically.
- Only the temporary child's private SID scalar changes.
- Block type, major/minor and size are verified before reading.
- The open file's credential reference follows 2 → 3 → 2.
- A header read is stable; each full backup agrees with an independent complete second read.
- Output begins only after identity restoration.
- Binary transport is non-PTY, exact-length, and separated from diagnostic status.
- Private backup checksums remain stable during analysis and image preparation.

## Signed-image construction

- Complete authorized public keys match throughout the actual trust chain.
- Stock and candidate signatures and hashes verify independently.
- Candidate differences are restricted to the intended kernel option/CRCs, new CPIO, necessary Android fields and rebuilt BOOT metadata.
- Official payloads, device trees, external metadata, salt, flags and rollback relationships retain their required identities.
- Every format extent, page alignment, CRC, image ID, hash, signature and loader bound is checked.
- Parser and signature validation reject deliberately altered covered data and an unrelated key.

## Flash and runtime

- All current BOOT bytes match the expected pre-write image.
- The complete candidate is written, synchronized and read back identically.
- Private helper identity and global enforcement are restored.
- Temporary executables and helper processes are absent afterward.
- Bounded logs contain no new matching kernel panic, Oops, BUG, GPU fault/reset or helper fatal signal.
- A new normal boot completes and `su -c id` actually returns UID 0 in the Magisk domain.
- SELinux reports Enforcing and verified boot reports locked/green.
- Repeat the normal boot/root check to establish persistence across boot; the demonstrated chain reached three consecutive successful boots.
- Verify the final pre-init block device/configuration and require the official Magisk environment check to return 0.
- Check the projector's physical controls and media functions separately from shell-level assertions.

## Verified outcome

| Capability | Demonstrated evidence |
|---|---|
| Read-only physical-memory access | Validated live kernel Image and linked structures |
| Reversible temporary root | Root IDs/capabilities, followed by exact restoration |
| Useful privileged reads | Verified BOOT/recovery reads through a private init identity |
| Complete stock backups | Full captures and independent byte-for-byte second reads |
| Existing AVB chain compatibility | Exact keys, trust callback, hashes and signatures |
| Signed Magisk BOOT installation | Complete partition write, fsync and full readback |
| Boot-persistent root | Magisk 30.7 UID 0 verified across three consecutive successful normal boots |
| Completed Magisk environment | Measured `PREINITDEVICE=userdata`, matching pre-init block device, official environment check exit 0 |
| Security state after successful boot | Enforcing, locked and green |

These are confirmed results on the listed Chinese-market M146FCN firmware. The international V10, other firmware, replacement operating systems, recovery entry and future updates remain outside this tested scope.
