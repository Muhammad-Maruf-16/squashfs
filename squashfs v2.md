# Cara Flash Armbian + SquashFS + OverlayFS ke STB B860H v2

Tutorial ini berlaku untuk dua versi firmware. Langkahnya sama, perbedaan ada di nama file,
tata letak partisi, dan beberapa angka. Bagian yang berbeda ditandai **[20.10]** dan **[21.08.1]**.

| | **20.10** | **21.08.1** |
|---|---|---|
| File image | `Armbian_20.10_Arm-64_focal_current_5.9.0.img.xz` | `Armbian_21.08.1_Amlogic-GXL_focal_current_5.10.60.img.xz` |
| Kernel | 5.9.0 | 5.10.60 |
| Partisi BOOT bawaan | 512 MB | **128 MB** |
| Awal partisi rootfs (sektor) | `1081344` | **`294913`** (selalu cek dengan `fdisk -l`) |
| File kernel di BOOT | `zImage` | **`Image`** |
| File initramfs | `initrd.img-5.9.0-arm-64` | **`initrd.img-5.10.60-amlogic-meson-lts`** |
| File u-boot yang dipakai | `u-boot-s905x-s912` | **`u-boot-old`** (ukuran sama, 650183 byte) |
| Edit `extlinux.conf` (Tahap 2) | Perlu (aktifkan blok amlogic) | **Tidak perlu** (sudah aktif) |
| Ukuran partisi squashfs (sdX2) | 700 MB | **1 GB** (disarankan) |
| Kompresi squashfs | `lz4` | `lz4` (kernel mendukung `CONFIG_SQUASHFS_LZ4`) |
| Audio analog | Tidak tersedia di DTB ini | **Uji dulu sebelum konversi** (lihat Tahap 3) |
| Nama device kartu SD di STB | `mmcblk0` | **Berubah tiap boot** (`mmcblk1`, `mmcblk2`, ...). Jangan menulis `/dev/mmcblkXpN` tetap, pakai `PARTUUID`/`LABEL` |
| Dengung di jack audio saat HDMI dicabut | - | Matikan video komposit: `video=Composite-1:d` (Tahap 3b) |
| Setelan Node-RED kembali ke default tiap reboot | - | Opsional, lihat **Tahap 3b** |

## Persiapan

**File yang dibutuhkan:** salah satu dari image di tabel atas.

**Tools:**
- Laptop Linux
- SD card minimal 8GB Class 10 (hindari UHS-I/U1/U3, pakai SanDisk atau Kingston biasa)
- `dd` atau balenaEtcher
- Package di laptop: `squashfs-tools`, `u-boot-tools`, `cpio`

> **Penting:** Setiap kali SD card dicabut dan dipasang kembali, nama device bisa berubah
> (misal dari `/dev/sdc` jadi `/dev/sdd`). Selalu cek dengan `lsblk` sebelum menjalankan
> perintah apapun yang melibatkan `/dev/sdX`. Jangan tertukar dengan SD card lama yang
> berisi data.

> **Penting (penyebab squashfs rusak di percobaan sebelumnya):** matikan automount desktop
> selama mengerjakan tutorial ini. Kalau tidak, desktop bisa me-mount partisi lama sebagai
> ext4 dan menimpa data yang baru ditulis dengan `dd`.
> ```bash
> gsettings set org.gnome.desktop.media-handling automount false
> gsettings set org.gnome.desktop.media-handling automount-open false
> # cek (harus false):
> gsettings get org.gnome.desktop.media-handling automount
> # setelah semua selesai, kembalikan:
> gsettings set org.gnome.desktop.media-handling automount true
> gsettings set org.gnome.desktop.media-handling automount-open true
> ```
> Jalankan **tanpa `sudo`** (dengan `sudo` pengaturan masuk ke akun root dan tidak berefek).
> Cabut-colok ulang SD card setelahnya, lalu pastikan kolom MOUNTPOINTS di `lsblk` kosong.

---

## Cek Device SD Card

Selalu jalankan ini dulu setiap kali SD card dipasang ke laptop:

```bash
lsblk
```

Cari device dengan ukuran 7.3G (atau sesuai ukuran SD card). Catat nama devicenya
(contoh: `sdc`, `sdd`). Ganti semua contoh `/dev/sdX` di tutorial ini dengan device yang sesuai.

---

## Tahap 1 — Flash Image Armbian ke SD Card

```bash
# Cek integritas image dulu (tidak boleh ada error)
xz -t <NAMA_IMAGE>.img.xz

# Sesuaikan /dev/sdX dengan hasil lsblk
sudo umount /dev/sdX1 /dev/sdX2 2>/dev/null

xz -dc <NAMA_IMAGE>.img.xz | sudo dd of=/dev/sdX bs=1M status=progress conv=fsync
sudo sync
```

Atau pakai **balenaEtcher** — pilih file `.img.xz`, pilih device SD card, flash.

---

## Tahap 2 — Setting DTB dan U-Boot untuk B860H

