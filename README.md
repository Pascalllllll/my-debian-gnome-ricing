# Debian Linux Install & macOS-Style Rice

A start-to-finish log of how this machine was built: a minimal Debian base, GNOME on top, WhiteSur theming for the macOS look, a themed boot sequence, and a development setup. Each step says what it does and what tends to go wrong.

Tested target: **Debian 13 "trixie"** (current stable, ships GNOME 48). Most steps also work on Debian 12, with the differences noted inline.

---

<img width="2560" height="1600" alt="image" src="https://github.com/user-attachments/assets/a1180f15-de98-4fa8-8c0e-c4f478531d6e" />

## Contents
1. [Base system install](#1-base-system-install)
2. [Post-install setup](#2-post-install-setup)
3. [Terminal and shell](#3-terminal-and-shell)
4. [GNOME desktop](#4-gnome-desktop)
5. [macOS look](#5-macos-look)
6. [GRUB and Plymouth](#6-grub-and-plymouth)
7. [Development setup](#7-development-setup)
8. [Undoing things](#8-undoing-things)

---

## 1. Base system install

### Bootable USB
1. Download the **netinst** ISO from [debian.org](https://www.debian.org/distrib/netinst). It is small and pulls current packages during install.
2. Write it to a USB stick with Rufus, balenaEtcher, or on Linux: `sudo dd if=debian-*.iso of=/dev/sdX bs=4M status=progress oflag=sync` (check `sdX` with `lsblk` first; `dd` overwrites whatever you point it at).

### Installer choices
1. Boot from the USB and pick **Graphical install**.
2. Hostname: `deb`
3. **Root password:** leave it **empty**. The installer then locks the root account, installs `sudo`, and adds your user to the `sudo` group for you. If you set a root password, you have to do that yourself (see section 2).
4. User: `pascal`, with a password.
5. Partitioning: *Guided - use entire disk*. Pick the *encrypted LVM* variant if this is a laptop.
6. Software selection: **untick** *Debian desktop environment* and *GNOME*. Keep **standard system utilities** only. Installing GNOME later with `gnome-core` gives a lighter desktop than the installer's full `gnome` task (no games, no LibreOffice, fewer background services).
7. Finish and reboot.

---

## 2. Post-install setup

Log in on the TTY as `pascal`.

### sudo (only if you set a root password)
```bash
su -                      # the "-" loads root's PATH, which is where usermod lives
apt update && apt install -y sudo
usermod -aG sudo pascal
exit
```
Log out and back in; group membership is read at login. Check with `groups`.

### APT sources
Open `/etc/apt/sources.list` (or `/etc/apt/sources.list.d/debian.sources` on trixie) and make sure the components include `main contrib non-free non-free-firmware`. Without `non-free-firmware`, Wi-Fi cards and some GPUs will not get their firmware.

### Base tools
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git unzip tar neovim build-essential fontconfig fastfetch zsh
```
Changes from the original list:
- `journalctl` removed. It is not a package; it ships with `systemd` and is already installed.
- `fontconfig` added. `fc-cache` in the next section needs it, and a no-desktop install may not have it yet.
- `fastfetch` is in trixie. On Debian 12 it is only in `bookworm-backports`.

---

## 3. Terminal and shell

### A. Nerd Font (JetBrains Mono)
Nerd Fonts patch icons into a normal font. Prompt themes like Powerlevel10k draw their separators and icons from these glyphs, so without one the prompt shows boxes.

```bash
mkdir -p ~/.local/share/fonts/JetBrainsMonoNerd
cd /tmp
wget https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip
unzip -o JetBrainsMono.zip -d ~/.local/share/fonts/JetBrainsMonoNerd
fc-cache -f
fc-list : family | grep -i "jetbrainsmono nerd" | sort -u
```
The last command prints the exact family names fontconfig sees. You get three families:

| Family | Use |
|---|---|
| `JetBrainsMono Nerd Font` | General use; icons may overflow one cell |
| `JetBrainsMono Nerd Font Mono` | Terminals; every glyph is exactly one cell wide |
| `JetBrainsMono Nerd Font Propo` | Proportional UI text (also listed as `JetBrainsMono NFP`) |

Use the **Mono** variant in terminals. Set it in your terminal's profile preferences, not only in the shell config; the shell cannot change the font the terminal draws with.

### B. Zsh, Oh My Zsh, Powerlevel10k
```bash
# Install Oh My Zsh. It offers to make zsh your login shell; say yes.
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# If you skipped that prompt:
chsh -s "$(command -v zsh)"

# Powerlevel10k theme
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k"

# Point .zshrc at it
sed -i 's|^ZSH_THEME=.*|ZSH_THEME="powerlevel10k/powerlevel10k"|' ~/.zshrc
exec zsh    # starts the p10k configuration wizard
```
Rerun the wizard any time with `p10k configure`.

Two optional plugins that are packaged in Debian, so they update with the rest of the system:
```bash
sudo apt install -y zsh-autosuggestions zsh-syntax-highlighting
cat >> ~/.zshrc <<'EOF'
source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh   # keep this line last
EOF
```

> Powerlevel10k's author has put the project on limited support. It still works fine. If you want something actively developed, [Starship](https://starship.rs) is the usual replacement and works across shells.

---

## 4. GNOME desktop

```bash
sudo apt install -y gnome-core gnome-tweaks gnome-shell-extension-manager gnome-browser-connector
sudo reboot
```
- `gnome-core` brings GNOME Shell, GDM (the login screen), Nautilus, Settings, and NetworkManager.
- `gnome-shell-extension-manager` is a desktop app for browsing and installing extensions, so you do not need the browser route at all.
- `gnome-browser-connector` replaces the old `chrome-gnome-shell` package, which no longer exists on Debian 12+. Only needed if you want to install extensions from extensions.gnome.org in a browser.
- The **User Themes** extension (needed in section 5) already comes with `gnome-core` through `gnome-shell-extensions`.

**Network shows "unmanaged" after reboot?** The installer wrote your connection to `/etc/network/interfaces`, and NetworkManager ignores interfaces listed there. Comment out everything except the `lo` lines, then:
```bash
sudo systemctl restart NetworkManager
```

---

## 5. macOS look

User themes go in `~/.themes` (GTK and GNOME Shell) and `~/.icons` (icons and cursors). The WhiteSur scripts create and fill these for you. Settings are applied with `gsettings`, which writes to the same dconf keys GNOME Settings and Tweaks use, so the result is identical and you can script it.

### A. Themes, icons, cursor
```bash
cd ~
git clone --depth=1 https://github.com/vinceliuice/WhiteSur-gtk-theme.git
git clone --depth=1 https://github.com/vinceliuice/WhiteSur-icon-theme.git
git clone --depth=1 https://github.com/vinceliuice/McMojave-cursors.git

# GTK 3 + GNOME Shell theme, with the glassy Nautilus sidebar
cd ~/WhiteSur-gtk-theme
./install.sh -N glassy

# GTK 4 / libadwaita apps (Settings, Files, Text Editor...), light variant
./install.sh -l -c light

# Login screen (GDM)
sudo ./tweaks.sh -g

# Icons (-a = alternative icons for some apps)
cd ~/WhiteSur-icon-theme
./install.sh -a

# Cursor
cd ~/McMojave-cursors
./install.sh
```
Corrections to the original commands:
- `-s 220` was removed. In the current script `-s` selects the color scheme (`standard` or `nord`), not a sidebar width, so `220` is rejected.
- The manual `ln -s ... ~/.config/gtk-4.0/...` symlinks are replaced by `./install.sh -l`, which does the same job. The symlinks also failed on a fresh account because `~/.config/gtk-4.0` does not exist yet.

**About the GTK 4 step:** libadwaita does not support themes. `-l` works by copying CSS into `~/.config/gtk-4.0/`, which every GTK 4 app loads as a user override. The result looks right, but the light/dark switch in Settings stops affecting those apps. To switch, rerun `./install.sh -l -c dark`. To undo it, delete `~/.config/gtk-4.0/gtk.css`, `gtk-dark.css`, and `assets`.

Optional extras from the same repo:
```bash
./tweaks.sh -d    # restyle Dash to Dock to match (run after installing the extension)
./tweaks.sh -F    # apply the theme to Flatpak apps
./tweaks.sh -f    # macOS-style Firefox theme
```

### B. Apply with gsettings
Run these inside the GNOME session (not on a TTY or over SSH, where there is no session bus).
```bash
gsettings set org.gnome.desktop.interface gtk-theme      'WhiteSur-Light'
gsettings set org.gnome.desktop.interface icon-theme     'WhiteSur'
gsettings set org.gnome.desktop.interface cursor-theme   'McMojave-cursors'
gsettings set org.gnome.desktop.interface color-scheme   'prefer-light'
gsettings set org.gnome.desktop.wm.preferences button-layout 'close,minimize,maximize:'

# Fonts: a proportional UI font, the Nerd Font for monospace
sudo apt install -y fonts-inter
gsettings set org.gnome.desktop.interface font-name           'Inter 10'
gsettings set org.gnome.desktop.interface monospace-font-name 'JetBrainsMono Nerd Font Mono 10'

# GNOME Shell (top bar, menus, overview) theme
gnome-extensions enable user-theme@gnome-shell-extensions.gcampax.github.com
gsettings set org.gnome.shell.extensions.user-theme name 'WhiteSur-Light'
```
Why the font change: the original set a monospace coding font as the interface font. It works, but menus and labels end up wide and nothing like macOS. Inter is close to San Francisco in shape and is in Debian's repos. The Nerd Font belongs in the monospace slot, which terminals and editors read.

`button-layout 'close,minimize,maximize:'` puts the buttons on the left. The colon separates the left group from the right group.

### C. Top bar (replacing YASB)
The original guide used YASB. **YASB is a Windows-only program** and does not run on Linux. Other Linux bars such as Waybar or Polybar do not fit either: GNOME's compositor does not support the layer-shell protocol that Wayland bars need.

On GNOME, the macOS menu bar is the GNOME top bar itself, restyled:
- The WhiteSur shell theme from step A already makes it translucent and thin, like the macOS bar.
- Rerunning the GTK theme installer with `--shell` plus its sub-options changes the panel height, the top-left icon, and the panel font. Run `./install.sh --help` to see the exact sub-flags in your checkout.
- Use **Just Perfection** to hide the parts macOS does not have (the Activities button, for example) instead of hiding the whole bar.
- **Logo Menu** adds an Apple-style menu in the top-left corner for shortcuts like System Settings and Lock.

### D. Extensions
Install from **Extension Manager** (search by name, click Install):

| Extension | What to set |
|---|---|
| **Dash to Dock** | Position: bottom. Turn off *Panel mode* so the dock floats. Then run `./tweaks.sh -d`. |
| **Blur my Shell** | Blurs the top bar, overview, and dock background. |
| **Just Perfection** | Hide Activities button; adjust top bar elements. |
| **Compiz alike magic lamp effect** | Genie-style minimize animation. |
| **Logo Menu** | macOS-style top-left menu. |
| **User Themes** | Already enabled in step B. Required for the shell theme. |

Extensions are tied to GNOME Shell versions. If one shows as incompatible after a GNOME upgrade, wait for an update rather than forcing it; a broken extension can freeze the shell. If that happens, disable all extensions from a TTY with `gsettings set org.gnome.shell disable-user-extensions true`.

---

## 6. GRUB and Plymouth

### A. GRUB theme
The WhiteSur GRUB theme lives in the [grub2-themes](https://github.com/vinceliuice/grub2-themes) repo. There is no separate `WhiteSur-grub-theme` repo.
```bash
cd ~
git clone --depth=1 https://github.com/vinceliuice/grub2-themes.git
cd grub2-themes
sudo ./install.sh -t whitesur -i whitesur -c 2560x1600
```
- `-t whitesur` picks the theme, `-i whitesur` the matching OS icons.
- Use `-s 1080p`, `-s 2k`, or `-s 4k` for standard screens. This laptop's panel is 2560x1600, which is not a preset, so `-c` sets it directly.
- The script runs `update-grub` itself.

If GRUB is hidden on boot (single-OS installs with a zero timeout), set `GRUB_TIMEOUT=3` and `GRUB_TIMEOUT_STYLE=menu` in `/etc/default/grub`, then `sudo update-grub`.

### B. Plymouth boot splash
Plymouth replaces the scrolling kernel log with a graphical splash while the system starts.

There is **no** `WhiteSur-plymouth-theme` repo, so the original clone step fails. Debian ships usable themes:
```bash
sudo apt install -y plymouth plymouth-themes
plymouth-set-default-theme -l            # list installed themes
sudo plymouth-set-default-theme -R bgrt  # -R also rebuilds the initramfs
```
`bgrt` shows your laptop maker's firmware logo with a spinner under it, which is the closest built-in match to a Mac boot screen. `spinner` is the same without the logo. Community themes (for example [adi1090x/plymouth-themes](https://github.com/adi1090x/plymouth-themes)) install to `/usr/share/plymouth/themes/<name>/` and are selected the same way.

### C. Kernel parameters
Edit `/etc/default/grub` (`sudo nvim /etc/default/grub`):
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 udev.log_level=3 vt.global_cursor_default=0"
```
Then `sudo update-grub`.

| Parameter | Effect |
|---|---|
| `quiet` | Fewer kernel messages |
| `splash` | Tells Plymouth to show the graphical splash |
| `loglevel=3` | Only errors and worse reach the console |
| `udev.log_level=3` | Same for udev. The original used `rd.udev.log_priority`, which only applies to dracut; Debian uses initramfs-tools by default. |
| `vt.global_cursor_default=0` | Hides the blinking console cursor |

If the splash never appears and you only see a black screen, the graphics driver is loading too late. This is common with the NVIDIA proprietary driver; add `nvidia-drm.modeset=1` to the line above.

---

## 7. Development setup

### Python
```bash
sudo apt install -y python3-venv python3-pip python3-dev
mkdir -p ~/Projects/my-app && cd ~/Projects/my-app
python3 -m venv .venv
source .venv/bin/activate
```
Debian marks its system Python as externally managed (PEP 668), so `pip install` outside a venv fails on purpose. Install packages inside a venv, use `apt install python3-<name>` for system-wide libraries, and `pipx` for command-line tools.

### PySpark
PySpark runs on the JVM, so a Python venv alone is not enough:
```bash
sudo apt install -y default-jdk-headless
source ~/Projects/my-app/.venv/bin/activate
pip install pyspark
python -c "from pyspark.sql import SparkSession; print(SparkSession.builder.getOrCreate().version)"
```
The last line starts a local Spark session and prints its version. If it fails with a Java error, run `java -version` to confirm the JDK is on your PATH.

### Node.js
```bash
sudo apt install -y nodejs npm
```
Debian's Node is a long-term-support release that stays fixed for the whole Debian cycle. That is fine for most frontend work. If a project needs a newer or different version, use a version manager such as [fnm](https://github.com/Schniz/fnm) or [nvm](https://github.com/nvm-sh/nvm) instead, and do not mix it with the apt package on the same PATH.

### SQL clients
```bash
sudo apt install -y postgresql-client sqlite3
```
These are clients only. For a local PostgreSQL server, install `postgresql` as well.

---

## 8. Undoing things

| To revert | Run |
|---|---|
| GTK/Shell theme | `~/WhiteSur-gtk-theme/install.sh -r` |
| GTK 4 override | `rm -r ~/.config/gtk-4.0/{gtk.css,gtk-dark.css,assets}` |
| GDM theme | `sudo ~/WhiteSur-gtk-theme/tweaks.sh -g -r` |
| GRUB theme | `sudo ~/grub2-themes/install.sh -r -t whitesur` |
| Plymouth | `sudo plymouth-set-default-theme -R spinner` (or remove `splash` from the kernel line) |
| All GNOME appearance settings | `gsettings reset-recursively org.gnome.desktop.interface` |
| All extensions | `gsettings set org.gnome.shell disable-user-extensions true` |
