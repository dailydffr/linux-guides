# About This Guide

This document began as a personal reference for building what I consider the *ideal Arch Linux system*—lean, secure, and tuned for modern hardware. I’ve chosen to publish it in case others find value in the same approach.

⚠️ **Note:** This is *not* a beginner’s Linux tutorial. The guide assumes you already know how to:
- Download the official Arch Linux ISO.  
- Create a bootable USB/DVD.  
- Boot from installation media.  
- Work comfortably in a Linux shell.  

Network setup inside Arch is covered here, but your underlying router, Wi-Fi, or Ethernet configuration is outside the scope.

---

## Design Goals

This guide focuses on laptops and other modern systems (roughly last 4–5 years) with SSD/NVMe storage, UEFI firmware, and TPM modules. My priorities are:

- **Security** – Full-disk encryption, including `/boot`, with TPM-sealed keys (via Clevis) where possible.  
- **Portability** – Safe to carry into the world; theft or drive loss should not expose sensitive data.  
- **Reliability** – Use of Btrfs snapshots for rollbacks; minimizing risk from Arch’s bleeding-edge updates.  
- **Fault tolerance** – A separate, encrypted recovery partition capable of restoring or reinstalling the OS if the main system becomes unbootable.  
- **Performance** – SSD/NVMe-optimized layout and best practices for modern hardware.

Some legacy systems (e.g. BIOS-only machines or those with spinning IDE drives) may still work with these instructions if additional hardware—such as an aftermarket TPM module—is installed. However, such modules are rare, often case-specific, and not covered here. In short: anything older than ~2020 falls outside the scope of this guide and is **not supported**.

---

## Scope & Assumptions

- **Single-boot only.** Dual-booting with Windows or other OSes is not covered.  
- **All commands** are run as `root` (prompt shown as `#`). If you see a `$` prompt, you are probably **not** in the Arch ISO. Triple-check what environment you’ve booted into before proceeding.  

---

> ## ⚠️ Important Encryption Note
> This guide uses full disk encryption with Clevis.  
> - The EFI partition remains unencrypted (a limitation of UEFI).  
> - Without TPM, you will be prompted *twice* for your root partition password at boot.  
> - This is acceptable, but weigh the trade-offs carefully before proceeding.

---

###   ⚠️ Security Consideration: Clevis & Evil Maid Attacks

This setup integrates Clevis with TPM sealing. While this improves convenience and security, it comes with an important caveat:

- If your laptop is ever out of your physical control, an attacker could tamper with **GRUB** or even your **UEFI firmware**.  
- When this happens, TPM unsealing will fail. You’ll see policy errors (generic warnings about what changed), followed by a *second* prompt for your LUKS password during boot.  

This double password prompt is a **red flag**. It doesn’t always mean your system is compromised—kernel or system updates can also cause TPM policy changes—but it does mean the trust chain has been altered.  

In maximum-paranoia terms: if an attacker somehow *had* access to your encrypted root (and therefore your initramfs), your system is already lost. The second prompt would still be a clue, but realistically you wouldn’t have the laptop back in that scenario unless the attacker wanted something they couldn’t get without tricking you.  

Either way—benign update or hostile tampering—the second prompt deserves your full attention.

---

With expectations set, let’s begin the initial setup.

---

# SECTION 1: SYSTEM CHECKS

### **Task 1 - Verify boot mode**

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

---

### **Task 2: Identify available storage devices**
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
- In this example the target disk is `/dev/sda` and it is a 256GB SSD.
- NVMe drives use a different naming scheme (e.g., `/dev/nvme0n1`).  
- This guide will use `/dev/sda`. **replace with your actual device!**  
- Copy/paste blindly at your own peril.

---

### **Task 3: Detect network interfaces**

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

---

### **Task 4: Check the TPM**
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

---

# SECTION 2: WIFI & NETWORKING

NOTE: If your system already has an active Ethernet connection (discovered in "SYSTEM CHECKS" → "Detect network interfaces"), you can skip the WiFi and DHCP/Manual IP tasks.

---

### **Task 1: WiFi**
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

---

### **Task 2: DHCP/Manual IP**
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

---

### **Task 3: Check Internet Connection**
Whether via Ethernet or WiFi, confirm Internet connectivity:
```bash
ping -c 3 www.archlinux.org
```
Alternative if `archlinux.org` is unreachable:
```bash
ping -c 3 8.8.8.8
```
Successful replies indicate you have a working Internet connection.

---

### **Task 4: Enable SSH access**
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

---

# SECTION 3: PARTITIONS

NOTE: The astute will notice some strangeness in my partitioning... There is method to my madness.
We will be building a recovery partition, standing right next to the EFI partition. This recovery partition is not just for emergencies — it’s a second, bootable Arch installation. However, for the sake of security it too will get the full encryption treatment.

---