```bash
# Cek device SD card dulu
lsblk

sudo mount /dev/sdX1 /mnt
ls /mnt
```

### [20.10]

```bash
# Copy u-boot yang sesuai untuk S905X
sudo cp /mnt/u-boot-s905x-s912 /mnt/u-boot.ext

# Edit extlinux.conf
sudo nano /mnt/extlinux/extlinux.conf
```

Ubah bagian aktif jadi seperti ini:
- Tambah `#` di depan baris `FDT` dan `APPEND` yang rockchip (yang aktif)
- Hapus `#` di depan baris `FDT` dan `APPEND` yang amlogic

```
LABEL Armbian
LINUX /zImage
INITRD /uInitrd

FDT /dtb/amlogic/meson-gxl-s905x-p212.dtb
APPEND root=LABEL=ROOTFS rootflags=data=writeback rw console=ttyAML0,115200n8 ...
```

> Baris `APPEND` di nano bisa terlihat terpotong (tanda `>` di ujung). Itu hanya tampilan,
> isinya tetap utuh. Jangan ikut menyalin tanda `>` itu.

Simpan: **Ctrl+X → Y → Enter**

### [21.08.1]

File `u-boot-s905x-s912` **tidak ada** di image ini. Yang ada `u-boot.ext` (bawaan, 663582 byte) dan
`u-boot-old` (650183 byte, sama dengan u-boot yang terbukti jalan di 20.10).

```bash
sudo cp /mnt/u-boot.ext /mnt/u-boot.ext.shipped   # simpan bawaan image
sudo cp /mnt/u-boot-old /mnt/u-boot.ext           # pakai u-boot yang sudah terbukti
```

Kalau layar tetap hitam dengan `u-boot-old`, kembalikan yang bawaan:
`sudo cp /mnt/u-boot.ext.shipped /mnt/u-boot.ext`.

**`extlinux.conf` tidak perlu diedit.** Di image ini baris `FDT /dtb/amlogic/meson-gxl-s905x-p212.dtb`
dan `APPEND ... ttyAML0` sudah aktif (tanpa `#`), dan `LINUX /Image`. Cukup periksa dengan
`cat /mnt/extlinux/extlinux.conf`.

### Kedua versi

```bash
sudo umount /mnt
sudo sync
```

---

## Tahap 3 — Boot STB dari SD Card

1. Cabut SD card dari laptop
2. Pasang SD card ke slot STB sampai **klik**
3. Cabut power STB
4. Colok power STB kembali
5. Tunggu boot — STB otomatis baca SD card saat cold boot

> **Penting:** Harus cabut power (cold boot), bukan reboot biasa dari Android.
> Bootloader Amlogic hanya cek SD card saat cold boot.

SSH masuk setelah boot (login pertama `root` / `1234`, lalu buat user sesuai wizard):
```bash
ssh me@<IP_STB>
```

> Di tahap ini STB hanya dipakai untuk boot pertama (Armbian akan resize rootfs dan
> membuat user). Tidak perlu install apapun di STB, semua pembuatan SquashFS dilakukan
> di laptop pada tahap berikutnya.

### [21.08.1] Uji audio analog SEKARANG, sebelum konversi

Konversi ke squashfs memakan waktu. Pastikan dulu fitur yang dituju memang ada:

```bash
cat /proc/asound/cards
aplay -l
dmesg | grep -iE "audio|sound|acodec|aiu|hdmi-tx"
# colok speaker/headphone ke jack AV 3.5 mm, lalu:
speaker-test -c2 -t wav
```

- Kalau kartu analog muncul dan bersuara: lanjut ke Tahap 4.
- Kalau hanya HDMI: coba DTB lain di `/boot/dtb/amlogic/` (ubah baris `FDT` di
  `/boot/extlinux/extlinux.conf`, lalu reboot) sebelum menyerah.
- Kalau tidak ada yang berhasil: pertimbangkan USB sound card dan tetap di versi lama.

**Dengung/"gereng" di speaker saat kabel HDMI dicabut?** Sebabnya bukan ground. Saat HDMI dicabut,
kernel memindahkan tampilan ke output **video komposit** yang berbagi jack AV 3.5 mm dengan audio,
dan sinyal videonya bocor ke audio. Uji tanpa reboot (HDMI dicabut, noise sedang terdengar):

```bash
echo off | sudo tee /sys/class/drm/card0-Composite-1/status     # noise hilang = terbukti
# kembalikan: echo detect | sudo tee /sys/class/drm/card0-Composite-1/status
```

Perintah itu hanya sementara (hilang saat reboot). Cara permanen ada di **Tahap 3b**. HDMI tidak terpengaruh.
Dampaknya: video lewat jack AV tidak tersedia lagi (audio L/R tetap jalan).

---

## Tahap 3b — Siapkan sistem SEBELUM dibungkus (di STB, masih ext4)

Apapun yang terpasang atau diatur di STB sekarang akan **ikut terbungkus** ke squashfs sebagai
dasar sistem. Kerjakan semua ini sebelum Tahap 4, lalu matikan STB dengan `poweroff`.

