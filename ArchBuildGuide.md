

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

Either way—benign update or hostile tampering—anytime you see a second prompt for a password at boot time, deserves your full attention.  

---

With expectations set, let’s begin the initial setup.

# SECTION 1: SYSTEM CHECKS

---

###  **Task 1 - Verify boot mode**

Check whether the system is booted in **UEFI mode** (required for this guide):
```bash
ls -l /sys/firmware/efi/efivars
```
Expected output (truncated):
```text
total 0
-rw-r--r-- 1 root root   66 Aug 22 13:05 Boot0000-8be4df61-93ca-11d2-aa0d-00e098032b8c
-rw-r--r-- 1 root root  112 Aug 22 13:05 Boot0002-8be4df61-93ca-11d2-aa0d-00e098032b8c
-rw-r--r-- 1 root root   24 Aug 22 13:05 LoaderFirmwareType-8be4df61-93ca-11d2-aa0d-00e098032b8c
-rw-r--r-- 1 root root    5 Aug 22 13:05 SecureBoot-8be4df61-93ca-11d2-aa0d-00e098032b8c
...
```

These files are UEFI variables.  
If the directory is empty, or does not exist, stop here and reboot the installer in **UEFI mode**.


---

###  **Task 2: Identify available storage devices**
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
- In this example the target disk is `/dev/nvme0n1p` and it is a 256GB SSD.
- NVMe drives use a different naming scheme (e.g., `/dev/nvme0n1`).  
- This guide will use `/dev/nvme0n1p`. **replace with your actual device!**  
- Copy/paste blindly at your own peril.

---

###  **Task 3: Detect network interfaces**

Let’s check which devices exist, and whether one is already online:
```bash
ip a
```

Example output:
```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens18: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether a7-dc-2c-ce-e2-fe brd ff:ff:ff:ff:ff:ff
3: wlan0: <BROADCAST,MULTICAST> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 22-f5-7b-d6-25-ea brd ff:ff:ff:ff:ff:ff
```
Interpretation:  
• `ens18` = Ethernet device (currently UP).  
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

###  **Task 4: Check the TPM**
Let’s confirm you have a working **Trusted Platform Module (TPM):**
```bash
ls /dev/tpm*
```
If your system has a TPM, you should see output similar to:
```text
crw-rw---- 1 tss  root  10,   224 Aug 22 13:05 /dev/tpm0
crw-rw---- 1 root tss  253, 65536 Aug 22 13:05 /dev/tpmrm0
```
> ⚠️ If you do not see these devices, you do not have a working TPM module. It is *not* recommended to proceed. ⚠️



# SECTION 2: WIFI & NETWORKING

NOTE: If your system already has an active Ethernet connection (discovered in "SYSTEM CHECKS" → "Detect network interfaces"), you can skip the WiFi and DHCP/Manual IP tasks.

---

###  **Task 1: WiFi**
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
  wlan0                 disconnected  
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
      YOUR-NETWORK-SSID                 psk                 ****    
      It-Burns-When-IP                  psk                 ****    
      ATTjBKEIbz                        psk                 ****    
      this_one_jake                     psk                 ****    
      BlueNose                          psk                 ****    
      OldPeople12345                    psk                 ****    
      in2deep2care                      psk                 ****    
```
One of those should be your router/ap. Otherwise, we're outside the scope of this guide. Have a look [at the wiki for useful information](https://wiki.archlinux.org/title/Network_configuration/Wireless#Troubleshooting). 

Connect to *your* network:
```bash
station wlan0 connect YOUR-NETWORK-SSID
```
Check you're connected:
```bash
station list
```
`wlan0` should now appear "connected":
```text
                             Devices in Station Mode                           *
--------------------------------------------------------------------------------
  Name                  State            Scanning
--------------------------------------------------------------------------------
  wlan0                 connected
```
Exit `iwctl`:
```bash
exit
```

---

###  **Task 2: DHCP/Manual IP**
Most networks use DHCP, which Arch ISO's NetworkManager handles automatically. 

If your network does not provide DHCP, manually assign an IP address:
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

###  **Task 3: Check Internet Connection**
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

###  **Task 4: Enable SSH access**
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

---

###  **Task 1: Build the Partition Table**
Launch the partitioning tool:
```bash
cfdisk /dev/nvme0n1p
```
NOTE: remember to change `/dev/nvme0n1p` to match your drive designation.

Use "GPT" for partition type, then follow these steps:

> ">> Free space" -> [New] -> 512M -> [Type] -> EFI System

> ">> Free space" -> [New] -> 12G

> ">> Free space" -> [New] -> MAX SIZE

> [Write] -> type "yes" -> [Quit]

NOTE: the second partition is the future recovery drive. I set it to 12GB. you can expand or shrink this if you like. 
- For recovery, 6~8GB is enough for CLI-only.
- 10~12GB if you want to add a light GUI.
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

###  **Task 2: Encrypt All the things!**
Encrypt the non-EFI partitions:
```bash
cryptsetup luksFormat --type luks1 --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 50 --sector-size 512 --use-random --verify-passphrase /dev/nvme0n1p2
cryptsetup luksFormat --type luks1 --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 50 --sector-size 512 --use-random --verify-passphrase /dev/nvme0n1p3
```
NOTES: All options in the commands above are chosen to ensure compatibility with GRUB, SSDs, and strong encryption. Note: GRUB does not support LUKS2 yet, hence `--type luks1`, and it's very picky about iter-time and sector-size. Unless you know what you’re doing, just leave the command as-is.

Follow the prompts, but as a security note, this guide expects you to use a **STRONG** password. A weak password will negate any efforts we go through to secure your computer. You might be tempted to use the same password for both drives. This is not entirely discouraged. **However**, you need to be aware that if a hacker gets access to one password, they have access to both drives.

What qualifies as a strong password? 
- Here's a fast and humorous explanation: [XKCD webcomic # 936](https://xkcd.com/936/)
- If you'd like help picking a strong password: [xkpasswd.net](https://www.xkpasswd.net/)
  
Now, Open the newly encrypted partitions:
```bash
cryptsetup luksOpen /dev/nvme0n1p2 cryptrec
cryptsetup luksOpen /dev/nvme0n1p3 cryptsys
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