### **Task 1: Build the Partition Table**
Launch the partitioning tool:
```bash
cfdisk /dev/sda
```
NOTE: remember to change `/dev/sda` to match your drive designation.

Use "GPT" for partition type, then follow these steps:

> ">> Free space" -> [New] -> 512M -> [Type] -> EFI System

> ">> Free space" -> [New] -> 12G

> ">> Free space" -> [New] -> MAX SIZE

> [Write] -> type "yes" -> [Quit]

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

---

### **Task 2: Encrypt All the things!**
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
```
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

---

### **Task 3: Swap Calculation**
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

---

### **Task 4: Setup LVMs**
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
```

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

---

### **Task 5: Format Partitions**
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

---

### **Task 6: Setup btrfs on root**
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

---

### **Task 7: Prepare the build space**
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

---

# SECTION 4: INSTALL & CONFIGURATION

---

### **Task 1: Install the base packages**
Refresh pacman/repos (may not be necessary):
```bash
pacman -Syy
```
Install base packages into /mnt:
```bash
pacstrap -i /mnt base linux linux-firmware nano grub efibootmgr
```
NOTES:
- When prompted, install iptables-nft (2)
- When prompted, install mkinitcpio (1)

---

### **Task 2: Run pre-chroot configuration**
Generate fstab:
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Get UUID of the luks container and add it to the grub config file (**THIS MUST BE DONE OUTSIDE CHROOT**):
```bash
cryptsetup luksUUID /dev/sda3 >> /mnt/etc/default/grub`
```
NOTE: We capture the luksUUID here to simplify editing GRUB later inside the chroot. It will be used for GRUB_CMDLINE_LINUX’s cryptdevice parameter.

Reminder: `/dev/sda3` is the main system (cryptsys). 
We’ll capture `/dev/sda2` (recovery) later when we set up *its* GRUB.

Double check system config files:
```bash
cat /mnt/etc/fstab
cat /mnt/etc/pacman.d/mirrorlist
cat /mnt/etc/default/grub
cat /mnt/etc/mkinitcpio.conf
```
Note: you're mostly making sure they're where they belong.

---

### **Task 3: chroot into the build area**
```bash
arch-chroot /mnt /bin/bash
```

You are now inside your new system. All following commands affect the installed OS, not the live ISO.

---

### **Task 4: Install needed tools**
Refresh the repositories.
```bash
pacman -Syy
```
Install packages:
```bash
pacman -S reflector git rsync clevis luksmeta tpm2-tools lvm2 cryptsetup iwd networkmanager openssh sudo
```

---

### **Task 5: Enable Services**
```bash
systemctl enable sshd.service
systemctl enable NetworkManager.service
```

---

### **Task 6: Configure Date, Time, and Location**
Set the region info:
```bash
ln -sf /usr/share/zoneinfo/America/Boise /etc/localtime
```
NOTE: I'm in mountain time so I used America/Boise. See available timezones with `ls /usr/share/zoneinfo`

Check system time:
```bash
date
```
Set the system time to ntp:
```bash
timedatectl set-ntp true
```

```note
$$$NOTE TO SELF (or chatbot if I gave this to you): I would like an option to set the RTC time/date if it is not correct. I know I can use 'hwclock --systohc' to set the RTC from the systemclock, and I know `timedatectl set-ntp true` *should* set the system clock to the correct time... but I can see scenerios where neither are correct and for some reason ntp isn't working. so... how to set the system clock and update RTC?$$$
```

Generate Locale info:
```bash
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
echo LC_MESSAGES=en_US.UTF-8 >> /etc/locale.conf
```
NOTE: I'm in the US so I used "en_US.UTF-8". See [this wiki page](https://wiki.archlinux.org/title/Locale#Generating_locales) for more information on your own location details.

---

### **Task 7: Set the computer name**
```bash
echo archlinux > /etc/hostname`
echo 127.0.0.1	archlinux >> /etc/hosts`
echo 127.0.1.1	archlinux.local.net	archlinux >> /etc/hosts`
```
NOTES:
- Obviously replace "archlinux" with your preferred computer name.
- Also, feel free to set your domain name, if you have one.

---