**1. Zona waktu**
```bash
sudo timedatectl set-timezone Asia/Jakarta
```

**2. Matikan video komposit (mencegah dengung di jack audio)** — hanya kalau video lewat jack AV tidak dipakai.
Tambahkan `video=Composite-1:d` di akhir baris `APPEND` yang aktif di `/boot/extlinux/extlinux.conf`:
```bash
sudo cp /boot/extlinux/extlinux.conf /boot/extlinux/extlinux.conf.bak-composite
sudo sed -i '/^APPEND root=/s/$/ video=Composite-1:d/' /boot/extlinux/extlinux.conf
grep '^APPEND' /boot/extlinux/extlinux.conf          # pastikan ada di ujung
```

**3. Pasang aplikasi dan aktifkan layanannya** (contoh)
```bash
sudo apt update
sudo apt install -y mosquitto mosquitto-clients curl git build-essential python3
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
sudo systemctl enable --now nodered mosquitto
```

**4. Simpan volume audio**
```bash
sudo alsactl store
```

**5. (Opsional, disarankan) Node-RED selalu kembali ke setelan default saat reboot**

Tujuan: konfigurasi Node-RED yang sekarang jadi **default** (ikut terbungkus di squashfs). Selama STB hidup
Anda bebas mengubah, Deploy, atau memasang node. Setelah reboot semuanya kembali ke default. Cara kerjanya:
folder kerja Node-RED dipindah ke **RAM** dan diisi ulang dari salinan default setiap `nodered` start.
Bonus: Deploy tidak menulis ke SD card.

```bash
# salinan default (ikut terbungkus) + titik mount RAM
sudo mkdir -p /srv/nodered-userdir /usr/local/share/nodered-default
sudo rsync -a --delete --exclude='*.bak-*' --exclude='*.backup' --exclude='.npm' ~/.node-red/ /usr/local/share/nodered-default/
sudo chown -R me:me /usr/local/share/nodered-default      # ganti 'me' kalau user Anda berbeda

# skrip: isi ulang folder kerja dari default (dijalankan sebagai root tiap nodered start)
sudo tee /usr/local/sbin/nodered-reset-default > /dev/null << 'EOF'
#!/bin/sh
# Siapkan folder kerja Node-RED di RAM (tmpfs khusus) dan isi dari default, setiap nodered start.
# JANGAN pakai /dev/shm (dihapus systemd-logind saat sesi user logout) atau /run (noexec).
set -e
SRC=/usr/local/share/nodered-default
DST=/srv/nodered-userdir
U=$(id -u me); G=$(id -g me)
mountpoint -q "$DST" || mount -t tmpfs -o size=256m,mode=0755,uid=$U,gid=$G tmpfs "$DST"
find "$DST" -mindepth 1 -delete
cp -a "$SRC"/. "$DST"/
chown -R me:me "$DST"
EOF

# skrip: jadikan isi folder kerja SAAT INI sebagai default baru (tersimpan permanen di overlay)
sudo tee /usr/local/sbin/nodered-save-default > /dev/null << 'EOF'
#!/bin/sh
set -e
[ -d /srv/nodered-userdir/lib ] || { echo "folder kerja kosong, nodered belum jalan?"; exit 1; }
rsync -a --delete --exclude='*.backup' /srv/nodered-userdir/ /usr/local/share/nodered-default/
echo "Default baru tersimpan di /usr/local/share/nodered-default"
EOF
sudo chmod 755 /usr/local/sbin/nodered-reset-default /usr/local/sbin/nodered-save-default

# sambungkan ke layanan nodered
sudo mkdir -p /etc/systemd/system/nodered.service.d
sudo tee /etc/systemd/system/nodered.service.d/default-userdir.conf > /dev/null << 'EOF'
[Service]
ExecStartPre=+/usr/local/sbin/nodered-reset-default
Environment="NODE_RED_OPTIONS=--userDir /srv/nodered-userdir"
EOF
sudo systemctl daemon-reload
sudo systemctl restart nodered
```

Uji (harus sesuai komentar):
```bash
journalctl -u nodered -b --no-pager | grep "User directory"    # /srv/nodered-userdir
findmnt /srv/nodered-userdir                                    # tmpfs
# ubah sesuatu di editor lalu Deploy, kemudian:
sudo systemctl restart nodered                                  # perubahan hilang = kembali ke default
```

Catatan penting:
- Kenapa bukan `/dev/shm`: `systemd-logind` (`RemoveIPC=yes`) **menghapus isi `/dev/shm` milik user** saat sesi
  terakhirnya logout, jadi folder kerja lenyap ketika Anda menutup SSH. Kenapa bukan `/run`: ter-mount `noexec`,
  sehingga modul native (mis. `serialport`) gagal dimuat (`failed to map segment from shared object`).
- Mengubah default: edit di editor sampai sesuai, lalu `sudo nodered-save-default`. Hasilnya tersimpan di overlay
  (bertahan antar reboot), bukan di squashfs dasarnya.
