Inspired by:
- https://gist.github.com/gutoandreollo/e12455886149a6c85a70
- https://github.com/egara/arch-btrfs-installation
- https://www.gloriouseggroll.tv/arch-linux-efi-install-guide/

# Init
Internet (tethering phone via USB) :
```
ls /sys/class/net
dhcpcd interface
ping -c 3 www.google.com
efivar -l
lsblk
```
# Drive wipe and setting
```
gdisk /dev/sdX (x representing your drive. mine is sda)
x
z
y
y
cgdisk /dev/sdX
```
cdisk options
```
[New] Press Enter
First Sector: Leave this blank ->press Enter
Size in sectors: 1024MiB ->press Enter
Hex Code: EF00 press Enter
Enter new partition name: boot ->press Enter
```
```
[New] Press Enter
First Sector: Leave this blank ->press Enter
Size in sectors: 8GiB ->press Enter
Hex Code: 8200 ->press Enter
Enter new partition name: swap ->press Enter
```
```
[New] Press Enter
First Sector: Leave this blank ->press Enter
Size in sectors: Leave this blank ->press Enter
Hex Code: Leave this blank ->press Enter
Enter new partition name: root ->press Enter
```
Arrow over to [Write] to save your new partitions, hit enter, type “yes”, hit enter again.
Lastly, Arrow over to [Quit] and press enter.
reboot
# Creating partitions
```
mkfs.fat -F32 /dev/sda1
mkswap /dev/sda2
swapon /dev/sda2
mkfs.btrfs -L arch /dev/sda3
mount /dev/sda3 /mnt
cd /mnt
btrfs subvolume create _active
btrfs subvolume create _active/rootvol
btrfs subvolume create _active/tmp
btrfs subvolume create _active/home
btrfs subvolume create _snapshots
cd
umount /mnt
mount -o relatime,space_cache,compress=zstd,subvol=_active/rootvol /dev/sdaX /mnt
mkdir /mnt/{home,tmp,boot}
mount -o relatime,space_cache,compress=zstd,subvol=_active/home /dev/sdaX /mnt/home
mount -o relatime,space_cache,compress=zstd,subvol=_active/tmp /dev/sdaX /mnt/tmp
mount /dev/sdaX /mnt/boot
```
# Configure pacman
```
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
sudo pacman -Sy  
sudo pacman -S pacman-contrib
```
Rank best mirrors
```
sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.backup
rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
```
# Install Base files
```
pacstrap -i /mnt base base-devel
```
# Generate fstab
```
genfstab -U -p /mnt >> /mnt/etc/fstab
nano /mnt/etc/fstab // verify that the drives are correctly mapped
```
# chroot into system
```
arch-chroot /mnt
sudo pacman -S nano // if you don't want to use vim or vi
nano /etc/locale.gen // uncomment locales refering to your language eg. en_US-UTF-8
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
ls /usr/share/zoneinfo/
ln -s /usr/share/zoneinfo/your-time-zone > /etc/localtime //select your zone
hwclock --systohc --utc
echo PC_NAME > /etc/hostname
nano /etc/pacman.conf
```
uncomment multilib
# Setup users
```
pacman -Sy
passwd
useradd -m -g users -G wheel,storage,power -s /bin/bash someusername
passwd someusername
EDITOR=nano visudo
```
uncomment and add, to make sudoers use root password
```
%wheel ALL=(ALL) ALL
Defaults rootpw
```
# Setups
```
pacman -S bash-completion
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
bootctl install
nano /boot/loader/entries/arch.conf
```
add
```
title Arch Linux
linux /vmlinuz-linux
initrd  /intel-ucode.img
initrd /initramfs-linux.img
options root=LABEL=arch rw,nvidia-drm.modeset=1 rootflags=subvol=_active/rootvol
```
get uuid, if you dont wanna use the label
```
echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/sdb3) rw" >> /boot/loader/entries/arch.conf
```
# Install apliances
```
pacman -S intel-ucode
pacman -S networkmanager
systemctl enable NetworkManager
pacman -S linux mkinitcpio linux-headers btrfs-progs
pacman -S nvidia-dkms libglvnd nvidia-utils opencl-nvidia lib32-libglvnd lib32-nvidia-utils lib32-opencl-nvidia nvidia-settings
pacman -S mesa lib32-mesa xf86-video-intel vulkan-intel #if intel graphics
sudo nano /etc/mkinitcpio.conf
```
Change in modules
```
MODULES="nvidia nvidia_modeset nvidia_uvm nvidia_drm"
```
Add ```/usr/bin/btrfsck``` to BINARIES
In HOOKS
- remove "fsck" and 
- add "btrfs" at the end
```
mkinitcpio -p linux
```
Create pacman nvidia HOOK
```
mkdir /etc/pacman.d/hooks
nano /etc/pacman.d/hooks/nvidia.hook

[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia

[Action]
Depends=mkinitcpio
When=PostTransaction
Exec=/usr/bin/mkinitcpio -P
```
# Reboot into the system
```
exit
umount -a
reboot
```
See if the system boots correctly from your drive.
# Install the DE
It's done.
