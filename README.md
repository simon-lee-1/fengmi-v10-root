# Fengmi / Formovie V10: verified software root

A prescriptive engineering guide to the software-only rooting method demonstrated on the **Fengmi V10 M146FCN, FengOS 2.1.5.3936**.

The verified result is **Magisk 30.7 root across two normal boots**, with SELinux **Enforcing** and Android reporting verified boot **green** and the bootloader **locked**. The work used network ADB and retained the stock projector software and device trees.

The working chain is:

1. Match the exact hardware, firmware and Mali driver.
2. Use the vendor Mali DMA-buffer exporter to read the running kernel.
3. Recover symbols and apply the kernel's MStar packed-RAM translation.
4. Give a temporary child private credentials and let it modify and restore its own credentials through observer-created mappings.
5. Use a dynamically identified init SELinux identity inside that child to obtain verified stock partition backups.
6. Validate the existing AVB trust chain and construct a correctly wrapped, signed Magisk BOOT image.
7. Write and verify the complete BOOT partition, restore enforcing mode, and verify actual Magisk root after normal startup.

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

The runtime claim here is boot-persistent Magisk root. Complete validation of Magisk's later pre-init environment configuration is outside this edition's demonstrated runtime result. Firmware replacement, recovery entry and future Magisk updates are separate qualifications.

## Credits and publication scope

This work depends on upstream research and tools. Start with [the credits and source map](CREDITS.md): GitHub Security Lab, Pro-me3us, the Philips/MStar research community, Marin Moulinier's kernel tooling, Android Open Source Project, and topjohnwu/Magisk all have distinct roles.

This publication contains a curated account of verified working steps. [Privacy scope](PRIVACY.md) explains the information intentionally kept out of the repository.