- Membatalkan mekanisme ini: `sudo rm /etc/systemd/system/nodered.service.d/default-userdir.conf`, lalu
  `sudo systemctl daemon-reload && sudo systemctl restart nodered` (Node-RED kembali memakai `~/.node-red`).

**6. Bersihkan dan matikan dengan benar**
```bash
sudo apt clean
sudo rm -rf /var/lib/apt/lists/* ~/.npm /tmp/*
sudo journalctl --vacuum-size=1M
history -c
sudo poweroff          # tunggu lampu mati total. Jangan cabut power saat masih menyala
```

> Ukuran terpakai root biasanya sekitar 2 GB setelah pembersihan. Hasil squashfs (lz4) sekitar 1 GB,
> jadi untuk 21.08.1 siapkan partisi sdX2 sebesar `+1G` atau `+1500M`. Selalu cek `ls -lh rootfs.squashfs`.

---

## Tahap 4 — Buat File SquashFS dari Laptop

Pasang SD card ke laptop, cek device:

```bash
lsblk
```

```bash
# Pastikan sdX2 tidak ter-mount, lalu backup rootfs dulu (opsional tapi disarankan)
sudo umount /dev/sdX2 2>/dev/null
sudo dd if=/dev/sdX2 of=~/Documents/rootfs_backup.img bs=4M status=progress

# Baru mount READ-ONLY untuk dipack
sudo mount -o ro /dev/sdX2 /mnt

# Pack jadi SquashFS dengan kompresi lz4
sudo mksquashfs /mnt ~/Documents/rootfs.squashfs -comp lz4 -b 131072 -noappend

sudo umount /mnt

# Pastikan ukurannya MUAT di partisi squashfs (20.10: 700M, 21.08.1: 1G)
ls -lh ~/Documents/rootfs.squashfs
```

> Jangan pakai `-comp lz4` kalau kernel tidak mendukung. Cek dulu di config kernel
> (`grep SQUASHFS_LZ4 /media/$USER/BOOT/config-*`). Kedua versi di tutorial ini mendukung.

---

## Tahap 5 — Partisi Ulang SD Card

SD card perlu 3 partisi:
- sdX1 → BOOT (FAT32) — sudah ada, jangan dihapus
- sdX2 → SquashFS — rootfs read-only (**20.10: 700MB, 21.08.1: 1GB**)
- sdX3 → OVERLAY (sisa kartu, ±6GB) — writable persistent

```bash
# Unmount semua partisi SD card dulu sebelum fdisk
sudo umount /dev/sdX1 /dev/sdX2 /dev/sdX3 2>/dev/null

# CATAT dulu kolom "Start" milik sdX2 (ini sektor awal yang akan dipakai lagi)
sudo fdisk -l /dev/sdX
```

> **Sektor awal sdX2 berbeda per versi.** [20.10] = `1081344`, [21.08.1] = `294913`.
> Selalu ambil dari output `fdisk -l` di atas, jangan mengetik dari ingatan.

> **Catat juga `Disk identifier`** dari output `fdisk -l` (contoh `0x16520000`). Nilainya **tidak berubah**
> setelah partisi dibuat ulang dengan `fdisk`, dan dipakai di Tahap 6 untuk `root=PARTUUID=<ID>-02`
> (partisi 2) dan `<ID>-03` (partisi 3). Ini pengganti nama device `/dev/mmcblkXpN` yang berubah tiap boot.
> Setelah boot, cek dengan `blkid`.

```bash
sudo fdisk /dev/sdX
```

Langkah fdisk (ganti `<START>` dan `<UKURAN>` sesuai tabel):
```
d → 2 → Enter                          (hapus sdX2)
n → p → 2 → <START> → <UKURAN> → Enter
Y                                       (HAPUS signature ext4 lama. Jangan jawab N!)
p                                       (CATAT angka End sector sdX2)
n → p → 3 → (End sdX2 + 1) → Enter
p                                       (verifikasi sdX3 sekitar 6GB)
w                                       (simpan)
```

| | `<START>` | `<UKURAN>` |
|---|---|---|
| [20.10] | `1081344` | `+700M` |
| [21.08.1] | `294913` | `+1G` |

> **Catatan penting fdisk:**
> - First sector sdX3 = End sector sdX2 + 1 → **lihat dari output `p` sebelum buat sdX3**
> - Jangan ketik angka sembarang untuk first sector sdX3
> - fdisk bisa memperingatkan sektor awal tidak rata (`294913`). Itu aman, lanjutkan.
> - **Jawab `Y` (hapus signature), bukan `N`.** Kalau signature ext4 lama dibiarkan,
>   desktop mengenali sdX2 sebagai ext4 dan otomatis me-mount-nya. Saat `dd` menulis
>   squashfs, ext4 yang ter-mount ikut meng-update superblock-nya di byte 1024-2047
>   dan merusak blok pertama squashfs. Akibatnya file kecil seperti `/etc/apt/*`
>   tidak bisa dibaca dan `apt update` hang (CPU 100%, tanpa output).

