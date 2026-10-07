# Debian Linux Install & macOS-Style Rice

A start-to-finish log of how this machine was built: a minimal Debian base, GNOME on top, WhiteSur and Tahoe themes for the macOS look, fish with Starship in the terminal, a themed boot sequence, and a development setup. Each step says what it does and what tends to go wrong.

Target: **Debian 13 "trixie"** (GNOME 48). Most steps also work on Debian 12, with the differences noted inline.

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
4. User: `hosee`, with a password.
5. Partitioning: *Guided - use entire disk*. Pick the *encrypted LVM* variant if this is a laptop.
6. Software selection: **untick** *Debian desktop environment* and *GNOME*. Keep **standard system utilities** only. Installing GNOME later with `gnome-core` gives a lighter desktop than the installer's full `gnome` task (no games, no LibreOffice, fewer background services).
7. Finish and reboot.

---

## 2. Post-install setup

Log in on the TTY as `hosee`.

### sudo (only if you set a root password)
```bash
su -                      # the "-" loads root's PATH, which is where usermod lives
apt update && apt install -y sudo
usermod -aG sudo hosee
exit
```
Log out and back in; group membership is read at login. Check with `groups`.

### APT sources
Open `/etc/apt/sources.list` (or `/etc/apt/sources.list.d/debian.sources` on trixie) and make sure the components include `main contrib non-free non-free-firmware`. Without `non-free-firmware`, Wi-Fi cards and some GPUs will not get their firmware.

### Base tools
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git unzip tar neovim build-essential fontconfig fastfetch
```
- `journalctl` is not a package; it ships with `systemd` and is already installed.
- `fontconfig` provides `fc-cache`, used in the next sections.
- `fastfetch` is in trixie. On Debian 12 it is only in `bookworm-backports`.

---

## 3. Terminal and shell

The shell is **fish** with the **Starship** prompt. Fish gives autosuggestions, syntax highlighting, and history search out of the box, which on zsh takes three separate plugins (`zsh-autosuggestions`, `zsh-syntax-highlighting`, `zsh-history-substring-search`) plus a framework to load them.

### A. Nerd Font (JetBrains Mono)
Starship draws its icons and separators from Nerd Font glyphs. Without one, the prompt shows empty boxes.

```bash
mkdir -p ~/.local/share/fonts/JetBrainsMonoNerd
cd /tmp
wget https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip
unzip -o JetBrainsMono.zip -d ~/.local/share/fonts/JetBrainsMonoNerd
fc-cache -f
fc-list : family | grep -i "jetbrainsmono nerd" | sort -u
```
The last command prints the family names fontconfig sees:

| Family | Use |
|---|---|
| `JetBrainsMono Nerd Font` | General use; icons may overflow one cell |
| `JetBrainsMono Nerd Font Mono` | Terminals; every glyph is exactly one cell wide |
| `JetBrainsMono Nerd Font Propo` | Proportional text (also listed as `JetBrainsMono NFP`) |

Pick the **Mono** variant in your terminal's profile preferences. The shell cannot change the font the terminal draws with.

### B. fish + Starship
```bash
sudo apt install -y fish
curl -sS https://starship.rs/install.sh | sh     # installs the binary to /usr/local/bin
chsh -s "$(command -v fish)"                    # takes effect at next login

mkdir -p ~/.config/fish
cat >> ~/.config/fish/config.fish <<'EOF'
if status is-interactive
    starship init fish | source
