# Chad's Guide in Installing Arch

This guide will show step-by-step how to Install Arch Linux on UEFI mode like a Chad.

---

## Bootable Flash Drive
First of all, you need the Arch Linux image, that can be downloaded from the [Official Website](https://www.archlinux.org/download/).
After that, you should create the bootable flash drive.

If you're on a GNU/linux distribution, you can use the `dd` command for it.
```sh
$ dd bs=4M if=/path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync && sync
```
> Note that you need to update the `of=/dev/sdx` with your USB device location (it can be discovered with the `lsblk` command).

Otherwise, if you're on Windows, you can follow this [tutorial](https://wiki.archlinux.org/index.php/USB_flash_installation_media#In_Windows).

---

## BIOS
We'll install Arch on UEFI mode, so you should enable the UEFI mode and disable the secure boot option on your BIOS system.
> Also remember to change the boot order to boot through your USB device.

---

## Pre installation
I'm presuming that you're already in the Arch Linux zsh shell prompt.

### 1. Check boot mode
To check if the UEFI mode is enabled, run:

```sh
# ls /sys/firmware/efi/efivars
```

If the directory does not exists, the system may be booted in BIOS.

### 2. Update System Clock
Ensures that the system clock is accurate.
```sh
# timedatectl set-ntp true
```

### 3. Internet Connection
First, test if you alredy have internet connection, so run:
```sh
# ping google.com
```
If you're not connected, follow one of these steps:

#### Connect to Wifi with iwd
1. Check network interface name
   ```sh
   # ip link
   ```
   The response will be something like:
   ```
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: enp2s0f0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
        link/ether 00:11:25:31:69:20 brd ff:ff:ff:ff:ff:ff
    3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT qlen 1000
        link/ether 01:02:03:04:05:06 brd ff:ff:ff:ff:ff:ff
    ```
2. Run iwd
   ```sh
   # iwctl
   ```
   if $interface is off, then `quit` iwd and run:
   ```sh
   # rfkill unblock all
   # ip link set $interface up
   ```
3. Connect to wifi SSID
  
   run `device list` to list all network interface and run the following commands:
   ```sh
   # station $interface scan
   # station $interface get-networks
   # station $interface connect SSID
   ```
---

#### Wired Connection
**Warning:** Make sure the DHCP is deactivated by running `systemctl stop dhcpcd.service`

1. Find the network interface name

    ```sh
    # ip link
    ```

    The response will be something like:
    ```
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: enp2s0f0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
        link/ether 00:11:25:31:69:20 brd ff:ff:ff:ff:ff:ff
    3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT qlen 1000
        link/ether 01:02:03:04:05:06 brd ff:ff:ff:ff:ff:ff
    ```

1. Activate Network interface

    Using the `enp2s0f0` for example:
    ```sh
    # ip link set enp2s0f0 up
    ```

1. Add IP addresses

    The command to do that is `ip addr add [ip_address]/[mask] dev [interface]` applying to our example:
    ```sh
    # ip addr add 192.168.1.2/24 dev enp2s0f0
    ```

1. Add the Gateway

    The command is `ip route add default via [gateway]` then:
    ```sh
    # ip route add default via 192.168.1.1
    ```

1. Change DNS

    Using the Google DNS, open the file `/etc/resolv.conf` (you can use `nano` or `vi` to do that) and write down these lines:
    ```sh
    nameserver 1.1.1.1
    nameserver 8.8.8.8
    nameserver 8.8.4.4
    ```

After that, test your internet connection again with the `ping` command.

### Partitioning

First, define your partitions size. There's no rules about this process.

> Tip: If you use a SSD drive, it's recommended to leave 25% of his storage free. More info [here](https://wiki.archlinux.org/index.php/Solid_State_Drives#TRIM).

My SSD has 256GB of storage. I want to have Dual-Boot with Windows10. If Windows was installed first, then you could see it's partitions. For that example, I have 4 Windows partitions already created:
(in my case, I'll work with `/dev/nvme0n1` disk. Use `fdisk -l /dev/nvme0n1` to list partitions)

| Name |      Size       | Type |
| :--: | :-------------: | :--: |
| nvme0n1p1 | 499M       | Windows recovery environment  |
| nvme0n1p2 | 100M       | EFI System |
| nvme0n1p3 | 16M        | Microsoft Reserved |
| nvme0n1p3 | 97.1       | Microsoft Basic Data |

> 100GB were allocated for Windows in total

EFI partition was created by Windows, so we don't need to care about it. We need to create additional partitions for Linux installation.

| Name |    Mount     |      Size       | Type |
| :--: | :----------: | :-------------: | :--: |
| nvme0n1p5 | `swap`  | 1G       | Linux Swap  |
| nvme0n1p6 | `/`     | 32G      | Linux Root x86-64 (Ext4) |
| nvme0n1p7 | `/home` | Remaining Space        | Linux Home (Ext4) |

Look at partitioning layout examples: https://wiki.archlinux.org/index.php/partitioning#Example_layouts

#### Create Partitions

Use [fdisk](https://wiki.archlinux.org/index.php/Fdisk) to create partitions.

To create partitions, I'll use `gdisk` since to work on UEFI mode we need GPT partitions.

First, list partitions (Informational only) with the following command
```sh
# fdisk -l /dev/nvme0n1
```

Here's a table with some handy gdisk commands

| Command | Description            |
| :-----: | ---------------------- |
| p       | Print partitions table |
| n       | Add a new partition       |
| d       | Delete a partition       |
| w       | Write table to disk and exit        |
| l       | List known partition types                   |
| t       | Change a partition type                   |
| m       | Help                   |

1. Enter in the interactive menu
    ```sh
    # fdisk /dev/nvme0n1
    ```

1. Create boot partition (If not Dual-Boot)
    - Type `n` to create a new partition
    - Partition Number: default (return)
    - First Sector: default
    - Last Sector: `+512M`
    - Type: `1` - EFI System

1. Create root partition
    - Type `n` to create a new partition
    - Partition Number: default
    - First Sector: default
    - Last Sector: `+32G`
    - Type: `24` - Linux Root (x86-64)

1. Create swap partition
    - Type `n` to create a new partition
    - Partition Number: default
    - First Sector: default
    - Last Sector: `+1G`
    - Type: `19` - Linux Swap

1. Create home partition
    - Type `n` to create a new partition
    - Partition Number: default
    - First Sector: default
    - Last Sector: default
    - Type: `28` - Linux Home

1. Save changes with `w`

#### Format partitions
Once the partitions have been created, each (except swap) should be formatted with an appropriated file system. So run:

```sh
# mkfs.ext4 /dev/nvme0n1p6              #-- root partition
# mkfs.ext4 /dev/nvme0n1p7              #-- home partition
```

If not Dual Boot format partition for EFI boot
```sh
# mkfs.fat -F32 -n BOOT /dev/nvme0n1p2  #-- boot partition
```

The process for swap partition is slight different:
```sh
# mkswap -L swap /dev/nvme0n1p5
# swapon /dev/nvme0n1p5
```

> To check if the swap partition is working, run `swapon -s` or `free -h`.

#### Mount file system
1. Mount root partition:
    ```sh
    # mount /dev/nvme0n1p6 /mnt
    ```

1. Mount home partition:
    ```sh
    # mkdir -p /mnt/home
    # mount /dev/nvme0n1p6 /mnt/home
    ```

1. Mount boot partition: (to use `grub-install` later)
    ```sh
    # mkdir -p /mnt/boot
    # mount /dev/nvme0n1p2 /mnt/boot
    ```

---

## Installation
Now we'll install arch on disk

### Select Mirror
Before installation, is recommended to select the best mirror servers.
So open the file `/etc/pacman.d/mirrorlist` (again, you can use `nano` or `vi` to do that) and move the best mirror to the top of the file.

> **Tip**: That [link](https://www.archlinux.org/mirrorlist/) generates a mirror list based on your location, you can use them as reference.

### Install Base Packages
Now that the mirrors are already set, use `pacstrap` to install the base package group:
```sh
# pacstrap /mnt base base-devel linux linux-firmware
```

### Generate fstab
Now you should generate the fstab with the `genfstab` script:
```sh
# genfstab -p /mnt >> /mnt/etc/fstab
```

> Optional: You can add `noatime,commit=60,barrier=0` to the generated `fstab` file (on root and home partitions) to increase IO performance. https://wiki.archlinux.org/index.php/ext4#Improving_performance

### Chroot
Now, we'll change root into the new system
```sh
# arch-chroot /mnt
```

### Check pacman keys
```sh
# pacman-key --init
# pacman-key --populate archlinux
```

---

## Configure System

### Install basic packages
```sh
pacman -S neovim man-db man-pages texinfo elinks
```

> Now, if you want to install some additional package, do it with `pacman -S <package_name>`

### Timezone

Follow: https://wiki.archlinux.org/index.php/Installation_guide#Time_zone

### Locale and Language

Follow: https://wiki.archlinux.org/index.php/Installation_guide#Localization

### Network
```sh
# echo myhostname > /etc/hostname
```

> Change `myhostname` to your hostname (Computer Name)

After that, open the file `/etc/hosts` and write (remember to change the `myhostname` to your own)

```
# IPv4 Hosts
127.0.0.1	localhost myhostname

# Machine FQDN
127.0.1.1	myhostname.localdomain	myhostname

# IPv6 Hosts
::1		localhost	ip6-localhost	ip6-loopback
ff02::1 	ip6-allnodes
ff02::2 	ip6-allrouters
```

#### IPTables
Write to file `/etc/modules-load.d/firewall.conf`:
```
# iptables modules to run on boot
ip_tables
#Enable conntrack only if NAT used
#nf_conntrack_netbios_ns
#nf_conntrack
```

### Blacklists
**Warning**: this part is optional.

#### No Beep
To avoid the beep on boot, Write to file `/etc/modprobe.d/nobeep.conf`:
```
# Dont run pcpkr module on boot
blacklist pcspkr
```

#### No watchdog
If you don't want a watchdog service running, write to file `/etc/modprobe.d/nowatchdog.conf`
```
blacklist iTCO_wdt
```

### Initramfs

https://wiki.archlinux.org/index.php/Installation_guide#Initramfs

> Creating a new initramfs is usually not required, because mkinitcpio was run on installation of the kernel package with pacstrap.

```sh
# mkinitcpio -p linux
```

### Bootloader

Using [systemd-boot](https://wiki.archlinux.org/index.php/systemd-boot) to install EFI boot manager:
```sh
# bootctl --path=/boot install
```

Then add following content to `/boot/loader/entries/arch.conf`
```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=UUID=ROOT_PART_UUID rw
```

`ROOT_PART_UUID` must be replaced with UUID found using `blkid` or `lsblk -f` (More here: https://wiki.archlinux.org/index.php/Persistent_block_device_naming)

and following content to `/boot/loader/loader.conf`
```
timeout 5
default arch
```

### Root password
```sh
# passwd
```

### Set-up network manager

https://wiki.archlinux.org/index.php/Network_configuration

Select network manager: https://wiki.archlinux.org/index.php/Network_configuration#Network_managers

### Using systemd-networkd + systemd-resolved

Install required packages with `pacman`
```sh
# pacman -S wpa_supplicant
```

See: https://bbs.archlinux.org/viewtopic.php?pid=1393759#p1393759

#### Setup systemd-resolved

Follow: https://wiki.archlinux.org/index.php/Systemd-resolved

#### Setup systemd-networkd

This assumes that your NIC is `wlp2s0`, your SSID is `MyNetwork`, and the password is `SuperSecretPassphrase`.

You need to create a `wpa_supplicant-wlp2s0.conf`.  So use `wpa_passphrase` to generate one:
```sh
# set +o history
# wpa_passphrase MyNetwork SuperSecretPassphrase > /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
# set -o history
```

Enable it so that it runs on boot:
```sh
# systemctl enable wpa_supplicant@wlp2s0
```

Now make networkd configuration files.

File `/etc/systemd/network/20-wired.network`:
```
[Match]
Name=enp1s0

[Network]
DHCP=yes

[DHCP]
RouteMetric=10
```

File `/etc/systemd/network/25-wireless.network`:
```
[Match]
Name=wlp2s0

[Network]
DHCP=yes

[DHCP]
RouteMetric=20
```

Now ensure that `systemd-networkd.service` is enabled.
```sh
# systemctl enable systemd-networkd.service
```

It should be working after reboot.


https://wiki.archlinux.org/index.php/Systemd-networkd#Wired_and_wireless_adapters_on_the_same_machine

https://bbs.archlinux.org/viewtopic.php?pid=1393759#p1393759


### Reboot
Exit chroot environment by pressing `Ctrl + D` or typing `exit`

Unmount system mount points:
```sh
# umount -R /mnt
```

Reboot system:
```sh
# reboot
```

> Remember to remove USB stick on reboot

## Window Systems

There are basically three layers that can be included in the Linux desktop:

**X Windows** – This is the foundation that allows for graphic elements to be drawn on the display. [X Windows](http://en.wikipedia.org/wiki/X_Windows) builds the primitive framework that allows moving of windows, interactions with keyboard and mouse, and draws windows. This is required for any graphical desktop.

**Window Manager** – The Window Manager is the piece of the puzzle that controls the placement and appearance of windows. Window Managers include: Enlightenment, Afterstep, FVWM, Fluxbox, IceWM, etc. Requires X Windows but not a desktop environment.

**Desktop Environment** – This is where it begins to get a little fuzzy for some. A Desktop Environment includes a Window Manager but builds upon it. The Desktop Environment typically is a far more fully integrated system than a Window Manager. Requires both X Windows and a Window Manager. Examples of desktop environments are GNOME, KDE, Cinnamon, Xfce among others

See: https://askubuntu.com/questions/18078/what-is-the-difference-between-a-desktop-environment-and-a-window-manager
See: https://www.ghacks.net/2008/12/09/get-to-know-linux-desktop-environment-vs-window-manager/

### Xorg
Install Xorg Server: (use default options)
```sh
# pacman -S xorg-server
```

Define your keyboard layout on `/etc/X11/xorg.conf.d/10-keyboard.conf` file:
```
Section "InputClass"
	Identifier "keyboard default"
	MatchIsKeyboard "yes"
	Option  "XkbLayout" "br"
	Option  "XkbVariant" "abnt2"
EndSection
```

### Video
Install your GPU driver
```sh
# pacman -S xf86-video-vesa
```

### Audio
Install audio driver
```sh
# pacman -S alsa-utils
```

Configure and save:
```sh
# alsamixer
# alsactl store
```

### Users
Install sudo package
```sh
# pacman -S sudo
```

Configure sudo (uses `vim` as default editor) by running `visudo` and uncommenting the line:
```
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL
```

Now we're going to add a new user by running: (change `myuser` to your username)
```sh
# useradd -m -G wheel -s /bin/zsh myuser
```

Change the new user passord:
```sh
# passwd myuser
```


---

## Post Installation
Now you're on your successfull Arch Linux installation.

Login with your user and follow the next steps.


Now We're gonna install the Window Manager.


I'll show the steps to install [Gnome](https://www.gnome.org/).

First of all, run the installation command with `pacman`:
```sh
$ sudo pacman -S gnome gnome-extra
```

When the installation finishes, enable `gdm` to be started with system on boot:
```sh
$ sudo systemctl enable gdm.service
```

### Network Manager and services
Now we'll remove the previously enabled service from `netctl` and the `wifi-menu` settings.

First ensures that the NetworkManager package is installed:
```sh
$ sudo pacman -S networkmanager
```

Enable and start NetworkManager service:
```sh
$ sudo systemctl enable NetworkManager.service
$ sudo systemctl start NetworkManager.service
```

Go to `/etc/netctl` folder and see the connection files (the ones that starts with something like `wlp1s0...`)

Disable the netctl service that you've been enable previously:
```sh
$ sudo netctl diable wlp1s0-MyWiFi
```

Then, remove all `/etc/netctl` folder and remove your connection file (the one that starts with something like `wlp1s0...`)
```sh
$ sudo rm wlp1s0...  #-- replace with you wifi connection file
```

Now you can reboot the system (by running `reboot`) and everyting should be working fine.

---

## Extras
### Set-up TTF Fonts
Follow [this tutorial](https://gist.github.com/cryzed/e002e7057435f02cc7894b9e748c5671)

### Bluetooth Headphone
To connect the headphone:

1. Install required packages:
    ```sh
    $ sudo pacman -S pulseaudio pulseaudio-bluetooth pavucontrol bluez-utils
    ```
1. Edit `/etc/pulse/system.pa` and add:
    ```sh
    load-module module-bluez5-device
    load-module module-bluez5-discover
    ```
1. For GNOME users:
    ```sh
    $ sudo mkdir -p ~gdm/.config/systemd/user
    $ ln -s /dev/null ~gdm/.config/systemd/user/pulseaudio.socket
    ```
1. Connect to bluetooth device
    ```sh
    $ bluetoothctl
    # power on
    # agent on
    # default-agent
    # scan on
    # pair HEADPHONE_MAC
    # trust HEADPHONE_MAC
    # connect HEADPHONE_MAC
    # quit
    ```
To auto switch to A2DP mode:

1. Edit `/etc/pulse/default.pa` and add:
    ```
    .ifexists module-bluetooth-discover.so
    load-module module-bluetooth-discover
    load-module module-switch-on-connect  # Add this line
    .endif
    ```
1. Modify (or create) `/etc/bluetooth/audio.conf` to auto select AD2P profile:
    ```
    [General]
    Disable=Headset
    ```
1. Reboot PC to apply changes


## References
- https://wiki.archlinux.org/index.php/installation_guide
- https://forum.archlinux-br.org/viewtopic.php?id=4453
- https://itsfoss.com/install-arch-linux/
- https://www.ostechnix.com/install-arch-linux-latest-version/
- https://github.com/jieverson/dotfiles/wiki/arch-linux-for-dummies
- http://ticki.github.io/blog/setting-up-archlinux-on-a-lenovo-yoga/