Bersihkan signature, format overlay, lalu tulis squashfs:

```bash
# Pastikan tidak ada yang ter-mount (MOUNTPOINTS sdX2 dan sdX3 harus kosong)
sudo umount /dev/sdX1 /dev/sdX2 /dev/sdX3 2>/dev/null
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS /dev/sdX

# Hapus sisa signature filesystem lama di sdX2
sudo wipefs -a /dev/sdX2

# Format partisi overlay sebagai ext4
sudo mkfs.ext4 -L OVERLAY /dev/sdX3

# Tulis squashfs ke sdX2 (conv=fsync agar benar-benar tertulis)
sudo dd if=~/Documents/rootfs.squashfs of=/dev/sdX2 bs=4M conv=fsync status=progress
sudo sync
```

**Verifikasi WAJIB sebelum lanjut** (jangan hanya percaya pesan `dd` sukses):

```bash
# Buang cache supaya yang dibandingkan benar-benar isi SD card
echo 3 | sudo tee /proc/sys/vm/drop_caches

# Harus mencetak IDENTIK
sudo cmp -n $(stat -c%s ~/Documents/rootfs.squashfs) ~/Documents/rootfs.squashfs /dev/sdX2 && echo IDENTIK

# Pastikan sdX2 tidak ter-mount oleh siapapun
lsblk -o NAME,FSTYPE,MOUNTPOINTS /dev/sdX     # sdX2 harus bertipe squashfs, tanpa mountpoint
```

Kalau `cmp` melaporkan perbedaan, **jangan lanjut**: ulangi `wipefs` dan `dd`.

---

## Tahap 6 — Setup OverlayFS via Initramfs

Nama file initramfs berbeda per versi. Cari dulu namanya:

```bash
sudo mkdir -p /media/$USER/BOOT
sudo mount /dev/sdX1 /media/$USER/BOOT
ls /media/$USER/BOOT
```

| | File initramfs |
|---|---|
| [20.10] | `initrd.img-5.9.0-arm-64` |
| [21.08.1] | `initrd.img-5.10.60-amlogic-meson-lts` |

```bash
# Simpan nama initramfs yang SEBENARNYA ada di variabel
INITRD=$(ls /media/$USER/BOOT/initrd.img-* | head -1); echo "$INITRD"

# Pastikan formatnya gzip (kalau bukan gzip, perintah zcat di bawah tidak berlaku)
file "$INITRD"

# Ekstrak initramfs (tanpa menyembunyikan error)
sudo rm -rf /tmp/initrd2
mkdir -p /tmp/initrd2
cd /tmp/initrd2
sudo sh -c 'zcat "$1" | cpio -idmv' sh "$INITRD"
```

Buat script overlay (pakai `sudo tee` karena hasil ekstrak dimiliki root).

> **Kenapa skrip ini mencari partisi lewat LABEL.** Di kernel 5.10 urutan `mmcblk*` berubah antar boot
> (root pernah di `mmcblk1p2`, lalu `mmcblk2p2`). Skrip lama yang menulis `/dev/mmcblk0p3` tetap akan gagal
> mount overlay di boot tertentu, dan STB boot tanpa overlay (root `squashfs ro`, paket baru "hilang").
> Skrip di bawah memakai `/dev/disk/by-label/OVERLAY` (label dibuat saat `mkfs.ext4 -L OVERLAY` di Tahap 5).
>
> Dasar teknis yang sudah diperiksa di initramfs 21.08.1: `blkid`, `readlink`, dan `sleep` (pecahan detik)
> tersedia, dan initramfs sudah menangani `root=PARTUUID=`/`LABEL=` (terbukti: `root=LABEL=ROOTFS` jalan).
> **Status:** versi LABEL ini belum pernah dijalankan end-to-end sampai boot. Versi lama dengan
> `/dev/mmcblk0p3` hanya terbukti di 20.10. Kalau overlay tidak aktif setelah boot, STB tetap bisa di-SSH
> (lihat Troubleshooting) dan datanya aman.

