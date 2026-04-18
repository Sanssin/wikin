# Deployment Guide - Wikin v2 (Laravel 10)

Panduan ini mencakup:
1. Menjalankan project secara **local**
2. Deploy ke **VPS/server Linux (Nginx + PHP-FPM)**
3. Deploy ke **Railway** (fokus koneksi database)

## 1) Menjalankan Local

### Prasyarat
- PHP 8.1+
- Composer 2+
- Node.js + npm
- Untuk opsi SQLite: aktifkan extension `pdo_sqlite` dan `sqlite3` di `php.ini`

### Setup awal
```powershell
composer install
npm install
Copy-Item .env.example .env
```

Jika pakai SQLite:
```powershell
New-Item -ItemType File database\database.sqlite -Force
```

Set `.env` (minimal):
```env
APP_ENV=local
APP_DEBUG=true
APP_URL=http://127.0.0.1:8000

DB_CONNECTION=sqlite
DB_DATABASE=database/database.sqlite
```

Lalu jalankan:
```powershell
php artisan key:generate
php artisan migrate --seed --force
php artisan storage:link
```

Start aplikasi:
```powershell
# Terminal 1
php artisan serve --host=127.0.0.1 --port=8000

# Terminal 2
npm run dev -- --host 127.0.0.1 --port 5173
```

## 2) Deploy ke VPS / Server Linux

Contoh stack: Ubuntu + Nginx + PHP 8.1 + MySQL/MariaDB.

### Install dependency server
```bash
sudo apt update
sudo apt install -y nginx git unzip curl
sudo apt install -y php8.1 php8.1-fpm php8.1-mysql php8.1-xml php8.1-mbstring php8.1-curl php8.1-zip
```

Install Composer dan Node.js (sesuaikan versi sesuai server Anda), lalu clone project:
```bash
git clone <repo-url> /var/www/wikin-v2
cd /var/www/wikin-v2
```

### Build aplikasi
```bash
composer install --no-dev --optimize-autoloader
npm ci
npm run build
cp .env.example .env
php artisan key:generate
```

Set `.env` production (minimal):
```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://domain-anda.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=wikin_v2
DB_USERNAME=<user_db>
DB_PASSWORD=<password_db>
```

Migrasi:
```bash
php artisan migrate --seed --force
php artisan storage:link
php artisan config:cache
php artisan view:cache
```

Permission:
```bash
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R ug+rwx storage bootstrap/cache
```

### Nginx config (contoh)
`/etc/nginx/sites-available/wikin-v2`
```nginx
server {
    listen 80;
    server_name domain-anda.com;
    root /var/www/wikin-v2/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Aktifkan site:
```bash
sudo ln -s /etc/nginx/sites-available/wikin-v2 /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 3) Deploy ke Railway (fokus database)

## A. Buat service app
1. Push code ke GitHub.
2. Di Railway: **New Project -> Deploy from GitHub Repo**.
3. Pilih repository ini.

## B. Build dan start command
Set di Railway service settings:

- **Build Command**
```bash
composer install --no-dev --optimize-autoloader && npm ci && npm run build
```

- **Start Command**
```bash
php artisan serve --host=0.0.0.0 --port=$PORT
```

## C. Environment variables penting
Di Railway App Service -> Variables:

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://<domain-railway-anda>
APP_KEY=base64:...   # generate dengan: php artisan key:generate --show

CACHE_DRIVER=file
SESSION_DRIVER=cookie
QUEUE_CONNECTION=sync
```

> `SESSION_DRIVER=cookie` dipakai agar tidak bergantung pada file session persistent di container.

## D. Koneksi database di Railway

Project ini mendukung `DATABASE_URL` pada semua driver (`mysql`, `pgsql`, `sqlite`) melalui `config/database.php`.

### Opsi 1 (disarankan): pakai `DATABASE_URL`
1. Tambah plugin database di Railway: **MySQL** atau **PostgreSQL**.
2. Di App Service, set:
```env
DB_CONNECTION=mysql   # atau pgsql jika pakai PostgreSQL
DATABASE_URL=<url-koneksi-dari-service-db>
```

Contoh:
- MySQL URL biasanya format `mysql://user:pass@host:port/dbname`
- PostgreSQL URL biasanya format `postgresql://user:pass@host:port/dbname`

### Opsi 2: set variabel DB satu per satu
```env
DB_CONNECTION=mysql   # atau pgsql
DB_HOST=<host>
DB_PORT=<port>
DB_DATABASE=<database>
DB_USERNAME=<username>
DB_PASSWORD=<password>
```

## E. Jalankan migrasi di Railway

Setelah deploy sukses, jalankan sekali:
```bash
php artisan migrate --seed --force
php artisan storage:link
```

Jika pakai Railway CLI:
```bash
railway run php artisan migrate --seed --force
railway run php artisan storage:link
```

## F. Integrasi Google Login (jika dipakai)

Jika fitur Google Login aktif, tambahkan:
```env
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_CLIENT_REDIRECT=https://<domain-anda>/auth/google/callback
```

Pastikan callback URL di Google Console sama persis.

## G. Catatan penting untuk Railway
- Storage lokal (`storage/app`) di container bersifat ephemeral.
- Untuk upload file production, pertimbangkan object storage (S3-compatible).
- Jika ada perubahan `.env`, lakukan redeploy.

## Troubleshooting singkat
- `could not find driver` -> extension PDO DB belum aktif (mis. `pdo_sqlite`, `pdo_mysql`, `pdo_pgsql`).
- Error 500 setelah ubah env -> jalankan `php artisan config:clear`.
- `APP_KEY` kosong -> generate key lalu set ke variable Railway.
