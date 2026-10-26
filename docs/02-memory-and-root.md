# 2. Obtain memory access and prove temporary root

[Guide index](../README.md) · [Previous](01-device-and-preparation.md) · [Next: backups and AVB](03-backups-and-avb.md)

## Qualify the exact Mali interface

Open `/dev/mali0` and negotiate the interface before asking it to export memory:

| Operation | Request | Qualified arguments / result |
|---|---:|---|
| Version check | `0xc0048000` | Two `u16` values: major 11, minor 13 |
| Context flags | `0x40048001` | One `u32`, zero |
| GPU properties | `0x40108003` | Query required size, then supply a bounded buffer |

The two caller architectures returned the same 711-byte property set. Useful profile values are product ID `0x7402`, raw GPU ID `0x74021000`, and MMU features `0x2823`.

The working memory primitive is the vendor **`mem_block_export_dma_buf`** path. It exports a caller-selected, page-aligned bus/physical range as a DMA-buffer file. The exact inspected path checks alignment; its accepted shell call and read-only mappings were demonstrated directly.

The request is `0xc0188027`, with this 24-byte native layout:

```rust
#[repr(C)]
struct Export {
    address: u64,
    length: u32,
    skip_translation: u8,
    padding: [u8; 3],
    fd: i32,
    reserved: u32,
}
```

Set both padding fields to zero and the initial output descriptor to `-1`. Use flag **1** to supply an already established bus address; the flag's verified meaning is to bypass the exporter's own MIU conversion. Address and length must be multiples of 4096.

Map the returned descriptor with `PROT_READ | MAP_SHARED` for discovery. Its vendor mmap callback passes the stored PFN and VMA protection to `remap_pfn_range`. The release callback frees the export metadata. No GPU job, JIT allocation or race is needed by this path.

**Attribution:** GitHub Security Lab's Mali research and Pro-me3us's Raven implementation supplied the vulnerability and architectural background. The V10 route documented here uses this vendor export interface; the successful chain did not run Raven's CVE-2022-38181 race. Credit those works for their actual contribution rather than describing this as an unmodified Raven port.

## Capture and identify the running kernel

Start with a single read-only 4 KiB page at physical `0x20280000`. Require two equal copies of the first 64 bytes and validate the AArch64 Image header:

- Magic at byte 56: `0x41 0x52 0x4d 0x64` (the AArch64 Image magic).
- Text offset: `0x80000`.
- Image size: `18,407,424` bytes (`0x118e000`).

Once that page validates, capture exactly the image-declared span. Compare its header to the qualified page and verify the exact kernel build banner. Keep this RAM capture private: it includes mutable runtime data and is an analysis input, not a stock flash image.

Use **Marin Moulinier's [vmlinux-to-elf](https://github.com/marin-m/vmlinux-to-elf)**, version 1.4.0 in the recorded work, to recover kallsyms and an analysis ELF. The captured kernel yielded 39,311 symbols. Cross-check the critical names against the kernel's actual exported-symbol records and disassembly before using their addresses.

## Apply the MStar packed-RAM mapping

This kernel's `mstar_virt_to_phy` implements a packed linear map. Subtract `PAGE_OFFSET = 0xffffffc000000000`, select the correct interval, then translate its offset into that RAM bank:

| Linear offset interval, end exclusive | Physical/bus start | Length |
|---|---:|---:|
| `[0x00000000, 0x22100000)` | `0x20200000` | `0x22100000` |
| `[0x22200000, 0x69000000)` | `0x56c00000` | `0x46e00000` |
| `[0x69000000, 0xe9000000)` | `0x180000000` | `0x80000000` |

The bank boundary includes 2 MiB rounding. Reject holes and any record whose last byte would cross a bank boundary. Read the corresponding boot globals from the exact live kernel and compare them to the profile before using this table.

Kernel-image addresses use their own translation:

```text
image virtual base = 0xffffff8008080000
image physical base = 0x20280000
kimage_voffset = 0xffffff7fe7e00000
```

Validate translation through actual linked structures: `init_task`, PID 1 with comm `init`, list backlinks and group-leader identity. Match the helper by both its syscall-reported PID and complete comm value.