```bash
sudo tee /tmp/initrd2/scripts/init-bottom/overlay > /dev/null << 'EOF'
#!/bin/sh
PREREQ=""
prereqs() { echo "$PREREQ"; }
case $1 in prereqs) prereqs; exit 0;; esac
. /scripts/functions
OVERLAY="no"
for arg in $(cat /proc/cmdline); do
  case "$arg" in
  overlay=*) OVERLAY=${arg#overlay=} ;;
  esac
done
if [ "$OVERLAY" != "yes" ]; then
  log_warning_msg "overlay not enabled"
  exit 0
fi
log_begin_msg "Mounting overlay on ext4 partition"
# Cari partisi overlay lewat LABEL (nama mmcblkX berubah tiap boot). Tunggu udev maks ±10 detik.
OVDEV=""
i=0
while [ $i -lt 20 ]; do
  if [ -e /dev/disk/by-label/OVERLAY ]; then
    OVDEV=$(readlink -f /dev/disk/by-label/OVERLAY)
    break
  fi
  sleep 0.5
  i=$((i + 1))
done
if [ -z "$OVDEV" ]; then
  log_failure_msg "partisi dengan LABEL=OVERLAY tidak ditemukan"
  exit 0
fi
mkdir -p /rw
if ! mount -t ext4 "$OVDEV" /rw; then
  log_failure_msg "can't mount ext4 overlay partition"
  exit 0
fi
mkdir -p /rw/upper /rw/work /ro
if ! mount -o move "${rootmnt}" /ro; then
  log_failure_msg "can't move root fs to /ro"
  exit 0
fi
if ! mount -t overlay -o lowerdir=/ro,upperdir=/rw/upper,workdir=/rw/work overlay-root "${rootmnt}"; then
  log_failure_msg "can't mount overlayfs"
  mount -o move /ro "${rootmnt}"
  exit 0
fi
mkdir -p "${rootmnt}/ro" "${rootmnt}/rw"
mount -o move /ro "${rootmnt}/ro"
mount -o move /rw "${rootmnt}/rw"
log_end_msg
exit 0
EOF

sudo chmod +x /tmp/initrd2/scripts/init-bottom/overlay
```

Daftarkan ke ORDER. **Lihat dulu isi ORDER aslinya, jangan menimpa buta** (isinya bisa beda per versi):

```bash
cat /tmp/initrd2/scripts/init-bottom/ORDER

# Tambahkan baris overlay di PALING ATAS, hanya kalau belum ada
grep -q 'init-bottom/overlay' /tmp/initrd2/scripts/init-bottom/ORDER || \
  sudo sed -i '1i /scripts/init-bottom/overlay "$@"' /tmp/initrd2/scripts/init-bottom/ORDER

cat /tmp/initrd2/scripts/init-bottom/ORDER     # baris pertama harus overlay
```

> Hasil akhir ORDER di 20.10 yang sudah terbukti jalan:
> ```
> /scripts/init-bottom/overlay "$@"
> /scripts/init-bottom/udev "$@"
> [ -e /conf/param.conf ] && . /conf/param.conf
> ```

Repack initramfs (pakai `sudo` supaya owner file tetap root):

```bash
sudo apt install u-boot-tools

cd /tmp/initrd2
sudo sh -c 'find . | cpio -o -H newc | gzip > /tmp/uInitrd-new.gz'
mkimage -A arm64 -T ramdisk -C gzip -n uInitrd -d /tmp/uInitrd-new.gz /tmp/uInitrd-new

# Copy ke partisi BOOT (masih ter-mount dari langkah di atas)
sudo cp /tmp/uInitrd-new /media/$USER/BOOT/uInitrd

sudo nano /media/$USER/BOOT/extlinux/extlinux.conf
```

Ubah baris `APPEND` yang **aktif** (satu baris penuh). Ganti `root=LABEL=ROOTFS ... rw` jadi squashfs read-only,
dan tambahkan parameter overlay di ujung:

**[21.08.1]** (pakai `PARTUUID`, **bukan** `/dev/mmcblk...`; ganti `16520000` dengan `Disk identifier` kartu Anda):

```
APPEND root=PARTUUID=16520000-02 rootfstype=squashfs ro console=ttyAML0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 overlay=yes overlay.index=0 overlay.nfs_export=0 video=Composite-1:d
```

**[20.10]** (terbukti jalan di 20.10; boleh juga memakai bentuk `PARTUUID` di atas):

```
APPEND root=/dev/mmcblk0p2 rootfstype=squashfs ro console=ttyAML0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 overlay=yes overlay.index=0 overlay.nfs_export=0
```

> **Bawa semua parameter kustom dari baris lama.** Baris `APPEND` ditulis ulang, jadi parameter yang sudah
> Anda tambahkan di Tahap 3b (contoh `video=Composite-1:d`) harus ikut ditulis lagi. Kalau terlewat,
> dengung di jack audio saat HDMI dicabut muncul lagi.

> **Wajib: `overlay.index=0 overlay.nfs_export=0`.** Tanpa dua parameter ini overlayfs memakai
> fitur `index`. Setelah instalasi besar (contoh: Node.js 22 + Node-RED, yang menghapus dan
> memasang ulang ribuan file) indeks itu bisa menyisakan entri yatim. Saat reboot berikutnya
> overlay GAGAL dimount (`dmesg`: `orphan index entry`, `failed index dir cleanup (-5)`).
> Skrip initramfs lalu lanjut tanpa overlay, sehingga STB boot dengan root squashfs
> read-only dan semua paket yang baru dipasang terlihat "hilang".
> Datanya sebenarnya tetap utuh di `mmcblk0p3`. Dengan `index=0` masalah ini tidak terjadi.
> Parameter ini berlaku di [20.10] karena overlayfs kernel 5.9 built-in dengan fitur index.
> Di [21.08.1] `CONFIG_OVERLAY_FS_INDEX` tidak terdaftar di config kernel, jadi parameter ini
> kemungkinan tidak berpengaruh. Tidak berbahaya, tetap boleh dipasang.