### **Task 8: Configure users**
Set the root password:
```bash
`# passwd`
```
Configure sudo:
```bash
`# EDITOR=nano visudo`
```
Look for and uncomment the following:
```text
`%wheel      ALL=(ALL) ALL`
```
Configure your user:
```bash
useradd -m -G lp,users,games,wheel -s /bin/bash $USERNAME
passwd $USERNAME
```
NOTE: Change "$USERNAME" to your preferred username.

---

### **Task 9: Setup clevis**
Clone the mkinitcpio-clevis-hook from git:
```bash
cd /tmp
git clone https://github.com/kishorviswanathan/arch-mkinitcpio-clevis-hook.git
cd arch-mkinitcpio-clevis-hook/
sh install.sh
```
NOTES:
- DO NOT ACTUALLY BUILD THE HOOK FROM THE AUR!
- All we need from this specific git clone are the hook and install files (2 short scripts). 
- The install.sh script we ran simply copies these files to the relevant locations for mkinitcpio.
- I was very tempted to rip off kishorviswanathan and paste the scripts here directly... but proper attribution is in order!
- kishorviswanathan is the GOAT!

---

### **Task 10: Edit mkinitcpio.conf**
NOTE: there is a LOT going on here.
- We are configuring zstd kernel compression
- We are adding our primary hooks.
- We are adding the clevis hook (to unlock cryptsys)
- We are enabling kms and the associated modules (if applicable)

Open mkinitcpio.conf:
```bash
nano /etc/mkinitcpio.conf
```

---

For KMS for AMD/Radeon GPUs, modify MODULES to include the following:
```text
`MODULES=(amdgpu)`
```
NOTE: DO NOT USE THIS IF YOU HAVE AN AMD/RADEON/NVIDIA GPU!
Won't hurt. Just won't work.

For KMS for Intel Graphics, modify MODULES to include the following:
```bash
`MODULES=(i915)`
```
NOTE: DO NOT USE THIS IF YOU HAVE AN INTEL/NVIDIA GPU!
Won't hurt. Just won't work.

For KMS for nvidia/nouveau GPUs...
>======================================================================
>
>    "Fuck you nvidia!"
>	
>		            --Linus Torvalds
>				
>======================================================================`
For proprietary nvidia driver users, early KMS is not supported. You’ll get modesetting later in the boot, but that's all nvidia will deign to give you.

NOTE: For those installing on virtualbox, vmware, proxmox and other such hypervisors, the kms HOOK is **essential** for the initial boot stages, else you will get a black screen and no indicator of what's going on. Otherwise, the associated `MODULES` are unnecessary.
	
