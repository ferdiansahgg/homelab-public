# Pelajaran dari Lapangan

Masalah nyata yang saya temui selama membangun homelab ini. Formatnya: **gejala → penyebab → perbaikan → pelajaran**.

---

### 1. ISP diam-diam membajak DNS

- **Gejala:** `dig @1.1.1.1 google.com` dijawab `forcesafesearch.google.com`, padahal yang ditanya Cloudflare.
- **Penyebab:** ISP mencegat semua lalu lintas ke port 53 dan menjawabnya sendiri.
- **Perbaikan:** Unbound meneruskan lewat DNS-over-TLS (port 853), yang terenkripsi dan tidak bisa diubah di jalan.
- **Pelajaran:** "Saya pakai 1.1.1.1" belum tentu berarti benar-benar sampai ke 1.1.1.1. Selalu uji dengan bukti.

### 2. Firewall Proxmox: urutan aturan menentukan segalanya

- **Gejala:** setelah menambah aturan DROP ke LAN, DNS kontainer ikut mati dan tunnel putus.
- **Penyebab:** Proxmox membaca aturan dari atas dan berhenti di aturan pertama yang cocok, sedangkan aturan baru ditaruh di **paling atas**. DROP berada di atas ACCEPT DNS.
- **Perbaikan:** seret ACCEPT DNS ke atas DROP. Untuk aturan `out`, IP tujuan diisi di kolom **Destination**, dan protocol pada aturan DROP dikosongkan agar TCP/UDP/ICMP semuanya tertutup.
- **Pelajaran:** firewall dibaca berurutan; selalu uji yang diizinkan *dan* yang diblokir.

### 3. Paket repo gagal karena `curl` tidak ada

- **Gejala:** `apt update` menolak repo Cloudflare dengan `Missing key ... not signed`.
- **Penyebab:** template Debian 13 tidak membawa `curl`. Perintah `curl ... | tee file.gpg` gagal, tapi `tee` tetap membuat file kunci **kosong**.
- **Perbaikan:** `apt install curl`, unduh ulang kunci, pastikan ukurannya bukan 0.
- **Pelajaran:** pipeline bisa gagal diam-diam. Cek hasil antara (`ls -l`), jangan hanya pesan error terakhir.

### 4. Teleporter mengganti password dasbor

- **Gejala:** setelah impor konfigurasi dari Pi-hole utama, password dasbor Pi-hole cadangan berubah.
- **Penyebab:** Teleporter Pi-hole v6 membawa seluruh `pihole.toml`, termasuk hash password.
- **Perbaikan:** samakan password kedua server.
- **Pelajaran:** pahami apa saja isi sebuah backup/export sebelum mengimpornya ke tempat lain.

### 5. DNS host hanya satu, kontainer ikut rapuh

- **Gejala:** kontainer berpotensi kehilangan DNS saat server DNS utama mati, walau router sudah membagikan dua DNS.
- **Penyebab:** host Proxmox hanya punya satu nameserver, dan kontainer dengan DNS kosong menyalin DNS host saat start.
- **Perbaikan:** tambahkan DNS kedua di host, reboot kontainer.
- **Pelajaran:** redundansi harus dicek di setiap lapisan, bukan hanya di router.

### 6. Site nginx tidak muncul

- **Gejala:** subdomain baru menampilkan isi situs lain.
- **Penyebab:** file site belum di-symlink ke `sites-enabled`, sehingga nginx jatuh ke `default_server`.
- **Perbaikan:** `ln -s` ke `sites-enabled`, `nginx -t`, reload.
- **Pelajaran:** `default_server` adalah jaring pengaman yang bisa menyembunyikan kesalahan konfigurasi.

### 7. Tunnel vs public hostname

- **Gejala:** membuat tunnel kedua untuk setiap subdomain.
- **Penyebab:** salah paham konsep. Satu tunnel (satu konektor) bisa melayani banyak hostname.
- **Perbaikan:** hapus tunnel kedua; tambahkan hostname baru di tunnel yang sama. Jangan menjalankan `service install` dua kali.
- **Pelajaran:** pahami model objek sebuah layanan sebelum mengklik.

### 8. Email di luar policy tidak menerima OTP

- **Gejala:** kode login Cloudflare Access tidak pernah sampai untuk email tertentu.
- **Penyebab:** disengaja. Cloudflare tidak mengirim kode ke email yang tidak ada di policy, tapi tetap menampilkan halaman yang sama agar daftar tamu tidak bisa ditebak.
- **Pelajaran:** perilaku yang terlihat seperti bug bisa jadi fitur keamanan.

### 9. Tailscale di LXC: `tailscaled` tidak mau jalan

- **Gejala:** `failed to connect to local tailscaled`, service terus restart.
- **Penyebab:** kontainer unprivileged tidak punya `/dev/net/tun`. Dua baris izin `lxc.*` sudah ditambahkan, tapi di **paling bawah** file config, yaitu di dalam bagian snapshot terakhir, bukan di config aktif.
- **Perbaikan:** pindahkan baris ke bagian atas (sebelum header snapshot `[nama]` pertama), verifikasi dengan `pct config <ID>`, reboot kontainer.
- **Pelajaran:** di Proxmox, bagian berkurung siku dalam file config adalah milik snapshot. Juga: nama service-nya `tailscaled` (daemon), bukan `tailscale`.

### 10. Reverse proxy & Nextcloud

- **Tantangan:** Nextcloud dilayani lewat `tailscale serve` (HTTPS) dan langsung lewat LAN (HTTP) sekaligus.
- **Solusi:** `trusted_proxies = 127.0.0.1`, `overwriteprotocol = https` hanya jika `overwritecondaddr` cocok dengan `127.0.0.1`. Cek dulu isi `trusted_domains` sebelum menambah slot agar entri lama tidak tertimpa.
- **Pelajaran:** aplikasi di balik proxy perlu diberi tahu siapa proxy-nya dan protokol aslinya.

### 11. Tidak semua klaim tutorial berlaku untuk perangkatmu

- **Konteks:** video pendek mempromosikan flash OpenWrt lewat halaman upgrade firmware bawaan router.
- **Temuan:** router saya memakai platform yang tidak didukung OpenWrt; mencobanya berisiko *brick* dan kehilangan fitur mesh.
- **Pelajaran:** cek dukungan resmi (Table of Hardware, versi hardware) sebelum mengikuti tutorial.

---

## Kebiasaan yang terbukti menyelamatkan

1. **Satu perubahan dalam satu waktu.** Kalau rusak, penyebabnya jelas.
2. **Snapshot sebelum langkah berisiko.** Rollback hitungan detik.
3. **Jangan tutup sesi SSH lama** sebelum sesi baru terbukti berhasil (terutama saat mengubah SSH/firewall/IP).
4. **Baca skrip installer sebelum dijalankan** (`less install.sh`).
5. **Checkpoint dengan bukti**, bukan asumsi.
6. **Dokumentasikan segera** selagi detailnya masih segar.
