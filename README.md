# RIRI STORE SCRIPT - MEWING VPN INSTALLER

Selamat datang di repositori instalasi VPN MEWING. Script ini dirancang untuk kemudahan dan otomatisasi instalasi berbagai protokol VPN populer seperti Xray (Vmess, Vless, Trojan), SSH, OpenVPN, dan Dropbear pada VPS Ubuntu / Debian.

## 🔥 Pembaruan Keamanan & Penguatan Sistem (Security Hardening)
Script ini baru saja melalui proses audit dan penambalan (patching) besar-besaran untuk menjamin keamanan VPS serta kenyamanan Anda dalam mengelola klien. Berikut adalah daftar hal-hal yang telah diubah dan diperkuat:

### 1. Penambalan Kebocoran Data (Data Leak Patched)
- **Kredensial SMTP Email**: Telah dihapus dari hardcode di dalam file `main.sh`. Email dan sandi Gmail yang digunakan untuk backup log sebelumnya terbuka secara publik. (Sekarang diubah menjadi placeholder yang aman).
- **Token API Telegram**: Kunci bot Telegram Anda sekarang juga sudah di-redaksi dengan rapi.

### 2. Perbaikan Bug Instalasi (Bug Fixes)
- **Instalasi Dropbear & DDOS Deflate**: Memperbaiki bug kritis di mana skrip utama bisa mengalami force-close / exit otomatis apabila mendeteksi folder instalasi versi sebelumnya.
- **Bug Syntax Typo**: Memperbaiki deklarasi `$EROR` yang sebelumnya menyebabkan output _blank_ di layar instalasi. 
- **Bug wget URL**: Memperbaiki _broken link_ pada perintah `wget` (modul udp-custom) yang berpotensi mematikan fungsi instalasi VPN berbasis UDP.
- **Systemctl Error Handling**: Menambahkan _error handling_ pada servis Nginx/Apache ketika meminta SSL certifikasi via Acme.sh sehingga proses _stop daemon_ lebih aman dan _clean_.

## 🚀 Fitur Tambahan Baru
Sesuai dengan _request_, repositori ini juga sekarang dilengkapi dengan fitur-fitur baru berikut:

### 1. Custom UUID untuk Vmess, Vless, dan Trojan
Pada menu pembuatan akun Vmess, Vless, dan Trojan (`m-vmess`, `m-vless`, `m-trojan`), Anda tidak lagi dipaksa menggunakan UUID otomatis dari sistem kernel. Kini Anda bisa **menyalin-tempel (copy-paste) UUID kustom** milik Anda sendiri saat prompt pembuatan UUID muncul. Apabila Anda malas mengisi, cukup kosongkan dan tekan Enter, maka sistem akan membuatkannya (auto-generate) untuk Anda!

### 2. Pengubah Versi Dropbear Terintegrasi (Custom Version Changer)
Telah disediakan _tools_ compile Dropbear otomatis dari source resmi! 
- Akses melalui opsi **[15] GANTI DROPBEAR** pada **Utility Menu** (`menu-x`).
- Versi yang dapat Anda pasang / ubah secara bebas dan otomatis:
  - `Dropbear v2018.76`
  - `Dropbear v2019.78`
  - `Dropbear v2020.81`
  - `Dropbear v2022.83`
  - `Dropbear v2024.85`
- Fitur ini sangat berguna apabila klien Anda memiliki kebutuhan akan versi keamanan / tipe *cipher* lawas atau terbaru.

---
**RIRI STORE AUTO SCRIPT - 2024**
