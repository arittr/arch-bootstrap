## A simple guide to bootstrap Arch Linux on an XPS 15 9570

**Expect this to take at approximately 2 hours. Go slow, be patient, have fun.** 

*See https://wiki.archlinux.org/index.php/Dell_XPS_15_9560 for additional info as needed.*


## Prework

0. Download the Arch installer and write the ISO to a USB/external disk using Etcher: https://etcher.io/.
1. Boot into Windows and install any BIOS updates that might exist.
2. Dual boot prework - **skip this step if you plan to single boot Linux (or install Windows later)**.
	1. Switch Windows 10 from RAID/IDE to AHCI: https://triplescomputers.com/blog/uncategorized/solution-switch-windows-10-from-raidide-to-ahci-operation/
	2. Resize your Windows partition: https://www.lifewire.com/how-to-open-disk-management-2626080
3. Boot into the UEFI/BIOS (e.g. hit F-12 during POST).
	1. Change the SATA Mode from "RAID" to "AHCI" (unless you already did so in 2.1).
	2. Disable Secure Boot.


## Start the installer

4. Boot up the computer and hit F-12 during POST. Select the external disk/USB that has the Arch installer. Wait for the installer to boot.
5. Once started, run `wifi-menu` and connect to WiFi.


## Wipe, Partition, and Format the disk

6. Figure out the name and current partition structure of your disk.
	1. `fdisk -l`
	2. Take note of the name of your disk. For NVMe drives this is often `/dev/nvme0n1`, so we'll be using that for this guide.
	3. Take note of any existing partitions. This is especially important if dual-booting.

7. Wipe the disk. **If dual booting Windows skip this step**.
	1. `gdisk /dev/nvme0n1`
	2. Hit `x` (for advanced options), then `z` (wipe the disk), then `y` and `y` (or whatever confirmation is required).

8. Partition the disk.
	1. `cfdisk /dev/nvme0n1`
	2. Select `gpt`
	3. Create the EFI partition. **If dual booting Windows skip this step**.
		1. Select `New`.
		2. Make the partition size `300M`.
		3. Select `Type` - `EFI System`.
	4. Create the Swap partition.
		1. Select `New`.
		2. Make the partition size `16G` (or `32G` depending on preference).
		3. Select `Type` - `Linux swap`.
	5. Create the Root partition.
		1. Select `New`.
		2. Hit Enter to make the partition size equal to the rest of the disk.
	6. Select `Write` and confirm with `yes`.
	7. Select `Quit`.

9. Encrypt, Format, and Mount the Root partition.
	1. `fdisk -l`
	2. Take note of the new partition structure. Make sure you know the ids of the EFI, Swap, and Root partitions (e.g. `/dev/nvme0n1p1` for the EFI partition). You'll		 need the Root partition (e.g. `/dev/nvme0n1p3` for the next step).
	3. `cryptsetup -y -v luksFormat --type luks2 /dev/nvme0n1p3` (assuming your Root partition is at `/dev/nvme0n1p3`).
	4. Input a password.
	5. `cryptsetup open /dev/nvme0n1p3 cryptroot`
	6. Input your password.
	7. `mkfs.ext4 /dev/mapper/cryptroot` - this formats the partition.
	8. `mount /dev/mapper/cryptroot /mnt` - this mounts the partition to prepare for installation.

10. Format and Mount the Swap partition.
	1. `mkswap /dev/nvme0n1p2` (assuming your Swap partition is at `/dev/nvme0n1p2`).
	2. `swapon /dev/nvme0n1p2`

11. Format and Mount the EFI partition.
	1. `mkfs.vfat -F32 /dev/nvme0n1p1` (assuming your EFI partition is at `/dev/nvme0n1p1`). **If dual booting Windows skip this step**.
	2. `mkdir /mnt/boot`
	3. `mount /dev/nvme0n1p1 /mnt/boot`


## Bootstrap the Arch installation

12. Ensure system clock is accurate
	1. `timedatectl set-ntp true`

13. Install the base system
	1.  `pacstrap /mnt base base-devel refind-efi zsh vim git efibootmgr dialog wpa_supplicant intel-ucode sudo`

14. Generate the fstab - this automounts your partitions on boot.
	1. `genfstab -pU /mnt >> /mnt/etc/fstab`
	2. Make minor changes to fstab via `vim /mnt/etc/fstab` or similar.
		1. Change `relatime` for `/dev/mapper/cryptroot` to `noatime` (this supposedly reduces wear on your SSD).
		2. (OPTIONAL) Make `/tmp` a ramdisk by adding the following line to /mnt/etc/fstab. This ensures `/tmp` erases on reboot.
			1. `tmpfs    /tmp    tmpfs    defaults,noatime,mode=1777    0    0`


