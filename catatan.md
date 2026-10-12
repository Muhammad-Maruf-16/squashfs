# Catatan: `apt update` hang di STB B860H v2 (Armbian + SquashFS + OverlayFS)

Tanggal: 2026-10-12 · STB: `me@192.168.24.48` · Armbian 20.10 Focal ARM64

## Ringkasan

| | |
|---|---|
| **Gejala** | `sudo apt update` hang tanpa output, load average 1.0 padahal idle. Juga muncul `/etc/bash.bashrc: Input/output error` saat SSH |
| **Penyebab** | 1 KB pertama (byte 1024–2047) partisi SquashFS `/dev/mmcblk0p2` tertimpa superblock ext4 saat proses flash |
| **Bukan** | Jaringan, permission user `_apt`, atau SD card rusak |
| **Status** | **Selesai.** Partisi identik dengan `rootfs.squashfs` asli, `apt update` normal, 0 error SquashFS |

## 1. Diagnosa

1. `apt-get update` dalam status `R` (CPU 100%), bukan menunggu jaringan. File yang sedang dibuka: `/etc/apt/apt.conf.d/01-vendor-ubuntu`. Apt berputar terus saat membaca config yang menghasilkan EIO.
2. `dmesg`: `SQUASHFS error: lz4 decompression failed, data probably corrupt` dan `Failed to read block 0x6a: -5`.
3. Blok mentah (offset `0x6a`, 10.966 byte) saya ambil dan dekompres di PC. Stream LZ4 valid sampai ±1,6 KB pertama lalu rusak di byte ke-935. Dibaca 3 kali hasilnya identik, jadi korupnya permanen, bukan error baca sesaat.
4. Blok itu adalah fragment block pertama, berisi ekor 140 file kecil (115 di antaranya masih aktif di tampilan overlay): `/etc/apt/*`, `/etc/bash.bashrc`, `trusted.gpg*`, `/etc/NetworkManager/*`, `/etc/cron.*`, `/etc/console-setup/*`, dst. Inode dan ukuran file masih utuh, hanya isinya tidak terbaca.
5. Jaringan normal: `apt update` terisolasi mengunduh 18 MB dalam 14 detik.

## 2. Akar masalah (dibuktikan)

Membandingkan `~/Documents/rootfs.squashfs` (asli) dengan isi partisi di STB:

- md5 seluruh isi **setelah 4096 byte pertama identik**. Artinya `mksquashfs` dan `dd` benar.
- Hanya **82 byte** berbeda, **semuanya di byte 1024–2047** partisi, yaitu lokasi superblock ext4.
- Field yang berubah: waktu tulis (`s_wtime`), jumlah blok/inode bebas, `s_state`, dll. Magic ext4 tidak tertulis. Itu pola ext4 yang ter-mount lalu meng-update superblock di tempat, di atas data squashfs yang baru di-`dd`.

Rantai kejadiannya:

1. Di `fdisk`, signature ext4 lama **dibiarkan** (jawab `N` pada "remove signature").
2. Desktop mengenali sdX2 sebagai ext4 lalu **otomatis me-mount**-nya. Petunjuknya: error `mkfs.ext4 ... apparently in use`.
3. `dd` menulis squashfs ke partisi yang sedang ter-mount.
4. ext4 yang ter-mount meng-update superblock-nya dan merusak blok pertama squashfs.

## 3. Perbaikan

### Tahap A: workaround sementara (menyelamatkan sistem)

Overlay menolak `mv`/`rm` ke file lower yang korup (EIO). Jalan keluarnya: tulis file pengganti **langsung ke `/rw/upper/<path>`**. Upper layer selalu menang atas lower, lalu reboot supaya cache overlay segar.

- 77 file dipulihkan dari paket `.deb` Ubuntu (ukuran cocok dengan inode asli, md5 conffile cocok dengan database dpkg).
- 3 file NetworkManager dipulihkan persis dari awal blok korup yang masih valid.
- File kembar disalin dari `/usr/share/...`.
- Kunci GPG Armbian dipasang ulang, sertifikat dan konfigurasi console digenerate ulang.
- Beberapa file Armbian diganti file fungsional (paket aslinya sudah dihapus dari mirror).

