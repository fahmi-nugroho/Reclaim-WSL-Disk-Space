# Reclaim-WSL-Disk-Space

# Mengosongkan Ruang Penyimpanan WSL2 Secara Umum

Ketika file atau paket dihapus di dalam WSL2, ruang penyimpanan **belum otomatis kembali ke Drive C**. Hal ini karena WSL2 menyimpan seluruh filesystem Linux di dalam file virtual `ext4.vhdx` yang hanya akan membesar secara otomatis, tetapi tidak mengecil secara otomatis.

Untuk mengembalikan ruang penyimpanan ke Windows, lakukan langkah-langkah berikut.

---

## Langkah 1 — Hapus File atau Paket yang Tidak Diperlukan

Hapus terlebih dahulu file, direktori, atau paket yang sudah tidak digunakan dari dalam distribusi Linux Anda.

Contoh menghapus paket Python:

```bash
pip uninstall nama-paket -y
```

Contoh menghapus paket APT:

```bash
sudo apt remove nama-paket
sudo apt autoremove
sudo apt clean
```

Contoh menghapus file:

```bash
rm -rf /path/yang/tidak/diperlukan
```

> Pastikan ruang yang ingin dikembalikan memang sudah benar-benar dihapus dari dalam WSL.

---

## Langkah 2 — Jalankan TRIM

Setelah proses penghapusan selesai, jalankan perintah berikut:

```bash
sudo fstrim -v /
```

Perintah ini akan memberi tahu sistem penyimpanan virtual bahwa blok-blok tersebut sudah kosong dan boleh dilepaskan.

Contoh output:

```text
/: 8.4 GiB (9023453184 bytes) trimmed
```

Semakin besar angka yang muncul, semakin banyak ruang yang dapat dikembalikan saat proses compact.

---

## Langkah 3 — Tutup Seluruh Instance WSL

Keluar dari terminal Ubuntu, kemudian di Windows buka **PowerShell** atau **Command Prompt** sebagai **Administrator**, lalu jalankan:

```powershell
wsl --shutdown
```

Perintah ini memastikan file `ext4.vhdx` tidak sedang digunakan sehingga dapat diproses.

---

## Langkah 4 — Temukan Lokasi `ext4.vhdx`

Jika belum mengetahui lokasi file virtual disk WSL, jalankan:

```powershell
Get-ItemProperty Registry::HKCU\Software\Microsoft\Windows\CurrentVersion\Lxss\* |
Select-Object DistributionName, BasePath
```

Contoh hasil:

```text
DistributionName BasePath
Ubuntu           C:\Users\<User>\AppData\Local\wsl\{GUID}
```

File yang akan diproses adalah:

```text
C:\Users\<User>\AppData\Local\wsl\{GUID}\ext4.vhdx
```

---

## Langkah 5 — Compact Virtual Disk

Masuk ke utilitas DiskPart:

```powershell
diskpart
```

Kemudian jalankan:

```powershell
select vdisk file="C:\Users\<User>\AppData\Local\wsl\{GUID}\ext4.vhdx"

attach vdisk readonly

compact vdisk

detach vdisk

exit
```

Tunggu hingga proses `compact vdisk` selesai 100%.

---

## Langkah 6 — Jalankan Kembali WSL

Buka kembali Ubuntu seperti biasa.

WSL akan otomatis menggunakan file `ext4.vhdx` yang ukurannya sudah lebih kecil.

---

# Ringkasan Alur

```text
Hapus file/paket
        │
        ▼
Jalankan fstrim
        │
        ▼
wsl --shutdown
        │
        ▼
Cari lokasi ext4.vhdx
        │
        ▼
diskpart
        │
        ▼
select vdisk
        │
        ▼
attach vdisk readonly
        │
        ▼
compact vdisk
        │
        ▼
detach vdisk
        │
        ▼
exit
        │
        ▼
Ruang Drive C kembali tersedia
```

---

# Catatan Penting

- `fstrim` **tidak mengecilkan** ukuran `ext4.vhdx`, tetapi hanya menandai blok kosong.
- `compact vdisk` adalah proses yang benar-benar menyusutkan ukuran file `ext4.vhdx`.
- Jalankan `compact vdisk` hanya ketika WSL sudah dimatikan menggunakan `wsl --shutdown`.
- Proses ini **aman** dan tidak menghapus data selama dilakukan pada file `ext4.vhdx` yang benar.
- Semakin banyak data yang telah dihapus sebelum menjalankan `fstrim`, semakin besar potensi ruang yang dapat dikembalikan ke Windows.