```bash
sudo umount /media/$USER/BOOT
sudo sync
```

> **Catatan:** Setelah copy uInitrd, verifikasi `overlay=yes`, `overlay.index=0`, dan
> `overlay.nfs_export=0` sudah ada di `extlinux/extlinux.conf` bagian APPEND.
> Kalau belum ada, mount lagi dan edit.

> **Peringatan upgrade kernel:** skrip overlay ini hanya ada di `uInitrd` hasil edit manual.
> Kalau suatu saat `apt upgrade` memasang kernel baru, initramfs dibuat ulang dan hook overlay
> **hilang** (STB boot tanpa overlay). Sebelum upgrade kernel, taruh skrip yang sama di
> `/etc/initramfs-tools/scripts/init-bottom/overlay` (`chmod +x`) supaya ikut masuk setiap
> regenerasi. (Cara ini belum diuji di setup ini, uji dulu di SD card cadangan.)

> **SSH warning "REMOTE HOST IDENTIFICATION HAS CHANGED"** setelah flash SD card baru
> adalah normal. Hapus host key lama dengan:
> ```bash
> ssh-keygen -R <IP_STB>
> ```
> Lalu SSH lagi seperti biasa.

---

## Verifikasi Hasil

Setelah boot, SSH masuk dan jalankan:

```bash
# Cek overlay aktif
mount | grep " / "
```

Hasilnya harus:
```
overlay-root on / type overlay (rw,relatime,lowerdir=/ro,upperdir=/rw/upper,workdir=/rw/work)
```

```bash
# Cek squashfs di bawahnya
mount | grep squash
```

Hasilnya harus:
```
/dev/mmcblk0p2 on /ro type squashfs (ro,relatime)
```

```bash
# Cek parameter kernel lengkap (harus ada overlay=yes overlay.index=0 overlay.nfs_export=0,
# dan [21.08.1] root=PARTUUID=... serta video=Composite-1:d)
cat /proc/cmdline

# Partisi overlay ter-mount di /rw (nama mmcblk boleh berbeda tiap boot, itu normal)
findmnt /rw /ro

# [21.08.1] Komposit mati dan Node-RED memakai folder default di RAM
cat /sys/class/drm/card0-Composite-1/status        # disconnected
findmnt /srv/nodered-userdir                       # tmpfs
journalctl -u nodered -b --no-pager | grep "User directory"

# Cek TIDAK ADA masalah overlay (harus mencetak 0)
dmesg | grep -ciE "orphan index|failed index"

# Cek TIDAK ADA error squashfs (harus mencetak 0)
dmesg | grep -ci "squashfs error"

# Cek apt benar-benar jalan (harus selesai dalam hitungan detik-menit, bukan hang)
sudo timeout 300 apt update
```

Kalau `dmesg` menampilkan `SQUASHFS error` atau `apt update` hang tanpa output,
squashfs di sdX2 rusak. Ulangi Tahap 5 dengan verifikasi `cmp`.

**[21.08.1]** Cek audio sekali lagi di sistem akhir (`aplay -l`, `speaker-test -c2`).

---

## Uji Coba Persistent Storage

Setelah boot dengan overlay aktif, test apakah file tersimpan persistent:

```bash
# Buat file test
touch ~/testfile
ls ~/
```

Reboot STB:
```bash
sudo reboot
```

SSH masuk lagi setelah boot:
```bash
ssh me@<IP_STB>
```

Cek file masih ada:
```bash
ls ~/
```

> File yang dibuat tersimpan di partisi OVERLAY (mmcblk0p3) dan survive reboot.
> Base system squashfs tidak bisa corrupt karena read-only. Tapi partisi OVERLAY
> (ext4) tetap bisa rusak kalau power dicabut paksa saat sedang menulis, jadi
> sesekali cadangkan isi `/rw`.

---

## Pindah dari 20.10 ke 21.08.1 (data aplikasi)

Jangan restore seluruh `/rw/upper` lama ke sistem baru. Itu mencampur file 20.10 dan 21.08 dan bisa
merusak. Sebaliknya: pasang ulang paket di sistem baru (`apt install mosquitto ...`, Node-RED), lalu
kembalikan hanya data aplikasinya (contoh: `~/.node-red/flows.json`, `~/.node-red/settings.js`,
`/etc/mosquitto/`). Cadangkan data lama dulu (partisi sdX3 → `dd`/`tar`) sebelum SD card lama dipakai ulang.

---

## Troubleshooting

**Layar hitam total:**
- Pastikan `u-boot.ext` sudah dicopy dari file u-boot yang benar ([20.10] `u-boot-s905x-s912`, [21.08.1] `u-boot-old`)
- Pastikan DTB di `extlinux.conf` sudah mengarah ke amlogic, bukan rockchip
- Coba cabut pasang SD card, pastikan masuk sampai klik
- [21.08.1] Kalau masih hitam, coba `u-boot.ext.shipped` (u-boot bawaan image)

