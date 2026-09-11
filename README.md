# RIRI STORE SCRIPT - MEWING VPN INSTALLER

Selamat datang di repositori instalasi VPN MEWING. Script ini dirancang untuk kemudahan dan otomatisasi instalasi berbagai protokol VPN populer seperti Xray (Vmess, Vless, Trojan), SSH, OpenVPN, dan Dropbear pada VPS Ubuntu / Debian.

## 📡 Arsitektur Koneksi (Connection Flow)

Berikut adalah diagram alur koneksi dari klien VPN ke backend services di VPS:

```
┌──────────────────────────────────────────────────────────────────────┐
│                        KLIEN VPN / USER                             │
└───────────┬──────────────┬───────────────┬───────────────────────────┘
            │              │               │
     Port 222-442    Port 443 (TLS)   Port 80/8080/8880/55
     Port 444-999                     (Non-SSL Direct)
     Port 8443                              │
            │              │                │
            ▼              ▼                ▼
┌───────────────────────────────────────────────────────────────────────┐
│                         HAPROXY (Frontend)                           │
│                                                                      │
│  ┌─────────────────┐  ┌──────────────┐  ┌─────────────────────────┐  │
│  │ ft: multiport   │  │ ft: ft_ssl   │  │ ft: ft_http_direct      │  │
│  │ (TCP inspect)   │  │ (TCP, :443)  │  │ (TCP, :80/8080/8880/55) │  │
│  │                 │  │              │  │                         │  │
│  │ HTTP? ──► http  │  │  TLS? ──►   │  │ HTTP? ──► nginx_http    │  │
│  │ TLS?  ──► https │  │   https     │  │ else  ──► OpenVPN       │  │
│  └────┬────────┬───┘  └──────┬──────┘  └───────┬──────────┬──────┘  │
│       │        │             │                  │          │         │
│       ▼        ▼             ▼                  ▼          ▼        │
│  ┌─────────┐ ┌──────────────────────────┐  ┌────────┐ ┌─────────┐  │
│  │ recir   │ │    ft: ft_https_terminated│  │nginx   │ │OpenVPN  │  │
│  │ _http   │ │    (mode http, SSL off)   │  │ _http  │ │ :1194   │  │
│  └────┬────┘ │                           │  └───┬────┘ └─────────┘  │
│       │      │  WebSocket? ──► nginx_ws  │      │                   │
│       ▼      │  gRPC?      ──► nginx_grpc│      │                   │
│  ┌─────────┐ │  Path /?    ──► nginx_ws  │      │                   │
│  │ ft_http │ └──────┬────────────┬───────┘      │                   │
│  │_terminat│        │            │               │                   │
│  │ (HTTP)  │        ▼            ▼               │                   │
│  │ WS? ──► │   ┌─────────┐ ┌──────────┐         │                   │
│  │nginx_ws │   │nginx_ws │ │nginx_grpc│         │                   │
│  └────┬────┘   └────┬────┘ └─────┬────┘         │                   │
│       │              │           │               │                   │
└───────┼──────────────┼───────────┼───────────────┼───────────────────┘
        │              │           │               │
        ▼              ▼           ▼               ▼
┌───────────────────────────────────────────────────────────────────────┐
│                         NGINX (Reverse Proxy)                        │
│                                                                      │
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐  │
│  │  Server :1010 (WS Handler)  │  │ Server :1013 (gRPC Handler)  │  │
│  │                              │  │                              │  │
│  │  /vless      ──► Xray:10001 │  │ /vless-grpc  ──► Xray:10005 │  │
│  │  /vmess      ──► Xray:10002 │  │ /vmess-grpc  ──► Xray:10006 │  │
│  │  /trojan-ws  ──► Xray:10003 │  │ /trojan-grpc ──► Xray:10007 │  │
│  │  /ss-ws      ──► Xray:10004 │  │ /ss-grpc     ──► Xray:10008 │  │
│  │  /  (default)──► WS:10015   │  │                              │  │
│  │              (SSH WebSocket) │  │                              │  │
│  └──────────────────────────────┘  └──────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────┐                                    │
│  │  Server :81 (HTTPS Static)  │                                    │
│  │  SSL cert: xray.crt/key     │                                    │
│  │  root: /var/www/html        │                                    │
│  └──────────────────────────────┘                                    │
└──────────────────┬───────────────────────────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────────────────────────────┐
│                      BACKEND SERVICES                                │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    XRAY CORE (localhost)                        │  │
│  │  :10001 VLESS WS    │ :10005 VLESS gRPC                       │  │
│  │  :10002 VMESS WS    │ :10006 VMESS gRPC                       │  │
│  │  :10003 TROJAN WS   │ :10007 TROJAN gRPC                      │  │
│  │  :10004 SS WS       │ :10008 SS gRPC                          │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────┐  ┌───────────────┐  ┌──────────────────────────┐ │
│  │  OpenSSH      │  │  Dropbear     │  │  ePro WS Proxy (ws)     │ │
│  │  :22 :2222    │  │  :143 :109    │  │  tun.conf:              │ │
│  │  :2223        │  │               │  │  SSH→109 (:10015)       │ │
│  └───────────────┘  └───────────────┘  │  OVPN→1194 (:10012)    │ │
│                                         └──────────────────────────┘ │
│  ┌───────────────┐  ┌───────────────┐                                │
│  │  OpenVPN      │  │  UDP Mini     │                                │
│  │  TCP :1194    │  │  :1-65535     │                                │
│  └───────────────┘  └───────────────┘                                │
└───────────────────────────────────────────────────────────────────────┘
```