end
EOF
```
Wrapping the init in `status is-interactive` stops Starship from running for scripts and non-interactive shells.

Prompt layout and colours live in `~/.config/starship.toml`. Start from a preset instead of an empty file:
```bash
starship preset nerd-font-symbols -o ~/.config/starship.toml
starship preset --list        # other presets: pastel-powerline, tokyo-night, ...
```

Notes:
- fish is not POSIX. One-line commands copied from guides mostly work, but `export VAR=value` becomes `set -gx VAR value`, and `source venv/bin/activate` becomes `source venv/bin/activate.fish`.
- fish keeps its history in `~/.local/share/fish/fish_history` (YAML-like, one entry per command).
- If you prefer zsh, the same Starship config works; add `eval "$(starship init zsh)"` to `~/.zshrc`.

---

## 4. GNOME desktop

```bash
sudo apt install -y gnome-core gnome-tweaks gnome-shell-extension-manager gnome-browser-connector
sudo reboot
```
- `gnome-core` brings GNOME Shell, GDM (the login screen), Nautilus, Settings, and NetworkManager.
- `gnome-shell-extension-manager` is the **apt** build of Extension Manager. Use this one, not the Flatpak from Flathub: the Flatpak reads its GTK 4 styling from its own sandbox folder (`~/.var/app/com.mattjakeman.ExtensionManager/config/gtk-4.0`), so it ignores the theme in `~/.config/gtk-4.0` and has to be themed separately.
- `gnome-browser-connector` replaces the old `chrome-gnome-shell` package. Only needed if you install extensions from extensions.gnome.org in a browser.
- The **User Themes** extension comes with `gnome-core` through `gnome-shell-extensions`.

### Extra apps
`gnome-core` leaves out several apps that have macOS counterparts (Calendar, Weather, Maps, Clock). Add them, plus an archive tool and a media player:
```bash
sudo apt install -y file-roller gnome-calendar gnome-contacts gnome-weather gnome-maps gnome-clocks gnome-connections vlc
```

### Network shows "unmanaged"?
The installer wrote your connection to `/etc/network/interfaces`, and NetworkManager ignores interfaces listed there. Comment out everything except the `lo` lines, then `sudo systemctl restart NetworkManager`.

---

## 5. macOS look

The look is built from two sources:
- **WhiteSur** (vinceliuice): GNOME Shell theme, login screen, Flatpak theming.
- **gnome-macos bundle**: Tahoe-Neo GTK themes, icons and cursors, fonts, preconfigured extensions, MacTahoe wallpapers, and a config script. *(Add the bundle's download link here.)*

### Where themes live
GTK and GNOME Shell search both of these:

| Path | Notes |
|---|---|
| `~/.local/share/themes`, `~/.local/share/icons` | The XDG locations. The bundle installs here. |
| `~/.themes`, `~/.icons` | Older locations, still supported. WhiteSur installs here by default. |
| `/usr/share/themes`, `/usr/share/icons` | System-wide. Used when you run an installer with `sudo`. GDM and Flatpak apps can see it. |

Any of them works for your own session. The difference matters for Flatpak apps (section E), which only see the folders you grant them.

### A. WhiteSur GTK and Shell theme
```bash
cd ~
git clone --depth=1 https://github.com/vinceliuice/WhiteSur-gtk-theme.git
cd WhiteSur-gtk-theme
./install.sh --help          # options change between releases; check before copying flags

./install.sh -n WhiteSur -t all -m -N stable -l --shell -i apple -h bigger --round
```
| Flag | Effect |
|---|---|
| `-t all` | Install every accent colour |
| `-m` | Monterey style |
| `-N stable` | Nautilus sidebar style |
| `-l` | Also write the GTK 4 / libadwaita theme into `~/.config/gtk-4.0` |
| `--shell -i apple` | Apple logo in the top-left of the top bar |
| `--shell -h bigger` | Taller top bar, closer to the macOS menu bar |
| `--round` | Rounded corners on maximized windows |

To make the theme visible to GDM and Flatpak, install a system-wide copy as well:
```bash
sudo ./install.sh -o normal     # into /usr/share/themes
```

### B. Login screen (GDM)
```bash
sudo ./tweaks.sh -g -nd -b ~/Downloads/SequoiaLight.png
```
- `-b` sets the login wallpaper. `-nd` keeps it from being darkened. Add `-nb` to skip the blur.
- Run it from inside the `WhiteSur-gtk-theme` folder as `./tweaks.sh`. `/tweaks.sh` (no dot) looks in the root directory and fails.
- `-n` and `-N` are not `tweaks.sh` options; they belong to `install.sh`.
- Rerun after GNOME Shell updates, which can replace the GDM theme file.

### C. gnome-macos bundle
Download the bundle zips to `~/Downloads`, then:
```bash
cd ~/Downloads

