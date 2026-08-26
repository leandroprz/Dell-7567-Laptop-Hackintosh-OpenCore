# Dell 7567 laptop - Hackintosh triple boot

[![rEFInd boot picker](assets/triple-boot-refind-picker.webp "rEFInd boot picker")](assets/triple-boot-refind-picker.webp)

[![macOS](https://img.shields.io/badge/macOS-Sequoia%2015.7.9-d3a94e?style=for-the-badge)](#) [![Windows](https://img.shields.io/badge/Windows%2011-25H2-61a0d3?style=for-the-badge)](#) [![Linux](https://img.shields.io/badge/Linux-CachyOS-2d8c8c?style=for-the-badge)](#)

Dual or triple booting a hackintosh is not a difficult task when you get it right from the start. I'll briefly explain how I got the three systems running on a single drive on the **Dell 7567** laptop from the start, without the need to resize any partition after the installation.

Keep in mind this is a companion guide for the main one on [this repository](https://github.com/leandroprz/Dell-7567-Laptop-Hackintosh-OpenCore) to install macOS on the previously mentioned laptop.

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
6. Boot using the macOS USB stick and copy the EFI folder from this repo into your EFI system partition
7. Boot into Linux and install [rEFInd](https://www.rodsbooks.com/refind/index.html), it will allow you to boot into macOS, Windows and Linux
8. [Download](https://github.com/leandroprz/Dell-7567-Laptop-Hackintosh-OpenCore/archive/refs/heads/main.zip) or clone this repository: `git clone https://github.com/leandroprz/Dell-7567-Laptop-Hackintosh-OpenCore.git` to get my rEFInd theme and configuration
9. Copy the folder `minimal-black` to `/boot/efi/EFI/refind/themes/`
10. Configure rEFInd to your needs from Linux or macOS by modifying the `refind.conf` and `theme.conf` files located in the EFI system partition

**If you want to learn more or need to do some troubleshooting, keep reading.**

# Overview

Three things are important for this to work: installation order of the operating systems, partition sizes before installing them and their respective file systems.

I'll explain the steps you need to follow and how to get all three systems to show up on a themed boot picker.

## Installation order

The installation order that will make this work is the following:

1. macOS
2. Windows
3. Linux

This laptop has space for two physical drives. I wanted to use one for the operating systems and the other for data. I'll explain how to install all the systems on a single drive.

## Partition sizes

We need to know how much space we'll asign to each system from the start, this is to avoid resizing partitions after each installation as that can cause booting issues.

Let's do some simple math. I have a 2TB NVme drive, so I decided each OS will get a third of the drive:
```
(2TB x 1024) ÷ 3 systems = ~680GB
```
Each OS will have around **680GB**. This is just an estimate, since:
- The exact formatted capacity depends slightly on the filesystem
- There's a difference between decimal terabytes (TB) used by manufacturers and binary tebibytes (TiB) used internally by operating systems
- We also need to account space for the `EFI/ESP` partition

## The EFI/ESP partition

The size of this partition will be determined by the boot manager we'll be using for our three systems. In our case it will be [rEFInd](https://www.rodsbooks.com/refind/index.html).

Some guides recommend the size of the EFI partition to be [1GB](https://wiki.archlinux.org/title/EFI_system_partition#Create_the_partition), others [2GB](https://wiki.cachyos.org/installation/installation_on_root/#size-at-least-2048-mib) or even more, because they are adding the size of the Linux kernel into it. But we'll configure the Linux installation to not use the EFI system partition to store the kernel. Instead we'll use the Linux partition itself, where `/home` and `/` are located.

For our setup we can just use **200MB** (the macOS default) and it will work just fine. You can, however, follow the advice of the linked guides above if you prefer and adjust your partition sizes accordingly. You can also add the size of this partition to the math we did before, but it's so small you won't notice the difference practically (if using 200MB).

# Partitioning the hard drive

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
    - macOS
        - Name: _macOS_
        - Format: `APFS`
    - Windows
        - Name: _Windows_
        - Format: `ExFAT`
    - Linux
        - Name: _Linux_
        - Format: `ExFAT`

    Click on _Apply_.
    
    An identifying name for each partition will be useful on the next steps. Don't mind the `ExFAT` partitions for now, they are serving as placeholders.

# Installing the OSes

## 1. macOS

1. Install macOS as described in the [main guide of this repository](https://github.com/leandroprz/Dell-7567-Laptop-Hackintosh-OpenCore) using the bootable macOS USB along with the custom EFI for this laptop
2. Once finished, shutdown the computer and plug the Windows USB installer

## 2. Windows

1. Boot using the Windows installer and when asked, select _Custom install_
2. You should see the partitions you created in macOS:

    [![Windows installer](assets/win-install.webp "Windows installer")](assets/win-install.webp)

3. Choose the one you previously named _Windows_, format it (the default file system is `NTFS`, this is selected automatically) and install Windows. The installer will probably overwrite the macOS EFI partition, but we'll fix that later
4. Once finished, shutdown the computer and plug the Linux USB installer

## 3. Linux

This step will vary depending on your distro. I chose [CachyOS](https://cachyos.org/) for this laptop. Since CachyOS uses _Calamares_ as the installer, I'll be showing that. But no matter the distro, you basically need to install _GRUB_ as the bootloader and partition your drive as shown in the next steps. The partitioning can also be done using [GParted](https://gparted.org/).

1. Boot using your Linux USB. The CachyOS installer will ask you to select a bootloader, choose _GRUB_. We won't be using it, but since CachyOS doesn't have an option to not install a bootloader, I'm selecting that. Plus, it is a good idea to have GRUB as a fallback in case _rEFInd_ fails after an update
2. When asked how you want to partition the drive, select _Manual partitioning_. You'll see something like this:
    
    [![CachyOS installer](assets/cachyos-install.webp "CachyOS installer")](assets/cachyos-install.webp)
    
    Double click on the partition you previously named _Linux_. We need to change the partition as follow:
    
    - Content: Format
    - File system: `ext4` or `Btrfs`
    - Mount point: `/`
    - FS Label: _CachyOS_ for me
    
    If needed, you can resize it to make room for a `swap` partition. In that case, change the _Size_ at the top.

3. Now double click on the _EFI_ partition. Make sure you have the following:
    - Content: Keep
    - File System: `FAT32`
    - Mount Point: `/boot/efi`
    - FS Label: _EFI_
    - Flags: `boot`

4. Click on _Next_ to continue with the installation. You might get a warning about the EFI partition size, but you can ignore it. Select your desktop environment and finish the installation

# Boot manager configuration

Since Windows probably replaced the macOS configuration on the EFI system partition, we'll fix that and also get the three operating systems working using a single boot manager (rEFInd) to keep things simple.

## Fixing OpenCore

1. Boot using the macOS USB stick and from the OpenCore boot picker select the macOS installation that's already present in your internal SSD drive
2. Once at the desktop run [MountEFI](https://github.com/corpnewt/MountEFI) and mount both the USB and internal SSD EFI partitions
3. Copy the EFI folder from the USB to the internal SSD EFI partition. Your _Linux distro_ and _Microsoft_ folders should already be present in there

## Installing and configuring rEFInd

Why [rEFInd](https://www.rodsbooks.com/refind/index.html) instead of OpenCore? Simply put, the OpenCore bootloader will apply some macOS specific unwanted patches to the other systems when booting and we want to avoid that. You can read more about it [here](https://chriswayg.gitbook.io/opencore-visual-beginners-guide/advanced-topics/dual-boot-options).

Also, I found it easier to setup rEFInd as a boot manager and was able to get it exactly as I wanted (theme included).

### Installing the package

1. Boot into your Linux installation
2. Install rEFInd. The command to install it depends on your distro:
    
    **Arch based**:
    ```
    sudo pacman -S refind
    sudo refind-install
    ```
    **Debian based**:
    ```
    sudo apt update
    sudo apt install refind
    ```
    **Fedora based**:
    ```
    sudo dnf install refind
    sudo refind-install
    ```
For other distros or manual installation, refer to the [official rEFInd website](https://www.rodsbooks.com/refind/installing.html).

### Configuring rEFInd

This repository contains my custom `theme.conf` file. You can [download](https://github.com/leandroprz/Dell-7567-Laptop-Hackintosh-OpenCore/archive/refs/heads/main.zip) that and use it as a starting point. I recommend you to keep reading and modify it to your needs, especially if you are using a different distro than me. The theme I'm using is slightly a modified version of [rEFInd minimal black](https://github.com/andersfischernielsen/rEFInd-minimal-black).

1. You need to get the `UUID` for your Linux partition. Open a Terminal and type `lsblk -f`. You'll see something like this:
    
    ```
    NAME        FSTYPE FSVER LABEL   UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
    sda                                                                                   
    ├─sda1                                                                                
    └─sda2      ntfs         Datos   F6FE4410FE43C797                                     
    zram0       swap   1     zram0   73176885-3bf6-4627-9184-eac74ddfe4bb                [SWAP]
    nvme0n1                                                                               
    ├─nvme0n1p1 vfat   FAT32 EFI     67E3-17ED                             114,8M    42% /boot/efi
    ├─nvme0n1p2 apfs                 f1fc7a76-0380-4e02-9180-954089400214                 
    ├─nvme0n1p3 ntfs         Windows F4F212CAF21290CA                                     
    ├─nvme0n1p4 btrfs        CachyOS 09354f10-a3e8-4ec6-a95c-2adbad1735ea  588,4G     4% /var/tmp
    │                                                                                    /var/log
    │                                                                                    /var/cache
    │                                                                                    /srv
    │                                                                                    /root
    │                                                                                    /home
    │                                                                                    /
    ├─nvme0n1p5                                                                           
    └─nvme0n1p6 swap   1             0860ff70-86a6-4a00-9625-53bdac8c4473                [SWAP]
    ```
    From this output you need to copy the `UUID` for CachyOS.
2. Open the `refind/themes/minimal-black/theme.conf` file downloaded from this repo using a text editor
3. Near the end of the file there's an entry for Linux that says `menuentry "CachyOS" { ...`
4. Inside that block find the line that starts with `options` and replace the `UUID` string with the one you got on the Terminal. For me it is `09354f10-a3e8-4ec6-a95c-2adbad1735ea`
5. Change anything else in the file to your liking. Read the file comments to understand what every line is doing
6. Copy the folder `minimal-black` to `/boot/efi/EFI/refind/themes/`
7. Use a text editor to change the default rEFInd configuration file located in `/boot/efi/EFI/refind/refind.conf`. Add the following at the end of the file:
    
   ```
    # Custom config
    include themes/minimal-black/theme.conf
    ```
8. Inside the EFI folder you got from this repo, there's a folder called `tools`. Copy that folder to `/boot/efi/EFI/` to get _UEFI Shell_ working in the rEFInd boot picker
9. After copying everything your EFI system partition should look like this:
    
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
Now you'll be able to boot into Windows, Linux and macOS directly from the rEFInd boot picker.

# Finishing touches

We need to change our OpenCore `config.plist` to get a cleaner boot picker. This isn't just a cosmetic thing, it's to make sure OpenCore always boots into our macOS installation from rEFInd.

1. Boot into macOS. Go to `System Settings > General > Startup Disk` and select the drive where you installed macOS
2. Mount the SSD's EFI partition using [MountEFI](https://github.com/corpnewt/MountEFI) and open the `config.plist` file using [ProperTree](https://github.com/corpnewt/ProperTree)
3. Change the following value: `Misc > Security > ScanPolicy: 2687747 (number)` to only keep macOS related disks in the OpenCore boot picker
4. Change the following value: `Misc > Boot > ShowPicker: False`. This is to avoid showing the OpenCore boot picker. We don't need to see it since we are booting "directly" from rEFInd into macOS

# References and credits

- [Hackintosh & multiboot on my 2016 laptop - my tips](https://www.reddit.com/r/hackintosh/comments/1n8eqmh/hackintosh_multiboot_on_my_2016_laptop_my_tips/)
- [How to remove Windows and hide Tools and have a Clean OpenCore Boot Menu?](https://www.reddit.com/r/hackintosh/comments/mgt0ow/how_to_remove_windows_and_hide_tools_and_have_a/)
- [Dualbooting on the same disk](https://dortania.github.io/OpenCore-Multiboot/empty/samedisk.html)
- [Dual-Boot Options](https://chriswayg.gitbook.io/opencore-visual-beginners-guide/advanced-topics/dual-boot-options)
- [rEFInd minimal black theme](https://github.com/andersfischernielsen/rEFInd-minimal-black)
- [REFIND - Hackintosh Dual Boot SIN PASAR POR OPENCORE](https://www.youtube.com/watch?v=xZUH0UCL6AI)