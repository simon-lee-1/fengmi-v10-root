# Chinese-market Fengmi / Formovie V10: confirmed software root

A prescriptive engineering guide to the software-only rooting method demonstrated on the **Chinese-market Fengmi V10 M146FCN, FengOS 2.1.5.3936**.

**The confirmed procedure applies to the Chinese model. International V10 compatibility remains untested; see the [portability hypothesis](docs/01-device-and-preparation.md#international-v10-portability-hypothesis).**

The verified result is **Magisk 30.7 root across three consecutive successful normal boots**, with SELinux **Enforcing** and Android reporting verified boot **green** and the bootloader **locked**. The work used network ADB and retained the stock projector software and device trees.

To our knowledge, this is the **first publicly documented end-to-end software-root procedure for the Chinese-market V10 on this firmware**. This model-specific result builds on the upstream work credited below.

The working chain is:

1. Match the exact hardware, firmware and Mali driver.
2. Use the vendor Mali DMA-buffer exporter to read the running kernel.
3. Recover symbols and apply the kernel's MStar packed-RAM translation.
4. Give a temporary child private credentials and let it modify and restore its own credentials through observer-created mappings.
5. Use a dynamically identified init SELinux identity inside that child to obtain verified stock partition backups.
6. Validate the existing AVB trust chain and construct a correctly wrapped, signed Magisk BOOT image.
7. Temporarily clear the profiled kernel enforcement variable through the Mali mapping, write and verify the complete BOOT partition, restore Enforcing through normal SELinuxfs, and verify actual Magisk root after normal startup.

```mermaid
flowchart LR
    A[Network ADB shell] --> B[Vendor Mali memory export]
    B --> C[Exact kernel and RAM mapping]
    C --> D[Private self-restoring root helper]
    D --> E[Verified partition backups]
    E --> F[Compatible signed Magisk BOOT]
    F --> G[Verified Magisk root after reboot]
```

## Read the guide in order

| Guide | What it covers |
|---|---|
| [1. Device and preparation](docs/01-device-and-preparation.md) | Exact compatibility profile, ADB baseline, projector controls and useful vendor inspection |
| [2. Memory access and temporary root](docs/02-memory-and-root.md) | Export ABI, address translation, credential ownership, SCM_RIGHTS and restoration |
| [3. Backups and verified boot](docs/03-backups-and-avb.md) | Privileged reads, complete backups and the actual AVB chain |
| [4. Booting Magisk](docs/04-magisk-boot.md) | ARM32 payloads, legacy system-as-root, image construction, authenticated command line and verified installation |
| [5. Verification](docs/05-verification.md) | Acceptance criteria for each working stage and the demonstrated result |

This is the documented procedure and compatibility profile. The purpose-built Rust helpers, private firmware images and signing material are not distributed in this repository. Recreating the method requires implementing the described guards and deriving the exact profile from the target firmware. A different device using the name `nemo` is a separate target.

Boot-persistent root and the completed Magisk pre-init environment are verified: the final configuration selects `userdata`, its pre-init block device matches that partition, and the official environment check returns 0. Firmware replacement, recovery entry and future Magisk updates are separate qualifications.

## International V10 hypothesis

The same method may transfer to an international V10 if it retains the relevant Mali driver interface, kernel behavior and boot trust chain. Language, launcher and bundled-app changes alone would not remove the kernel mechanism. This is an **untested hypothesis**, with no verified international firmware comparison or successful international root report in the sources reviewed. Temporary root and persistent Magisk require separate checks; [the preparation chapter](docs/01-device-and-preparation.md#international-v10-portability-hypothesis) lists the evidence needed.

For the initial Chinese-model flash, `setenforce 0` did **not** succeed. The working fallback and subsequent normal-interface restoration are explained in [the SELinux section](docs/04-magisk-boot.md#handle-selinux-for-the-initial-installation).

## Credits and publication scope

This work depends on upstream research and tools. Start with [the credits and source map](CREDITS.md): GitHub Security Lab, Pro-me3us, the Philips/MStar research community, Marin Moulinier's kernel tooling, Android Open Source Project, and topjohnwu/Magisk all have distinct roles.

This publication contains a curated account of verified working steps. [Privacy scope](PRIVACY.md) explains the information intentionally kept out of the repository.
