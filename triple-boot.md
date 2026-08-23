# Hackintosh triple boot

[![rEFInd boot picker](assets/triple-boot-refind-picker.webp "rEFInd boot picker")](assets/triple-boot-refind-picker.webp)

Dual or triple booting a hackintosh is easy if you keep in mind a few things before starting the whole process. 
Here I'll briefly explain how I got the three systems running on a single drive on the **Dell 7567** laptop from the start, without the need to resize any partition after the installation.

# TL;DR

1. Installation order: macOS -> Windows -> Linux
2. When booting into macOS, create all your partitions with _Disk Utility_:
    - For macOS: use `APFS`, `GUID Partition Map` and name it _macOS_
    - For Windows: use `ExFAT` and name it _Windows_
    - For Linux: use `ExFAT` and name it _Linux_
3. Install macOS following the guide on [this repository](https://github.com/leandroprz/Dell-7567-Laptop-Hackintosh-OpenCore)
4. Install Windows. It will be formatted to `NTFS` automatically
5. Install your Linux distro of choice:
    - Install _GRUB_ on your distro
    - Use `ext4` or `Btrfs` for the OS and mount it on `/`
    - For the existing EFI partition created by macOS, keep `FAT32`, mount it on `/boot/efi` and use the flag `boot`. **Do not** format this one!
6. Boot using the macOS USB stick and copy the EFI folder from this repo into the EFI system partition
7. Boot into Linux and install [rEFInd](https://www.rodsbooks.com/refind/index.html), it will allow you to boot into macOS, Windows and Linux
8. Configure rEFInd to your needs from Linux or macOS by modifying the `refind.conf` file located in the EFI system partition

**If you want to learn more or need to do some troubleshooting, keep reading.**

# Overview

Three things are important for this to work: installation order of the operating systems, partition sizes before installing them and their respective file systems.

Below I'll explain the steps you need to follow and how to get all three systems to show up on a boot picker.

## Installation order

The installation order that will make this work is the following:

1. macOS
2. Windows
3. Linux

This laptop has space for two physical drives. I wanted to use one for the operating systems and the other for data. I'll explain how to install all the systems on a single drive.

## Partition sizes

We need to know how much space we'll asign to each system from the start, this is to avoid resizing partitions after the installation as that can cause booting issues.

Let's do some simple math. I have a 2TB NVme drive, so I decided each OS will get a third of the drive: `(2TB x 1024) ÷ 3 systems = ~680GB`. Each OS will have around **680GB**.

This is just an estimate, since:
- The exact formatted capacity depends slightly on the filesystem
- There's a difference between decimal terabytes (TB) used by manufacturers and binary tebibytes (TiB) used internally by operating systems
- We also need to account space for the `EFI/ESP` partition

## The EFI/ESP partition

The size of this partition will be determined by the boot manager we'll be using for our three systems. In our case it will be [rEFInd](https://www.rodsbooks.com/refind/index.html).

Some guides recommend the size of the EFI partition to be [1GB](https://wiki.archlinux.org/title/EFI_system_partition#Create_the_partition), others [2GB](https://wiki.cachyos.org/installation/installation_on_root/#size-at-least-2048-mib) or even more, because they are adding the size of the Linux kernel into it. But we'll configure the Linux installation to not use the EFI partition to store the kernel. Instead we'll use the Linux partition itself, where `/home` and `/` are located.

For our setup we can just use **200MB** (the macOS default) and it will work just fine. You can, however, follow the advice of the linked guides above if you prefer and adjust your partition sizes accordingly.

You can also add the size of this partition to the math we did before, but it's so small you won't notice the difference practically (if using 200MB).

# Getting started

## Partitioning the hard drive

We'll use the macOS installer to create the partitions and set their sizes from the start. Then we'll deal with the file systems for each OS.

1. Boot using your macOS USB stick. Once you get to the macOS installer, choose _Disk Utility_. On the top left click on _View > Show All Devices_. Select your internal SSD drive and click on _Erase_. Format it:
    - Name: _macOS_
    - Format: `APFS`
    - Scheme: `GUID Partition Map`

2. Select the SSD drive and click on _Partition_. You will see a new window where you can choose how many partitions you want.
    
    When clicking on the `+` button to add a new partition, you might get a message like _"Do you want to add a volume to the APFS container or do you want to divide the container’s storage into separate partitions?"_, if that's the case, click on _Add Partition_.
    
3. Create three partitions to get something like this:

    [![Partition the drive in macOS](assets/macos-disk-util-partition.webp "Partition the drive in macOS")](assets/macos-disk-util-partition.webp)

    Select the sizes you want for your partitions and set them up:
    1. macOS
        - Name: _macOS_
        - Format: `APFS`
    2. Windows
        - Name: _Windows_
        - Format: `ExFAT`
    3. Linux
        - Name: _Linux_
        - Format: `ExFAT`

    Click on _Apply_.
    
    An identifying name for each partition will be useful on the next steps. Don't mind the ExFAT partitions for now, they are serving as placeholders.

# Installation

## 1. macOS

1. Install macOS as described in the [main guide of this repository](https://github.com/leandroprz/Dell-7567-Laptop-Hackintosh-OpenCore) using the bootable macOS USB along with the custom EFI for this laptop
2. Once finished, shutdown the computer and plug the Windows USB installer

## 2. Windows

1. Boot using the Windows installer and when asked, select _Custom install_
2. You should see the partitions you created in macOS:

    [![Windows installer](assets/win-install.webp "Windows installer")](assets/win-install.webp)

3. Choose the one you previously named _Windows_, format it (the default file system is NTFS, this is automatic) and install Windows. The installer will probably overwrite the macOS EFI partition, but we'll fix that later
4. Once finished, shutdown the computer and plug the Linux USB installer

## 3. Linux

This step will vary depending on your distro. I chose [CachyOS](https://cachyos.org/) for this laptop. Since CachyOS uses _Calamares_ as the installer, I'll be showing that. But no matter the distro, you basically need to install _GRUB_ as the bootloader and partition your drive as shown in the next steps. The partitioning can also be done using [GParted](https://gparted.org/).

1. Boot using your Linux USB. The CachyOS installer will ask you to select a bootloader, choose _GRUB_. We won't be using it, but since CachyOS doesn't have an option to not install a bootloader, I'm selecting that. Plus, it is a good idea to have GRUB as a fallback in case _rEFInd_ fails
2. When asked how you want to partition the drive, select _Manual partitioning_. You'll see something like this:
    
    [![CachyOS installer](assets/cachyos-install.webp "CachyOS installer")](assets/cachyos-install.webp)
    
    Double click on the partition you previously named _Linux_. We need to change the partition as follow:
    
    - Content: Format
    - File system: `ext4` or `Btrfs`
    - Mount point: `/`
    - FS Label: _CachyOS_ for me
    
    If needed, you can resize it to make room for a **swap** partition. In that case, change the _Size_ at the top.

3. Now double click on the _EFI_ partition. Make sure you have the following:
    
    - Content: Keep
    - File System: `FAT32`
    - Mount Point: `/boot/efi`
    - FS Label: _EFI_
    - Flags: `boot`

4. Click on _Next_ to continue with the installation. You might get a warning about the EFI partition size, but you can ignore it. Select your desktop manager and finish the installation

# Boot manager configuration

Since Windows probably replaced the macOS configuration on the EFI partition, we'll fix that and also get the three operating systems working using a single boot manager to keep it as simple as possible.

## Fixing OpenCore

Boot using the macOS USB stick and from the OpenCore boot picker select the macOS installation that's already present in your internal SSD drive.

1. Once at the desktop run [MountEFI](https://github.com/corpnewt/MountEFI) and mount both the USB and internal SSD EFI partitions
2. Copy the EFI folder from the USB to the internal SSD EFI partition. Your EFI system partition should look like this:

```
EFI System Partition/
└── EFI/
    ├── BOOT/
    │   └── BOOTx64.efi
    ├── cachyos/
    │   └── grubx64.efi
    ├── Microsoft
    │   └── Boot/
    │   └── Recovery/
    └── OC/
        └── ACPI/
        └── Drivers/
        └── Kexts/
        └── Resources/
        └── Tools/
        └── config.plist
        └── OpenCore.efi
```

Your EFI partition now has almost everything to boot into each OS without using any USB stick.

## Installing and configuring rEFInd

Why [rEFInd](https://www.rodsbooks.com/refind/index.html) instead of OpenCore? Simply put, the OpenCore bootloader will apply some macOS specific unwanted patches to the other systems when booting them and we want to avoid that. You can read more about it [here](https://chriswayg.gitbook.io/opencore-visual-beginners-guide/advanced-topics/dual-boot-options).

Also, I found it easier to setup rEFInd as a boot manager and was able to get it exactly as I wanted (themes included).

You'll have to boot into your Linux installation (this is where GRUB comes handy), install rEFInd and finally copy and modify the configuration files from this repository to the folders `refind` and `tools` in your EFI system partition.

### Installing the package

The command to install rEFInd depends on your distro:

Arch based:
```
sudo pacman -S refind
sudo refind-install
```
Debian based:
```
sudo apt update
sudo apt install refind
```
Fedora based:
```
sudo dnf install refind
sudo refind-install
```
For other distros, refer to the [official rEFInd website](https://www.rodsbooks.com/refind/installing.html).

### Configuring rEFInd

check refind/drivers_x64/

add tools/shellx64.efi

choose default OS: refind/refind.conf

choose theme: refind/themes/minimal-black

```
EFI System Partition/
└── EFI/
    ├── BOOT/
    │   └── BOOTx64.efi
    ├── cachyos/
    │   └── grubx64.efi
    ├── Microsoft
    │   └── Boot/
    │   └── Recovery/
    ├── OC/
    │   └── ACPI/
    │   └── Drivers/
    │   └── Kexts/
    │   └── Resources/
    │   └── Tools/
    │   └── config.plist
    │   └── OpenCore.efi
    ├── refind/
    └── tools/
```

# Finishing touches

choose the startup disk in macOS from system config

Only keep macOS related disks in OpenCore boot picker: Security > ScanPolicy > 2687747 (number)

Misc > Boot > ShowPicker > False

# References and credits

- [Hackintosh & multiboot on my 2016 laptop - my tips](https://www.reddit.com/r/hackintosh/comments/1n8eqmh/hackintosh_multiboot_on_my_2016_laptop_my_tips/)
- [How to remove Windows and hide Tools and have a Clean OpenCore Boot Menu?](https://www.reddit.com/r/hackintosh/comments/mgt0ow/how_to_remove_windows_and_hide_tools_and_have_a/)
- [Dualbooting on the same disk](https://dortania.github.io/OpenCore-Multiboot/empty/samedisk.html)
- [Dual-Boot Options](https://chriswayg.gitbook.io/opencore-visual-beginners-guide/advanced-topics/dual-boot-options)

https://github.com/chriswayg/theme-minimal-black/

https://github.com/andersfischernielsen/rEFInd-minimal-black

https://www.youtube.com/watch?v=xZUH0UCL6AI