## Use the audited structure layout

These offsets belong to this exact kernel profile:

| Structure / field | Offset |
|---|---:|
| `task_struct.tasks` | `0x320` |
| PID | `0x420` |
| Group leader | `0x460` |
| `real_cred` / `cred` | `0x5d8` / `0x5e0` |
| Comm | `0x5e8` |
| Credential object size | `0xa8` bytes |
| Credential usage | `0x00`, `u32` |
| Eight UID/GID fields | `[0x04, 0x24)` |
| Inheritable / permitted / effective capabilities | `0x28` / `0x30` / `0x38` |
| Bounding / ambient capabilities | `0x40` / `0x48` |
| Security pointer | `0x78` |
| SELinux credential-security object | 24 bytes; active SID is `u32` at `+4` |

Derive the credential offsets from `commit_creds` and the allocation/copy paths. Derive the security layout from the SELinux hooks. The audited `selinux_cred_prepare` duplicates the 24-byte security object, giving the new credential a separate security allocation.

## Separate observation from the credential being measured

Each memory-export file holds an **`f_cred` reference to the process that created it**. That reference is released through `file_free_rcu`, so closing an export descriptor does not immediately remove its effect on credential usage.

Use a parent observer and a temporary child:

1. Create synchronization pipes and a Unix socketpair before forking.
2. Let the child identify itself, then wait on an existing pipe.
3. Validate the parent and child task records and their separate credential/security objects.
4. Ask the child to call `setresuid(-1, -1, -1)`. In this exact kernel, the call prepares and commits a fresh credential without changing IDs.
5. Have the child open no new files while the parent observes it.
6. Require `real_cred == cred`, distinct new credential and security pointers, usage **exactly 2**, eight IDs equal to 2000, and zero initial inheritable/permitted/effective capabilities.
7. Require three identical full credential/security snapshots, separated by 50 ms, with unchanged task pointers.

All observation exports now reference the parent's credentials. The child's two task references remain directly countable.

## Make the child the sole writer

After validation, the parent exports only the child's identified task, credential and security pages and passes those same file descriptors using **`SCM_RIGHTS`**. The child maps the existing file objects, whose `f_cred` references still belong to the parent.

```mermaid
sequenceDiagram
    participant P as Observer parent
    participant C as Temporary child
    P->>C: Existing pipes and socketpair
    C->>C: Clone private credentials; wait
    P->>P: Validate child identity, ownership and snapshots
    P->>C: Exported task/cred/security descriptors via SCM_RIGHTS
    C->>C: Independently validate imported mappings
    C->>C: Change own scalar fields; observe root; restore
    C->>P: Report after restoration
    P->>P: Independently verify original bytes
    P->>C: Exit and reap
```

This ownership matters: a physical mapping does not pin a credential allocation. Keeping the writer in the target process ensures that target death also stops all subsequent writes. Use parent-death handling, pipe EOF, bounded execution and explicit child reaping.

## Prove root and restore exactly

The tested original bounding set is **`0xc0`**. Preserve it exactly. Linux's bounding set governs later capability acquisition across exec; current permitted/effective sets can exceed it. The bounded proof performs no exec, fork or credential-changing syscall while elevated.

`cap_last_cap` is 37, giving a valid capability mask of **`0x3fffffffff`**. Modify only these 48 scalar bytes in the child's proven-private credential:

- All eight UID/GID fields become zero.
- Permitted and effective capabilities become the full valid mask.

Leave reference counts, securebits, inheritable/bounding/ambient capabilities, pointers and the security object unchanged for this first proof. Use aligned volatile stores, compiler barriers and AArch64 `dmb ish`. Set permitted before effective; restore effective before permitted.

Observe root using `getuid`, `geteuid`, `getgid`, `getegid`, `getresuid`, `getresgid` and `capget`. Also check all eight ID fields directly, including fsuid/fsgid. Then restore the original scalar values unconditionally before reporting.

Require the complete 168-byte credential and 24-byte security records to match the originals, usage to remain 2, syscalls to report UID/GID 2000 and zero original capabilities, and an independent parent read to confirm restoration. The demonstrated temporary-root proof passed these checks.
