# Perjalanan Proyek

Setiap tahap diakhiri **checkpoint**: bukti bahwa tahap itu benar-benar bekerja (output perintah, header HTTP, log, uji dari jaringan seluler) sebelum lanjut ke tahap berikutnya.

---

## Bagian 1: Pi-hole di Raspberry Pi

**Tujuan:** memblokir iklan, pelacak, telemetri, dan situs judi online di semua perangkat rumah tanpa konfigurasi per perangkat.

- Raspberry Pi OS Lite, IP statis, update sistem, hardening SSH (login kunci saja, `AllowUsers`, password login dimatikan).
- Unbound sebagai resolver lokal di `127.0.0.1:5335` dengan DNS-over-TLS ke Quad9 dan Cloudflare Security.
- Pi-hole dengan upstream Unbound, blocklist iklan/pelacak/judi, regex untuk pola domain judol, daftar anti-DoH.
- Firewall UFW: hanya SSH, DNS, dasbor, mDNS dari LAN.
- Router membagikan Pi-hole sebagai DNS lewat DHCP; IPv6 dimatikan agar tidak ada jalur DNS yang lolos.

**Checkpoint:** `doubleclick.net` → `0.0.0.0`; `google.com` lewat DNS polos ISP dijawab `forcesafesearch` (bukti pembajakan), lewat Unbound dijawab normal (bukti enkripsi bekerja); `isc.org` mendapat flag `ad` (DNSSEC); `dnssec-failed.org` → `SERVFAIL`.

## Bagian 2: Migrasi ke STB Armbian

**Tujuan:** Raspberry Pi milik kantor harus dikembalikan, jadi Pi-hole dipindahkan ke STB bekas (HG680P) yang di-flash Armbian.

- Verifikasi checksum image, flash ke microSD, lalu instal ke eMMC internal.
- Konfirmasi klaim penjual dari perangkat sendiri (`/proc/device-tree/compatible`, RAM, eMMC).
- Membuat ulang SSH host key dan `machine-id` (image yang sama dipakai ribuan orang).
- Memindahkan konfigurasi Unbound, hardening SSH, dan seluruh isi Pi-hole lewat **Teleporter**.
- Batasi umur database query 30 hari karena eMMC hanya ±8 GB.
- Migrasi bertahap: STB jadi cadangan dulu (uji failover), baru mengambil alih IP utama.
- Pensiunkan Pi: kartu memori ditulis ulang sebelum dikembalikan (berisi riwayat DNS keluarga).

**Checkpoint:** enam tes `dig` lulus, juga setelah reboot; failover teruji dengan mematikan `pihole-FTL` di server utama.

## Bagian 3: Proxmox VE (Tahap 1–5.5)

**Tujuan:** satu mesin kecil yang bisa menjalankan banyak layanan terisolasi.

- Instalasi Proxmox VE di Lenovo M710q, IP statis, nama `pve.home.arpa`.
- Hardening host dan SSH berbasis kunci.
- Kontainer latihan untuk belajar snapshot dan rollback.
- Backup harian otomatis.
- **Uji restore**: backup dipulihkan ke CT ID baru, diverifikasi identik, lalu dibersihkan. Backup yang belum pernah dipulihkan hanyalah harapan.

## Bagian 4: Pi-hole cadangan (Tahap 6A)

- Kontainer LXC dengan Pi-hole + Unbound, isi diimpor dari STB via Teleporter.
- Penyesuaian khas kontainer: NTP Pi-hole dimatikan (jam milik host), firewall diatur Proxmox.
- Router membagikan dua DNS; host Proxmox juga memakai keduanya.

**Checkpoint:** failover teruji, query pindah ke dasbor cadangan saat server utama dimatikan.

## Bagian 5: Nextcloud (Tahap 6B)

- LXC dengan disk sistem 16 GB + disk data terpisah 200 GB (thin provisioning).
- MariaDB dengan database & user khusus, `mariadb-secure-installation`.
- Apache + PHP + Redis (file locking) + APCu (cache lokal), `memory_limit` 512M.
- Unduhan Nextcloud diverifikasi dengan SHA-256, instalasi lewat `occ`.
- Cron sistem untuk tugas latar belakang, jendela maintenance di luar jam aktif.

**Checkpoint:** login, unggah foto, Administration → Overview sebagai daftar perbaikan.

## Bagian 6: Website publik (Tahap 6C)

- nginx dengan `server_tokens off` dan real IP dari header Cloudflare.
- Cloudflare Tunnel (`cloudflared` dari repo resmi), public hostname, Always Use HTTPS.
- Cloudflare Access: policy berbasis email + One-Time PIN untuk halaman pribadi.
- Firewall keluar: kontainer publik tidak bisa menjangkau LAN, kecuali DNS ke Pi-hole.

**Checkpoint:** situs terbuka dari data seluler dengan gembok; log berisi IP asli pengunjung; halaman pribadi mengembalikan `302` ke halaman login Access; email di luar policy tidak pernah menerima kode; dari kontainer, `curl` ke server LAN timeout.

## Bagian 7: Tailscale & HTTPS Nextcloud (Tahap 7)

- Kontainer gateway dengan izin `/dev/net/tun`, IP forwarding, mengiklankan `192.168.0.0/24`.
- Split DNS `home.arpa` → Pi-hole rumah.
- Tailscale langsung di kontainer Nextcloud + `tailscale serve` untuk HTTPS resmi `ts.net`.
- Konfigurasi reverse proxy di Nextcloud (`trusted_proxies`, `overwriteprotocol`, `overwritecondaddr`, `overwrite.cli.url`).
- HSTS dan penyembunyian versi Apache.

**Checkpoint:** Proxmox dan Pi-hole terbuka dari HP lewat data seluler; `curl -sI https://nextcloud.<tailnet>.ts.net/status.php` → `HTTP/2 200` dengan header HSTS dan `server: Apache` tanpa versi; akses LAN `http://` tetap `200 OK`; peringatan HTTPS/HSTS di Nextcloud hilang; aplikasi ponsel tersambung lewat HTTPS.