For more useful info on Kernel Mode Setting (KMS) See the [wiki page](https://wiki.archlinux.org/title/Kernel_mode_setting#Early_KMS_start)

---


Next, modify HOOKS similar to this:
```text
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap block clevis encrypt lvm2 resume filesystems)
```
NOTES: 
- I find it easiest to simply comment out the existing HOOKS line, then just paste in this line right below it.
- Bear in mind, this is *MY* hooks section! You should definitly read [the wiki page](https://wiki.archlinux.org/title/Mkinitcpio#Common_hooks) on hooks.


Finally, for zstd compression, simply uncomment the following:
```bash
COMPRESSION="zstd"
```
done!

Save and exit nano:
> CTRL+X >> CTRL+Y >> [ENTER]

---

### **Task 11: Edit grub config**
```bash
nano /etc/default/grub
```
Modify GRUB_CMDLINE_LINUX to inform clevis which device to decrypt at boot: 
```text
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<luksUUID-of-/dev/sda3>:cryptsys resume=/dev/mapper/lvm-swap"
```
Uncomment/Add the following:
```text
GRUB_ENABLE_CRYPTODISK=y
```
NOTES:
- The UUID should be at the bottom of the file. 
- The UUID was added to the file earlier using `cryptsetup luksUUID /dev/sda3 >> /mnt/etc/default/grub`
- I find it best to move the cursor to the bottom line, use CTRL-K to cut the line, then reposition to where you want to paste it, and use CTRL-U. 
- You'll need to remove the carraige return it likes to add after the UUID pastes in.
- Make sure you delete the UUID from the bottom of the file if you don't use CTRL+K. Leaving it will cause errors!

Save and exit nano:
> CTRL+X >> CTRL+Y >> [ENTER]

---

### **Task 12: Install ucode for KMS**
For AMD/Radeon graphics cards and iGPUs:
```bash
pacman -S amd-ucode
```
For Intel graphics cards and iGPUs:
```bash
pacman -S intel-ucode
```
NOTES:
- It's best to only install the ucode applicable to your GUP/iGPU however, it won't hurt if both get installed.
- For nvidia/nouveau, check the [wiki article on early kms start](https://wiki.archlinux.org/title/Kernel_mode_setting#Early_KMS_start) ...and may god have mercy on your soul.

---

### **Task 13: Build/Install initramfs and grub**
Build the kernel image:
```bash
mkinitcpio -P linux
```
NOTES:
- It **should not** be necessary to build the kernel image if you installed a ucode package. A hook triggers mkinitcpio during package installation, so the initramfs should already be generated.
- If errors occured during the afformentioned mkinitrd (and you've already corrected them), or if you skipped ucode installation, then run this before continuing.

Install the boot loader:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ARCH
grub-mkconfig -o /boot/grub/grub.cfg
```

---

### **Task 14: Configure crypttab**

Generate a random key:
```bash
dd if=/dev/urandom of=/etc/cryptsetup-keys.d/recovery.key bs=4096 count=1
chmod 0400 /etc/cryptsetup-keys.d/recovery.key
```
Notes:
- bs=4096 count=1 creates a 4KB random key (adjust if you want a larger key).
- Permissions are restricted to root (0400) for security.

Add the key to the recovery LUKS header:
```bash
cryptsetup luksAddKey /dev/sda2 /etc/cryptsetup-keys.d/recovery.key
```

- You will be prompted for the current LUKS password (from initial encryption).
- This adds a new key slot using the keyfile.

Now lets update crypttab to unlock the drive automatically:
```bash
UUID=$(cryptsetup luksUUID /dev/sda2)
echo "cryptrec $UUID /etc/cryptsetup-keys.d/recovery.key luks" >> /etc/crypttab
```

NOTE: we will *only* do this for recovery. we NEVER want to store keys for the main OS in the open. Even if encrypted, the main drive can be scraped and keys aquired by maliscious individuals.

---

At this point, you have a fully functional Arch install with full disk encryption.
You *could* exit chroot, unmount the partitions, and continue on to the recovery build... But lets see how much more we can get done first!

---
> # **IMPORTANT NOTE: DO NOT STOP HERE!!!**
> If you opt to skip SECTION 5 and 6, you *absolutely* need to continue to section 7! The recovery build is NOT optional and clevis and the TPM seal is only half installed at this point!
---

---

# SECTION 5: INSTALL ALL THE THINGS!!!

I will divide this section into options: GNOME, KDE Plasma, XFCE, Cinnamon, and LXQt.

---

## **Option 1: GNOME Desktop**

This option installs the **GNOME desktop environment** with the **GDM login manager**, plus a standard suite of applications and utilities to provide a complete desktop experience.  

---

###  Step 1: Install GNOME and GDM
```bash
pacman -S gnome gdm
```

---

###  Step 2: Enable and Start GDM
```bash
systemctl enable gdm.service
```

Optional: Configure **automatic login** by editing:
```bash
/etc/gdm/custom.conf
```
Uncomment and set:
```ini
[daemon]
AutomaticLoginEnable=True
AutomaticLogin=your_username
```

---

###  Step 3: Install Common Utilities
These applications provide the typical desktop tools you’d expect on any modern distribution.  

```bash
pacman -S gnome-calculator gedit \
    vlc rhythmbox \
    cups system-config-printer \
    networkmanager network-manager-applet \
    blueman \
    pavucontrol \
    flameshot
```

Enable printing and NetworkManager:
```bash
systemctl enable cups.service
systemctl enable NetworkManager.service
```

---

###  Step 4: Office / Productivity Suite (Optional)
```bash
pacman -S libreoffice-fresh \
    hunspell-en_us hyphen-en \
    evince
```

---

###  Step 5: Internet Suite (Optional)
```bash
pacman -S firefox thunderbird \
    chromium \
    transmission-gtk
```

---

###  Step 6: Admin / Tools Suite
```bash
pacman -S gvfs gvfs-smb nfs-utils cifs-utils sshfs \
    gparted \
    timeshift
```

---

## **Option 2: KDE Plasma Desktop**

This option installs the **KDE Plasma desktop environment** with the **SDDM login manager**, plus a standard suite of applications and utilities to provide a complete desktop experience.

---

###  Step 1: Install KDE Plasma and SDDM
```bash
pacman -S plasma sddm

---

###  Step 2: Enable and Start SDDM
```bash
systemctl enable sddm.service
```

Optional: Configure **automatic login** by editing:
```bash
/etc/sddm.conf
```
If the file doesn’t exist, generate it:
```bash
sddm --example-config > /etc/sddm.conf
```
Then set:
```ini
[Autologin]
User=your_username
Session=plasma.desktop
```

---

###  Step 3: Install Common Utilities
These applications provide the typical desktop tools you’d expect on any modern distribution.  

```bash
pacman -S kcalc kate vlc elisa \
    cups system-config-printer \
    networkmanager plasma-nm \
    bluedevil pavucontrol spectacle
```

Enable printing and NetworkManager:
```bash
systemctl enable cups.service
systemctl enable NetworkManager.service
```

---

###  Step 4: Office / Productivity Suite (Optional)
```bash
pacman -S libreoffice-fresh hunspell-en_us hyphen-en okular
```

---

###  Step 5: Internet Suite (Optional)
```bash
pacman -S firefox chromium thunderbird ktorrent
```

---

###  Step 6: Admin / Tools Suite
```bash
pacman -S gvfs gvfs-smb nfs-utils cifs-utils sshfs \
    partitionmanager timeshift
```

---

## **Option 3: XFCE Desktop**

This option installs the **XFCE desktop environment** with the **LightDM login manager**, plus a standard suite of applications and utilities to provide a complete desktop experience.

---

###  Step 1: Install XFCE and LightDM
```bash
pacman -S xfce4 xfce4-goodies lightdm lightdm-gtk-greeter
```

---

###  Step 2: Enable LightDM
```bash
systemctl enable lightdm.service
```

Optional: Configure **automatic login** by editing:
```bash
/etc/lightdm/lightdm.conf
```
Uncomment and edit:
```ini
[Seat:*]
autologin-user=your_username
autologin-session=xfce
```

---

###  Step 3: Install Common Utilities
```bash
pacman -S galculator mousepad vlc parole \
    cups system-config-printer \
    networkmanager network-manager-applet \
    blueman pavucontrol xfce4-screenshooter
```

Enable printing and networking:
```bash
systemctl enable cups.service
systemctl enable NetworkManager.service
```

---

###  Step 4: Office / Productivity Suite (Optional)
```bash
pacman -S libreoffice-fresh hunspell-en_us hyphen-en evince
```

---

###  Step 5: Internet Suite (Optional)
```bash
pacman -S firefox chromium thunderbird transmission-gtk
```

---

###  Step 6: Admin / Tools Suite
```bash
pacman -S gvfs gvfs-smb nfs-utils cifs-utils sshfs \
    gparted timeshift
```

---

## **Option 4: Cinnamon Desktop**

This option installs the **Cinnamon desktop environment** with the **LightDM login manager**, plus a standard suite of applications and utilities to provide a complete desktop experience.

---

###  Step 1: Install Cinnamon and LightDM
```bash
pacman -S cinnamon lightdm lightdm-slick-greeter
```

---

###  Step 2: Enable LightDM
```bash
systemctl enable lightdm.service
```

Optional: Configure **automatic login** by editing:
```bash
/etc/lightdm/lightdm.conf
```
Uncomment and edit:
```ini
[Seat:*]
autologin-user=your_username
autologin-session=cinnamon
```

---

###  Step 3: Install Common Utilities
```bash
pacman -S gnome-calculator gedit vlc celluloid \
    cups system-config-printer \
    networkmanager network-manager-applet \
    blueman pavucontrol gnome-screenshot
```

Enable printing and networking:
```bash
systemctl enable cups.service
systemctl enable NetworkManager.service
```

---

###  Step 4: Office / Productivity Suite (Optional)
```bash
pacman -S libreoffice-fresh hunspell-en_us hyphen-en evince
```

---

###  Step 5: Internet Suite (Optional)
```bash
pacman -S firefox chromium thunderbird transmission-gtk
```

---

###  Step 6: Admin / Tools Suite
```bash
pacman -S gvfs gvfs-smb nfs-utils cifs-utils sshfs \
    gparted timeshift
```

---

## **Option 5: LXQt Desktop**

This option installs the **LXQt desktop environment** with the **SDDM login manager**, plus a standard suite of applications and utilities. LXQt is a lightweight alternative ideal for older or low-resource machines, but still user-friendly.

---

###  Step 1: Install LXQt and SDDM
```bash
pacman -S lxqt sddm
```

---

###  Step 2: Enable SDDM
```bash
systemctl enable sddm.service
```

Optional: Configure **automatic login** by editing:
```bash
/etc/sddm.conf
```
Add (or edit):
```ini
[Autologin]
User=your_username
Session=lxqt
```

---

###  Step 3: Install Common Utilities
LXQt doesn’t include much by default, so we’ll add the basics:
```bash
pacman -S gnome-calculator featherpad vlc celluloid \
    cups system-config-printer \
    networkmanager network-manager-applet \
    blueman pavucontrol lximage-qt
```

Enable printing and networking:
```bash
systemctl enable cups.service
systemctl enable NetworkManager.service
```

---

###  Step 4: Office / Productivity Suite (Optional)
```bash
pacman -S libreoffice-fresh hunspell-en_us hyphen-en okular
```

---

###  Step 5: Internet Suite (Optional)
```bash
pacman -S firefox chromium thunderbird transmission-qt
```

---

###  Step 6: Admin / Tools Suite
```bash
pacman -S gvfs gvfs-smb nfs-utils cifs-utils sshfs \
    gparted timeshift
```

---

# SECTION 6: PLYMOUTH ROCKS!

Plymouth provides a graphical splash screen during boot and shutdown. It smooths out the startup experience and hides the initial wall of text, while still letting you switch to verbose output if needed (`Esc` key). It is entirely optional and will have no effect on the rest of this build guide. In other words, this is strictly cosmetic and thus optional!

Me, personally? I like the wall of text. Makes me feel like a hackerman! But that's just me.

---

### **Step 1: Install Plymouth**
```bash
pacman -S plymouth
```

---

### **Step 2: Add Plymouth to initramfs**
Edit:
```bash
/etc/mkinitcpio.conf
```

In the `HOOKS=` line, add **`plymouth`** *after* `base` and `udev`, but **before** `encrypt`:
```bash
HOOKS=(base udev plymouth autodetect microcode modconf kms keyboard keymap block clevis encrypt lvm2 filesystems resume)
```

Rebuild the initramfs:
```bash
mkinitcpio -P
```

---

### **Step 3: Configure GRUB for Plymouth**
Edit:
```bash
/etc/default/grub
```

Find the `GRUB_CMDLINE_LINUX_DEFAULT=` line and add:
```text
splash
```

Optional (to hide most messages and make it *really* clean):
```text
quiet splash
```

Update GRUB:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

### **Step 4: Enable Plymouth Service (for shutdown/reboot splash)**
```bash
systemctl enable plymouth-quit.service
systemctl enable plymouth-quit-wait.service
systemctl enable plymouth-start.service
```

---

### **Step 5: Choose a Theme**
List available themes:
```bash
plymouth-set-default-theme -l
```

Pick one (e.g. *bgrt*, *spinner*, *solar*, *fade-in*, *tribar*):
```bash
plymouth-set-default-theme spinner
```

Rebuild initramfs after changing theme:
```bash
mkinitcpio -P
```

Now your system boots up with a smooth graphical splash!

---

# SECTION 7: RECOVERY

---

### **Task 0: Prepare the recovery build space**

Exit chroot and unmount the arch build space:
```bash
exit
umount -R /mnt
swapoff /dev/mapper/lvm-swap
```
Mount recovery partition:
```bash
mount /dev/mapper/recovery /mnt
```
Create EFI mount point:
```bash
mkdir -p /mnt/boot/efi
```
Mount efi (secure, minimal writes):
```bash
mount -o umask=0077 /dev/sda1 /mnt/boot/efi
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
│ └─cryptrec   253:0    0    12G  0 crypt /mnt
└─sda3           8:3    0 243.5G  0 part  
  └─cryptsys   253:1    0 243.5G  0 crypt 
    ├─lvm-root 253:2    0 234.7G  0 lvm   
    └─lvm-swap 253:3    0   7.8G  0 lvm   
sr0             11:0    1   1.3G  0 rom   /run/archiso/bootmnt
```

---

NOTES:
- We need to effectively repeat everything we did in Section 4. It's a very tedious process so I'm going to just blitz through it. I'm not even going to stop for sanity checks. If you need a full explanation, you can go back to section 4 and just repeat it exactly, here!
- We will NOT be repeating SECTION 5 or 6. Though we will be installing some additional recovery tools.

---

### **Task 1: Install the base packages**
```
pacman -Syy
pacstrap -i /mnt base linux linux-firmware nano grub efibootmgr
```

---

### **Task 2: Run pre-chroot configuration**
```
genfstab -U /mnt >> /mnt/etc/fstab
cryptsetup luksUUID /dev/sda2 >> /mnt/etc/default/grub
```

---

### **Task 3: chroot into the build area**
```
arch-chroot /mnt /bin/bash
```

---

### **Task 4: Install needed tools**
```
pacman -Syy
pacman -S reflector git rsync clevis luksmeta tpm2-tools lvm2 cryptsetup iwd networkmanager openssh sudo
```

---

### **Task 5: Enable Services**
```
systemctl enable sshd.service
systemctl enable NetworkManager.service
```

---

### **Task 6: Configure Date, Time, and Location**
```
ln -sf /usr/share/zoneinfo/America/Boise /etc/localtime
timedatectl set-ntp true
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
echo LC_MESSAGES=en_US.UTF-8 >> /etc/locale.conf
```
> REMINDER: set your own zoneinfo and locale!

---

### **Task 7: Set the computer name**
```
echo recovery > /etc/hostname
echo 127.0.0.1	recovery >> /etc/hosts
echo 127.0.1.1	recovery.local.net	recovery >> /etc/hosts
```

---

### **Task 8: Configure users**
```
passwd
EDITOR=nano visudo
```
- Uncomment:
```
`%wheel ALL=(ALL) ALL`
```

```
useradd -m -G lp,users,games,wheel -s /bin/bash $USERNAME
passwd $USERNAME
```
> REMINDER: set your own username!

---

### **Task 9: Setup clevis**
```
cd /tmp
git clone https://github.com/kishorviswanathan/arch-mkinitcpio-clevis-hook.git
cd arch-mkinitcpio-clevis-hook/
sh install.sh
```

---

### **Task 10: Edit mkinitcpio.conf**
```
nano /etc/mkinitcpio.conf
```
- Set MODULES as needed for your GPU and system
  - AMD/Radeon: amdgpu
  - Intel: i915
- Set HOOKS:
```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap block clevis encrypt lvm2 resume filesystems)
```
- Add `COMPRESSION="zstd"`

---

### **Task 11: Edit grub config**
```
nano /etc/default/grub
```
- Set GRUB_CMDLINE_LINUX:  
```
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<luksUUID-of-/dev/sda2>:cryptsys resume=/dev/mapper/lvm-swap"
```
- Set `GRUB_ENABLE_CRYPTODISK=y`
> REMINDER: the UUID of /dev/sda2 will be at the bottom of the file. CTRL+K to cut. CTRL+U to paste.

---

### **Task 12: Install ucode for KMS**
```
pacman -S amd-ucode       # For AMD/Radeon GPUs
pacman -S intel-ucode     # For Intel GPUs
```

---

### **Task 13: Build/Install initramfs and grub**
```
mkinitcpio -P linux
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ARCH
grub-mkconfig -o /boot/grub/grub.cfg
```

---

WHEW! That was a lot, fast! hopefully you got it!

---

### **Task 14: Install Recovery Tools**
```bash
pacman -S btrfs-progs lvm2 dosfstools ntfs-3g gdisk gptfdisk e2fsprogs rsync parted \
cryptsetup clevis luksmeta tpm2-tools networkmanager iwd openssh vim htop smartmontools \
timeshift grub efibootmgr pacman arch-install-scripts
```
⚠️ Note: Some of these tools may have been installed previously. This command ensures everything needed for recovery is present.

Once installed, the recovery system can:
- Inspect and repair partitions: LVM, Btrfs, ext2/3/4, FAT, NTFS
- Recover encrypted volumes using Clevis/LUKS
- Restore snapshots with Timeshift
- Access the network via Ethernet or Wi-Fi
- Reinstall GRUB or perform a full nuclear reinstall using pacstrap

> ⚠️ Important: While the tools are present on the recovery partition, nothing is automated. Like the Arch system itself, everything is in your hands. You must learn and remember how to use these tools!

> ⚠️ Nuclear reinstall considerations:
> - The recovery partition cannot be mounted into the main OS during reinstall.
> - The EFI partition must be unmounted from the recovery boot environment before mounting it into the build space.

> These steps are only needed in absolute worst-case scenarios — covering situations where the main OS is completely unbootable.

---

# SECTION 8: BOOTLOADERS AND YOU!

---

### **Task 1: Set the boot device in EFI firmware**

Check the current EFI boot setup:
```bash
efibootmgr
```
Example output:
```text
BootCurrent: 0001
Timeout: 3 seconds
BootOrder: 0005,0004,0003,0002,0001,0000
Boot0000* UiApp ...
Boot0001* UEFI QEMU DVD-ROM QM00003 ...
Boot0002* UEFI QEMU QEMU HARDDISK ...
Boot0003* Windows Boot Manager ...
Boot0004* ARCH ...
Boot0005* RECOVERY ...

```
Notes:
- The entries for ARCH and RECOVERY were added in that order; currently RECOVERY is the top boot priority.
- We need to change the boot order to load ARCH first, then RECOVERY, then everything else in their existing order.

Change the EFI boot order:
```bash
efibootmgr -o 0004,0005,0003,0002,0001,0000
```
⚠️ Warning: Do not omit or skip any entries listed in the current BootOrder. Firmware can behave unpredictably if entries are missing.

> Optional cleanup: It is safe to remove superfluous entries if desired. For example, to delete the Windows Boot Manager entry above:
```bash
efibootmgr -b 0003 -B
```
> The BootOrder will still reference the deleted number until explicitly reordered, but the firmware will automatically skip it.

Make these changes now before moving on, or ensure you are at peace with the current setup.

---

### **Task 2: Configure LUKS/TPM2 bind with Clevis**

Bind the root partition to TPM2:
```bash
clevis luks bind -d /dev/sda3 tpm2 '{"pcr_bank":"sha256","pcr_ids":"0,4,7"}'
```
Notes:
- pcr_bank specifies the hash algorithm; sha256 is recommended.
- pcr_ids correspond to firmware and boot measurements:
  - PCR 0: Changes if system firmware (BIOS/UEFI) is updated.
  - PCR 4: Changes if bootloader or kernel is modified (including updates).
  - PCR 7: Captures “boot policy” measurements, including:
    - Default EFI boot entry selection
    - EFI Boot Manager variables (BootOrder, BootNext, etc.)
    - Secure Boot state and policy keys (KEK/PK/DB/DBX) if enabled
    - Certain firmware-specific boot path settings
  - PCR 7 usually remains stable, which is why we ensured the boot manager was correctly set before running this command.

---

### **Task 3: Unmount and reboot**
```bash
exit
umount -R /mnt
swapoff /dev/mapper/lvm-swap
reboot
```

Your Arch system is now fully installed, encrypted, and ready to boot. The recovery partition is also configured for future emergency use.

---

# SECTION 9: BUTTON IT UP!

This section configures a staged, user-mediated recovery update system. Recovery updates are applied only after at least one safe reboot of the main OS, and the user explicitly confirms applying the updates. Only packages that exist on the recovery partition are staged.
Unlike the previous *entire guide* this section expects you to have booted successfully into your new and shiney arch for paranoid people. You should therefore open up a terminal, konsole, or just slip on over to console 2 (CTRL+ALT+F2). Log in as your regular user, and then run the following with sudo!

---

### **Task 1: Prepare staging directories**

Create the directory to track recovery update state:
```bash
sudo mkdir -p /var/cache/recovery-updates
sudo touch /var/cache/recovery-updates/staged-packages.txt
```
State files:
- /var/cache/recovery-updates/staged-packages.txt → Staged package list for recovery.
- /var/cache/recovery-updates/reboot-ok → Marker for at least one safe main OS reboot.
- /var/cache/recovery-updates/last-applied → Timestamp of last recovery update.

---

### **Task 2: Create a list of recovery packages**

This list defines which packages should ever be updated in recovery:
```bash
sudo pacstrap -Qq /recovery > /etc/recovery-pkg-list
```
- Only the packages listed in `/etc/recovery-pkg-list` will be considered for staged updates.

---

### **Task 3: Configure pacman hook to stage updates**

Create `/etc/pacman.d/hooks/recovery-update.hook`:

```bash
sudo tee /etc/pacman.d/hooks/recovery-update.hook > /dev/null << EOF
[Trigger]
Operation = Upgrade
Type = Package
Target = *

[Action]
Description = Stage updates for recovery partition
When = PostTransaction
Exec = /usr/local/bin/stage-recovery-updates.sh
EOF
```

---

### **Task 4: Create staging script**

Create `/usr/local/bin/stage-recovery-updates.sh`:
```bash
sudo tee /usr/local/bin/stage-recovery-updates.sh > /dev/null << EOF
#!/bin/bash

STAGE_DIR="/var/cache/recovery-updates"
STAGED="$STAGE_DIR/staged-packages.txt"
REBOOT_OK="$STAGE_DIR/reboot-ok"
RECOVERY_LIST="/etc/recovery-pkg-list"

mkdir -p "$STAGE_DIR"

# If staged cache exists and a reboot occurred, ask user
if [[ -f "$STAGED" && -f "$REBOOT_OK" ]]; then
    AGE=$(( $(date +%s) - $(stat -c %Y "$STAGED") ))
    echo "Recovery update cache is $AGE seconds old."
    read -p "Apply updates to the recovery partition now? (y/n) " yn
    case $yn in
        [Yy]* ) /usr/local/bin/apply-recovery-updates.sh ;;
        * ) echo "Recovery update postponed." ;;
    esac
fi

# Stage new updates for packages in recovery
for pkg in "$@"; do
    if grep -qx "$pkg" "$RECOVERY_LIST"; then
        grep -qx "$pkg" "$STAGED" || echo "$pkg" >> "$STAGED"
    fi
done
EOF
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/stage-recovery-updates.sh
```

---

### **Task 5: Mark successful reboot**
Create a systemd one-shot service /etc/systemd/system/recovery-reboot-ok.service:
```bash
sudo tee /etc/systemd/system/recovery-reboot-ok.service > /dev/null << EOF
[Unit]
Description=Mark recovery updates safe to apply after reboot
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/touch /var/cache/recovery-updates/reboot-ok

[Install]
WantedBy=multi-user.target
EOF
```
Enable the service:
```bash
systemctl enable recovery-reboot-ok.service
```

---

### **Task 6: Apply staged updates**
Create `/usr/local/bin/apply-recovery-updates.sh`:
```bash
sudo tee /usr/local/bin/apply-recovery-updates.sh > /dev/null << EOF
#!/bin/bash
STAGE_DIR="/var/cache/recovery-updates"
STAGED="$STAGE_DIR/staged-packages.txt"
RECOVERY_MNT="/recovery"

# Ensure recovery is mounted
mount /dev/mapper/cryptrec $RECOVERY_MNT

if [[ -f "$STAGED" ]]; then
    echo "Applying recovery updates..."
    while read -r pkg; do
        pacman -S --noconfirm --root "$RECOVERY_MNT" "$pkg"
    done < "$STAGED"

    rm -f "$STAGED"
    touch "$STAGE_DIR/last-applied"
fi

umount $RECOVERY_MNT
echo "Recovery partition updated successfully."
EOF
```
Make it executable:
```bash
chmod +x /usr/local/bin/apply-recovery-updates.sh
```

Summary:
- On first run after a reboot: recovery updates are staged but not applied.
- After at least one safe reboot: user is prompted to apply updates.
- Only packages installed on recovery are staged and updated.
- Updates do not pile on automatically — the user is informed and must confirm.
