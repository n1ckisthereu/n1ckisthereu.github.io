---
title: Single GPU Passthrough
published: 2025-03-05
description: "How can pass your GPU for a virtual machine and get a bare metal
  performance"
image: "./cover.jpg"
tags: ["linux", "vm", "passthrough", "virtualization"]
category: "virtualization"
draft: false
---

:::warning
This content is intended for experienced linux users and **NOT** advisable for
new linux users.
:::

## Introduction

In this post, let's set up GPU passthrough with a single GPU. First, I would
like to make it clear that my intention is not to copy, but to share what
worked for me. Some text has been copied to avoid redoing work from the
original posts. I will share the content I used to create this post in the
[references](#references) section and throughout the post.

### Attention

- Your configuration and the one in this post can, and will, be different. Do
  not take my configuration as your truth.

- Your GPU must support UEFI, and so must your system. Be sure to have
  installed your distro with UEFI; otherwise, you may need to apply additional
  workarounds that are not explained in this guide

- If you want to run macOS, understand that it can be complex. If you can
  passthrough your GPU, the process doesn’t differ much from a typical
  Hackintosh. Check the following lists for compatible GPUs if you don’t want
  to deal with complicated workarounds, for [NVIDIA](https://elitemacx86.com/threads/nvidia-gpu-compatibility-list-for-macos.614/),
  [AMD](https://elitemacx86.com/threads/amd-gpu-compatibility-list-for-macos.617/)

### Why use single gpu

## Preparation

- Check if your motherboard BIOS version is up to date. (This can help you
  have better IOMMU groups for the next steps)
- Check if your system is installed in UEFI mode, do not use CSM, as it will
  install your system as Legacy, which doesn't work with GPU passthrough.
  (Force UEFI in your BIOS when installing your distro)

### Bios configuration

Depending on your machine, you may need to enable certain options in your BIOS
for the passthrough to succeed. Enable the settings listed in the table below.

| AMD CPU  | Intel CPU |
| -------- | --------- |
| IOMMU    | VT-D      |
| NX mode  | VT-X      |
| SVM mode |           |

**If you use intel:** You may not have both options, in that case, just enable
the one available to you.

**If you not have any virtualization settings** like said make sure your BIOS
is up to date, and that your CPU and motherboard support virtualization.

## Configure your Bootloader

### Enable IOMMU

Many distros use GRUB as the default bootloader, but these instructions also
work for systemd-boot.

Set the appropriate parameter for your boot system.

| AMD CPU                                         | Intel CPU      |
| ----------------------------------------------- | -------------- |
| Kernel auto-detects if IOMMU is enabled in BIOS | intel_iommu=on |

For more informations consult: [Arch Wiki Enabling IOMMU](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Enabling_IOMMU)

For AMD users, you can use the following parameter: video=efifb:off. This can
help prevent issues when returning from the guest to the host. It is
recommended that you add it.

Enable **IOMMU** for [systemd-boot](#for-systemd-boot), [grub](#for-grub)

:::important
after edit your bootloader reboot your system, for verify if the parameters has
been loaded, run

```sh
cat /proc/cmdline
```

:::

### For systemd-boot

We need to edit the bootloader. For systemd-boot, it is usually found at
`/boot/efi/loader/entries/something.conf`, but this may vary. In my case, for
example, it is located at `/boot/loader/entries/something.conf`.

My setup has two entries, one for _initramfs_. I will edit the one I use to
boot: `/boot/loader/entries/2024-12-04_17-24-11_linux.conf`.

Now, let's edit the options parameter. In systemd-boot, you should have
something like this:

```conf
title   Arch
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options cryptdevice=PARTUUID=4ca7758c-d6eb-41ca-807b-10d6b28ebfb3:root root=/dev/mapper/root zswap.enabled=0 rootflags=subvol=@ rw rootfstype=btrfs splash quiet
```

In options, add `video=efifb:off` if you have an AMD GPU, and
`intel_iommu=on` if you have an Intel CPU.

In my case, using AMD, it looks like this:

```conf
options cryptdevice=PARTUUID=4ca7758c-d6eb-41ca-807b-10d6b28ebfb3:root root=/dev/mapper/root zswap.enabled=0 rootflags=subvol=@ rw rootfstype=btrfs splash quiet video=efifb:off
```

### For grub

For GRUB, let's edit the `GRUB_CMDLINE_LINUX_DEFAULT` parameter in the
`/etc/default/grub` file. This example shows intel_iommu=on, but if you need
to add `video=efifb:off`, the process is the same.

Before edit:

```conf
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="Arch"
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
GRUB_CMDLINE_LINUX="cryptdevice=UUID=b5f6fc67-c239-40b9-8263-beb2d40682a5:root zswap.enabled=0 rootfstype=btrfs"

```

After edit:

```conf
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="Arch"
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet intel_iommu=on"
GRUB_CMDLINE_LINUX="cryptdevice=UUID=b5f6fc67-c239-40b9-8263-beb2d40682a5:root zswap.enabled=0 rootfstype=btrfs"
```

Now run `sudo grub-mkconfig -o /boot/grub/grub.cfg`

## IOMMU Groups

Neste passo vamos olhar os grupos IOMMU para coletar informações sobre sua GPU.
seus Dispositivos PCI, são divididos em grupos chamados IOMMU Groups. Sua GPU
está em um ou multiplos desses grupos, e você precisa

Use o seguinte script para listar todos os grupos (copie-cole isso na
sua linha de comando)

```bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

## Bypass anti-cheat

### Vanguard

## Troubleshoot

### Internet does not work (WIFI)

### Black screen

### Group "X" is not viable

## References

I used a mix from these repositories

- Single GPU passthrough by [risingprismtv](https://gitlab.com/risingprismtv/single-gpu-passthrough)
- Single GPU passthrough by [ilayna](https://github.com/ilayna/Single-GPU-passthrough-amd-nvidia)
