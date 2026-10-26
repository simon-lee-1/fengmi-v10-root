# Credits and source map

[Guide index](README.md)

The V10-specific work combines existing research, public tools and direct analysis of the tested firmware. This source map identifies each upstream contribution; it does not imply that any upstream author tested or endorsed this V10 procedure.

## Mali research and architectural precedent

- **Man Yue Mo and GitHub Security Lab** — [GHSL-2022-054 Arm Mali advisory](https://securitylab.github.com/advisories/GHSL-2022-054_Arm_Mali/) and [the technical write-up](https://github.blog/2023-01-23-pwning-the-all-google-phone-with-a-non-google-bug/). These establish CVE-2022-38181 and its mitigation.
- **Pro-me3us** — [CVE_2022_38181_Raven](https://github.com/Pro-me3us/CVE_2022_38181_Raven). Its Android 9, ARM32 userspace, Bifrost r16p0 and Linux 4.9 work supplied a close architectural reference.
- **Arm / Android GPU-source contributors** — the [pre-mitigation source snapshot](https://android.googlesource.com/kernel/google-modules/gpu/+/5eb4e28d03ffb5e035bf7d39f24d2d0e33daa07f/mali_kbase/mali_kbase_mem_linux.c) and [mitigated snapshot](https://android.googlesource.com/kernel/google-modules/gpu/+/0a0cf9d342933afcea17c73845c3f5605789e5e7/mali_kbase/mali_kbase_mem_linux.c#954) used for exact driver comparison.

The demonstrated V10 chain uses the vendor `mem_block_export_dma_buf` interface, custom packed-RAM translation, private credentials and observer-created descriptors. The earlier Mali publications are credited as prior research; this publication does not claim their CVE discovery or label the V10 vendor interface with that CVE.

## Kernel reconstruction and implementation

- **Marin Moulinier (`marin-m`)** — [vmlinux-to-elf](https://github.com/marin-m/vmlinux-to-elf), version 1.4.0 used here for kallsyms recovery and creation of an analysis ELF.
- **Linux kernel contributors** — [Linux 4.9 source](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/?h=v4.9.118), including credentials, file lifetime, RCU, descriptor passing and SELinux. The exact vendor Image supplied the final V10 offsets and behavior.
- **Rust and LLVM contributors** — [Rust](https://www.rust-lang.org/) and [LLD](https://lld.llvm.org/) for the standalone ARM/AArch64 `no_std` helpers. The recorded compiler version was Rust 1.90.0.

## Vendor-interface and bootloader research

- **yath** — [tpm171e](https://github.com/yath/tpm171e), especially its [pinned README](https://github.com/yath/tpm171e/blob/3c777a08ae0c85d780acaa8fad67cbc1b2bcbc63/README.md), for the Philips/MediaTek kernel-CLI research.
- **Said Sürücü** — [philips-tpm171e-root](https://github.com/saidsurucu/philips-tpm171e-root), an additional documented reproduction and protocol reference.
- **Published MediaTek DirectFB source** — [the mtcli implementation](https://github.com/unitedcolorsofg/vizio_oss/blob/74270002d874391148119b48882db6816e7deedc/directfb/MediaTek/DirectFB/inputdrivers/mtcli/mtcli.c#L196-L203) corroborates the command-interface shape.
- **MStar, U-Boot and vendor-source contributors; davift's source archive** — [the pinned MStar bootloader tree](https://github.com/davift/FireOS_FireTV_and_FireStick/tree/224f8510bd734acafb1a5bc9d4e2de5dfd0686f0/platform/bootable/bootloader/uboot-mstar/t31/u-boot-2011.06) supplied comparison material for boot recipes, image checking and optional bootloader facilities. Exact V10 MBOOT captures were used for final device-specific conclusions.
- **dipcore** — [mstar-bin-tool](https://github.com/dipcore/mstar-bin-tool), for MStar firmware and signing research context. The tested V10 key identity was established through AVB-format comparison against AOSP and the actual trusted-key storage.

## Magisk and Android Verified Boot

- **topjohnwu and Magisk contributors** — [Magisk v30.7](https://github.com/topjohnwu/Magisk/tree/v30.7), the official runtime, manager, `magiskinit`, `magiskboot`, `magiskpolicy`, scripts and documentation. The persistent-root payload and systemless-root implementation are their work.
- Relevant pinned Magisk sources: [installation documentation](https://github.com/topjohnwu/Magisk/blob/v30.7/docs/install.md), [boot_patch.sh](https://github.com/topjohnwu/Magisk/blob/v30.7/scripts/boot_patch.sh), [architecture selection](https://github.com/topjohnwu/Magisk/blob/v30.7/scripts/util_functions.sh#L502), [init argument parsing](https://github.com/topjohnwu/Magisk/blob/v30.7/native/src/init/getinfo.cpp), [initialization](https://github.com/topjohnwu/Magisk/blob/v30.7/native/src/init/init.rs), and [boot-image handling](https://github.com/topjohnwu/Magisk/blob/v30.7/native/src/boot/bootimg.cpp).
- **Android Open Source Project contributors** — [external/avb](https://android.googlesource.com/platform/external/avb/), `avbtool`, the AVB test fixtures and [libavb verification](https://android.googlesource.com/platform/external/avb/+/refs/heads/android11-release/libavb/avb_slot_verify.c), for the signing formats, trusted-key serialization and verification semantics.
- **Python cryptography contributors** — [cryptography](https://cryptography.io/) for an independent signature-verification implementation alongside explicit RSA/DER validation.

## Projector preparation

- **CleanYourTV contributors** — [cleanyourtv-app](https://github.com/udhaykumarbala/cleanyourtv-app) and [symi-daguo's FengOS adaptation](https://github.com/symi-daguo/cleanyourtv/commit/2a80be2cf5a5ddb14a9cbf8ded423c02967edbeb) for the clean-launcher approach and explicit stock-Home redirection.
- **spocky and the Projectivy/projector community** — [Projectivy documentation](https://xdaforums.com/t/app-android-tv-projectivy-launcher.4436549/) and [projector research](https://xdaforums.com/t/guide-recovery-rom-xiaomi-fengmi-wemax-mi-led-laser-projector-tv.3851755/) for device-specific context and preserving remote behavior.

Upstream projects retain their own licenses. This repository links to their primary sources and describes how they informed the verified V10 procedure; it does not redistribute their firmware, APKs, binaries or private signing material.