# Themes: check the zip layout first, then extract the theme folders
unzip -l gnome-macos-themes-g48.zip | head
unzip -o gnome-macos-themes-g48.zip .face .face.icon -d ~      # user avatar for GDM and Settings

# Icons and cursors
unzip -o gnome-macos-icon-cursors.zip -d ~/.local/share/icons

# Fonts (macOS-style UI fonts)
unzip -o gnome-macos-fonts.zip -d ~/.local/share/fonts
fc-cache -f

# Preconfigured GNOME Shell extensions
unzip -o gnome-macos-extensions.zip -d ~/.local/share/gnome-shell/
unzip -o gnome-macos-extensions.zip ".commands.json" -d ~

# Config script (dconf settings, wallpapers)
unzip -o gnome-macos-config.zip -d ~/Downloads/
cd ~/Downloads/gnome-macos-config && ./install.sh
```
`g48` in the file name means the themes and extensions target GNOME 48. Use the matching build if your GNOME version is different (`gnome-shell --version`).

Check the results:
```bash
ls ~/.local/share/themes | grep Tahoe
ls ~/.local/share/icons
ls ~/.local/share/gnome-shell/extensions
ls /usr/share/backgrounds/MacTahoe /usr/share/gnome-background-properties/MacTahoe.xml
```
The last line confirms the wallpapers registered with GNOME, so they show up in Settings → Appearance.

On Wayland, GNOME Shell only scans the extensions folder at startup. **Log out and back in** before enabling the new extensions.

### D. GTK 4 / libadwaita apps
libadwaita apps (Settings, Files, Text Editor, Extension Manager) ignore the `gtk-theme` setting. The only way to restyle them is a user stylesheet in `~/.config/gtk-4.0/`:

| File | Loaded when |
|---|---|
| `gtk.css` | Always |
| `gtk-dark.css` | Dark style is active |
| `assets/` | Images the stylesheets reference |

With both CSS files in place, the light/dark switch in Settings keeps working. Point them at the theme with `@import` rather than symlinks:
```bash
mkdir -p ~/.config/gtk-4.0
rm -f ~/.config/gtk-4.0/gtk.css ~/.config/gtk-4.0/gtk-dark.css

T=~/.local/share/themes
echo "@import url('file://$T/Tahoe-Neo-Light-solid/gtk-4.0/gtk.css');" > ~/.config/gtk-4.0/gtk.css
echo "@import url('file://$T/Tahoe-Neo-Dark-solid/gtk-4.0/gtk.css');"  > ~/.config/gtk-4.0/gtk-dark.css
cp -r "$T/Tahoe-Neo-Dark-solid/gtk-4.0/assets" ~/.config/gtk-4.0/
```
Why `@import` instead of `ln -s`:
- `ln -s` fails with "File exists" when `gtk.css` is already there (WhiteSur's `-l` creates one). You need `rm` first or `ln -sf`.
- A one-line `@import` file is easy to read back. `cat ~/.config/gtk-4.0/gtk.css` tells you which theme is active.
- Switching themes means editing one line instead of re-linking three files.

To use WhiteSur instead of Tahoe-Neo, swap the paths for `~/.themes/WhiteSur-Light-solid` and `~/.themes/WhiteSur-Dark-solid`. Close and reopen the apps to see the change.

### E. Flatpak apps
Flatpak apps run in a sandbox and cannot see your theme folders by default.
```bash
sudo apt install -y flatpak ostree appstream
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

