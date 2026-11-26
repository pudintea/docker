# NGINX DOCKER COMPOSE, REVERSE PROXY

Saya sarankan menggunakan Nginx + Docker Compose dengan konfigurasi virtual host terpisah untuk setiap domain.

Dengan struktur ini, Anda akan mendapatkan beberapa keuntungan:

- Port `80` dan `443` dibuka dari container Nginx.
- SSL Cloudflare (`.key` dan `.pem`) bisa digunakan langsung oleh Nginx.
- Menambah domain cukup dengan membuat satu file konfigurasi baru.
- Mengganti SSL cukup dengan mengganti file sertifikat, tanpa mengubah Docker Compose.
- Konfigurasi Nginx disimpan di luar container sehingga tidak hilang ketika container dibuat ulang.
- Bisa melakukan reload Nginx tanpa menghentikan container.

Saya akan menggunakan pendekatan Nginx sebagai reverse proxy utama, dengan satu file konfigurasi untuk setiap domain.

## 1. Struktur folder

Buat struktur direktori seperti berikut:

text

```text
nginx-proxy/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       ├── domain1.conf
│       └── domain2.conf
├── ssl/
│   ├── domain1/
│   │   ├── namadomain1.pem
│   │   └── namadomain1.key
│   └── domain2/
│       ├── namadomain2.pem
│       └── namadomain2.key
└── logs/
```

Dengan struktur ini, Anda bisa menambahkan domain sebanyak yang diperlukan.

## 2. Docker Compose

Buat file `docker-compose.yml`:

yaml

```yaml
services:
  nginx:
    image: nginx:stable-alpine
    container_name: nginx-proxy
    restart: unless-stopped

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./ssl:/etc/nginx/ssl:ro
      - ./logs:/var/log/nginx

    networks:
      - proxy

networks:
  proxy:
    name: nginx-proxy
    external: true
```

Buat network terlebih dahulu:

bash

```bash
docker network create nginx-proxy
```

Network ini nantinya bisa digunakan oleh container aplikasi lain agar Nginx dapat mengakses aplikasi berdasarkan nama container atau service Docker.

## 3. Konfigurasi utama Nginx

Buat `nginx/nginx.conf`:

nginx

```nginx
user nginx;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 2048;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;

    server_tokens off;

    # WebSocket support
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      '';
    }

    include /etc/nginx/conf.d/*.conf;
}
```

## 4. Konfigurasi domain pertama

Misalnya Anda memiliki domain:

`app.domainanda.com`

Aplikasi berjalan di container lain dengan nama `myapp` pada port `3000`.

Buat file:

`nginx/conf.d/app.domainanda.com.conf`

nginx

```nginx
# HTTP -> HTTPS
server {
    listen 80;
    server_name app.domainanda.com;

    return 301 https://$host$request_uri;
}

# HTTPS Reverse Proxy
server {
    listen 443 ssl;
    server_name app.domainanda.com;

    # SSL Cloudflare
    ssl_certificate /etc/nginx/ssl/domain1/namadomain1.pem;
    ssl_certificate_key /etc/nginx/ssl/domain1/namadomain1.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Reverse Proxy
    location / {
        proxy_pass http://myapp:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```

Ganti:

- `app.domainanda.com` dengan domain Anda.
- `myapp` dengan nama container atau service aplikasi.
- `3000` dengan port aplikasi.
- Path SSL sesuai lokasi sertifikat Anda.

Penting: Container aplikasi harus terhubung ke network `nginx-proxy` yang sama agar nama `myapp` bisa di-resolve oleh Nginx.

Contoh pada Docker Compose aplikasi:

yaml

```yaml
services:
  app:
    image: your-app-image
    networks:
      - nginx-proxy

networks:
  nginx-proxy:
    external: true
```

## 5. Menambahkan domain kedua

Misalnya Anda ingin menambahkan:

`api.domainanda.com`

Cukup buat file baru:

`nginx/conf.d/api.domainanda.com.conf`

nginx