###  **Task 3: Swap Calculation**
How much swap do you need?
The old rule of thumb was 2.5× your RAM—back when 4 GB was considered “friken' *HUGE*.” Nowadays, 64 GB of RAM isn’t entirely unreasonable. As of this writing, most systems only need a couple of gigabytes of swap—unless you want **hibernation/resume**. In that case, you need roughly the same amount of swap space as your total RAM size, plus a little overhead.

Let’s calculate it with a simple one-liner:
```bash
awk '/MemTotal/ {ram=$2/1024; swap=(ram<2048?ram*2:(ram>65536?65536:ram+4096)); printf "%.0f\n", swap}' /proc/meminfo
```
This simply prints out a number. **Take note of it!**

The logic behind this number:
- If RAM < 2 GB → swap = 2× RAM (small systems need more swap).
- If RAM > 64 GB → swap capped at 64 GB (no need for enormous swap). 
- Otherwise → swap = RAM + 4 GB (covers hibernation and typical system usage).

If you *do* have 64 GB or more of RAM… congratulations! You probably don’t even need swap for system use! However, If you **want** more swap with 64GB+ of RAM… **why?**

Either way, make sure you take note of the number, you'll need it in a moment.


---

###  **Task 4: Setup LVMs**
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

###  **Task 5: Format Partitions**
Format EFI (Fat32):
```bash
mkfs.fat -F32 /dev/nvme0n1p1
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

###  **Task 6: Setup btrfs on root**
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
ID 257 gen 9 top level 5 path @home
ID 258 gen 9 top level 5 path @varlog
ID 259 gen 9 top level 5 path @varcache
ID 260 gen 9 top level 5 path @varlibpacman
ID 261 gen 9 top level 5 path @vartmp
```
Unmount `/dev/mapper/lvm-root`:
```bash
umount /mnt
```
> ⚠️ THIS IS NECESSARY! Do not skip! ⚠️

---

###  **Task 7: Prepare the build space**
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
mount -o umask=0077 /dev/nvme0n1p1 /mnt/boot/efi
```
Mount @home (compressed):
```bash
mount -o ssd,noatime,compress=zstd:3,space_cache=v2,discard=async,subvol=@home /dev/mapper/lvm-root /mnt/home
```
Mount @varlog (no compression; keep CoW/checksums):
```bash
mount -o ssd,noatime,compress=no,space_cache=v2,discard=async,subvol=@varlog /dev/mapper/lvm-root /mnt/var/log
```
Mount @varcache/@vartmp/@varlibpacman (no CoW, no compression):
```bash
mount -o ssd,noatime,nodatacow,compress=no,space_cache=v2,discard=async,subvol=@varcache /dev/mapper/lvm-root /mnt/var/cache
mount -o ssd,noatime,nodatacow,compress=no,space_cache=v2,discard=async,subvol=@vartmp /dev/mapper/lvm-root /mnt/var/tmp
mount -o ssd,noatime,nodatacow,compress=no,space_cache=v2,discard=async,subvol=@varlibpacman /dev/mapper/lvm-root /mnt/var/lib/pacman
```
Make NOCOW persistent for new files in these subvolumes (Optional):
```bash
chattr +C /mnt/var/cache
chattr +C /mnt/var/tmp
chattr -R +C /mnt/var/lib/pacman
```
Mount recovery (quiet, fewer writes; safe journaling):
```bash
mount -o noatime,discard,data=ordered,commit=120 /dev/mapper/cryptrec /mnt/recovery
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
    ├─lvm-root 253:2    0 234.7G  0 lvm   /mnt/var/lib/pacman
    │                                     /mnt/var/tmp
    │                                     /mnt/var/cache
    │                                     /mnt/var/log
    │                                     /mnt/home
    │                                     /mnt
    └─lvm-swap 253:3    0   7.8G  0 lvm   [SWAP]
sr0             11:0    1   1.3G  0 rom   /run/archiso/bootmnt
```

> NOTE: the order of the btrfs subpartitions does **not** matter here. As long as they are mounted, we're fine. **however**, the order *does* matter (slightly) in fstab, which we will check on later.

---

The build space is partitioned, encrypted, formatted, and mounted.
You are ready to install the base system!

# SECTION 4: INSTALL & CONFIGURATION

---

###  **Task 1: Install the base packages**

NOTE: the next section (Rebuild pacman's config and mirrorlist) is entirely unnecessary. I can see a single use case where these instructions would be useful, but you've got to have done something truly unwise to get here (like attempting to install arch from a manjaro install disk... Yeah... Mistakes were made. **don't be like me**). Unless you're in such a situation, just move on to "Install base packages into /mnt":

---

Rebuild pacman's config and mirrorlist:
Reload pacman's config file:
```bash
curl -o /etc/pacman.conf https://gitlab.archlinux.org/pacman/pacman/-/raw/master/etc/pacman.conf.in
```
Edit the new config file:
```bash
nano /etc/pacman.conf
```
Add the `[core]` and `[extra]` stansas (**only** [core] and [extra] are required here):
```text
[core]
Include = /etc/pacman.d/mirrorlist