# Let every Flatpak app read the theme folders and your GTK 4 stylesheet
flatpak override --user --filesystem=~/.themes:ro
flatpak override --user --filesystem=~/.local/share/themes:ro
flatpak override --user --filesystem=xdg-config/gtk-4.0:ro

# Package WhiteSur as a Flatpak theme (needs ostree + appstream)
cd ~/WhiteSur-gtk-theme && ./tweaks.sh -F
```
Avoid `flatpak override --env=GTK_THEME=...`. `GTK_THEME` forces one GTK 3 theme on every app and breaks libadwaita's light/dark switching. If you set it earlier, remove it:
```bash
flatpak override --user --unset-env=GTK_THEME
flatpak override --user --show          # review what is set
```

### F. Apply with gsettings
Run these inside the GNOME session (not on a TTY or over SSH, where there is no session bus).
```bash
gsettings set org.gnome.desktop.interface gtk-theme      'Tahoe-Neo-Dark-solid'
gsettings set org.gnome.desktop.interface color-scheme   'prefer-dark'
gsettings set org.gnome.desktop.interface icon-theme     '<icon theme from the bundle>'
gsettings set org.gnome.desktop.interface cursor-theme   '<cursor theme from the bundle>'
gsettings set org.gnome.desktop.interface monospace-font-name 'JetBrainsMono Nerd Font Mono 10'
gsettings set org.gnome.desktop.wm.preferences button-layout 'close,minimize,maximize:'

