# Dokumentasi Instalasi & Ricing Debian Linux (macOS Style)

Dokumentasi komprehensif ini mencakup instalasi *base system* Debian, setup *environment development*, hingga tahapan *ricing* secara mendalam untuk mencapai estetika macOS, menggunakan metode standar sesuai arsitektur XDG dan panduan resmi GNOME/GTK.

---

## Daftar Isi
1. [Persiapan & Instalasi Base System](#1-persiapan--instalasi-base-system)
2. [Pasca-Instalasi & Sistem Dasar](#2-pasca-instalasi--sistem-dasar)
3. [Ricing Terminal & Shell](#3-ricing-terminal--shell)
4. [Instalasi Desktop Environment](#4-instalasi-desktop-environment)
5. [Proses Ricing UI (Tampilan macOS)](#5-proses-ricing-ui-tampilan-macos)
6. [Kustomisasi Bootloader (GRUB) & Plymouth](#6-kustomisasi-bootloader-grub--plymouth)
7. [Setup Development](#7-setup-development)

---

## 1. Persiapan & Instalasi Base System

### Persiapan Bootable
1. Unduh file ISO Debian Netinst (Network Installer) dari situs resmi Debian.
2. Buat *bootable USB* (via Rufus/BalenaEtcher).

### Proses Instalasi
1. Boot PC/Laptop dari USB dan pilih **Graphical Install**.
2. Atur *Hostname*: `deb`
3. Atur *Username*: `pascal` beserta *password*.
4. **Partisi Disk**: Pilih *Guided - use entire disk*.
5. **Software Selection**: **Hilangkan** centang *Debian desktop environment* dan *GNOME*. Cukup centang **Standard system utilities**.
6. Selesaikan instalasi dan *reboot*.

---

## 2. Pasca-Instalasi & Sistem Dasar

Login ke tty (terminal) menggunakan `pascal`.

### Konfigurasi Sudo
```bash
su -
apt update && apt install sudo
usermod -aG sudo pascal
exit
```
*Logout* dan *login* kembali.

### Instalasi Tools Esensial
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip tar neovim build-essential journalctl fastfetch zsh
```

---

## 3. Ricing Terminal & Shell

Mengganti antarmuka terminal dasar dengan Zsh untuk produktivitas *command-line*.

### A. Font (JetBrainsMono NFP)
```bash
mkdir -p ~/.local/share/fonts
cd /tmp
wget https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip
unzip JetBrainsMono.zip -d ~/.local/share/fonts/
fc-cache -fv
```

### B. Oh-My-Zsh & Tema (Powerlevel10k)
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k

# Edit ~/.zshrc dan ubah: ZSH_THEME="powerlevel10k/powerlevel10k"
```

---

## 4. Instalasi Desktop Environment

```bash
# Instal GNOME minimal
sudo apt install -y gnome-core gnome-tweaks gnome-shell-extensions chrome-gnome-shell
```
Setelah instalasi selesai, `sudo reboot`.

---

## 5. Proses Ricing UI (Tampilan macOS)

Mengacu pada arsitektur XDG dan panduan GTK, direktori standar untuk kustomisasi pengguna adalah `~/.themes` dan `~/.icons`. Kita akan menggunakan utilitas bawaan GNOME (`gsettings`) untuk mengatur preferensi tema, yang merupakan standar antarmuka API GTK/GNOME dibandingkan menggunakan *third-party tweaking tool*.

### A. Unduh Tema, Ikon, & Kursor (McMojave)
```bash
# Buat direktori standar XDG
mkdir -p ~/.themes ~/.icons

# Clone repositori
cd ~
git clone https://github.com/vinceliuice/WhiteSur-gtk-theme.git
git clone https://github.com/vinceliuice/WhiteSur-icon-theme.git
git clone https://github.com/vinceliuice/McMojave-cursors.git

# Instal Tema GTK
cd WhiteSur-gtk-theme
./install.sh -t all -N glassy -s 220
sudo ./tweaks.sh -g  # Modifikasi GDM (Login Screen)

# Instal Ikon
cd ../WhiteSur-icon-theme
./install.sh -a

# Instal Kursor macOS
cd ../McMojave-cursors
./install.sh
```

### B. Implementasi via GTK Settings (GSettings)
Gunakan perintah resmi GNOME untuk mengaplikasikan kustomisasi:
```bash
gsettings set org.gnome.desktop.interface gtk-theme 'WhiteSur-Light'
gsettings set org.gnome.desktop.interface icon-theme 'WhiteSur'
gsettings set org.gnome.desktop.interface cursor-theme 'McMojave-cursors'
gsettings set org.gnome.desktop.interface font-name 'JetBrainsMono NFP 10'
gsettings set org.gnome.desktop.wm.preferences button-layout 'close,minimize,maximize:'
```
*Tweak Libadwaita (GTK4): Hubungkan CSS WhiteSur ke konfigurasi GTK4 standar:*
```bash
ln -s ~/.themes/WhiteSur-Light/gtk-4.0/assets ~/.config/gtk-4.0/assets
ln -s ~/.themes/WhiteSur-Light/gtk-4.0/gtk.css ~/.config/gtk-4.0/gtk.css
ln -s ~/.themes/WhiteSur-Light/gtk-4.0/gtk-dark.css ~/.config/gtk-4.0/gtk-dark.css
```

### C. Status Bar (YASB)
```bash
# Integrasikan YASB (Yet Another Status Bar)
# Edit file ~/.config/yasb/styles.css:
# Pastikan deklarasi font menggunakan: font-family: 'JetBrainsMono NFP';
```

### D. Ekstensi GNOME Desktop
Gunakan GNOME Extensions (atau Extension Manager) untuk menginstal:
- **Dash to Dock**: Posisikan di bawah, nonaktifkan *panel mode*.
- **Blur my Shell**: Menambah efek kaca pada elemen UI.
- **Just Perfection**: Menyembunyikan Top Bar GNOME bawaan untuk memberi ruang pada YASB.
- **Compiz alike magic lamp effect**: Animasi minimize jendela (mirip *Genie Effect*).
- **User Themes**: Memungkinkan ekstensi memuat tema ke GNOME Shell.

---

## 6. Kustomisasi Bootloader (GRUB) & Plymouth

### A. Tema GRUB (WhiteSur)
```bash
cd ~
git clone https://github.com/vinceliuice/WhiteSur-grub-theme.git
cd WhiteSur-grub-theme
sudo ./install.sh -t all -s 1080p
```

### B. Boot Animation (Plymouth)
Menyembunyikan log kernel dengan logo *boot* kustom.
```bash
sudo apt install -y plymouth plymouth-themes
cd ~
git clone https://github.com/vinceliuice/WhiteSur-plymouth-theme.git
cd WhiteSur-plymouth-theme
sudo ./install.sh
sudo update-alternatives --config default.plymouth
sudo update-initramfs -u
```
*Terapkan pada GRUB kernel:*
Buka file `/etc/default/grub` (gunakan `sudo nvim /etc/default/grub`), lalu modifikasi baris ini:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 rd.udev.log_priority=3 vt.global_cursor_default=0"
```
Simpan dan jalankan: `sudo update-grub`.

---

## 7. Setup Development

Siapkan lingkungan komprehensif.

```bash
# Python (Virtual Environment & PySpark Readiness)
sudo apt install -y python3-venv python3-pip python3-dev
mkdir -p ~/Projects/my-app
cd ~/Projects/my-app
python3 -m venv venv
source venv/bin/activate

# Node.js (JavaScript / CSS framework)
sudo apt install -y nodejs npm

# SQL Tools
sudo apt install -y postgresql-client sqlite3
```