```nginx
server {
    listen 80;
    server_name api.domainanda.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name api.domainanda.com;

    ssl_certificate /etc/nginx/ssl/domain2/namadomain2.pem;
    ssl_certificate_key /etc/nginx/ssl/domain2/namadomain2.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://api-app:8080;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Anda tidak perlu mengubah `docker-compose.yml`.

Cukup:

1. Buat file konfigurasi domain.
2. Masukkan file SSL ke folder `ssl/domain2/`.
3. Pastikan aplikasi terhubung ke network `nginx-proxy`.
4. Reload Nginx.

## 6. Cara mengganti SSL

Misalnya sertifikat domain pertama sudah diperbarui oleh Cloudflare.

Anda cukup mengganti:

text

```text
ssl/domain1/namadomain1.pem
ssl/domain1/namadomain1.key
```

Pastikan file `.pem` dan `.key` merupakan pasangan sertifikat yang sama.

Kemudian jalankan:

bash

```bash
docker exec nginx-proxy nginx -t
```

Jika hasilnya:

text

```text
syntax is ok
test is successful
```

Lakukan reload:

bash

```bash
docker exec nginx-proxy nginx -s reload
```

Nginx akan membaca sertifikat baru tanpa perlu menghentikan container.

## 7. Menjalankan Nginx

Dari folder `nginx-proxy`:

bash

```bash
docker compose up -d
```

Melihat log:

bash

```bash
docker compose logs -f nginx
```

Memeriksa konfigurasi:

bash

```bash
docker exec nginx-proxy nginx -t
```

Reload setelah perubahan:

bash

```bash
docker exec nginx-proxy nginx -s reload
```

## 8. Pengaturan Cloudflare

Untuk konfigurasi ini, saya menyarankan menggunakan Cloudflare Origin Certificate.

Di dashboard Cloudflare:

1. Pilih domain.
2. Masuk ke SSL/TLS → Origin Server.
3. Pilih Create Certificate.
4. Pilih RSA atau ECC.
5. Masukkan domain, misalnya `*.domainanda.com` dan `domainanda.com`.
6. Download Origin Certificate (`.pem`) dan Private Key (`.key`).

Selanjutnya, di Cloudflare pilih:

Recommended SSL/TLS Encryption Mode: Full (strict)

Alur koneksinya:

Pengunjung

Cloudflare Proxy

SSL dari pengunjung ke Cloudflare

Docker Nginx Reverse Proxy

Port 443 + Cloudflare Origin Certificate

Backend Applications

myapp:3000, api-app:8080, dan lainnya

Pastikan DNS Cloudflare mengarah ke IP server Docker dan Proxy Status aktif (awan oranye).

Catatan keamanan: Origin Certificate hanya dipercaya oleh Cloudflare, bukan browser yang mengakses server secara langsung. Sebaiknya batasi akses port 80 dan 443 pada firewall agar hanya IP Cloudflare yang dapat mengakses origin server jika seluruh trafik memang harus melalui Cloudflare.

## 9\. Rekomendasi pengelolaan

Dengan struktur ini, workflow Anda akan sangat sederhana:

| Aktivitas | Yang dilakukan |
| --- | --- |
| Tambah domain | Buat file `.conf` baru |
| Tambah SSL | Simpan `.pem` dan `.key` |
| Ganti SSL | Replace file sertifikat |
| Ubah backend | Edit `proxy_pass` |
| Terapkan perubahan | `nginx -t` lalu reload |
| Restart Nginx | `docker compose restart` |

Saran tambahan: Jangan menyimpan private key di Git repository. Berikan permission yang sesuai pada folder SSL dan batasi akses ke server.

## 10\. Satu pengembangan yang saya rekomendasikan

Kalau nantinya Anda mengelola puluhan domain, kita bisa membuat struktur yang lebih modular, misalnya:
```
nginx-proxy/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       ├── domain-a.conf
│       ├── domain-b.conf
│       ├── domain-c.conf
│       └── domain-d.conf
├── ssl/
│   ├── domain-a/
│   ├── domain-b/
│   └── domain-c/
└── logs/
```

Setiap domain independen, sehingga perubahan konfigurasi domain A tidak akan memengaruhi domain B.

Menurut saya, pendekatan ini paling cocok untuk kebutuhan Anda: satu container Nginx sebagai reverse proxy sentral, banyak domain, masing-masing punya konfigurasi dan SSL sendiri, serta semua perubahan dapat dilakukan tanpa rebuild Docker image.