[extra]
Include = /etc/pacman.d/mirrorlist
```
Download a fresh mirrorlist:
```bash
curl -o /etc/pacman.d/mirrorlist https://archlinux.org/mirrorlist/all/
```
Alternatively, for just US (switch for your country here):
```bash
curl -o /etc/pacman.d/mirrorlist "https://archlinux.org/mirrorlist/?country=US&protocol=https&use_mirror_status=on"
```
Uncomment all `Server` lines:
```bash
sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist
```
Force refresh the keyring (important if switching to Arch):
```bash
pacman -Sy archlinux-keyring
```
Rank mirrors for speed (optional):
```bash
pacman -Sy pacman-contrib
rankmirrors -n 10 /etc/pacman.d/mirrorlist > /etc/pacman.d/mirrorlist.new
mv /etc/pacman.d/mirrorlist.new /etc/pacman.d/mirrorlist
```
Sync pacman:
```bash
pacman -Syyu
```

---

Install base packages into /mnt:
```bash
pacstrap -i /mnt base linux linux-firmware base-devel nano nano-syntax-highlighting grub efibootmgr
```
NOTES:
- When prompted, install iptables-nft (2)
- When prompted, install mkinitcpio (1)


---

###  **Task 2: Run pre-chroot configuration**
Generate fstab:
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Get UUID of the luks container and add it to the bottom of the grub config file (**THIS MUST BE DONE OUTSIDE CHROOT**):
```bash
cryptsetup luksUUID /dev/nvme0n1p3 >> /mnt/etc/default/grub
```
NOTE: We capture the luksUUID here to simplify editing GRUB later inside the chroot. It will be used for GRUB_CMDLINE_LINUX’s cryptdevice parameter. Aquiring this UUID later may add complications. It's best to do it now, when theres no doubt the UUID is correct.

Reminder: `/dev/nvme0n1p3` is the main system (cryptsys). We do NOT need to do this for /dev/nvme0n1p2 **yet**. We’ll capture `/dev/nvme0n1p2` (recovery) later, when we set up GRUB for recovery.

Check system config files:
```bash
cat /mnt/etc/pacman.d/mirrorlist
cat /mnt/etc/default/grub
cat /mnt/etc/mkinitcpio.conf
cat /mnt/etc/fstab
```
Note: you're mostly making sure they're where they belong. If the files are empty, or missing, you need to go back and see where they got missed (likely you need to re-run pacstrap and/or check where you set pacstrap to install)

IMPORTANT!
The output of `cat /mnt/etc/fstab` will look similar to this:
```text
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>
# /dev/mapper/lvm-root LABEL=root
UUID=4a40d049-b74f-4034-915a-a500f5404030	/         	btrfs     	rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=/@	0 0

# /dev/nvme0n1p1
UUID=73DC-6BAC      	/boot/efi 	vfat      	rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro	0 2

# /dev/mapper/lvm-root LABEL=root
UUID=4a40d049-b74f-4034-915a-a500f5404030	/home     	btrfs     	rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=/@home	0 0

# /dev/mapper/lvm-root LABEL=root
UUID=4a40d049-b74f-4034-915a-a500f5404030	/var/log  	btrfs     	rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=/@varlog	0 0

# /dev/mapper/lvm-root LABEL=root
UUID=4a40d049-b74f-4034-915a-a500f5404030	/var/cache	btrfs     	rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=/@varcache	0 0

# /dev/mapper/lvm-root LABEL=root
UUID=4a40d049-b74f-4034-915a-a500f5404030	/var/tmp  	btrfs     	rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=/@vartmp	0 0

# /dev/mapper/lvm-root LABEL=root
UUID=4a40d049-b74f-4034-915a-a500f5404030	/var/lib/pacman	btrfs     	rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=/@varlibpacman	0 0

# /dev/mapper/cryptrec LABEL=recovery
UUID=369de518-2a3d-4428-9a14-775345ac7e10	/recovery 	ext4      	rw,noatime,commit=120,data=ordered	0 2

# /dev/mapper/lvm-swap
UUID=ab1d9112-9d61-43e1-9f70-a935d08debb9	none      	swap      	defaults  	0 0

```
NOTE: Please make note of `# /dev/mapper/lvm-root LABEL=root` There are 6 of them... The one we care about right now is the entry for the actual root ("/")... It *should* already be at the top, but in roughly 25% of my test installs, it ends up on the bottom. Historically the order of mounts in `fstab` have not mattered, but for *some unknown reason*, in *this* build, it **has** mattered, every time! When issues arise, boot halts because the kernel claims it can't find lvm-root. Moving the mountpoint to the top, has resolved it every time.

If you want to make changes to `fstab`, do it now:
```bash
nano /mnt/etc/fstab
```

> I find it best to move the cursor to the line you want to move, use CTRL-K to cut the line, then reposition to where you want to paste it, and use CTRL-U. Rinse and repeat for each line. 

When you're done save and exit nano:
> CTRL+X >> "y" >> [ENTER]

---

###  **Task 3: chroot into the build area**
```bash
arch-chroot /mnt /bin/bash
```

You are now inside your new system. All following commands affect the installed OS, not the live ISO.

---

###  **Task 4: Install needed tools**
Refresh the repositories.
```bash
pacman -Syy
```
Install packages:
```bash
pacman -S reflector git rsync clevis luksmeta tpm2-tools lvm2 dialog networkmanager iw iwd wireless_tools dhcpcd wpa_supplicant openssh
```

---

###  **Task 5: Enable Services**
```bash
systemctl enable sshd.service
systemctl enable NetworkManager.service
systemctl enable bluetooth.service
```

---

###  **Task 6: Configure Date, Time, and Location**
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

If system time is wrong and for some reason NTP isn't working you can set it manually using:
```bash
timedatectl set-time "YYYY-MM-DD HH:MM:SS"
hwclock --systohc
```
> NOTE: replace "YYYY-MM-DD HH:MM:SS" with your *actual* time, ie: "2025-08-01 12:30:21"

