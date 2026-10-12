# Cara Flash Armbian + SquashFS + OverlayFS ke STB B860H v2

## Persiapan

**File yang dibutuhkan:**
- `Armbian_20.10_Arm-64_focal_current_5.9.0.img.xz`

**Tools:**
- Laptop Linux
- SD card minimal 8GB Class 10 (hindari UHS-I/U1/U3, pakai SanDisk atau Kingston biasa)
- `dd` atau balenaEtcher
- Package: `squashfs-tools`, `u-boot-tools`

> **Penting:** Setiap kali SD card dicabut dan dipasang kembali, nama device bisa berubah
> (misal dari `/dev/sdc` jadi `/dev/sdd`). Selalu cek dengan `lsblk` sebelum menjalankan
> perintah apapun yang melibatkan `/dev/sdX`.

---

## Cek Device SD Card

Selalu jalankan ini dulu setiap kali SD card dipasang ke laptop:

```bash
lsblk
```

Cari device dengan ukuran 7.3G (atau sesuai ukuran SD card). Catat nama devicenya
(contoh: `sdc`, `sdd`). Ganti semua contoh `/dev/sdc` di tutorial ini dengan device yang sesuai.

---

## Tahap 1 — Flash Image Armbian ke SD Card

```bash
# Sesuaikan /dev/sdX dengan hasil lsblk
sudo umount /dev/sdX1 /dev/sdX2

xz -dc Armbian_20.10_Arm-64_focal_current_5.9.0.img.xz | sudo dd of=/dev/sdX bs=1M status=progress conv=fsync
sudo sync
```

Atau pakai **balenaEtcher** — pilih file `.img.xz`, pilih device SD card, flash.

---

## Tahap 2 — Setting DTB dan U-Boot untuk B860H

```bash
# Cek device SD card dulu
lsblk

sudo mount /dev/sdX1 /mnt

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
APPEND root=/dev/mmcblk0p2 rootfstype=squashfs ro console=ttyAML0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 overlay=yes
```

Simpan: **Ctrl+X → Y → Enter**

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

SSH masuk setelah boot:
```bash
ssh me@<IP_STB>
```

---

## Tahap 4 — Install squashfs-tools di STB

```bash
sudo apt update && sudo apt install -y squashfs-tools
```

---

## Tahap 5 — Buat File SquashFS dari Laptop

Pasang SD card ke laptop, cek device:

```bash
lsblk
```

```bash
# Mount partisi rootfs (sdX2)
sudo mount /dev/sdX2 /mnt

# Backup rootfs dulu (opsional tapi disarankan)
sudo dd if=/dev/sdX2 of=~/Documents/rootfs_backup.img bs=4M status=progress

# Pack jadi SquashFS dengan kompresi lz4
sudo mksquashfs /mnt ~/Documents/rootfs.squashfs -comp lz4 -b 131072 -noappend

sudo umount /mnt
```

---

## Tahap 6 — Partisi Ulang SD Card

SD card perlu 3 partisi:
- sdX1 → BOOT (512MB, FAT32) — sudah ada, jangan dihapus
- sdX2 → SquashFS (700MB) — rootfs read-only
- sdX3 → OVERLAY (sisa ~6GB) — writable persistent

```bash
# Unmount BOOT dulu sebelum fdisk
sudo umount /media/$USER/BOOT

sudo fdisk /dev/sdX
```

Langkah fdisk:
```
d → 2 → Enter                     (hapus sdX2)
n → p → 2 → 1081344 → +700M → Enter
N                                  (jangan hapus squashfs signature)
p                                  (CATAT angka End sector sdX2)
n → p → 3 → (End sdX2 + 1) → Enter
p                                  (verifikasi sdX3 sekitar 6GB)
w                                  (simpan)
```

> **Catatan penting fdisk:**
> - First sector sdX2 = `1081344` (tepat setelah sdX1 512MB, selalu sama)
> - First sector sdX3 = End sector sdX2 + 1 → **lihat dari output `p` sebelum buat sdX3**
> - Jangan ketik angka sembarang untuk first sector sdX3

```bash
# Format partisi overlay sebagai ext4
sudo umount /media/$USER/BOOT 2>/dev/null
sudo mkfs.ext4 -L OVERLAY /dev/sdX3

# DD squashfs ke sdX2
sudo dd if=~/Documents/rootfs.squashfs of=/dev/sdX2 bs=4M status=progress
sudo sync
```

---

## Tahap 7 — Setup OverlayFS via Initramfs

```bash
# Mount BOOT dulu kalau belum ter-mount
sudo mkdir -p /media/$USER/BOOT
sudo mount /dev/sdX1 /media/$USER/BOOT

# Ekstrak initramfs
mkdir -p /tmp/initrd2
cd /tmp/initrd2
sudo zcat /media/$USER/BOOT/initrd.img-5.9.0-arm-64 | cpio -idmv 2>/dev/null
```

Buat script overlay:

```bash
cat > /tmp/initrd2/scripts/init-bottom/overlay << 'EOF'
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
mkdir -p /rw
if ! mount -t ext4 /dev/mmcblk0p3 /rw; then
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

chmod +x /tmp/initrd2/scripts/init-bottom/overlay
```

Tambahkan ke ORDER:

```bash
cat > /tmp/initrd2/scripts/init-bottom/ORDER << 'EOF'
/scripts/init-bottom/overlay "$@"
/scripts/init-bottom/udev "$@"
[ -e /conf/param.conf ] && . /conf/param.conf
EOF
```

Repack initramfs:

```bash
sudo apt install u-boot-tools

cd /tmp/initrd2
find . | cpio -o -H newc | gzip > /tmp/uInitrd-new.gz
mkimage -A arm64 -T ramdisk -C gzip -n uInitrd -d /tmp/uInitrd-new.gz /tmp/uInitrd-new

# Copy ke partisi BOOT
# Cek device SD card dulu
lsblk | grep sd

# Mount BOOT (sesuaikan sdX1 dengan hasil lsblk)
sudo mkdir -p /mnt
sudo mount /dev/sdX1 /mnt
sudo cp /tmp/uInitrd-new /mnt/uInitrd

sudo nano /mnt/extlinux/extlinux.conf
APPEND root=/dev/mmcblk0p2 rootfstype=squashfs ro console=ttyAML0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 overlay=yes

sudo umount /mnt
sudo sync


```

> **Catatan:** Setelah copy uInitrd, verifikasi `overlay=yes` sudah ada di
> `/mnt/extlinux/extlinux.conf` bagian APPEND. Kalau belum ada, mount lagi dan edit.

> **SSH warning "REMOTE HOST IDENTIFICATION HAS CHANGED"** setelah flash SD card baru
> adalah normal. Hapus host key lama dengan:
> ```bash
> ssh-keygen -f '~/.ssh/known_hosts' -R '<IP_STB>'
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
> Cabut power paksa pun aman karena base system squashfs tidak bisa corrupt.

---

## Troubleshooting

**Layar hitam total:**
- Pastikan `u-boot.ext` sudah dicopy dari `u-boot-s905x-s912`
- Pastikan DTB di `extlinux.conf` sudah mengarah ke amlogic, bukan rockchip
- Coba cabut pasang SD card, pastikan masuk sampai klik

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

**mkfs.ext4 gagal "apparently in use":**
- Umount semua partisi SD card dulu: `sudo umount /dev/sdX1 /dev/sdX2 /dev/sdX3`
- Cabut pasang SD card lalu coba lagi
- Cek nama device baru dengan `lsblk`

**Default login Armbian:**
```
user: root
pass: 1234
```
(akan diminta ganti password saat pertama login)

