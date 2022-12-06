# Arch Bootstrap

This guide aims to describe the procedures I take during my Arch installations to set up my computer just the way I'd like. Inspiration is garnered mainly from Arch Wiki and other Internet sources. Shoutout to [fanoplanes](https://github.com/fanoplanes/) for getting me into this.

## Preliminary setup

Disable the annoying PC speaker during the installation:

```
rmmod pcspkr
```

Loading the proper keyboard layout for me:

```
loadkeys uk
```

I make sure to verify my current boot mode is UEFI:

```
ls /sys/firmware/efi/efivars
```

The output should not be empty.

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
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs networkmanager vim sudo man-db man-pages texinfo zsh grub efibootmgr amd-ucode gcc gdb ntfs-3g git wget bat light pandoc build-essential linux-headers neofetch ncdu htop btop qemu-full beep bluez bluez-utils
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

For localization, I edit the `/etc/locale.gen`, where I uncomment `UTF-8` lines with `en-US` or `en-GB`. To generate them, I run:

```
locale-gen
```

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

I set up the root password with:

```
passwd
```

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

## Post-install

We are now able to boot into Arch Linux, and use the `root` user for our tinkering. We not have any other users or GUI installed.

### Connect to the Internet

Once again, we need to connect to the Internet. We do this by firstly enabling the `NetworkManager` service:

```
systemctl enable NetworkManager.service
systemctl start NetworkManager.service
```