Generate Locale info:
```bash
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
echo LC_MESSAGES=en_US.UTF-8 >> /etc/locale.conf
```
NOTE: I'm in the US so I used "en_US.UTF-8". See [this wiki page](https://wiki.archlinux.org/title/Locale#Generating_locales) for more information on your own location details.

---

###  **Task 7: Set the computer name**
```bash
echo archlinux > /etc/hostname
echo 127.0.0.1    archlinux.local.net  archlinux >> /etc/hosts
echo 127.0.1.1    archlinux.local.net  archlinux >> /etc/hosts
```
NOTES:
- Obviously replace "archlinux" with your preferred computer name.
- Also, feel free to set your domain name, if you have one.

---

###  **Task 8: Configure users**
Set the root password:
```bash
passwd
```
Configure sudo:
```bash
EDITOR=nano visudo
```
Look for and uncomment the following:
```text
%wheel      ALL=(ALL) ALL
```
Configure your user:
```bash
useradd -m -G lp,users,games,wheel -s /bin/bash archuser
passwd archuser
```
NOTE: Change "archuser" to your preferred username.

---

###  **Task 9: Configure NetworkManager to use iwd**

IWD supports WPA3 better than wpa_supplicant so lets make use of it!

Configure NetworkManager:
```bash
nano /etc/NetworkManager/NetworkManager.conf
```
Add the following:
```text
[device]
wifi.backend=iwd
```

Save and exit nano:
> CTRL+X >> "y" >> [ENTER]



###  **Task 10: Setup clevis**
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
- also, be smart, and go read the scripts we're copying over:
    - /tmp/arch-mkinitcpio-clevis-hook/hooks/clevis
    - /tmp/arch-mkinitcpio-clevis-hook/install/clevis

---

### **Task 11: Configure crypttab**

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
cryptsetup luksAddKey /dev/nvme0n1p2 /etc/cryptsetup-keys.d/recovery.key
```

- You will be prompted for the current LUKS password (from initial encryption).
- This adds a new key slot using the keyfile.

Now lets update crypttab to unlock the drive automatically:
```bash
UUID=$(cryptsetup luksUUID /dev/nvme0n1p2)
echo "cryptrec	UUID=$UUID	/etc/cryptsetup-keys.d/recovery.key luks" >> /etc/crypttab
ln -s /etc/cryptsetup-keys.d/recovery.key /etc/cryptsetup-keys.d/$UUID.key
```
NOTE: we will *only* do this for recovery. we NEVER want to store keys for the main OS in the open. Even if encrypted, the main drive can be scraped and keys aquired by maliscious individuals.

---

###  **Task 12: Edit mkinitcpio.conf**
NOTE: there is a LOT going on here.
- We are configuring zstd kernel compression
- We are adding our primary hooks.
- We are adding the clevis hook (to unlock cryptsys)
- We are enabling kms and the associated modules (if applicable)
- We are adding the keyfile for the recovery partition.

Open mkinitcpio.conf:
```bash
nano /etc/mkinitcpio.conf
```

---

For KMS for AMD/Radeon GPUs, modify MODULES to include the following:
```text
`MODULES=(amdgpu)`
```
NOTE: DO NOT USE THIS IF YOU HAVE AN INTEL/NVIDIA GPU!
Won't hurt. Just won't work.

For KMS for Intel Graphics, modify MODULES to include the following:
```bash
`MODULES=(i915)`
```
NOTE: DO NOT USE THIS IF YOU HAVE AN AMD/RADEON/NVIDIA GPU!
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
	
For more useful info on Kernel Mode Setting (KMS) See the [wiki page](https://wiki.archlinux.org/title/Kernel_mode_setting#Early_KMS_start)

---

Add your keyfile to the `FILES` section:
```text
FILES=(/etc/cryptsetup-keys.d/recovery.key)
```

Next, modify HOOKS similar to this:
```text
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap block clevis encrypt lvm2 resume filesystems)
```
NOTES: 
- I find it easiest to simply comment out the existing HOOKS line, then just paste in this line right below it.
- Bear in mind, this is *MY* hooks section! You should definitly read [the wiki page](https://wiki.archlinux.org/title/Mkinitcpio#Common_hooks) on hooks.

NOTE: For those installing on virtualbox, vmware, proxmox and other such hypervisors, the `kms` HOOK is **essential** for the initial boot stages, else you will get a black screen and no indicator of what's going on. Otherwise, the associated `MODULES` are unnecessary.


Finally, for zstd compression, simply uncomment the following:
```bash
COMPRESSION="zstd"
```
done!

Save and exit nano:
> CTRL+X >> "y" >> [ENTER]

---

###  **Task 13: Edit grub config**
```bash
nano /etc/default/grub
```
Modify GRUB_CMDLINE_LINUX to inform clevis which device to decrypt at boot: 
```text
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<luksUUID-of-/dev/nvme0n1p3>:cryptsys resume=/dev/mapper/lvm-swap"
```
Uncomment/Add the following:
```text
GRUB_ENABLE_CRYPTODISK=y
```
NOTES:
- The UUID should be at the bottom of the file. 
- The UUID was added to the file earlier using `cryptsetup luksUUID /dev/nvme0n1p3 >> /mnt/etc/default/grub`
- If you use the "CTRL+K" method, you'll need to remove the carraige return it likes to add after the UUID pastes in.
- If you *don't* use CTRL+K, and type/paste it in manually, make sure you delete the UUID from the bottom of the file. Leaving it will cause errors!

Save and exit nano:
> CTRL+X >> "y" >> [ENTER]

---

###  **Task 14: Install ucode for KMS**
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

###  **Task 15: Build/Install initramfs and grub**

NOTES:
- It **should not** be necessary to build the kernel image if you installed a ucode package. A hook triggers mkinitcpio during package installation, so the initramfs should already be generated.
- If errors occured during the afformentioned mkinitrd (and you've already corrected them), or if you skipped ucode installation, then run this before continuing.

Build the kernel image:
```bash
mkinitcpio -P linux
```

---

Install the boot loader:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ARCHLINUX
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# SECTION 5: INSTALL ALL THE THINGS!!!

NOTE: I personally use Cinnamon and lightdm for my setup. As this is primarily a guide for myself, that's where I'm going to focus. However, I will toss in some quick and dirty steps for installing GNOME, KDE Plasma, XFCE, and LXQt. These are NOT to be considered comprehensive, nor complete. For a more thourough guide for any of these environments, I recommend [reading the wiki](https://wiki.archlinux.org/title/Desktop_environment).

---

## **Cinnamon Desktop**

---

###  Step 1: Install Cinnamon and LightDM
```bash
pacman -S cinnamon lightdm lightdm-slick-greeter \
  gnome-calculator gnome-terminal gedit \
  vlc vlc-plugins-all celluloid \
  cups system-config-printer \
  network-manager-applet blueman \
  pavucontrol flameshot \
  libreoffice-still hunspell-en_us hyphen-en evince code geany \
  firefox chromium thunderbird transmission-gtk \
  gvfs gvfs-smb nfs-utils cifs-utils sshfs gparted \
  keepassxc veracrypt
