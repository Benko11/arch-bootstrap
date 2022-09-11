# Arch Bootstrap

These are the scripts I use during my Arch installation to set up my computer just the way I'd like.

## Preliminary setup

Loading the proper keyboard layout for me:

```
loadkeys uk
```

I make sure to verify my current boot mode is UEFI

```
ls /sys/firmware/efi/efivars
```

Next, I connect to my WiFi using:

```
# iwctl
[iwd]# station wlan connect "Mordecai and Rigby"
```

Then I enter the passphrase for the network and leave the `iwctl` environment.

It is good practice to make sure the network connection is working with `ping google.com`.

Lastly, I update the system clock:

```
timedatectl set-ntp true
```

## Disk partitioning

Given the drive on my laptop, I usually initialize partitioning with:

```
fdisk /dev/nvme0n1
```

Next up, I create three partfitions and configure them like so:

```
512M        EFI System partition        boot
2G          Swap partiton               swap
the rest    BTRFS partition             root
```

After leaving `fdisk`, I make sure to configure and mount each partition appropriately:

### Boot partition

```
mkfs.fat -F 32 /dev/boot_partition
mount --mkdir /dev/boot_partition /mnt/boot
```

### Swap partition

```
mkswap /dev/swap_partition
swapon /dev/swap_partition
```

### Root partition

```
mkfs.btrfs /dev/root_partition
mount /dev/root_partition /mnt
```

## Installing packages

Many packages are important to be able to bootstrap my new Arch install:

```
pacstrap /mnt base base-devel linux-zen linux-firmware btrfs-progs networkmanager vim sudo man-db man-pages texinfo zsh grub efibootmgr amd-ucode
```

Afterwards, I generate the `fstab` file:

```
genfstab -U /mnt >> /mnt/etc/fstab
```

Then I enter the newly created system:

```
arch-chroot /mnt
```

## Initial system configuration

I start with setting the time:

```
ln -sf /usr/share/zoneinfo/Europe/Bratislava /etc/localtime
hwclock --systohc
```

For localization, I create the `/etc/locale.gen`, where I uncomment `UTF-8` lines with `en-US` or `en-GB`. To generate them, I run `locale-gen`.

I create the `/etc/locale.conf` file with the contents:

```
LANG=en_GB.UTF-8
```

And another file `/etc/vconsole.conf`:

```
KEYMAP=uk
```

To identify the device on the network, I set up the hostname in the file `/etc/hostname`:

```
Benko11
```

I set up the root password with `passwd`.

Then I install and configure the bootloader:

```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

Everything is set up properly for the next boot:

```
exit
umount -R /mnt
reboot
```