**Error `-110` looping (SD card tidak terbaca):**
- Flash ulang image ke SD card
- Coba SD card berbeda (SanDisk/Kingston Class 10 non-UHS lebih kompatibel)
- SD card UHS-I/U1/U3 kadang tidak kompatibel dengan bootloader STB

**Boot masuk ke Android, bukan Armbian:**
- Pastikan cold boot (cabut power, bukan reboot)
- Pastikan SD card terpasang sebelum colok power

**Overlay tidak aktif (masih squashfs):**
- Pastikan `overlay=yes` ada di APPEND extlinux.conf
- Pastikan script overlay ada di ORDER initramfs
- Cek dengan: `cat /proc/cmdline`

**Setelah instalasi besar lalu reboot, paket/aplikasi "hilang" (`/` bertipe `squashfs ro`, `Usage of /: 100%`, `mkdir /rw` gagal "Read-only file system"):**
- Datanya TIDAK hilang, hanya overlay gagal dimount saat boot. Data masih ada di `/dev/mmcblk0p3`.
- Cek: `mount | grep " / "` (seharusnya `overlay-root`) dan `dmesg | grep -iE "overlayfs|orphan"`
- Gejala di `dmesg`: `orphan index entry`, `workdir/#N already exists`, `failed index dir cleanup (-5)`
- Perbaikan (bisa lewat SSH, root squashfs-nya tetap boot): edit `/boot/extlinux/extlinux.conf`, tambahkan
  `overlay.index=0 overlay.nfs_export=0` di akhir baris `APPEND` (backup dulu), lalu `sudo reboot`.
  `/boot` adalah partisi FAT yang bisa ditulis walau root sedang read-only.
- Jangan menginstal apapun selama overlay belum aktif: tulisan masuk ke tmpfs dan hilang saat reboot.
- Jangan format `mmcblk0p3`: datanya masih utuh.

**mkfs.ext4 gagal "apparently in use":**
- Itu tanda desktop me-mount partisi SD card secara otomatis. Matikan automount (lihat bagian Persiapan)
- Umount semua partisi SD card dulu: `sudo umount /dev/sdX1 /dev/sdX2 /dev/sdX3`
- Cabut pasang SD card lalu coba lagi
- Cek nama device baru dengan `lsblk`

**`apt update` hang tanpa output, CPU 100%, atau `Input/output error` saat baca file di `/etc`:**
- Gejala squashfs rusak. Cek `dmesg | grep -i squashfs` (cari `lz4 decompression failed`)
- Penyebab umum: sdX2 ter-mount sebagai ext4 saat `dd`, atau signature ext4 lama tidak dihapus
- Perbaikan: ulangi Tahap 5 (jawab `Y` di fdisk, `wipefs -a`, `dd`, lalu verifikasi `cmp`)
- Bukan masalah jaringan, jadi jangan habiskan waktu di DNS/mirror

**`cp: cannot stat '/mnt/u-boot-s905x-s912'` ([21.08.1]):**
- File itu hanya ada di image 20.10. Di 21.08.1 pakai `u-boot-old` (lihat Tahap 2)
- Selalu `ls /mnt` dulu untuk melihat file yang benar-benar ada

**Overlay gagal mount di boot tertentu (kadang jalan, kadang tidak) / `dmesg` tidak menemukan partisi:**
- Penyebab: nama `/dev/mmcblkXpN` berubah antar boot. Jangan menulis nama tetap di skrip overlay atau `APPEND`.
- Perbaikan: `root=PARTUUID=<DiskID>-02` di `APPEND`, dan skrip overlay mencari partisi lewat `LABEL=OVERLAY`
  (lihat Tahap 6). Cek ID disk dengan `sudo blkid` atau `sudo fdisk -l /dev/<disk>`.

**Dengung/"gereng" di jack audio hanya saat HDMI dicabut:**
- Output video komposit ikut aktif dan bocor ke audio. Tambahkan `video=Composite-1:d` di `APPEND` (Tahap 3b)
- Uji cepat: `echo off | sudo tee /sys/class/drm/card0-Composite-1/status` (sementara)

**Node-RED di folder RAM: perubahan hilang saat Anda logout SSH, atau modul `serialport` error `failed to map segment`:**
- Jangan pakai `/dev/shm` (dihapus `systemd-logind` saat sesi user terakhir logout) dan jangan `/run` (`noexec`).
- Pakai tmpfs yang di-mount sendiri seperti di Tahap 3b (`/srv/nodered-userdir`)

**Node-RED tidak kembali ke default setelah reboot / tidak mau start:**
- Cek drop-in: `systemctl cat nodered | tail -8` dan `journalctl -u nodered -b --no-pager | tail -30`
- Batalkan sementara: `sudo rm /etc/systemd/system/nodered.service.d/default-userdir.conf` lalu
  `sudo systemctl daemon-reload && sudo systemctl restart nodered`

**Default login Armbian:**
```
user: root
pass: 1234
```
(akan diminta ganti password saat pertama login)