```
NOTES: 
- When pacman asks; use pipewire-jack (2) unless you intend to use pro audo tools... which I am not.
- I use `libreoffice-still`... if you like a more "bleeding edge" you can use `libreoffice-fresh` instead.

---

Configure **automatic login** (Optional):
Setup the autologin group and add your user:
```bash
groupadd -r autologin 2>/dev/null || true
gpasswd -a archuser autologin
```
Reminder: change "archuser" to your username!

Edit lightdm.conf:
```bash
nano /etc/lightdm/lightdm.conf
```
Uncomment and edit:
```text
[Seat:*]
autologin-user=your_username
autologin-session=cinnamon
```
Save and exit:
> CTRL+X > "y" > ENTER

---

Enable services:
```bash
systemctl enable cups.service
systemctl enable lightdm.service
```

Enable Flameshot autostart:
```bash
su archuser      # change to your preferred username!
mkdir -p ~/.config/autostart
cp /usr/share/applications/org.flameshot.Flameshot.desktop ~/.config/autostart/
```

return to root prompt:
```bash
exit
```

---

Fun and games!

Install Vulkan support:
- For AMD graphics:
```bash
pacman -S amdvlk vulkan-radeon
```
- For Intel graphics:
```bash
pacman -S vulkan-intel
```
- For nvidia graphics (proprietary):
```bash
pacman -S nvidia-utils
```
- For nvidia graphics (nouveau):
```bash
pacman -S vulkan-nouveau
```
Please note: I'm not bothering with 32bit libraries and therefor steam will not be installable. They're being slowly phased out and this guide is long enough. If you care about 32bit support or steam, have a look at [the wiki](https://wiki.archlinux.org/title/Vulkan). You will need to enable [multilib] in `/etc/pacman.conf` as well.

Install some games:
```bash
pacman -S lutris
```
Yeah... I don't play a lot of games anymore. :/

---

## **GNOME Desktop**

Install packages:
```bash
pacman -S gnome gdm gnome-calculator gnome-terminal gedit vlc rhythmbox cups system-config-printer networkmanager network-manager-applet blueman pavucontrol flameshot libreoffice-fresh hunspell-en_us hyphen-en evince code firefox thunderbird chromium transmission-gtk gvfs gvfs-smb nfs-utils cifs-utils sshfs gparted
```

Configure **automatic login** (Optional):
```bash
/etc/gdm/custom.conf
```
Uncomment and set:
```ini
[daemon]
AutomaticLoginEnable=True
AutomaticLogin=your_username
```

Enable Services:
```bash
systemctl enable gdm cups NetworkManager
```

---

## **KDE Plasma Desktop**

Install packages:
```bash
pacman -S plasma sddm konsole kcalc kate vlc elisa cups system-config-printer networkmanager plasma-nm bluedevil pavucontrol spectacle libreoffice-fresh hunspell-en_us hyphen-en okular firefox chromium thunderbird ktorrent gvfs gvfs-smb nfs-utils cifs-utils sshfs partitionmanager
```

Configure **automatic login** (Optional):
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

Enable Services:
```bash
systemctl enable sddm cups NetworkManager
```

---

## **XFCE Desktop**

Install packages:
```bash
pacman -S xfce4 xfce4-goodies lightdm lightdm-gtk-greeter galculator mousepad vlc parole cups system-config-printer networkmanager network-manager-applet blueman pavucontrol xfce4-screenshooter libreoffice-fresh hunspell-en_us hyphen-en evince firefox chromium thunderbird transmission-gtk gvfs gvfs-smb nfs-utils cifs-utils sshfs gparted
```

Configure **automatic login** (Optional):
```bash
/etc/lightdm/lightdm.conf
```
Uncomment and edit:
```ini
[Seat:*]
autologin-user=your_username
autologin-session=xfce
```

Enable Services:
```bash
systemctl enable lightdm cups NetworkManager
```

---

## **Option 5: LXQt Desktop**

This option installs the **LXQt desktop environment** with the **SDDM login manager**, plus a standard suite of applications and utilities. LXQt is a lightweight alternative ideal for older or low-resource machines, but still user-friendly.

NOTE: I do not typically use this DE and WM. This section is basic, at best. Feedback is welcome, but YMMV.

---

Install packages:
```bash
pacman -S lxqt sddm gnome-calculator featherpad vlc celluloid cups system-config-printer networkmanager network-manager-applet blueman pavucontrol lximage-qt libreoffice-fresh hunspell-en_us hyphen-en okular firefox chromium thunderbird transmission-qt gvfs gvfs-smb nfs-utils cifs-utils sshfs gparted
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

