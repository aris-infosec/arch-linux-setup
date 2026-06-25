# Arch Linux — Complete Installation & Hyprland Setup Guide
Complete Arch Linux installation guide with Hyprland, Waybar, and full desktop setup

> **Target audience:** Users who want a clean, minimal, and fully customized Arch Linux installation with the Hyprland Wayland compositor and a polished Waybar status bar.
>
> This guide covers everything from booting the live ISO to a fully configured desktop environment with media controls, audio, notifications, and a custom status bar.

---

## Table of Contents

1. [Pre-Installation Checklist & ISO Verification](#1-pre-installation-checklist)
2. [Boot into Live Environment](#2-boot-into-live-environment)
3. [Connect to the Internet](#3-connect-to-the-internet)
4. [Optional: Remote Setup via SSH](#4-optional-remote-setup-via-ssh)
5. [Sync System Clock](#5-sync-system-clock)
6. [Verify Boot Mode (UEFI vs BIOS)](#6-verify-boot-mode-uefi-vs-bios)
7. [Partition the Disk](#7-partition-the-disk)
8. [Format the Partitions](#8-format-the-partitions)
9. [Mount the Partitions](#9-mount-the-partitions)
10. [Configure Pacman Mirrors](#10-configure-pacman-mirrors)
11. [Install the Base System](#11-install-the-base-system)
12. [Generate the Filesystem Table (fstab)](#12-generate-the-filesystem-table-fstab)
13. [Chroot into the New System](#13-chroot-into-the-new-system)
14. [Configure Time Zone](#14-configure-time-zone)
15. [Set Up Localization](#15-set-up-localization)
16. [Set the Hostname](#16-set-the-hostname)
17. [Enable Services & Set Passwords](#17-enable-services--set-passwords)
18. [Create a User Account](#18-create-a-user-account)
19. [Enable sudo for the User](#19-enable-sudo-for-the-user)
20. [Install and Configure the Bootloader (GRUB)](#20-install-and-configure-the-bootloader-grub)
21. [Install GNOME (Optional Desktop)](#21-install-gnome-optional-desktop)
22. [Reboot and First Login](#22-reboot-and-first-login)
23. [Post-Boot Setup](#23-post-boot-setup)
24. [Install Hyprland](#24-install-hyprland)
25. [Install Essential Hyprland Packages](#25-install-essential-hyprland-packages)
26. [Configure Hyprland](#26-configure-hyprland)
27. [Configure Waybar](#27-configure-waybar)
28. [Advanced Waybar: Custom Media Widgets](#28-advanced-waybar-custom-media-widgets)

---

## 1. Pre-Installation Checklist

Before you begin, make sure you have:

- [ ] Downloaded the latest Arch Linux ISO from [archlinux.org](https://archlinux.org/download/)
- [ ] Verified the ISO integrity and authenticity (see below)
- [ ] Written the ISO to a USB drive (use `dd`, Rufus, or Balena Etcher)
- [ ] Backed up all important data on the target machine
- [ ] A working internet connection (Ethernet is easiest; Wi-Fi instructions are included)
- [ ] Disabled Secure Boot in your UEFI/BIOS settings
- [ ] Noted your drive identifier (usually `/dev/nvme0n1` for NVMe or `/dev/sda` for SATA)

> **Note:** All commands in this guide assume an NVMe drive at `/dev/nvme0n1`. Substitute with your actual drive name where needed.

### Verify ISO Integrity (Recommended)

From the Arch Linux download page, also download:
- The `sha256sums.txt` file
- The `.sig` PGP signature file

**Step 1 — Verify the checksum** (confirms the file downloaded without corruption):

```bash
# On Linux/macOS (coreutils required; on macOS: brew install coreutils)
sha256sum -c sha256sums.txt
# Expected output: archlinux-*.iso: OK
```

**Step 2 — Verify the PGP signature** (confirms the ISO is authentic and signed by Arch):

```bash
# Install GnuPG if needed (macOS: brew install gnupg)
# Download the signing key (copy exact command from the Arch download page)
gpg --keyserver-options auto-key-retrieve --verify archlinux-*.iso.sig

# What you want to see:
# Good signature from "Pierre Schmitz <pierre@archlinux.org>"
# (The WARNING about the key not being certified is normal — ignore it)
```

> If the checksum or signature fails, delete the ISO and re-download from a different mirror, then verify again before flashing.

---

## 2. Boot into Live Environment

1. Insert your USB and reboot the machine.
2. Access the UEFI/BIOS boot menu (usually `F2`, `F12`, `Del`, or `Esc` on startup).
3. Select your USB drive and boot into the Arch Linux live environment.

You will land at a root shell prompt:

```
root@archiso ~ #
```

**Tip: Increase the font size** if the text is too small on your screen:

```bash
setfont ter-132b
```

**Prompt explanation:**
- `root` = logged-in as superuser
- `archiso` = hostname of the live environment
- `~` = current directory (home)
- `#` = root privilege indicator

**Useful shell shortcuts:**

| Shortcut | Action |
|---|---|
| `Ctrl + C` | Cancel / kill current command |
| `Ctrl + D` | Exit shell / logout |
| `clear` | Clear the terminal screen |

**If you need a German keyboard layout:**

```bash
loadkeys de
```

---

## 3. Connect to the Internet

### Ethernet
If using a wired connection, you are likely already online. Verify with:

```bash
ping archlinux.org
```

Press `Ctrl + C` to stop the ping.

### Wi-Fi (via iwctl)

```bash
iwctl
```

Inside the `iwctl` prompt:

```
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "YOUR_WIFI_NAME"
exit
```

You will be prompted for the Wi-Fi password. After connecting, verify:

```bash
ping archlinux.org
```

---

## 4. Optional: Remote Setup via SSH

Working from another machine via SSH can make copy-pasting commands much easier.

```bash
# Find your live environment's IP address
ip addr show
# Look under wlan0 or eth0 for the inet address (e.g. 192.168.1.87)

# Set a root password for the SSH session
passwd

# Check if SSH daemon is running
systemctl status sshd

# Enable and start if not already running
systemctl enable --now sshd
```

Now on your **other machine**, connect via:

```bash
ssh root@<YOUR_IP_ADDRESS>
```

---

## 5. Sync System Clock

Arch requires an accurate system clock for package signing and time-sensitive operations.

```bash
timedatectl status
```

If `NTP service: inactive` or `Synchronized: no`, enable it:

```bash
timedatectl set-ntp true
timedatectl status
```

Confirm `Synchronized: yes` before continuing.

---

## 6. Verify Boot Mode (UEFI vs BIOS)

```bash
cat /sys/firmware/efi/fw_platform_size
```

| Output | Meaning |
|---|---|
| `64` | 64-bit UEFI — use GPT partition table |
| `32` | 32-bit UEFI — use GPT partition table |
| `No such file or directory` | Legacy BIOS — use MBR partition table |

> This guide is written for **UEFI (GPT)**. If you have BIOS, the partition layout and GRUB installation commands will differ.

---

## 7. Partition the Disk

### View available disks

```bash
fdisk -l
```

Identify your target drive (e.g. `/dev/nvme0n1`).

### Recommended partition layout (UEFI/GPT)

| Partition | Size | Type | Mount Point |
|---|---|---|---|
| `/dev/nvme0n1p1` | 1 GB | EFI System | `/boot` |
| `/dev/nvme0n1p2` | 75 GB | Linux filesystem | `/` (root) |
| `/dev/nvme0n1p3` | 4 GB | Linux swap | swap |
| `/dev/nvme0n1p4` | Remaining | Linux filesystem | `/home` |

> **Swap size:** The Arch Wiki recommends **4 GB** as a sensible default for modern systems. The old rule of "2× your RAM" is outdated. Adjust up if you regularly run memory-heavy workloads or use hibernation.
>
> **Root size:** The Arch Wiki recommends at least 32 GB. 75 GB gives comfortable headroom for packages and larger tools. Getting this right now is easier than resizing later.

### Partition with cfdisk

```bash
cfdisk /dev/nvme0n1
```

**Steps inside cfdisk:**

1. Select `gpt` as the partition table type if prompted.
2. Use arrow keys to select any existing partitions and press `d` to **delete** them until all space is free.
3. Select `[ New ]` to create each partition:
   - Enter the size (e.g. `1G`, `75G`, `8G`, leave blank for the rest)
   - For p1: press `t` to change type → select `EFI System`
   - For p3: press `t` to change type → select `Linux swap`
   - p2 and p4 stay as `Linux filesystem`
4. Once all partitions are created, select `[ Write ]`, type `yes` to confirm.
5. Select `[ Quit ]`.

Verify your new layout:

```bash
fdisk -l
```

---

## 8. Format the Partitions

```bash
# Format root partition as ext4
mkfs.ext4 /dev/nvme0n1p2

# Format home partition as ext4
mkfs.ext4 /dev/nvme0n1p4

# Initialize swap
mkswap /dev/nvme0n1p3

# Format EFI partition as FAT32
mkfs.fat -F 32 /dev/nvme0n1p1
```

> Type `y` or `yes` if prompted to confirm overwriting existing filesystems.

---

## 9. Mount the Partitions

Mount partitions in the correct order — root first, then the others.

```bash
# Mount root partition
mount /dev/nvme0n1p2 /mnt

# Create and mount home directory
mkdir /mnt/home
mount /dev/nvme0n1p4 /mnt/home

# Create and mount boot directory
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot

# Enable swap
swapon /dev/nvme0n1p3
```

Verify everything is mounted correctly:

```bash
lsblk
```

You should see your partitions mounted at `/mnt`, `/mnt/home`, `/mnt/boot`, and `[SWAP]`.

---

## 10. Configure Pacman Mirrors

Pacman is Arch's package manager. Choosing fast, nearby mirrors speeds up the installation significantly.

```bash
# Preview current mirror list
less /etc/pacman.d/mirrorlist
# Press q to exit

# Automatically select the 5 fastest mirrors in Germany via HTTPS
reflector --latest 5 --country Germany --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist

# Verify the result
cat /etc/pacman.d/mirrorlist
```

> Change `--country Germany` to your own country for best results.

---

## 11. Install the Base System

This installs the Linux kernel, firmware, base utilities, and an Intel CPU microcode update. Adjust for your CPU:

```bash
pacstrap -K /mnt base linux linux-firmware networkmanager vim vi base-devel intel-ucode
```

**Package explanation:**

| Package | Purpose |
|---|---|
| `base` | Minimal Arch Linux base |
| `linux` | The Linux kernel |
| `linux-firmware` | Firmware blobs for hardware |
| `networkmanager` | Network management tool |
| `vim` / `vi` | Text editors |
| `base-devel` | Build tools (needed for AUR later) |
| `intel-ucode` | Intel CPU microcode updates (use `amd-ucode` for AMD) |

This step will take several minutes depending on your internet speed.

---

## 12. Generate the Filesystem Table (fstab)

fstab tells the system which partitions to mount at boot and where.

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Review the generated file to ensure all partitions are listed:

```bash
cat /mnt/etc/fstab
```

You should see entries for `/`, `/home`, `/boot`, and `swap`.

---

## 13. Chroot into the New System

Change root into your newly installed system to configure it from the inside:

```bash
arch-chroot /mnt
```

Your prompt changes — you are now working inside the installed system.

---

## 14. Configure Time Zone

```bash
# Set your timezone (adjust Region/City as needed)
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime

# Sync hardware clock to system clock
hwclock --systohc
```

List available timezones with `timedatectl list-timezones` if unsure.

---

## 15. Set Up Localization

### Generate locales

```bash
vim /etc/locale.gen
```

Inside vim, search for your locale:

```
/de_DE
```

Find the line `#de_DE.UTF-8 UTF-8` and remove the `#` to uncomment it (press `x` with cursor on the `#`).

Save and exit:

```
:wq
```

Generate the locale:

```bash
locale-gen
```

### Set the system language

```bash
# Option A — quick one-liner
echo "LANG=de_DE.UTF-8" > /etc/locale.conf

# Option B — explicit editor method (same result)
vim /etc/locale.conf
# Press i, type: LANG=de_DE.UTF-8
# Press Esc, then :wq

export LANG=de_DE.UTF-8
```

Verify it was set correctly:

```bash
cat /etc/locale.conf
# Should output: LANG=de_DE.UTF-8
```

> For English, use `en_US.UTF-8` instead.

### Set the keyboard layout permanently

```bash
echo "KEYMAP=de" > /etc/vconsole.conf
```

---

## 16. Set the Hostname

The hostname is your machine's name on the network.

```bash
vim /etc/hostname
```

Press `i` to enter insert mode, type your desired hostname (e.g. `arch`), press `Esc`, then save:

```
:wq
```

---

## 17. Enable Services & Set Passwords

Enable NetworkManager to start at boot:

```bash
systemctl enable NetworkManager
```

Set the root password:

```bash
passwd
```

---

## 18. Create a User Account

Always create a regular user — running your desktop as root is a security risk.

```bash
useradd -m -G wheel,users yourusername
passwd yourusername
```

Replace `yourusername` with your actual username.

---

## 19. Enable sudo for the User

Members of the `wheel` group need permission to use `sudo`.

```bash
visudo
```

Inside visudo, search for:

```
/wheel
```

Find this line and uncomment it by removing the `#`:

```
# %wheel ALL=(ALL:ALL) ALL
```

Should become:

```
%wheel ALL=(ALL:ALL) ALL
```

Save and exit:

```
:wq
```

---

## 20. Install and Configure the Bootloader (GRUB)

```bash
# Install GRUB and EFI tools
pacman -S grub efibootmgr

# Install GRUB to the EFI partition
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# Verify boot files exist
ls /boot

# Generate GRUB configuration
grub-mkconfig -o /boot/grub/grub.cfg
```

Install additional useful utilities:

```bash
pacman -S git man-db man-pages reflector
```

---

## 21. Install GNOME (Optional Desktop)

If you want a full graphical desktop environment as a fallback (recommended for new users), install GNOME:

```bash
pacman -S gnome
```

Press Enter to accept defaults when prompted.

Enable the display manager:

```bash
systemctl enable gdm
```

> You can skip this if you plan to use **only Hyprland** — set up a minimal display manager later (e.g. Ly).

---

## 22. Reboot and First Login

Exit the chroot, unmount all partitions, and reboot:

```bash
exit
umount -R /mnt
reboot
```

**Remove your USB drive** when the machine restarts.

### First boot troubleshooting

If you land in a TTY (text console) instead of a graphical login:

```bash
# Switch to TTY2
Ctrl + Alt + F2

# Log in as root
root

# Set keyboard layout
loadkeys de

# Reset passwords if needed
passwd root
passwd yourusername

# Set X11 keyboard layout for GUI
localectl set-x11-keymap de
localectl set-keymap de

reboot
```

Enable internet after boot:

```bash
sudo timedatectl set-ntp true
```

---

## 23. Post-Boot Setup

### Update all packages

```bash
sudo pacman -Syu
```

### Enable Bluetooth

```bash
sudo pacman -S bluez bluez-utils bluez-deprecated-tools
sudo systemctl enable --now bluetooth
```

---

## 24. Install Hyprland

Hyprland is a dynamic tiling Wayland compositor. It requires `yay` (AUR helper) for installation.

### Connect to Wi-Fi (if needed)

```bash
nmcli device wifi connect "WIFI_NAME" password "WIFI_PASSWORD"
```

### Install yay (AUR Helper)

```bash
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
cd ~
```

### Install Hyprland and a Terminal

```bash
yay -S hyprland kitty
```

### Install Fonts

```bash
yay -S noto-fonts noto-fonts-cjk noto-fonts-emoji
fc-cache -f
```

### Start Hyprland (test)

```bash
Hyprland
```

### Install a Display Manager (Ly — recommended over GDM for Hyprland)

```bash
yay -S ly

# Disable GDM if you installed GNOME earlier
sudo systemctl disable gdm

# Enable Ly
sudo systemctl enable ly@tty2

reboot
```

At the login screen: select **Wayland** and choose **Hyprland**.

---

## 25. Install Essential Hyprland Packages

```bash
# Notifications
yay -S mako

# Audio (PipeWire stack)
yay -S pipewire wireplumber pipewire-audio pipewire-pulse pipewire-alsa pipewire-jack pavucontrol

# Desktop portal (screen sharing, file picker)
yay -S xdg-desktop-portal-hyprland

# Authentication agent (for sudo popups in GUI)
yay -S hyprpolkitagent

# Qt Wayland support
yay -S qt5-wayland qt6-wayland

# Nerd font for icons
yay -S ttf-jetbrains-mono-nerd

# Useful desktop apps
yay -S nemo imv mpv

# App launcher
yay -S rofi

# Media control and brightness
yay -S playerctl brightnessctl

# Dark mode tools
yay -S nwg-look gnome-themes-extra qt6ct

# Chromium browser
yay -S chromium

# Waybar click-action tools (needed for module on-click actions)
yay -S bluetui btop htop nm-connection-editor
```

> **`playerctld`** is included in the `playerctl` package. It acts as a media daemon that tracks the most recently active media player — important for the Waybar media widgets to work correctly across multiple players. It is started via autostart in Hyprland (covered in section 28).

**App reference:**

| App | Purpose |
|---|---|
| `nemo` | File manager |
| `imv` | Image viewer |
| `mpv` | Video player |
| `rofi` | App launcher / command runner |
| `pavucontrol` | Audio mixer GUI |
| `mako` | Notification daemon |
| `playerctl` | Media playback control (CLI) |
| `brightnessctl` | Screen brightness control |

---

## 26. Configure Hyprland

The main config file is at:

```
~/.config/hypr/hyprland.conf
```

> **Note:** Hyprland *should* generate a default config on first launch, but sometimes it doesn't. If you land in a black screen or get errors about a missing config, you need to manually create it. Get the official default from the [Hyprland GitHub](https://github.com/hyprwm/Hyprland) and paste it in:

```bash
mkdir -p ~/.config/hypr
vim ~/.config/hypr/hyprland.conf
# Paste the default config, press Esc, then :wq
# Exit Hyprland with Super+M, then log back in
```

Edit it:

```bash
nvim ~/.config/hypr/hyprland.conf
```

### Essential configuration sections

#### Keyboard layout

```
input {
    kb_layout = de
}
```

#### Monitor scaling

```bash
hyprctl monitors all
```

Example config for a HiDPI display:

```
monitor=DP-1,preferred,auto,2
```

#### Application variables

```lua
$terminal = kitty
$fileManager = nemo
$browser = chromium
$menu = rofi -show drun -show-icons
```

#### Recommended keybinds

```
bind = SUPER, T, exec, $terminal
bind = SUPER, Q, killactive
bind = SUPER, F, exec, $fileManager
bind = SUPER, B, exec, $browser
bind = SUPER, SPACE, exec, $menu
bind = SUPER SHIFT, SPACE, exec, rofi -show run
bind = SUPER SHIFT, Q, exit

# Window management
bind = SUPER, H, movefocus, l
bind = SUPER, L, movefocus, r
bind = SUPER, K, movefocus, u
bind = SUPER, J, movefocus, d

bind = SUPER SHIFT, H, movewindow, l
bind = SUPER SHIFT, L, movewindow, r
bind = SUPER SHIFT, K, movewindow, u
bind = SUPER SHIFT, J, movewindow, d

bind = SUPER SHIFT, T, togglefloating
bind = SUPER SHIFT, F, fullscreen

# Workspaces
bind = SUPER, 1, workspace, 1
bind = SUPER, 2, workspace, 2
bind = SUPER, 3, workspace, 3
bind = SUPER, 4, workspace, 4
bind = SUPER, 5, workspace, 5

bind = SUPER SHIFT, 1, movetoworkspace, 1
bind = SUPER SHIFT, 2, movetoworkspace, 2

# Media keys
bind = , XF86AudioPlay, exec, playerctl play-pause
bind = , XF86AudioPrev, exec, playerctl previous
bind = , XF86AudioNext, exec, playerctl next
bind = , XF86AudioRaiseVolume, exec, wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%+
bind = , XF86AudioLowerVolume, exec, wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-
bind = , XF86AudioMute, exec, wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle
bind = , XF86MonBrightnessUp, exec, brightnessctl set +5%
bind = , XF86MonBrightnessDown, exec, brightnessctl set 5%-
```

#### Window appearance

```
general {
    gaps_in = 10
    gaps_out = 20
    border_size = 3
    col.active_border = rgba(7aa2f7ff)
    col.inactive_border = rgba(333333aa)
    layout = dwindle
}

decoration {
    rounding = 8
    inactive_opacity = 0.85
    blur {
        enabled = false
    }
}
```

#### Input settings

```
input {
    repeat_rate = 25
    repeat_delay = 300
    touchpad {
        scroll_factor = 0.2
    }
}
```

#### Autostart

```
exec-once = waybar
exec-once = mako
exec-once = hyprpolkitagent
exec-once = playerctld daemon
exec-once = hyprctl setcursor Bibata-Modern-Ice 30
```

### Install cursor theme

```bash
yay -S bibata-cursor-theme
```

### Enable dark mode

```bash
nwg-look
# Choose: Adwaita Dark, Color scheme: Prefer Dark

qt6ct
# Select a dark theme
```

Add to Hyprland config environment:

```
env = QT_QPA_PLATFORMTHEME,qt6ct
```

### Set up Rofi

```bash
mkdir -p ~/.config/rofi
rofi -dump-config > ~/.config/rofi/config.rasi
```

### Install Rofi themes (Spotlight Dark)

The default Rofi look is plain. Install a proper theme set:

```bash
# Make sure git is installed
sudo pacman -S git

# Clone the newt theme repository (link from the Rofi themes community)
git clone https://github.com/newt-sc/a-newt-rofi-theme.git
cd a-newt-rofi-theme

# Create the themes directory and copy the spotlight themes
mkdir -p ~/.local/share/rofi/themes
cp ./themes/spotlight* ~/.local/share/rofi/themes/
cd ~
```

Apply the theme:

1. Open the Rofi runner: `Super + Shift + Space`
2. Type `theme-selector` and press Enter
3. Use arrow keys to browse — select `spotlight-dark`
4. Press `Alt + A` to accept

Now `Super + Space` will open Rofi with the dark spotlight theme.

---

## 27. Configure Waybar

Waybar is the status bar for Hyprland.

```bash
yay -S waybar inotify-tools
mkdir -p ~/.config/waybar
cd ~/.config/waybar
```

### Create the config file

```bash
nvim config.jsonc
```

Paste the following complete config:

```jsonc
{
    "layer": "top",
    "position": "top",
    "height": 40,
    "spacing": 4,

    "modules-left": [
        "hyprland/workspaces",
        "hyprland/submap",
        "group/media"
    ],

    "modules-center": [
        "clock"
    ],

    "modules-right": [
        "group/volume",
        "network",
        "bluetooth",
        "cpu",
        "memory",
        "battery"
    ],

    "hyprland/workspaces": {
        "format": "{id}",
        "on-click": "activate"
    },

    "clock": {
        "format": "{:%H:%M  %d.%m.%Y}",
        "tooltip-format": "<big>{:%Y %B}</big>\n<tt><small>{calendar}</small></tt>",
        "on-click": "xdg-open https://calendar.google.com"
    },

    "battery": {
        "format": "{capacity}% {icon}",
        "format-icons": ["󰁺", "󰁼", "󰁾", "󰂀", "󰂂", "󰁹"],
        "format-charging": "{capacity}% 󰂄",
        "format-full": "󰁹 Full",
        "states": {
            "warning": 30,
            "critical": 15
        }
    },

    "network": {
        "format-wifi": "{essid} ({signalStrength}%) 󰤨",
        "format-ethernet": "Ethernet 󰈀",
        "format-disconnected": "Disconnected 󰤭",
        "on-click": "nm-connection-editor"
    },

    "bluetooth": {
        "format": "󰂯 {status}",
        "format-connected": "󰂱 {device_alias}",
        "on-click": "ghostty -e bluetui"
    },

    "cpu": {
        "format": "CPU {usage}% 󰍛",
        "on-click": "kitty -e btop"
    },

    "memory": {
        "format": "RAM {}% 󰍛",
        "on-click": "kitty -e htop"
    },

    "pulseaudio": {
        "format": "{volume}% {icon}",
        "format-muted": "󰝟 Muted",
        "format-icons": {
            "default": ["󰕿", "󰖀", "󰕾"]
        },
        "on-click": "pavucontrol"
    },

    "pulseaudio/slider": {},

    "group/volume": {
        "orientation": "horizontal",
        "modules": ["pulseaudio", "pulseaudio/slider"]
    },

    "group/media": {
        "orientation": "horizontal",
        "modules": [
            "custom/media-animation",
            "custom/media-now-playing",
            "custom/media-time"
        ]
    },

    "custom/media-animation": {
        "exec": "~/.config/waybar/custom_modules/media/media-animation.sh",
        "interval": "once",
        "signal": 5
    },

    "custom/media-now-playing": {
        "exec": "~/.config/waybar/custom_modules/media/media-now-playing.sh",
        "interval": 3,
        "on-click": "playerctl play-pause",
        "on-click-middle": "playerctl previous",
        "on-click-right": "playerctl next"
    },

    "custom/media-time": {
        "exec": "~/.config/waybar/custom_modules/media/media-time.sh",
        "interval": 5
    }
}
```

### Create the stylesheet

```bash
nvim ~/.config/waybar/style.css
```

```css
@define-color dark-9 #111111;
@define-color dark-8 #1a1a1a;
@define-color dark-7 #222222;
@define-color highlight #7aa2f7;
@define-color urgent #ff5555;

* {
    font-family: "JetBrainsMono Nerd Font Propo";
    font-size: 14px;
    border: none;
    border-radius: 0;
    min-height: 0;
}

window#waybar {
    background: @dark-9;
    color: white;
}

#workspaces button {
    padding: 3px 8px;
    margin: 3px;
    background: transparent;
    border-radius: 4px;
    color: white;
}

#workspaces button.active {
    background: @highlight;
    color: black;
    font-weight: 700;
}

#workspaces button.urgent {
    background: @urgent;
}

#battery,
#bluetooth,
#network,
#cpu,
#memory,
#clock,
#pulseaudio {
    padding: 5px 10px;
    margin: 5px 3px;
    border-radius: 8px;
    background: @dark-8;
}

#battery.warning {
    color: orange;
}

#battery.critical {
    color: @urgent;
}

#volume {
    padding: 5px 10px;
    margin: 5px 3px;
    border-radius: 8px;
    background: @dark-8;
}

#pulseaudio-slider {
    padding: 0;
    margin: 0;
    margin-left: 8px;
}

#pulseaudio-slider slider {
    min-height: 0;
    min-width: 0;
}

#media {
    color: @highlight;
    margin-left: 80px;
}

#custom-media-now-playing {
    color: @highlight;
    margin-left: 8px;
}

#custom-media-animation {
    font-size: 14px;
    color: @highlight;
}

#custom-media-time {
    color: gray;
    font-size: 12px;
}

tooltip {
    background: @dark-9;
    color: white;
    border-radius: 5px;
    padding: 5px;
}
```

---

## 28. Advanced Waybar: Custom Media Widgets

### Create the media modules directory

```bash
mkdir -p ~/.config/waybar/custom_modules/media
cd ~/.config/waybar/custom_modules/media
```

### media-now-playing.sh

```bash
nvim media-now-playing.sh
```

```bash
#!/bin/bash
playerctl metadata --format '{{ title }} - {{ artist }}' 2>/dev/null | zscroll --length 25 --delay 0.3 2>/dev/null || echo "No media"
```

### media-animation.sh

```bash
nvim media-animation.sh
```

```bash
#!/bin/bash
frames=("◐" "◓" "◑" "◒")
while true; do
    status=$(playerctl status 2>/dev/null)
    if [ "$status" = "Playing" ]; then
        for frame in "${frames[@]}"; do
            echo "$frame"
            sleep 0.2
        done
    else
        echo "⏸"
        sleep 1
    fi
done
```

### media-time.sh

```bash
nvim media-time.sh
```

```bash
#!/bin/bash
position=$(playerctl position 2>/dev/null)
length=$(playerctl metadata mpris:length 2>/dev/null)

if [ -n "$position" ] && [ -n "$length" ]; then
    pos_min=$(echo "$position / 60" | bc)
    pos_sec=$(echo "$position % 60" | bc | awk '{printf "%02d", $1}')
    len_sec=$((length / 1000000))
    len_min=$((len_sec / 60))
    len_sec_r=$((len_sec % 60))
    echo "${pos_min}:${pos_sec} / ${len_min}:$(printf '%02d' $len_sec_r)"
else
    echo ""
fi
```

### Make all scripts executable

```bash
chmod +x ~/.config/waybar/custom_modules/media/*
```

### Install zscroll (for scrolling text)

```bash
yay -S python-setuptools zscroll
```

### Enable automatic Waybar reload on config save

```bash
nvim ~/.config/waybar/auto-reload.sh
```

```bash
#!/bin/bash
while inotifywait -e close_write ~/.config/waybar/config.jsonc ~/.config/waybar/style.css; do
    killall -SIGUSR2 waybar
done
```

```bash
chmod +x ~/.config/waybar/auto-reload.sh
```

### Final Hyprland autostart entries

Add to `~/.config/hypr/hyprland.conf`:

```
exec-once = waybar
exec-once = ~/.config/waybar/auto-reload.sh
exec-once = mako
exec-once = hyprpolkitagent
exec-once = playerctld daemon
exec-once = hyprctl setcursor Bibata-Modern-Ice 30
```

---

## Final Result

After completing this guide, you will have:

- **Arch Linux** installed on bare metal with a proper UEFI/GPT layout
- **Hyprland** Wayland compositor with tiling window management
- **Waybar** status bar with workspace indicators, system stats, audio, media, and clock
- **Rofi** app launcher with keyboard-driven navigation
- **PipeWire** audio stack with volume control
- **mako** desktop notifications
- **Dark mode** across GTK and Qt applications
- **Vim-style navigation** for windows (H/J/K/L)
- **Multimedia key support** for volume and brightness

---

## Troubleshooting Reference

| Problem | Solution |
|---|---|
| No internet after reboot | `nmcli device wifi connect "NAME" password "PASS"` |
| Wrong keyboard layout | `loadkeys de` (TTY) or `localectl set-keymap de` |
| Hyprland won't start | Check logs: `cat /tmp/hyprland-0.log` |
| No audio | `systemctl --user enable --now pipewire pipewire-pulse wireplumber` |
| Screen too small / HiDPI | Set monitor scale in hyprland.conf: `monitor=NAME,preferred,auto,2` |
| Pacman keyring errors | `sudo pacman-key --init && sudo pacman-key --populate archlinux` |

---

## Useful Resources

- [Arch Wiki](https://wiki.archlinux.org/) — the definitive reference for everything Arch
- [Hyprland Wiki](https://wiki.hyprland.org/) — official Hyprland documentation
- [Waybar Wiki](https://github.com/Alexays/Waybar/wiki) — Waybar module reference
- [AUR](https://aur.archlinux.org/) — Arch User Repository

---

*Guide maintained by [your GitHub username]. Contributions and corrections welcome via pull request.*