### Port Map (Ringkasan)

| Port Publik | Protokol | Deskripsi |
|---|---|---|
| `22` | TCP | OpenSSH Direct |
| `55, 80, 8080, 8880` | TCP | HAProxy → HTTP/WS Handler |
| `109, 143` | TCP | Dropbear SSH (via HAProxy) |
| `222-442, 444-999` | TCP | HAProxy Multi-port (auto-detect TLS/HTTP) |
| `443` | TLS | HAProxy → SSL Termination → Xray WS/gRPC |
| `2222, 2223` | TCP | OpenSSH Alternate Ports |
| `8443` | TCP | HAProxy Multi-port (TLS) |

### Xray Inbound Map

| Port Internal | Protokol | Path / Service |
|---|---|---|
| `10001` | VLESS WS | `/vless` |
| `10002` | VMESS WS | `/vmess` |
| `10003` | TROJAN WS | `/trojan-ws` |
| `10004` | Shadowsocks WS | `/ss-ws` |
| `10005` | VLESS gRPC | `vless-grpc` |
| `10006` | VMESS gRPC | `vmess-grpc` |
| `10007` | TROJAN gRPC | `trojan-grpc` |
| `10008` | Shadowsocks gRPC | `ss-grpc` |

---

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

### 3. Perbaikan Kompatibilitas Ubuntu 22.04 / 24.04
- **HAProxy Config**: Konfigurasi HAProxy ditulis ulang total untuk memperbaiki bug routing fatal (HTTP ACL di TCP mode yang menyebabkan semua koneksi WS/gRPC gagal).
- **`bind-process` dihapus**: Directive yang sudah tidak didukung di HAProxy 2.4+.
- **`[Unit]` header xray.service**: Ditambahkan header `[Unit]` yang hilang pada definisi systemd service Xray.
- **SSL Fallback**: Ditambahkan fallback self-signed certificate jika ACME gagal, agar HAProxy tetap bisa start.
- **PAM Password Fix**: Menghapus overwrite `/etc/pam.d/common-password` yang menyebabkan password root VPS berubah setelah instalasi.
- **Package names**: Diperbarui ke `python3`, `python-is-python3`, `libcurl4-openssl-dev`, `netcat-openbsd` untuk Ubuntu 22.04+.

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

## 🛠️ Instalasi

```bash
wget -q -O main.sh https://raw.githubusercontent.com/TahuKhasBandung/MEWING/main/main.sh && chmod +x main.sh && ./main.sh
```

### Persyaratan Sistem
- **OS**: Ubuntu 22.04 / 24.04 (LTS)
- **Arsitektur**: x86_64 (amd64)
- **RAM**: Minimal 1 GB
- **Akses**: Root

---
**RIRI STORE AUTO SCRIPT - 2024**