Enable Services:
```bash
systemctl enable cups sddm NetworkManager
```

---

# SECTION 6: PLYMOUTH ROCKS!

Plymouth provides a graphical splash screen during boot and shutdown. It smooths out the startup experience and hides the initial wall of text, while still letting you switch to verbose output if needed (`Esc` key). It is entirely optional and will have no effect on the rest of this build guide. In other words, this is strictly cosmetic and thus optional!

---

###  **Step 1: Install Plymouth**
```bash
pacman -S plymouth
```

---

###  **Step 2: Add Plymouth to initramfs**
Edit mkinitcpio.conf:
```bash
nano /etc/mkinitcpio.conf
```

In the `HOOKS=` line, add **`plymouth`** *after* `base` and `udev`, but **before** `encrypt`:
```bash
HOOKS=(base udev plymouth autodetect microcode modconf kms keyboard keymap block clevis encrypt lvm2 filesystems resume)
```

---

###  **Step 3: Configure GRUB for Plymouth**
Edit grub config:
```bash
nano /etc/default/grub
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

###  **Step 5: Choose a Theme**
List available themes:
```bash
plymouth-set-default-theme -l
```

Pick one (e.g. *bgrt*, *spinner*, *solar*, *fade-in*, *tribar*):
```bash
plymouth-set-default-theme spinner
```

Rebuild initramfs (must be done after changing theme):
```bash
mkinitcpio -P linux
```

---

# SECTION 7: SNAPPER AND GRUB-BTRFS

Snapper manages Btrfs snapshots for easy rollbacks, while grub-btrfs integrates these snapshots into the GRUB boot menu for bootable recovery points. We'll set this up inside the chroot before exiting.

⚠️ **Note:** In chroot, Snapper may fail due to D-Bus not being available (as systemd isn't fully running). Use the `--no-dbus` flag for configuration commands to bypass this. No need to start or stop D-Bus manually—this flag handles it.

### Task 1: Install Snapper and grub-btrfs
```bash
pacman -S snapper grub-btrfs
```

---

### Task 2: Configure Snapper for Root
If `/.snapshots` exists (unlikely, but check):
```bash
umount /.snapshots
rm -r /.snapshots
```
Create the config:
```bash
snapper --no-dbus -c root create-config /
```
This sets up snapshots for the `@` subvolume (your root).

---

### Task 3: Configure Snapper for Home (Optional, but Recommended)
If `/home/.snapshots` exists:
```bash
umount /home/.snapshots
rm -r /home/.snapshots
```
Create the config:
```bash
snapper --no-dbus -c home create-config /home
```
This sets up snapshots for the `@home` subvolume.

---

### Task 4: Adjust Snapper Permissions (Optional, for Non-Root Access)
For root config:
```bash
chmod 750 /.snapshots
chown :wheel /.snapshots
```
Repeat for home if configured:
```bash
chmod 750 /home/.snapshots
chown :wheel /home/.snapshots
```

---

### Task 5: Enable Snapper Timers
These create hourly snapshots and clean up old ones:
```bash
systemctl enable snapper-timeline.timer
systemctl enable snapper-cleanup.timer
```

---

### Task 6: Create Initial Snapshot
```bash
snapper --no-dbus -c root create --description "Initial installation"
```
For home (if configured):
```bash
snapper --no-dbus -c home create --description "Initial installation"
```

---

### Task 7: Configure grub-btrfs
Enable the service to auto-update GRUB when snapshots change:
```bash
systemctl enable grub-btrfsd.service
```
Regenerate GRUB to include snapshots:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

### Task 8: Pacman Hook for Pre/Post Snapshots (Optional)
For automatic snapshots on package changes, install `snap-pac` (AUR package). Since we're in chroot:
```bash
pacman -S base-devel  # If not installed
su - archuser  # Switch to non-root user (reminder: use the user you suet up earlier)
cd /tmp
git clone https://aur.archlinux.org/snap-pac.git
cd snap-pac
makepkg -si
```
This adds hooks to create snapshots before/after `pacman` transactions.

NOTE: I **really** hate installing from the AUR unless absolutly necessary, so you know this package is truely special!

...and in that vein...

If you recieve the error "ERROR: One or more PGP signatures could not be verified!" when running `makepkg -si`, you need to import the key:
```bash
gpg --recv-keys E4B5E45AA3B8C5C3  # replace the keyID with the unknown public key.
```
Then inspect the PKGBUILD in the cloned directory for the validpgpkeys array:
```bash
cat PKGBUILD | grep validpgpkeys
```
You should see something like this:
```text
validpgpkeys=('8535CEF3F3C38EE69555BF67E4B5E45AA3B8C5C3')
```
Then re-run 
```bash
makepkg -si
```
Return to root shell:
```bash
exit
```

---

> # **IMPORTANT NOTE: DO NOT STOP HERE!!!**
At this point, you have a fully functional Arch install with full disk encryption and snapshots. You *could* exit chroot, unmount the partitions, and reboot into your new OS... But we're not done yet! You *absolutely* need to continue to section 8! The recovery build is NOT optional and clevis and the TPM seal is only half installed at this point! **DON'T YOU GIVE UP ON ME NOW!**
---

# SECTION 8: RECOVERY

This section is where we will build the recovery partition. Full disclosure: recovery isn't strictly necessary, but in the even your have a boot failure and the SSD is still alive, you'll be glad you have it!

---

###  **Task 0: Prepare the recovery build space**

Exit chroot and unmount the arch build space:
```bash
exit
umount -R /mnt
swapoff /dev/mapper/lvm-swap
```
Mount recovery partition:
```bash
mount /dev/mapper/cryptrec /mnt
```
Create EFI mount point:
```bash
mkdir -p /mnt/boot/efi
```
Mount efi (secure, minimal writes):
```bash
mount -o umask=0077 /dev/nvme0n1p1 /mnt/boot/efi
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