Then I use `wpa_supplicant` (if things don't work out):

```
wpa_cli
> add_network
> set_network 0 ssid "Mordecai and Rigby"
> set_network 0 psk "password"
> enable_network 0
> save_config
> q
```

Then I enter `nmtui` and connect to the network.

### Git and the documentation (this guide)

Afterwards, I set up `git`:

```
git config --global user.name "Benjamin Bergstrom"
git config --global user.email "my.email@gmail.com"
git config --global init.defaultBranch main
git config --global core.editor "vim"
```

Then I clone this document:

```
git clone https://github.com/benko11/arch-bootstrap
```

### Oh My Zsh!

Next step is installing Oh My Zsh (run either of the lines):

```
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### Sudo Privileges and a Regular User

I edit the `/etc/sudoers` file and uncomment the line with the `wheel`. To save in Vim, I apply `w!` (it will work under root).

I create a regular user:

```
useradd -m -G wheel,network -s /usr/bin/zsh benko11
passwd benko11
```

### Programme Configuration

Staying in the root user, I enable parallel downloads for Pacman by editing `/etc/pacman.conf` and uncommenting the line:

```
ParallelDownloads = 5
```

I want to disable hiding boot messages after login shows up:

```
mkdir /etc/systemd/system/getty@tty1.service.d
vim /etc/systemd/system/getty@tty1.service.d/noclear.conf
```

In the newly created file I add:

```
[Service]
TTYVTDisallocate=no
```

I then make sure to enforce a delay between failed login attempts in file `/etc/pam.d/system-login` (10s):

```
auth optional pam_faildelay.so=10000000
```

I then edit the `/etc/security/faillock.conf` file:

```
deny = 5
fail_interval = 900
unlock_time = 3600
```

I am going to disable the annoying PC speakers (this time on the installed system):

```
# rmmod pcspkr

[/etc/modprobe.d/nobeep.conf]
blacklist pcspkr
```

### Paru

I leave the `root` user, and sign in to the newly created one. Inside, I install the `paru` package.

```
git clone https://aur.archlinux.org/paru
cd paru
makepkg -si
cd
rm -rf paru
```

### Chaotic AUR

Enter `root` user and run:

```
pacman-key --recv-key FBA220DFC880C036 --keyserver keyserver.ubuntu.com
pacman-key --lsign-key FBA220DFC880C036
pacman -U 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-keyring.pkg.tar.zst' 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-mirrorlist.pkg.tar.zst'
```

Edit `/etc/pacman.conf` (add to the end of the file):

```
Append (adding to the end of the file) to /etc/pacman.conf:

[chaotic-aur]
Include = /etc/pacman.d/chaotic-mirrorlist
```

### Dev programmes/tools

#### Installation

```
sudo pacman -S mariadb php nodejs npm postgresql sqlite apache composer
```

#### MariaDB configuration

Configuring MariaDB (change the root password, disable remote access):

```
# mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
# systemctl enable mariadb.service
# systemctl start mariadb.service
# mysql_secure_installation
```

#### PostgreSQL configuration

Configuring PostgreSQL:

```
sudo chattr /var/lib/postgres/data
sudo -iu postgres
initdb -D /var/lib/postgres/data
```

#### MongoDB configuration

Install (through Paru) and configure MongoDB:

```
paru mongodb-bin
sudo systemctl enable mongodb.service
sudo systemctl start mongodb.service
```

#### PHP configuration

Configuring PHP:

```
[/etc/php/php.ini]
date.timezone = Europe/Bratislava
...
display_errors = On
```

Enable the following extensions (MongoDB doesn't need to be explicitly loaded):

```
extension=gd
extension=mysqli
extension=pdo_mysql
extension=pdo_pgsql
extension=pdo_sqlite
extension=pgsql
extension=sqlite3
```

Enable XDebug:

```
sudo pacman -S xdebug

[/etc/php/conf.d/xdebug.ini]
zend_extension=xdebug
```

Install the extensions:

```
sudo pacman -S php-pgsql php-sqlite php-gd phpmyadmin
```

#### NodeJS configuration

We are going to enable global package installs in NodeJS for the current user:

```
[~/.zshrc]
PATH="$HOME/.local/bin:$PATH"
export npm_config_prefix="$HOME/.local"
```

#### Java configuration

```
sudo pacman -S jre-openjdk jdk-openjdk javac
```

#### Miscellaneous

```
sudo pacman -S redis
```

#### Apache HTTP Server

Next, it is a good idea to configure Apache HTTP Server.

```
sudo systemctl enable httpd.service
sudo systemctl start httpd.service
```

```
[/etc/httpd/conf/httpd.conf]
...
Listen 127.0.0.1:80
...
ServerAdmin my.email@gmail.com
```

We are going to bind PHP to the Apache server.

```
sudo pacman -S php-apache
```

```
[/etc/httpd/conf/httpd.conf]
#LoadModule mpm_event_module modules/mod_mpm_event.so
...
LoadModule mpm_prefork_module modules/mod_mpm_prefork.so
...
LoadModule php_module modules/libphp.so
AddHandler php-script .php
...
Include conf/extra/php_module.conf
```

Place each new line at the end of the respective file section.

### Wallpapers and avatars

```
git clone https://gitea.com/benko11/wallpapers
git clone https://gitea.com/benko11/avatars
```

### More applications

These are graphical applications that are considered necessary:

```
sudo pacman -S qbittorrent vlc abiword calibre latex-mk steam file-roller dosbox dosbox-x discord pamac ghostwriter
```

```
paru procyon # aur/procyon-decompiler
paru vscode # chaotic-aur/visual-studio-code-bin
paru microsoft-edge # chaotic-aur/microsoft-edge-stable-bin
paru tor-browser # chaotic-aur/tor-browser
paru aqemu # aur/aqemu
paru pamac # chaotic-aur/archlinux-appstream-data-pamac
```

### Graphical setup

At long last, we are now ready to tackle the GUI part of the setup. This is customized for my currently used hardware:

```
sudo pacman -S xorg-server xf86-video-amdgpu xorg-xinit
cp /etc/X11/xinit/xinitrc ~/.xinitrc
```

Now we are going to install the K Desktop Enviroment (select Noto Fonts, and `eng` option):

```
sudo pacman -S plasma-desktop
sudo pacman -S noto-fonts noto-fonts-emoji noto-fonts-cjk
<!-- sudo pacman -S kde-applications -->
sudo pacman -S xfce4 xfce4-goodies
cd
wget https://github.com/grassmunk/Chicago95/archive/refs/tags/v3.0.1.zip
unzip v3.0.1.zip -d v3.0.1
sudo pacman -S viewnior
```

(`xorg-xinit` enables the `startx` command.)

```
sudo pacman -S sddm
sudo systemctl enable sddm.service
```

Preview the `/usr/lib/sddm/sddm.conf.d/default.conf` file.

Next, I make sure to update the `.xinitrc` file:

```
#tvm &
#xclock -geometry 50x50-1+1 &
#xclock -geometry 80x50+494+51 &
...
#exec xterm ...
exec startplasma-x11
```

Set up desktop environments (adding fonts, removing backgrounds):

```
sudo rm -rf /usr/share/backgrounds/xfce
sudo pacman -S network-manager-applet
sudo rm -rf /usr/share/wallpapers
sudo cp -r ~/arch-bootstrap/fonts/* /usr/local/share/fonts
fc-cache
```

More graphics drivers:

```
sudo pacman -S --needed lib32-mesa vulkan-radeon lib32-vulkan-radeon vulkan-icd-loader lib32-vulkan-icd-loader
```

### Touchpad settings

Make sure the touchpad works properly.

```
sudo ln -s /usr/share/X11/xorg.conf.d/40-libinput.conf /etc/X11/xorg.conf.d/40-libinput.conf

[/etc/X11/xorg.conf.d/40-libinput.conf]
Section "InputClass"
    Identifier "libinput touchpad catchall"
    MatchIsTouchpad "on"
    MatchDevicePath "/dev/input/event*"
    Option "Tapping" "on"
    Option "ClickMethod" "clickfinger"
    Option "NaturalScrolling" "true"
    Driver "libinput"
EndSection
```

### Cleaning up

It is time to remove some extraneous packages I don't use, but were installed in previous package groups. (This may be subject to change.)

```
sudo pacman -Rs akregator artikulate blinken bomber bovo cantor cervisia discover granatier juk k3b kaddressbook kajongg kalarm kalgebra kalzium kanagram kapman kapptemplate katomic kblackbox kblocks kbounce kbreakout kbruch kcachegrind itinerary kdevelop-php kdevelop kdf kfloppy kfourinline kgeography kget kgoldrunner khangman khelpcenter kig kigo killbots kimagemapeditor kiriki kiten kjumpingcube klettres klickety kmahjongg kmines kmouth knetwalk knights kolf kollision klines kompare konquest kontact kontrast konversation kopete korganizer kpat kreversi krdc krfb kruler kshisen ksirk ksnakeduel kspaceduel ksquares ksudoku kteatime ktimer ktouch kturtle kubrick kwordquiz lokalize lskat marble minuet knavalbattle palapeli parley picmi ktuberling rocs skanlite skanpage step sweeper yakuake zanshin umbrello telepathy-kde-contact-list kamoso telepathy-kde-text-ui kdiamond ktorrent ark dragon cantor kmail kmail-account-wizard kdepim-addons akonadi-import-wizard kwalletmanager pim-data-exporter pim-sieve-editor kalendar knotes akonadiconsole akonadi-calendar-tools pimcommon mailcommon
```

## Miscellaneous

After this, the system is ready to go. It is assumed that this guide is performed solely in the terminal, until a restart when the system should boot up nicely to a GUI. KDE and XFCE environments are still customized, but that is not covered here. KDE provides me a modern Mac-like experience, whereas XFCE keeps a nice retro design.
