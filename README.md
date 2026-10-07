# Debian Linux Install & macOS-Style Rice

How this laptop was set up: a minimal Debian base tracking testing, GNOME with WhiteSur and Tahoe themes for the macOS look, fish with Starship in the terminal, a few desktop extras, and a development setup. Each step says what it does and why, and notes where it went wrong the first time.

**Target:** Debian 14 "forky" (testing), kernel 7.x. Check your GNOME version with `gnome-shell --version`; themes and extensions below are tied to it.

---

<img width="2560" height="1600" alt="image" src="https://github.com/user-attachments/assets/a1180f15-de98-4fa8-8c0e-c4f478531d6e" />

## Contents
1. [Base system install](#1-base-system-install)
2. [Post-install setup](#2-post-install-setup)
3. [Terminal and shell](#3-terminal-and-shell)
4. [GNOME desktop](#4-gnome-desktop)
5. [macOS look](#5-macos-look)
6. [Desktop extras](#6-desktop-extras)
7. [GRUB and Plymouth](#7-grub-and-plymouth)
8. [Development setup](#8-development-setup)
9. [Undoing things](#9-undoing-things)

---

## 1. Base system install

### Bootable USB
1. Download the **netinst** ISO from [debian.org](https://www.debian.org/distrib/netinst). It is small and pulls current packages during install.
2. Write it to a USB stick with Rufus, balenaEtcher, or on Linux: `sudo dd if=debian-*.iso of=/dev/sdX bs=4M status=progress oflag=sync`. Check `sdX` with `lsblk` first, because `dd` overwrites whatever you point it at.

### Installer choices
1. Boot from the USB and pick **Graphical install**.
2. Hostname: `deb`
3. **Root password:** leave it empty. The installer then locks root, installs `sudo`, and adds your user to the `sudo` group.
4. User: any name you like, with a password. This guide writes it as `user`; replace it with yours wherever it appears.
5. Partitioning: *Guided - use entire disk*. Pick the *encrypted LVM* variant on a laptop.
6. Software selection: untick *Debian desktop environment* and *GNOME*. Keep **standard system utilities** only. Installing `gnome-core` later skips the games, LibreOffice, and extra background services of the full `gnome` task.
7. Finish and reboot.

---

## 2. Post-install setup

Log in on the TTY with the account you just created.

### sudo (only if you set a root password)
```bash
su -                      # the "-" loads root's PATH, which is where usermod lives
apt update && apt install -y sudo
usermod -aG sudo user     # replace "user" with your username; inside su, $USER is root
exit
```
Log out and back in, because group membership is read at login. Check with `groups`.

### Move to testing
This machine runs testing for a newer GNOME and kernel. Edit `/etc/apt/sources.list.d/debian.sources` (or `/etc/apt/sources.list`) so every suite reads `forky` and the components are `main contrib non-free non-free-firmware`. Without `non-free-firmware`, Wi-Fi, audio DSP, and some GPUs get no firmware.

Name the release (`forky`) rather than `testing`. With `testing`, the machine silently jumps to the next release cycle when forky becomes stable.

### Base tools
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git unzip tar neovim build-essential fontconfig fastfetch htop ncdu
```
`journalctl` is not a package; it comes with `systemd`. `fontconfig` provides `fc-cache`, used in the next section.

---

## 3. Terminal and shell

The shell is **fish** with the **Starship** prompt. Fish has autosuggestions, syntax highlighting, and history search built in. On zsh the same takes three plugins (`zsh-autosuggestions`, `zsh-syntax-highlighting`, `zsh-history-substring-search`) and a framework to load them.

### A. Nerd Font (JetBrains Mono)
Starship draws its icons from Nerd Font glyphs. Without one, the prompt shows empty boxes.

```bash
mkdir -p ~/.local/share/fonts/JetBrainsMonoNerd
cd /tmp
wget https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip
unzip -o JetBrainsMono.zip -d ~/.local/share/fonts/JetBrainsMonoNerd
fc-cache -f
fc-list : family | grep -i "jetbrainsmono nerd" | sort -u
```

| Family | Use |
|---|---|
| `JetBrainsMono Nerd Font` | General use; icons may overflow one cell |
| `JetBrainsMono Nerd Font Mono` | Terminals; every glyph is one cell wide |
| `JetBrainsMono Nerd Font Propo` | Proportional text (also listed as `JetBrainsMono NFP`) |

Set the **Mono** variant in the terminal's profile preferences. The shell cannot change the font the terminal draws with.

### B. fish + Starship
```bash
sudo apt install -y fish
curl -sS https://starship.rs/install.sh | sh     # installs to /usr/local/bin
chsh -s "$(command -v fish)"                    # takes effect at next login
starship preset nerd-font-symbols -o ~/.config/starship.toml
```
`starship preset --list` shows other starting points. Edit `~/.config/starship.toml` from there.

### C. config.fish
Keep everything in `~/.config/fish/config.fish`. A function typed at the prompt (like `function fish_greeting ... end`) only lasts for that session; in this file it loads every time.

```fish
# ~/.config/fish/config.fish
if status is-interactive
    starship init fish | source

    function fish_greeting
        set_color cyan
        echo "Welcome back, $USER"
        set_color normal
        fastfetch
    end
end
```
- `status is-interactive` keeps the prompt and greeting out of scripts.
- Reload after editing with `source ~/.config/fish/config.fish`, or open a new terminal.
- Add folders to PATH with `fish_add_path ~/some/bin`. It saves the change once and skips duplicates, unlike appending to `fish_user_paths` by hand.

Fish is not POSIX. `export VAR=value` becomes `set -gx VAR value`, and a venv is activated with `source .venv/bin/activate.fish`.

### D. fastfetch with a custom logo
`neofetch` is no longer maintained; `fastfetch` reads the same kind of info and starts faster. Remove neofetch if you installed it: `sudo apt purge neofetch`.

```bash
fastfetch --gen-config            # writes ~/.config/fastfetch/config.jsonc
nano ~/.config/fastfetch/logo.txt # paste ASCII art
```
Point the config at the logo:
```jsonc
"logo": {
    "type": "file",
    "source": "~/.config/fastfetch/logo.txt",
    "color": { "1": "cyan" }
},
```
Use `$1`, `$2`, ... inside `logo.txt` to switch to the colours listed under `color`.

---

## 4. GNOME desktop

```bash
sudo apt install -y gnome-core gnome-tweaks gnome-shell-extension-manager gnome-browser-connector
sudo reboot
```
- `gnome-core` brings GNOME Shell, GDM, Nautilus, Settings, and NetworkManager.
- Use the **apt** build of Extension Manager, not the Flatpak. The Flatpak reads GTK 4 styling from its own sandbox folder (`~/.var/app/com.mattjakeman.ExtensionManager/config/gtk-4.0`) and ignores `~/.config/gtk-4.0`, so it has to be themed separately.
- `gnome-browser-connector` (formerly `chrome-gnome-shell`) is only needed to install extensions from extensions.gnome.org in a browser.
- The **User Themes** extension comes with `gnome-core`.

### Extra apps
`gnome-core` leaves out apps with macOS counterparts. Add them, plus an archive tool, a media player, and a volume mixer:
```bash
sudo apt install -y file-roller gnome-calendar gnome-contacts gnome-weather gnome-maps gnome-clocks gnome-connections vlc pavucontrol
```

### Faster login
`systemd-analyze --user blame | head` lists what slows the session start. Two services that are safe to mask on this setup:
```bash
systemctl --user mask gnome-software.service           # updates come from apt, not GNOME Software
systemctl --user mask gvfs-gphoto2-volume-monitor.service  # camera auto-detection; not used here
```
Masking stops them from starting at all. `systemctl --user unmask <name>` reverses it.

### Network shows "unmanaged"?
The installer wrote your connection to `/etc/network/interfaces`, and NetworkManager ignores interfaces listed there. Comment out everything except the `lo` lines, then `sudo systemctl restart NetworkManager`.

### Chrome on Wayland
`echo $XDG_SESSION_TYPE` prints `wayland` on a default GNOME login. Chrome may still start under XWayland, which looks blurry with fractional scaling. Open `chrome://flags`, set **Preferred Ozone platform** to **Wayland** (or **Auto**), and relaunch.

This replaces copying `google-chrome.desktop` into `~/.local/share/applications` and editing its `Exec` line. That copy overrides the system file, so it stops receiving changes when Chrome updates its launcher.

---

## 5. macOS look

The look comes from two sources:
- **WhiteSur** (vinceliuice): GNOME Shell theme, login screen, Flatpak theming.
- **gnome-macos bundle**: Tahoe-Neo GTK themes, icons and cursors, fonts, preconfigured extensions, MacTahoe wallpapers, and a config script. *(Add the bundle's download link here.)*

### Where themes live

| Path | Notes |
|---|---|
| `~/.local/share/themes`, `~/.local/share/icons` | XDG locations. The bundle installs here. |
| `~/.themes`, `~/.icons` | Older locations, still read. WhiteSur installs here by default. |
| `/usr/share/themes`, `/usr/share/icons` | System-wide, from installers run with `sudo`. GDM and Flatpak apps can see it. |

Any of them works for your session. The difference matters for Flatpak apps (section E), which only see folders you grant them.

### A. WhiteSur GTK and Shell theme
```bash
sudo apt install -y git
cd ~
git clone --depth=1 https://github.com/vinceliuice/WhiteSur-gtk-theme.git
cd WhiteSur-gtk-theme
./install.sh --help          # flags change between releases; check before copying

./install.sh -n WhiteSur -t all -m -N stable -l --shell -i apple -h bigger --round
sudo ./install.sh -o normal  # system-wide copy for GDM and Flatpak
```
| Flag | Effect |
|---|---|
| `-t all` | Every accent colour |
| `-m` | Monterey style |
| `-N stable` | Nautilus sidebar style |
| `-l` | Also write the GTK 4 / libadwaita theme into `~/.config/gtk-4.0` |
| `--shell -i apple` | Apple logo in the top-left of the top bar |
| `--shell -h bigger` | Taller top bar, closer to the macOS menu bar |
| `--round` | Rounded corners on maximized windows |

Paste the clone URL as plain text. A markdown link (`[https://...](https://...)`) pasted into the terminal makes `git clone` fail.

### B. Login screen (GDM)
```bash
cd ~/WhiteSur-gtk-theme
sudo ./tweaks.sh -g -nd -b ~/Downloads/SequoiaLight.png
```
- `-b` sets the login wallpaper. `-nd` keeps it from being darkened; add `-nb` to skip the blur.
- Run it as `./tweaks.sh`. `/tweaks.sh` looks in the root directory and fails.
- `-n` and `-N` are `install.sh` options, not `tweaks.sh` options.
- Rerun after GNOME Shell updates, which can replace the GDM theme file.

### C. gnome-macos bundle
Download the bundle zips to `~/Downloads`, then:
```bash
cd ~/Downloads

# Themes: check the layout first, then extract the theme folders
unzip -l gnome-macos-themes-g48.zip | head
unzip -o gnome-macos-themes-g48.zip .face .face.icon -d ~      # avatar for GDM and Settings

unzip -o gnome-macos-icon-cursors.zip -d ~/.local/share/icons
unzip -o gnome-macos-fonts.zip -d ~/.local/share/fonts && fc-cache -f

unzip -o gnome-macos-extensions.zip -d ~/.local/share/gnome-shell/
unzip -o gnome-macos-extensions.zip ".commands.json" -d ~

unzip -o gnome-macos-config.zip -d ~/Downloads/
cd ~/Downloads/gnome-macos-config && ./install.sh
```
**Version check:** `g48` means the files target GNOME 48. Testing ships a newer GNOME, and GNOME Shell refuses to load an extension whose `metadata.json` does not list the running version. If an extension shows as incompatible, get the bundle build for your version instead of editing `metadata.json`; the version check exists because Shell APIs change between releases.

Check the results:
```bash
ls ~/.local/share/themes | grep Tahoe
ls ~/.local/share/icons
ls ~/.local/share/gnome-shell/extensions
ls /usr/share/backgrounds/MacTahoe /usr/share/gnome-background-properties/MacTahoe.xml
```
The last line confirms the wallpapers are registered, so they appear in Settings > Appearance.

On Wayland, GNOME Shell only scans the extensions folder at startup. **Log out and back in** before enabling the new extensions.

### D. GTK 4 / libadwaita apps
libadwaita apps (Settings, Files, Text Editor) ignore the `gtk-theme` setting. The only way to restyle them is a user stylesheet in `~/.config/gtk-4.0/`:

| File | Loaded when |
|---|---|
| `gtk.css` | Always |
| `gtk-dark.css` | Dark style is active |
| `assets/` | Images the stylesheets reference |

With both CSS files present, the light/dark switch in Settings keeps working. Point them at the theme with `@import`:
```bash
mkdir -p ~/.config/gtk-4.0
rm -f ~/.config/gtk-4.0/gtk.css ~/.config/gtk-4.0/gtk-dark.css

T=~/.local/share/themes
echo "@import url('file://$T/Tahoe-Neo-Light-solid/gtk-4.0/gtk.css');" > ~/.config/gtk-4.0/gtk.css
echo "@import url('file://$T/Tahoe-Neo-Dark-solid/gtk-4.0/gtk.css');"  > ~/.config/gtk-4.0/gtk-dark.css
cp -r "$T/Tahoe-Neo-Dark-solid/gtk-4.0/assets" ~/.config/gtk-4.0/
```
Why `@import` and not `ln -s`:
- `ln -s` fails with "File exists" when `gtk.css` is already there (WhiteSur's `-l` creates one).
- `cat ~/.config/gtk-4.0/gtk.css` shows which theme is active in one line.
- Switching themes means editing one line instead of relinking three files.

For WhiteSur instead of Tahoe-Neo, use `~/.themes/WhiteSur-Light-solid` and `~/.themes/WhiteSur-Dark-solid`. Reopen apps to see the change.

### E. Flatpak apps
Flatpak apps run in a sandbox and cannot see your theme folders by default.
```bash
sudo apt install -y flatpak ostree appstream
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

flatpak override --user --filesystem=~/.themes:ro
flatpak override --user --filesystem=~/.local/share/themes:ro
flatpak override --user --filesystem=xdg-config/gtk-4.0:ro

cd ~/WhiteSur-gtk-theme && ./tweaks.sh -F   # packages WhiteSur as a Flatpak theme; needs ostree + appstream
```
`:ro` gives read-only access, which is all a theme needs.

Do not use `flatpak override --env=GTK_THEME=...`. It forces one GTK 3 theme on every app and breaks libadwaita's light/dark switching. To remove it:
```bash
flatpak override --user --unset-env=GTK_THEME
flatpak override --user --show
```

### F. Apply with gsettings
Run inside the GNOME session. On a TTY or over SSH there is no session bus and the commands fail.
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
- Icon and cursor names: `ls ~/.local/share/icons`. A folder with a `cursors/` subfolder is a cursor theme.
- Interface font: use the one the bundle installed (`fc-list : family | sort -u`). `fonts-inter` from Debian is a free alternative close to the macOS font.
- `button-layout`: the colon separates left from right, so this puts the buttons on the left.
- `gtk-theme` affects GTK 3 apps only. GTK 4 apps follow section D.

### G. Editing the Shell theme
To change the top bar, menus, or overview beyond what the installer offers, edit the theme's stylesheet:
```bash
nano ~/.local/share/themes/<theme-name>/gnome-shell/gnome-shell.css
```
Wayland cannot restart GNOME Shell in place. Reload the theme by switching away and back:
```bash
gsettings set org.gnome.shell.extensions.user-theme name ''
gsettings set org.gnome.shell.extensions.user-theme name '<theme-name>'
```
Back up the file first. Reinstalling the theme overwrites your edits.

### H. Top bar
The macOS menu bar is the GNOME top bar, restyled. YASB, from the first version of this guide, is Windows-only, and Wayland bars like Waybar do not run on GNOME's compositor.
- WhiteSur's `--shell -i apple -h bigger` (section A) adds the Apple logo and sets the height.
- **Just Perfection** hides parts macOS does not have, such as the Activities button.
- **Logo Menu** turns the top-left logo into a menu with System Settings, Lock, and similar items.

### I. Extensions
Enable from **Extension Manager**, or with `gnome-extensions enable <uuid>`:

| Extension | What to set |
|---|---|
| **Dash to Dock** | Position: bottom. Turn off *Panel mode* so the dock floats. Then run `./tweaks.sh -d` in the WhiteSur folder. |
| **Blur my Shell** | Blurs the top bar, overview, and dock background. |
| **Just Perfection** | Hide Activities button; adjust top bar elements. |
| **Compiz alike magic lamp effect** | Genie-style minimize animation. |
| **Logo Menu** | macOS-style top-left menu. |
| **User Themes** | Enabled in section F. Required for the Shell theme. |

`gnome-extensions list --enabled` shows what is running. If an extension breaks the shell after an upgrade, disable all of them from a TTY: `gsettings set org.gnome.shell disable-user-extensions true`.

---

## 6. Desktop extras

### Rain Clock (desktop clock)
[rain-clock-gnome](https://github.com/hugo-sants/rain-clock-gnome) draws a clock on the desktop and picks its text colour from the wallpaper's brightness.
```bash
cd ~
git clone https://github.com/hugo-sants/rain-clock-gnome.git
cd rain-clock-gnome
make install              # extension plus bundled fonts
```
Log out and back in, then:
```bash
gnome-extensions enable rainclock@hugo-sants.github.com
gnome-extensions prefs rainclock@hugo-sants.github.com   # position, time and date format
```
`make install` already installs the fonts (Anurati, Poppins). `make fonts-install` is only for reinstalling them on their own.

### HyprQuickPaper (wallpaper picker)
[hyprquickpaper-gnome](https://github.com/hugo-sants/hyprquickpaper-gnome) opens a keyboard-driven wallpaper carousel.
```bash
sudo apt install -y python3 python3-gi python3-cairo python3-gi-cairo gir1.2-gtk-4.0 libglib2.0-bin jq imagemagick
cd ~
git clone https://github.com/hugo-sants/hyprquickpaper-gnome.git
cd hyprquickpaper-gnome
make install
```
The installer asks for a wallpaper folder (default `~/Pictures/Wallpapers`) and a shortcut (default **Super+W**). In the picker: arrows or J/K to move, Enter to apply, Esc to close.

### Spotify theme (Spicetify)
Spicetify patches the Spotify client's interface. It works with the apt build of Spotify (`spotify-client` from Spotify's repo). The Snap build is read-only and cannot be patched.
```bash
curl -fsSL https://raw.githubusercontent.com/spicetify/cli/main/install.sh | sh
fish_add_path ~/.spicetify                 # if `spicetify` is not found afterwards

sudo chmod a+wr /usr/share/spotify
sudo chmod -R a+wr /usr/share/spotify/Apps
spicetify backup apply
```
- The `chmod` lines let Spicetify write into Spotify's install folder without root. Any local user can then modify those files, which is acceptable on a single-user laptop.
- Spotify updates replace the patched files. After one, run `spicetify backup apply` again.
- There is no `spicetify` apt package; `sudo apt spicetify` is not a valid command.

---

## 7. GRUB and Plymouth

### A. GRUB theme
The WhiteSur GRUB theme lives in the [grub2-themes](https://github.com/vinceliuice/grub2-themes) repo.
```bash
cd ~
git clone --depth=1 https://github.com/vinceliuice/grub2-themes.git
cd grub2-themes
sudo ./install.sh -t whitesur -i whitesur -c 2560x1600
```
- `-t whitesur` picks the theme, `-i whitesur` the matching OS icons.
- `-c 2560x1600` matches this panel. Use `-s 1080p`, `-s 2k`, or `-s 4k` for standard screens.
- The script runs `update-grub` itself.

Keep the menu visible on testing: set `GRUB_TIMEOUT=3` and `GRUB_TIMEOUT_STYLE=menu` in `/etc/default/grub`. When a kernel update breaks audio or the GPU, *Advanced options* in that menu boots the previous kernel.

### B. Plymouth boot splash
```bash
sudo apt install -y plymouth plymouth-themes
plymouth-set-default-theme -l            # list installed themes
sudo plymouth-set-default-theme -R bgrt  # -R also rebuilds the initramfs
```
`bgrt` shows the laptop maker's firmware logo with a spinner, the closest built-in match to a Mac boot screen. `spinner` is the same without the logo. Community themes such as [adi1090x/plymouth-themes](https://github.com/adi1090x/plymouth-themes) install to `/usr/share/plymouth/themes/<name>/` and are selected the same way.

### C. Kernel parameters
This laptop uses the NVIDIA driver, so the kernel line combines the splash settings with the NVIDIA ones. Edit `/etc/default/grub`:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 udev.log_level=3 vt.global_cursor_default=0 modprobe.blacklist=nouveau nvidia-drm.modeset=1 nvidia-drm.fbdev=1"
```
Then `sudo update-grub`.

| Parameter | Effect |
|---|---|
| `quiet`, `loglevel=3` | Only errors reach the console |
| `splash` | Tells Plymouth to draw the splash |
| `udev.log_level=3` | Same for udev (`rd.udev.log_priority` is dracut-only; Debian uses initramfs-tools) |
| `vt.global_cursor_default=0` | Hides the blinking console cursor |
| `modprobe.blacklist=nouveau` | Stops the open-source driver from claiming the GPU first |
| `nvidia-drm.modeset=1`, `nvidia-drm.fbdev=1` | NVIDIA provides the boot framebuffer, so Plymouth has a screen to draw on |

`nouveau.modeset=0` is redundant once nouveau is blacklisted. Without the two `nvidia-drm` options, the splash usually shows as a black screen.

---

## 8. Development setup

### Code on the NTFS data drive
Projects live on a shared NTFS partition. Git refuses to work in a repo owned by another user, which is why every project needed `git config --global --add safe.directory ...`. Mount the partition with your user as owner instead, in `/etc/fstab`:
```
UUID=<from lsblk -f>  /mnt/data  ntfs3  uid=1000,gid=1000,windows_names  0  0
```
`1000` is the ID of the first account created on the system; check yours with `id -u`. Every file then belongs to your account, and `safe.directory` entries are no longer needed. If Windows was hibernated or fast-started, the mount is read-only or fails; turn off Fast Startup in Windows, or run `sudo ntfsfix /dev/<partition>` once.

### Python
```bash
sudo apt install -y python3-venv python3-pip python3-dev
python3 -m venv .venv
source .venv/bin/activate.fish
```
Debian marks its system Python as externally managed (PEP 668), so `pip install` outside a venv fails on purpose. Use a venv for project packages, `apt install python3-<name>` for system-wide libraries, and `pipx` for command-line tools. Install `python3-venv`, not `python3.14-venv`; the generic name follows Python upgrades.

### Node.js
```bash
sudo apt install -y nodejs npm
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
fish_add_path ~/.npm-global/bin
```
The prefix makes `npm install -g` write into your home folder, so global installs need no `sudo`. For per-project Node versions, use [fnm](https://github.com/Schniz/fnm) instead of the apt package.

### .NET
Debian testing has no `dotnet-sdk-8.0` package, and Microsoft's repo for Debian 12 does not install cleanly on testing. Use Microsoft's install script:
```bash
wget https://dot.net/v1/dotnet-install.sh -O /tmp/dotnet-install.sh
bash /tmp/dotnet-install.sh --channel 8.0
set -Ux DOTNET_ROOT ~/.dotnet
fish_add_path ~/.dotnet ~/.dotnet/tools
```
Apps started from the desktop (VS Code, for example) do not read fish's PATH. `sudo ln -s ~/.dotnet/dotnet /usr/local/bin/dotnet` makes `dotnet` visible to them too.

### Java
```bash
sudo apt install -y default-jdk
```
The JDK includes the runtime, so it covers both running `.jar` files and compiling.

### Docker
```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER     # log out and in afterwards
```
The group lets you run `docker` without `sudo`. Membership in `docker` is equivalent to root access, so add only your own account.

---

## 9. Undoing things

| To revert | Run |
|---|---|
| WhiteSur GTK/Shell theme | `~/WhiteSur-gtk-theme/install.sh -r` (add `sudo` for the system-wide copy) |
| GTK 4 override | `rm -r ~/.config/gtk-4.0/{gtk.css,gtk-dark.css,assets}` |
| GDM theme | `sudo ~/WhiteSur-gtk-theme/tweaks.sh -g -r` |
| Flatpak overrides | `flatpak override --user --reset` |
| Rain Clock | `cd ~/rain-clock-gnome && make uninstall` |
| HyprQuickPaper | `cd ~/hyprquickpaper-gnome && make uninstall`, then remove the shortcut in Settings > Keyboard |
| Spicetify | `spicetify restore` |
| Masked services | `systemctl --user unmask gnome-software.service gvfs-gphoto2-volume-monitor.service` |
| GRUB theme | `sudo ~/grub2-themes/install.sh -r -t whitesur` |
| Plymouth | `sudo plymouth-set-default-theme -R spinner`, or remove `splash` from the kernel line |
| GNOME appearance settings | `gsettings reset-recursively org.gnome.desktop.interface` |
| All extensions | `gsettings set org.gnome.shell disable-user-extensions true` |
| Login shell back to bash | `chsh -s /bin/bash` |