Create a minimal 512MB swap file
```bash
fallocate -l 512M /mnt/swapfile
chmod 600 /mnt/swapfile
mkswap /mnt/swapfile
swapon /mnt/swapfile
```

Sanity checks:
```bash
lsblk
```
and...
```bash
swapon --show
free -h
```

---

NOTES:
- Now we essentially need to repeat everything we did in Section 4. There are subtle differences so I'm not going to do the lazy "just go back and repeat". That said, it's a very tedious process so I'm just going to blitz through it.
- We will **NOT** be repeating SECTION 5 or 6. Though we will be installing some additional recovery tools.

---

###  **Task 1: Install the base packages**
```
pacstrap -i /mnt base linux linux-firmware nano nano-syntax-highlighting grub efibootmgr sudo
```
> Reminder: install `iptables-nft` when prompted.

---

###   **Task 2: Run pre-chroot configuration**
```
genfstab -U /mnt >> /mnt/etc/fstab
cryptsetup luksUUID /dev/nvme0n1p2 >> /mnt/etc/default/grub
```

---

###   **Task 3: chroot into the build area**
```
arch-chroot /mnt /bin/bash
```

---

###   **Task 4: Install needed tools**
```bash
pacman -Syy reflector git rsync clevis luksmeta tpm2-tools lvm2 dialog networkmanager iw iwd wireless_tools dhcpcd wpa_supplicant openssh
```

---

###   **Task 5: Enable Services**
```
systemctl enable sshd NetworkManager bluetooth
```

---

###   **Task 6: Configure Date, Time, and Location**
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

###   **Task 7: Set the computer name**
```
echo recovery > /etc/hostname
echo 127.0.0.1	recovery >> /etc/hosts
echo 127.0.1.1	recovery.local.net	recovery >> /etc/hosts
```

---

###   **Task 8: Configure users**
```
passwd
EDITOR=nano visudo
```
- Uncomment:
```
`%wheel ALL=(ALL) ALL`
```

```
useradd -m -G lp,users,games,wheel -s /bin/bash archrecovery
passwd archrecovery
```
> REMINDER: set your own username!

---

###   **Task 9: Setup clevis**
```
cd /tmp
git clone https://github.com/kishorviswanathan/arch-mkinitcpio-clevis-hook.git
cd arch-mkinitcpio-clevis-hook/
sh install.sh
```

---

###   **Task 10: Edit mkinitcpio.conf**
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

###   **Task 11: Edit grub config**
```
nano /etc/default/grub
```
- Set GRUB_CMDLINE_LINUX:  
```
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<luksUUID-of-/dev/nvme0n1p2>:cryptrec"
```
- Set `GRUB_ENABLE_CRYPTODISK=y`
> REMINDER: the UUID of /dev/nvme0n1p2 will be at the bottom of the file. CTRL+K to cut. CTRL+U to paste.

---

###   **Task 12: Install ucode for KMS**
```
pacman -S amd-ucode       # For AMD/Radeon GPUs
pacman -S intel-ucode     # For Intel GPUs
```

---

###   **Task 13: Build/Install initramfs and grub**
```
mkinitcpio -P linux
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=RECOVERY
grub-mkconfig -o /boot/grub/grub.cfg
```

---

WHEW! That was a lot, fast! hopefully you got it all!

---

### **Task 14: Install Additional Recovery Tools**
```bash
pacman -S btrfs-progs dosfstools ntfs-3g gdisk gptfdisk e2fsprogs parted vim htop smartmontools archinstall arch-install-scripts testdisk wipe tmux fsarchiver stress partimage nano-syntax-highlighting

Once installed, the recovery system can:
- Inspect and repair partitions: LVM, Btrfs, ext2/3/4, FAT, NTFS
- Recover encrypted volumes using Clevis/LUKS
- Restore btrfs snapshots
- Access the network via Ethernet or Wi-Fi
- Reinstall GRUB or perform a full nuclear reinstall using pacstrap

> ⚠️ Important: While the tools are present on the recovery partition, nothing is automated. Like the Arch system itself, everything is in your hands. You must learn and remember how to use these tools!

> ⚠️ Nuclear reinstall considerations:
> - The recovery partition cannot be mounted into the main OS during reinstall.
> - The EFI partition must be unmounted from the recovery boot environment before mounting it into the build space.

> These steps are only needed in absolute worst-case scenarios — covering situations where the main OS is completely unbootable.

# SECTION 9: BOOTLOADERS AND YOU!

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
Boot0004* ARCHLINUX ...
Boot0005* RECOVERY ...

```
Notes:
- The entries for ARCHLINUX and RECOVERY were added in that order; currently RECOVERY is the top boot priority.
- We need to change the boot order to load ARCH first, then RECOVERY, then everything else in their existing order.

Change the EFI boot order:
```bash
efibootmgr -o 0004,0005,0003,0002,0001,0000
```
⚠️ Warning: Do not omit or skip any entries listed in the current `BootOrder`. Firmware can behave unpredictably if entries are missing.

> Optional cleanup: It is safe to remove superfluous entries if desired. For example, to delete the Windows Boot Manager entry above:
```bash
efibootmgr -b 0003 -B
```
> The BootOrder will still reference the deleted number until explicitly reordered, but the firmware will automatically skip it.