gnome-extensions enable user-theme@gnome-shell-extensions.gcampax.github.com
gsettings set org.gnome.shell.extensions.user-theme name 'WhiteSur-Dark'
```
- Get the icon and cursor names from `ls ~/.local/share/icons`. A folder with a `cursors/` subfolder is a cursor theme.
- For the interface font, use the one the bundle installed (`fc-list : family | sort -u` to find its name). `fonts-inter` from Debian is a close free alternative.
- `button-layout 'close,minimize,maximize:'` puts the buttons on the left. The colon separates the left group from the right group.
- `gtk-theme` affects GTK 3 apps only. GTK 4 apps follow `~/.config/gtk-4.0` (section D).

### G. Top bar
The macOS menu bar is the GNOME top bar, restyled. YASB, from the original version of this guide, is Windows-only, and Wayland bars like Waybar do not run on GNOME's compositor.
- WhiteSur's `--shell -i apple -h bigger` (section A) adds the Apple logo and sets the height.
- **Just Perfection** hides parts macOS does not have, such as the Activities button.
- **Logo Menu** turns the top-left logo into a menu with System Settings, Lock, and similar items.

### H. Extensions
Enable from **Extension Manager**, or with `gnome-extensions enable <uuid>`:

| Extension | What to set |
|---|---|
| **Dash to Dock** | Position: bottom. Turn off *Panel mode* so the dock floats. Then run `./tweaks.sh -d` in the WhiteSur folder. |
| **Blur my Shell** | Blurs the top bar, overview, and dock background. |
| **Just Perfection** | Hide Activities button; adjust top bar elements. |
| **Compiz alike magic lamp effect** | Genie-style minimize animation. |
| **Logo Menu** | macOS-style top-left menu. |
| **User Themes** | Already enabled in section F. Required for the Shell theme. |

`gnome-extensions list --enabled` shows what is running. Extensions are tied to GNOME Shell versions. If one shows as incompatible after an upgrade, wait for an update instead of forcing it; a broken extension can freeze the shell. To recover from a TTY: `gsettings set org.gnome.shell disable-user-extensions true`.

### I. Terminal info
```bash
fastfetch
```
Add it to the end of `~/.config/fish/config.fish` (inside `if status is-interactive`) to show it in every new terminal.

---

## 6. GRUB and Plymouth

### A. GRUB theme
The WhiteSur GRUB theme lives in the [grub2-themes](https://github.com/vinceliuice/grub2-themes) repo.
```bash
cd ~
git clone --depth=1 https://github.com/vinceliuice/grub2-themes.git
cd grub2-themes
sudo ./install.sh -t whitesur -i whitesur -c 2560x1600
```
- `-t whitesur` picks the theme, `-i whitesur` the matching OS icons.
- Use `-s 1080p`, `-s 2k`, or `-s 4k` for standard screens. This laptop's panel is 2560x1600, which is not a preset, so `-c` sets it directly.
- The script runs `update-grub` itself.

If GRUB is hidden on boot, set `GRUB_TIMEOUT=3` and `GRUB_TIMEOUT_STYLE=menu` in `/etc/default/grub`, then `sudo update-grub`.

### B. Plymouth boot splash
Plymouth replaces the scrolling kernel log with a graphical splash while the system starts.
```bash
sudo apt install -y plymouth plymouth-themes
plymouth-set-default-theme -l            # list installed themes
sudo plymouth-set-default-theme -R bgrt  # -R also rebuilds the initramfs
```
`bgrt` shows your laptop maker's firmware logo with a spinner under it, the closest built-in match to a Mac boot screen. `spinner` is the same without the logo. Community themes (for example [adi1090x/plymouth-themes](https://github.com/adi1090x/plymouth-themes)) install to `/usr/share/plymouth/themes/<name>/` and are selected the same way.

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
| `udev.log_level=3` | Same for udev (`rd.udev.log_priority` only applies to dracut; Debian uses initramfs-tools) |
| `vt.global_cursor_default=0` | Hides the blinking console cursor |

If the splash never appears and you see a black screen, the graphics driver loads too late. With the NVIDIA driver, add `nvidia-drm.modeset=1` to the line above.

---

## 7. Development setup

### Python
```bash
sudo apt install -y python3-venv python3-pip python3-dev
mkdir -p ~/Projects/my-app && cd ~/Projects/my-app
python3 -m venv .venv
source .venv/bin/activate.fish     # bash/zsh: source .venv/bin/activate
```
Debian marks its system Python as externally managed (PEP 668), so `pip install` outside a venv fails on purpose. Install packages inside a venv, use `apt install python3-<name>` for system-wide libraries, and `pipx` for command-line tools.

### PySpark
PySpark runs on the JVM, so a Python venv alone is not enough:
```bash
sudo apt install -y default-jdk-headless
pip install pyspark          # inside the venv
python -c "from pyspark.sql import SparkSession; print(SparkSession.builder.getOrCreate().version)"
```

### Node.js
```bash
sudo apt install -y nodejs npm
```
Debian's Node is a long-term-support release that stays fixed for the Debian cycle. If a project needs another version, use [fnm](https://github.com/Schniz/fnm) or [nvm](https://github.com/nvm-sh/nvm) instead, and do not mix it with the apt package on the same PATH.

### SQL clients
```bash
sudo apt install -y postgresql-client sqlite3
```
These are clients only. For a local PostgreSQL server, install `postgresql` as well.

---

## 8. Undoing things

| To revert | Run |
|---|---|
| WhiteSur GTK/Shell theme | `~/WhiteSur-gtk-theme/install.sh -r` (add `sudo` for the system-wide copy) |
| GTK 4 override | `rm -r ~/.config/gtk-4.0/{gtk.css,gtk-dark.css,assets}` |
| GDM theme | `sudo ~/WhiteSur-gtk-theme/tweaks.sh -g -r` |
| Flatpak overrides | `flatpak override --user --reset` |
| GRUB theme | `sudo ~/grub2-themes/install.sh -r -t whitesur` |
| Plymouth | `sudo plymouth-set-default-theme -R spinner` (or remove `splash` from the kernel line) |
| GNOME appearance settings | `gsettings reset-recursively org.gnome.desktop.interface` |
| All extensions | `gsettings set org.gnome.shell disable-user-extensions true` |
| Login shell back to bash | `chsh -s /bin/bash` |