## Finish bootstrapping

15. Enter the new system.
	1. `arch-chroot /mnt`

16. Setup the system clock.
	1. `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime`
	2. `hwclock --systohc`

17. Set the hostname.
	1. `echo MYHOSTNAME > /etc/hostname` (assuming your hostname is `MYHOSTNAME`). This step is not particularly important.

18. Update the locale.
	1. `vim /etc/locale.gen`
	2. Find `en_US.UTF-8 UTF-8` and uncomment (remove the `#`).
	3. `locale-gen`
	4. `echo LANG=en_US.UTF-8 >> /etc/locale.conf`
	5. `echo LANGUAGE=en_US >> /etc/locale.conf`
	6. `echo LC_ALL=C >> /etc/locale.conf`

19. Set password for root.
	1. `passwd`

20. Add a real user. Remove `-s /bin/zsh` if you don't want to use `zsh` (but maybe you should).
	1. `useradd -m -g users -G wheel -s /bin/zsh MYUSERNAME` (assuming your username is `MYUSERNAME`).
	2. `passwd MYUSERNAME`

21. Add the `wheel` group to Sudoers.
	1. `visudo`
	2. Uncomment `%wheel ALL=(ALL) ALL`

22. Configure mkinitcpio with modules needed for the initrd image.
	1. `vim /etc/mkinitcpio.conf`
	2. Add `ext4` to `MODULES`.
	3. Add `encrypt` to `HOOKS` before `filesystems`.
	4. `mkinitcpio -p linux` to regenerate the image.

23. Set up the refind bootloader.
	1. `refind-install`
	2. Figure out the uuid of the encrypted partition
		1. `ls -la /dev/disk/by-uuid` - note the uuid linked to `/dev/nvme0n1p3`.
		3. `vim /boot/refind_linux.conf`
		4. Delete all lines beginning with `archisobasedir`.
		5. Edit `Boot with minimal options` to be:
			1. `cryptdevice=UUID=<device-UUID>:cryptroot root=/dev/mapper/cryptroot` where `<device-UUID` is the UUID you found in 23.2.1.
		6. Add `Boot to single user mode`:
			1. `cryptdevice=UUID=<device-UUID>:cryptroot root=/dev/mapper/cryptroot single` where `<device-UUID` is the UUID you found in 23.2.1.

24. `exit` and go into the installer shell

25. Unmount the partitions.
	1. `umount -R /mnt`
	2. `swapoff -a`

26. Ensure the EFI boot entry was written.
	1. `efibootmgr -v`
	2. Verify you see an entry for `rEFInd`.
	3. Verify `BootOrder` has that entry's number first (e.g. `0001,0002...` where `0001` is rEFInd).

27. `reboot`


## If the system does not boot into the rEFInd bootloader

*see https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface#Launching_UEFI_Shell*

28. Manually add the EFI boot entry.
	1. Boot back into the Arch USB, before the installer starts select `EFI manager v2` or similar.
	2. `map` - take note of the device entry for your disk (e.g. whatever one says `GPT` somewhere).
	3. `bcfg boot add 3 FS0:\EFI\refind\refind_x64.efi "rEFInd"` (assuming `FS0` is the device id found in 28.2).
	4. Exit and reboot.


## Rejoice
**You now have a bootable Arch installation, but this is where the real fun begins.**

29. Install the NVIDIA drivers. See https://wiki.archlinux.org/index.php/NVIDIA#Installation, but this is probably as simple as `sudo pacman -S nvidia`.

30. `nvidia-smi` - do you see your card? If so, be happy this worked the first time...

31. Install Bumblebee. See https://wiki.archlinux.org/index.php/bumblebee. Do not install `bbswitch` - it is currently broken. Check https://bbs.archlinux.org/viewtopic.php?id=238389 for potential fixes.

32. Install *everything else you need to have a usable computer*. See `pacman_list.txt` for suggestions. I **highly** suggest you begin with `LightDM` (https://wiki.archlinux.org/index.php/LightDM). `GDM` seems to be unstable on this system at this time. You can then proceed to installing a Desktop Environment (https://wiki.archlinux.org/index.php/desktop_environment) or i3 (https://wiki.archlinux.org/index.php/i3).

*Remember this is supposed to be fun - but also a little overwhelming. Go slow, think about your choices, and understand getting a usable daily system will likely be a long and evolving process. Be patient.*