Make these changes now before moving on, or ensure you are at peace with the current setup.

---

### **Task 15: leave chroot and unmount**
```bash
exit
swapoff /mnt/swapfile
umount -R /mnt
```

---

# SECTION 10: BUTTON IT UP!

### Task 1: Confirm recovery boot options are available

Simply reboot out of the archiso environment:
```bash
reboot
```

When you boot, you may notice right away the recovery partition is NOT listed in the grub boot menu (after entering your luks password). To load into the recovery partition, you must use your PCs boot functions to get into the UEFI boot device menu. This differes from manufacturer to manufacturer, but heres a table with the common methods:

```text
| Vendor (desktop/laptop) | Boot menu key                          | Setup (UEFI/BIOS) key |
| ----------------------- | -------------------------------------- | --------------------- |
| **ASUS**                | `F8`                                   | `Del` or `F2`         |
| **Gigabyte / Aorus**    | `F12`                                  | `Del`                 |
| **MSI**                 | `F11`                                  | `Del`                 |
| **ASRock**              | `F11`                                  | `Del` or `F2`         |
| **Dell**                | `F12`                                  | `F2`                  |
| **HP**                  | `Esc` → `F9`                           | `Esc` or `F10`        |
| **Lenovo (ThinkPad)**   | `F12`                                  | `Enter` → `F1`        |
| **Lenovo (consumer)**   | `F12`                                  | `F2`                  |
| **Acer**                | `F12` (enable in BIOS first sometimes) | `Del` or `F2`         |
| **Toshiba**             | `F12`                                  | `F2`                  |
| **Samsung**             | `Esc` or `F12`                         | `F2`                  |
| **Sony VAIO**           | `F11` or `Assist` button               | `F2`                  |
```

Once in the boot device menu, you will see two familiar entries: `ARCHLINUX` and `RECOVERY`... these *should* be self explanitory; `RECOVERY` will boot into your recovery partition. Doing this should be temporary, so on the next reboot your system will load into `ARCHLINUX` as we set previously with `efibootmgr`. The benefit of this setup is it adds to the obscurity of your recovery options for your typical evil maid attack. It's there, but not obvious.

Simply confirm you can see both `ARCHLINUX` and `RECOVERY` in your firmware's boot device menu.

### **Task 2: Configure LUKS/TPM2 bind with Clevis**

When you finally boot into your new archlinux install, you will notice it asks you for your luks password twice. Once when grub needs to open it to get to the initramfs, and once when the kernel asks for it to fully open the luks partition. This is something I warned you about and this is the behavior you should ALWAYS watch out for! In **this case**, it's merely because we haven't bound the luks partition to the tpm! I tried to make it work from within the chroot, but there were just too many hoops for my ADHD so Lets do it from within the freshly booted OS!

Bind the root partition to TPM2:

Simply open a terminal emulator, or switch to the console (CTRL+ALT+F{2-6}).
Then run this command:
```bash
clevis luks bind -d /dev/nvme0n1p3 tpm2 '{"pcr_bank":"sha256","pcr_ids":"0,4,7"}'
```
It will ask to initialize the drive. This is expected. Hit "y" > ENTER

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

And that's it! Your Arch system is now installed, encrypted, and ready to go safely out into the wild.

---

# WORKS IN PROGRESS!!!

Continue here at your own perill... I'm working on some hooks to allow you to update your recovery partition without haveing to chroot into it for update/upgrades. It'll be nice to have and not have to think about unless you absolutly need it.

# SECTION 11: RECOVERY UPDATES

This section configures a staged, user-mediated recovery update system. Recovery updates are applied only after at least one safe reboot of the main OS, and the user explicitly confirms applying the updates. Only packages that exist on the recovery partition are staged.

---

### **Task 1: Prepare staging directories**

Create the directory to track recovery update state:
```bash
mkdir -p /var/cache/recovery-updates
touch /var/cache/recovery-updates/staged-packages.txt
```
State files:
- /var/cache/recovery-updates/staged-packages.txt → Staged package list for recovery.
- /var/cache/recovery-updates/reboot-ok → Marker for at least one safe main OS reboot.
- /var/cache/recovery-updates/last-applied → Timestamp of last recovery update.

---

### **Task 2: Create a list of recovery packages**

This list defines which packages should ever be updated in recovery:
```bash
pacstrap -Qq /recovery > /etc/recovery-pkg-list
```
- Only the packages listed in `/etc/recovery-pkg-list` will be considered for staged updates.

> $$$ NOTE TO SELF: maybe use `pacman -Qe --root /recovery | cut -d' ' -f1 > /etc/recovery-pkg-list` instead of `pacstrap -Qq /recovery > /etc/recovery-pkg-list`. The former will check for dependancies and not just the installed.


---

### **Task 3: Configure pacman hook to stage updates**

Create /etc/pacman.d/hooks/recovery-update.hook:
```text
[Trigger]
Operation = Upgrade
Type = Package
Target = *

[Action]
Description = Stage updates for recovery partition
When = PostTransaction
Exec = /usr/local/bin/stage-recovery-updates.sh
```

### **Task 4: Create staging script**

Create /usr/local/bin/stage-recovery-updates.sh:
```text
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
```

Make it executable:
```bash
chmod +x /usr/local/bin/stage-recovery-updates.sh
```

### **Task 5: Mark successful reboot**
```text
[Unit]
Description=Mark recovery updates safe to apply after reboot
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/touch /var/cache/recovery-updates/reboot-ok

[Install]
WantedBy=multi-user.target
```
Enable the service:
```bash
systemctl enable recovery-reboot-ok.service
```
### **Task 6: Apply staged updates**
Create `/usr/local/bin/apply-recovery-updates.sh`:
```text
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

