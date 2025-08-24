## Table of Contents

- [About This Guide](#about-this-guide)
  - [Important Note](#important-note)
- [SECTION 1: SYSTEM CHECKS](#section-1-system-checks)
  - [Task 1 - Verify boot mode](#task-1---verify-boot-mode)
  - [Task 2 - Identify available storage devices](#task-2-identify-available-storage-devices)
  - [Task 3 - Detect network interfaces](#task-3-detect-network-interfaces)
  - [Task 4 - Check the TPM](#task-4-check-the-tpm)
- [SECTION 2: WIFI & NETWORKING](#section-2-wifi--networking)
  - [Task 1 - WiFi](#task-1-wifi)
  - [Task 2 - DHCP/Manual IP](#task-2-dhcpmanual-ip)
  - [Task 3 - Check Internet Connection](#task-3-check-internet-connection)
  - [Task 4 - Enable SSH access](#task-4-enable-ssh-access)
- [SECTION 3: PARTITIONS](#section-3-partitions)
  - [Task 1 - Build the Partition Table](#task-1-build-the-partition-table)
  - [Task 2 - Encrypt All the things!](#task-2-encrypt-all-the-things)
  - [Task 3 - Swap Calculation](#task-3-swap-calculation)
  - [Task 4 - Setup LVMs](#task-4-setup-lvms)
  - [Task 5 - Format Partitions](#task-5-format-partitions)
  - [Task 6 - Setup btrfs on root](#task-6-setup-btrfs-on-root)
  - [Task 7 - Prepare the build space](#task-7-prepare-the-build-space)

# About This Guide

This document began as a personal reference for installing and configuring Arch Linux the way *I* prefer it—lean, secure, and optimized for modern hardware. I’ve chosen to publish it publicly in case others find it useful.

That said, this is not a beginner’s Linux tutorial. The guide assumes you already have some familiarity with Linux and the command line. Specifically, it expects that you:

- Know how to download the official Arch Linux installation image.  
- Can create a bootable USB/DVD from that image.  
- Understand how to boot your system from that media.  
- Have a working network infrastructure ready (e.g., Ethernet cable connected, or a wireless access point available).  

I will provide guidance for connecting to the network during installation, but the underlying setup of your home or office network is outside the scope of this guide.

While these instructions can be adapted to many types of hardware, the guide is written with laptops in mind. Laptops are portable, and with portability comes a higher need for security. For that reason, particular attention is given to features such as:

- UEFI boot environments.  
- TPM-based security integrations.  
- Full-disk encryption and data protection against theft.  

Performance and reliability are also priorities. The guide assumes modern storage such as NVMe or SSD drives, and the configuration choices reflect best practices for SSD health and speed. If you are attempting this installation on legacy hardware—say, a 15+ year-old desktop with mechanical IDE drives—the steps may still work, but there are likely other guides better suited to that scenario.

⚠️ This guide assumes a single-boot system. Dual-booting with Windows or any other OS is not supported here. If that’s your goal, good luck… and keep a stress ball handy.

In short:

- This is a **technical, security-focused, laptop-oriented Arch Linux installation guide**.  
- It prioritizes modern hardware and best practices.  
- It is written for my own use first—but if it helps you, welcome aboard.

Note: All commands in this guide are run as `root` (shown with `#`).  
You’re in the Arch ISO environment — if you see a `$` prompt, something is wrong.  
Switch to root with `su` or restart the ISO and pick the correct boot option.




> ## ⚠️ Important Note ⚠️
> This setup uses full disk encryption with Clevis. The boot partition will be encrypted, and we will not integrate a keyfile into the initramfs. If you are using UEFI without TPM, the guide can still be followed, however:
>  - You will be prompted twice for your root partition password during boot.
>  - This can be an acceptable compromise, but think carefully before continuing.

Now that we have a general understanding of what we're working with, let's move on to the initial setup.



# SECTION 1: SYSTEM CHECKS

> ### **Task 1 - Verify boot mode**

Check whether the system is booted in **UEFI mode** (required for this guide):
```bash
ls -l /sys/firmware/efi/efivars
```
Expected output (truncated):
```text
total 0
-rw-r--r-- 1 root root   66 Aug 22 13:05 Boot0000-8be...
-rw-r--r-- 1 root root  112 Aug 22 13:05 Boot0002-8be...
-rw-r--r-- 1 root root   24 Aug 22 13:05 LoaderFirmwareType-4a6...
-rw-r--r-- 1 root root    5 Aug 22 13:05 SecureBoot-8be...
...
```

These files are UEFI variables.  
If the directory is empty, or does not exist, stop here and reboot the installer in **UEFI mode**.


> ### **Task 2: Identify available storage devices**
We need to confirm what disk(s) are present before partitioning:
```bash
lsblk
```

Example output:
```text
NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0   7:0    0 959.8M  1 loop /run/archiso/airootfs
sda     8:0    0   256G  0 disk 
sr0    11:0    1   1.3G  0 rom  /run/archiso/bootmnt
```

Notes:  
• In this example the target disk is `/dev/sda` and it is a 256GB SSD.
• NVMe drives use a different naming scheme (e.g., `/dev/nvme0n1`).  
• This guide will use `/dev/sda`. **replace with your actual device!**  
• Copy/paste blindly at your own peril.

> ### **Task 3: Detect network interfaces**

Let’s check which devices exist, and whether one is already online:
```bash
ip a
```

Example output:
```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc ... qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST> mtu 1500 qdisc ... qlen 1000
    link/ether a7-dc-2c-ce-e2-fe brd ff:ff:ff:ff:ff:ff
3: wlan0: <BROADCAST,MULTICAST> mtu 1500 qdisc ... qlen 1000
    link/ether 22-f5-7b-d6-25-ea brd ff:ff:ff:ff:ff:ff
```
Interpretation:  
• `ens18` = Ethernet device (currently DOWN).  
• `wlan0` = Wireless adapter (currently DOWN, to be configured later).  
Take note of your actual device names!

If you see something like this:
```text
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ... qlen 1000
    inet 192.168.0.149/24 brd 192.168.0.255 ...
```
Your Ethernet device is already connected. 
Congratulations! You should already have internet access.

If you see interfaces such as:
```text
4: br-bd78161b2e72: <NO-CARRIER,BROADCAST,MULTICAST,UP> ...
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> ...
```
> ⚠️ You are not running from the Arch ISO environment. ⚠️ Likely you booted into another OS. Stop here and re-examine your setup!



> ### **Task 4: Check the TPM**
Let’s confirm you have a working **Trusted Platform Module (TPM):**
```bash
ls /dev/tpm*
```
If your system has a TPM, you should see output similar to:
```text
crw-rw---- 1 tss  root  10,   224 Aug 22 13:05 /dev/tpm0
crw-rw---- 1 root tss  253, 65536 Aug 22 13:05 /dev/tpmrm0
```
> ⚠️ If you do not see these devices, it is *not* recommended to proceed. ⚠️



# SECTION 2: WIFI & NETWORKING

NOTE: If your system already has an active Ethernet connection (discovered in "SYSTEM CHECKS" → "Detect network interfaces"), you can skip the WiFi and DHCP/Manual IP tasks.

> ### **Task 1: WiFi**
If you do not have an active Ethernet link, use iwd to configure WiFi:
```bash
iwctl
station list
```
You will see something like this:
```text
                            Devices in Station Mode                           *
--------------------------------------------------------------------------------
  Name                  State            Scanning
--------------------------------------------------------------------------------
  wlan0                 disconnected     scanning  
```
Your WiFi adapter is typically `wlan0`

Next, scan for access points:
```bash
station wlan0 scan
station wlan0 get-networks
```
You will get back a list, like this:
```text
                               Available networks                              
--------------------------------------------------------------------------------
      Network name                      Security            Signal
--------------------------------------------------------------------------------
      It-Burns-When-IP                  psk                 ****    
      ATTjBKEIbz                        psk                 ****    
      this_one_Tod                      psk                 ****    
      BlueNose                          psk                 ****    
      OldPeople12345                    psk                 ****    
      2deep                             psk                 ****    
```
One of those should be your router/ap. Otherwise, it's outside the scope of this guide.

Connect to *your* network:
```bash
station wlan0 connect YOUR-NETWORK-SSID
exit
```

> ### **Task 2: DHCP/Manual IP**
Most networks use DHCP, which Arch ISO's NetworkManager handles automatically. If your network does not provide DHCP, manually assign an IP address:
```bash
ip addr add 192.168.1.50/24 dev ens18
ip link set ens18 up
ip route add default via 192.168.1.1
```
Assumptions:
- Configuring `ens18` (Ethernet adapter).  
- Default gateway at `192.168.1.1` (adjust for your network).  
- Replace values with your network's configuration.

> ### **Task 3: Check Internet Connection**
Whether via Ethernet or WiFi, confirm Internet connectivity:
```bash
ping -c 3 www.archlinux.org
```
Alternative if `archlinux.org` is unreachable:
```bash
ping -c 3 8.8.8.8
```
Successful replies indicate you have a working Internet connection.

> ### **Task 4: Enable SSH access**
Set a root password:
```bash
passwd
```
Start the sshd daemon:
```bash
systemctl start sshd
```
Confirm the installer's IP address:
```bash
ip a
```
SSH into the live installer from another PC:
```bash
ssh root@<installer-IP-address>
```

Why enable SSH? Copy/pasting commands from this guide remotely is much easier than typing each command manually.




# SECTION 3: PARTITIONS

NOTE: The astute will notice some strangeness in my partitioning... There is method to my madness.
We will be building a recovery partition, standing right next to the EFI partition. This recovery partition is not just for emergencies — it’s a second, bootable Arch installation. However, for the sake of security it too will get the full encryption treatment.

> ### **Task 1: Build the Partition Table**
Launch the partitioning tool:
```bash
cfdisk /dev/sda
```
NOTE: remember to change `/dev/sda` to match your drive designation.

Use "GPT" for partition type, then follow these steps:
<div style="font-size: 0.8em; display:inline-block; padding:2px 6px; border-radius:4px; margin-bottom:2px;">steps</div>

```none
">> Free space" -> [New] -> 512M -> [Type] -> EFI System
">> Free space" -> [New] -> 12G
">> Free space" -> [New] -> MAX SIZE
[Write] -> type "yes" -> [Quit]
```

NOTE: the second partition is the future recovery drive. I set it to 12GB. you can expand or shrink this if you like. 
- For recovery, 6–8GB is enough for CLI-only.
- 10–12GB if you want to add a light GUI.
- Anything larger is just comfort room.

Sanity check.
```bash
lsblk
```
Your drives should now look something like this:
```text
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 959.8M  1 loop /run/archiso/airootfs
sda      8:0    0   256G  0 disk 
├─sda1   8:1    0   512M  0 part 
├─sda2   8:2    0    12G  0 part 
└─sda3   8:3    0 243.5G  0 part 
sr0     11:0    1   1.3G  0 rom  /run/archiso/bootmnt
```

> ### **Task 2: Encrypt All the things!**
Encrypt the non-EFI partitions:
```bash
cryptsetup luksFormat --type luks1 --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 50 --sector-size 512 --use-random --verify-passphrase /dev/sda2
cryptsetup luksFormat --type luks1 --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 50 --sector-size 512 --use-random --verify-passphrase /dev/sda3
```
NOTE: All options in the commands above are chosen to ensure compatibility with GRUB, SSDs, and strong encryption. Note: GRUB does not support LUKS2 yet, hence `--type luks1`, and it's very picky about iter-time and sector-size. Unless you know what you’re doing, just leave the command as-is.

Follow the prompts, but as a security note, this guide expects you to use a **STRONG** password. A weak password will negate any efforts we go through to secure your computer.

What qualifies as a strong password? 
- Here's a fast and humorous explanation: [XKCD webcomic # 936](https://xkcd.com/936/)
- If you'd like help picking a strong password: [xkpasswd.net](https://www.xkpasswd.net/)
  
Now, Open the newly encrypted partitions:
```bash
cryptsetup luksOpen /dev/sda2 cryptrec
cryptsetup luksOpen /dev/sda3 cryptsys
```
And once again, let's check your work!
```bash
lsblk
``
Your drives should now look something like this:
```text
NAME         MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
loop0          7:0    0 959.8M  1 loop  /run/archiso/airootfs
sda            8:0    0   256G  0 disk  
├─sda1         8:1    0   512M  0 part  
├─sda2         8:2    0    12G  0 part  
│ └─cryptrec 253:0    0    12G  0 crypt 
└─sda3         8:3    0 243.5G  0 part  
  └─cryptsys 253:1    0 243.5G  0 crypt 
sr0           11:0    1   1.3G  0 rom   /run/archiso/bootmnt
```

> ### **Task 3: Swap Calculation**
How much swap do you need?
The old rule of thumb was 2.5× your RAM—back when 4 GB was considered “friken' *HUGE*.” Nowadays, 64 GB of RAM isn’t entirely unreasonable. As of this writing, most systems only need a couple of gigabytes of swap—unless you want **hibernation/resume**. In that case, you need roughly the same amount of swap space as your total RAM size, plus a little overhead.

Let’s calculate it with a simple one-liner:
```bash
awk '/MemTotal/ {ram=$2/1024; swap=(ram<2048?ram*2:(ram>65536?65536:ram+4096)); printf "%.0f\n", swap}' /proc/meminfo
```
This simply prints out a number. **Take note of it!**

The logic behind this number:
- If RAM < 2 GB → swap = 2× RAM (small systems need more swap)  
- If RAM > 64 GB → swap capped at 64 GB (no need for enormous swap)  
- Otherwise → swap = RAM + 4 GB (covers hibernation)

If you have 64 GB or more of RAM… congratulations! You probably don’t even need swap for general use! However, If you **want** more swap with 64GB+ of RAM… ***why?***


> ### **Task 4: Setup LVMs**
We’re now going to carve up cryptsys into logical volumes for root and swap.

Initialize the encrypted volume for LVM:
```bash
pvcreate /dev/mapper/cryptsys
```
Create a volume group named 'lvm' on the encrypted volume:
```bash
vgcreate lvm /dev/mapper/cryptsys
```
Create a root logical volume of 1 GB (we'll resize later if needed):
```bash
lvcreate -L 1G lvm -n root
```
Create a swap logical volume using the number you noted from the "Swap Calculation" step (Replace `<SWAP_SIZE>`):
```bash
lvcreate -L <SWAP_SIZE>M lvm -n swap
```
Replace `<SWAP_SIZE>` with the number output from the swap calculation command. Yes, copy/paste that number here.

Extend the root volume to use all remaining free space:
```bash
lvextend -l 100%FREE /dev/mapper/lvm-root
```
Sanity check:
```bash
lsblk
```text
NAME           MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
loop0            7:0    0 959.8M  1 loop  /run/archiso/airootfs
sda              8:0    0   256G  0 disk  
├─sda1           8:1    0   512M  0 part  
├─sda2           8:2    0    12G  0 part  
│ └─cryptrec   253:0    0    12G  0 crypt 
└─sda3           8:3    0 243.5G  0 part  
  └─cryptsys   253:1    0 243.5G  0 crypt 
    ├─lvm-root 253:2    0 234.7G  0 lvm   
    └─lvm-swap 253:3    0   7.8G  0 lvm   
sr0             11:0    1   1.3G  0 rom   /run/archiso/bootmnt
```

> ### **Task 5: Format Partitions**
Format EFI (Fat32):
```bash
mkfs.fat -F32 /dev/sda1
```
Format recovery (ext4):
```bash
mkfs.ext4 -L recovery /dev/mapper/cryptrec
```
Format lvm-root (Btrfs):
```bash
mkfs.btrfs -L root /dev/mapper/lvm-root
```
Format lvm-swap (swap):
```bash
mkswap /dev/mapper/lvm-swap
```

> ### **Task 6: Setup btrfs on root**
Temporarily mount lvm-root to create subvols:
```bash
mount /dev/mapper/lvm-root /mnt
```
Create the subvolumes:
```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@varlog
btrfs subvolume create /mnt/@varcache
btrfs subvolume create /mnt/@varlibpacman
btrfs subvolume create /mnt/@vartmp
```
Sanity check:
```bash
btrfs subvolume list /mnt
```
You should see something like this:
```text
ID 256 gen 9 top level 5 path @
ID 257 gen 10 top level 5 path @home
ID 258 gen 10 top level 5 path @varlog
ID 259 gen 10 top level 5 path @varcache
ID 260 gen 11 top level 5 path @varlibpacman
ID 261 gen 11 top level 5 path @vartmp
```
Unmount `/dev/mapper/lvm-root`:
```bash
umount /mnt
```
> ⚠️ THIS IS NECESSARY! Do not skip! ⚠️

> ### **Task 7: Prepare the build space**
Mount @ (compressed):
```bash
mount -o ssd,noatime,compress=zstd:3,space_cache=v2,discard=async,subvol=@ /dev/mapper/lvm-root /mnt
```
Create mount points:
```bash
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,var/lib/pacman,var/tmp,recovery}
```
Mount efi (secure, minimal writes):
```bash
mount -o umask=0077 /dev/sda1 /mnt/boot/efi
```
Mount @home (compressed):
```bash
mount -o ssd,noatime,compress=zstd:3,space_cache=v2,discard=async,subvol=@home /dev/mapper/lvm-root /mnt/home
```
Mount @varlog (no compression; keep CoW/checksums):
```bash
mount -o ssd,noatime,nocompress,space_cache=v2,discard=async,subvol=@varlog /dev/mapper/lvm-root /mnt/var/log
```
Mount @varcache/@vartmp/@varlibpacman (no CoW, no compression):
```bash
mount -o ssd,noatime,nodatacow,nocompress,space_cache=v2,discard=async,subvol=@varcache /dev/mapper/lvm-root /mnt/var/cache
mount -o ssd,noatime,nodatacow,nocompress,space_cache=v2,discard=async,subvol=@vartmp /dev/mapper/lvm-root /mnt/var/tmp
mount -o ssd,noatime,nodatacow,nocompress,space_cache=v2,discard=async,subvol=@varlibpacman /dev/mapper/lvm-root /mnt/var/lib/pacman
```
Make NOCOW persistent for new files in these subvolumes (Optional):
```bash
chattr +C /mnt/var/cache
chattr +C /mnt/var/tmp
chattr -R +C /mnt/var/lib/pacman
```
Mount recovery (quiet, fewer writes; safe journaling):
```bash
mount -o noatime,discard=async,data=ordered,commit=120 /dev/mapper/cryptrec /mnt/recovery
```
Enable swap
```bash
swapon /dev/mapper/lvm-swap
```
Sanity check:
```bash
lsblk
```
Your partition layout should look like this:
```text
NAME           MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
loop0            7:0    0 959.8M  1 loop  /run/archiso/airootfs
sda              8:0    0   256G  0 disk  
├─sda1           8:1    0   512M  0 part  /mnt/boot/efi
├─sda2           8:2    0    12G  0 part  
│ └─cryptrec   253:0    0    12G  0 crypt /mnt/recovery
└─sda3           8:3    0 243.5G  0 part  
  └─cryptsys   253:1    0 243.5G  0 crypt 
    ├─lvm-root 253:2    0 234.7G  0 lvm   /mnt
    └─lvm-swap 253:3    0   7.8G  0 lvm   [SWAP]
sr0             11:0    1   1.3G  0 rom   /run/archiso/bootmnt
```

The build space is partitioned, encrypted, formatted, and mounted.
You are ready to install the base system!