Hasilnya: `apt update` jalan normal (±10–30 detik).

### Tahap B: perbaikan sebenarnya (menimpa 4096 byte pertama)

Dilakukan lewat SSH, tanpa mencabut kartu:

```bash
# cadangan dulu (disimpan di laptop, lihat bagian Cadangan)
# kirim 4096 byte pertama dari file asli, lalu di STB:
sudo dd if=/tmp/hdr4k.bin of=/dev/mmcblk0p2 bs=4096 count=1 conv=fsync
sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
head -c <ukuran squashfs> /dev/mmcblk0p2 | md5sum     # harus 73c8778cb17b37e0dc837d488cccdf60
```

Setelah reboot: seluruh file di lower (SquashFS asli) terbaca 100%.

### Tahap C: bersih-bersih

Karena file asli sudah terbaca, **115 file pengganti di `/rw/upper` dihapus** (rm langsung di upper, lalu reboot) supaya sistem kembali ke isi aslinya. Catatan:

- `ca-certificates.conf` asli ternyata men-*deselect* sertifikat kedaluwarsa `!mozilla/AddTrust_External_Root.crt`. Versi regenerasi sebelumnya salah mengaktifkannya. Sekarang file asli dipakai dan `update-ca-certificates --fresh` dijalankan ulang.
- `trusted.gpg` asli sudah memuat kunci Armbian `…93D6889F9F0E78D5`, jadi kunci tambahan tidak perlu dipertahankan.
- Hasil `apt update` di `/var/lib/apt/lists` dan file runtime lain **tidak dihapus** karena memang sah.

## 4. Hasil akhir

- Partisi `/dev/mmcblk0p2` identik dengan `rootfs.squashfs` (md5 `73c8778cb17b37e0dc837d488cccdf60`).
- `dmesg | grep -ci "squashfs error"` = **0**, juga setelah membaca semua file.
- Pembacaan penuh seluruh file di `/ro` dan `/`: 0 file tak terbaca.
- `sudo apt update` selesai tanpa error maupun warning, repo Ubuntu dan Armbian terambil.
- `apt-get -s install htop` normal.

## 5. Cadangan

Disimpan di `~/Documents/stb-backup/` (dibuat sebelum langkah perbaikan):

| File | Isi |
|---|---|
| `rw-upper-before-fix.tgz` | Salinan lengkap `/rw/upper` sebelum perbaikan Tahap B dan C (termasuk semua file pengganti) |
| `p2-head-64k-BEFORE-fix.bin` | 64 KB pertama partisi sdX2 sebelum ditimpa (kondisi rusak) |

Berkas besar lain di `~/Documents/`: `rootfs.squashfs` (asli, 489 MiB) dan `rootfs_backup.img` (backup rootfs ext4 sebelum dipack, 7 GB). Jangan dihapus dulu.

## 6. Pelajaran / hal yang harus diingat

- **Matikan automount** desktop selama menulis ke SD card (`gsettings set org.gnome.desktop.media-handling automount false`).
- Di `fdisk`, **jawab `Y`** (hapus signature), lalu jalankan `wipefs -a /dev/sdX2` sebelum `dd`.
- **Selalu verifikasi sesudah `dd`**, jangan hanya percaya pesan sukses:
  ```bash
  echo 3 | sudo tee /proc/sys/vm/drop_caches
  sudo cmp -n $(stat -c%s rootfs.squashfs) rootfs.squashfs /dev/sdX2 && echo IDENTIK
  ```
- Gejala `apt update` hang + CPU 100% + `Input/output error` di file `/etc` = curigai squashfs, cek `dmesg | grep -i squashfs`. Bukan jaringan.
- Overlayfs: file lower yang korup tidak bisa di-`rm`/`mv` lewat `/`. Tulis pengganti langsung di `/rw/upper`, lalu reboot.
- Tutorial yang sudah direvisi: `~/Documents/squashfs fix.md`.
