<div align=center>

|    NRP     |                 Name                  |
| :--------: | :-----------------------------------: |
| 5025221218 |          Ricardo Supriyanto           |

# CI/CD Deployment with GitHub Actions

Repository ini berisi konfigurasi untuk mengimplementasikan proses CI/CD (Continuous Integration and Continuous Deployment) menggunakan GitHub Actions. Proses ini mencakup build dan deploy aplikasi, dengan menjaga keamanan sensitive data. Deploy dapat dilihat pada link ini http://20.243.200.197 atau http://20.243.200.197/seri

Link dapat diakses secara bebas hingga tanggal 19/05/2024, setelah itu vm nya akan saya stop.
</div>

## Fitur Utama

- **CI/CD Pipeline**: Menggunakan GitHub Actions untuk mengotomatisasi proses build dan deploy.
- **Keamanan**: Tidak menampilkan leak apapun yang bersifat sensitive dalam data pipeline CI/CD.
- **Build dan Push Docker Image**: Membuat dan mengupdate image Docker ke Docker Hub.
- **Deploy ke Server**: Menggunakan Docker Compose untuk pull dari Docker Hub dan menjalankan container baru di server.

## Secret Environment Variables

Untuk menjaga keamanan, kami menggunakan beberapa secret environment variables:
- `DOCKERHUB_PASSWORD`
- `DOCKERHUB_USERNAME`
- `SERVER_IP`
- `SERVER_USERNAME`
- `SSH_PRIVATE_KEY`

## Struktur Direktori


Berikut adalah dokumentasi dalam format markdown (README.md) yang menjelaskan pekerjaan Anda pada repository GitHub, dengan penjelasan rinci tentang setiap file dan hubungannya:

markdown
Copy code
# CI/CD Deployment with GitHub Actions

Repository ini berisi konfigurasi untuk mengimplementasikan proses CI/CD (Continuous Integration and Continuous Deployment) menggunakan GitHub Actions. Proses ini mencakup build dan deploy aplikasi, dengan menjaga keamanan sensitive data seperti key, IP, dan password.

## Fitur Utama

- **CI/CD Pipeline**: Menggunakan GitHub Actions untuk mengotomatisasi proses build dan deploy.
- **Keamanan**: Tidak me-leak sensitive data dalam pipeline CI/CD.
- **Build dan Push Docker Image**: Membuat dan mengupdate image Docker ke Docker Hub.
- **Deploy ke Server**: Menggunakan Docker Compose untuk pull dan menjalankan container baru di server.

## Secret Environment Variables

Untuk menjaga keamanan, kami menggunakan beberapa secret environment variables:
- `DOCKERHUB_PASSWORD`
- `DOCKERHUB_USERNAME`
- `SERVER_IP`
- `SERVER_USERNAME`
- `SSH_PRIVATE_KEY`

### Dockerfile

Dockerfile merupakan file yang bertujuan untuk `build` image agar aplikasi dapat dijalankan sebagai container. pada Dockerfile ini isinya untuk menjalankan `front end` dan `back end` dari project yang ada.

```Dockerfile
FROM php:8.1-fpm

WORKDIR /var/www/html

COPY composer.lock composer.json /var/www/html/

COPY .env.example .env

RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    jpegoptim optipng pngquant gifsicle \
    vim \
    unzip \
    git \
    curl \
    libzip-dev \
    nodejs \
    npm \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install gd pdo_mysql zip \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Install Yarn
RUN npm install -g yarn

# Add user for laravel application
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www

# Copy existing application directory contents
COPY . /var/www/html

RUN chmod -R 777 .

RUN chown -R www-data:www-data /var/www/html
# Change current user to www
USER www
# Install application dependencies
RUN composer install

# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
```

Dockerfile ini mendefinisikan environment untuk aplikasi PHP (nama containernya di [docker-compose.yml](./docker-compose.yml) adalah app) dengan dependencies yang diperlukan dan mengatur user serta permission yang sesuai.

### default.conf

File default.conf adalah file yang saya gunakan untuk konfigurasi nginx yang akan dimanfaatkan sebagai webserver. Directory dari file tersebut ada di `/nginx/conf.d/default.conf`.

```conf
server {
    listen 80;
    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/html/public;
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```

File ini mendefinisikan konfigurasi Nginx untuk menangani request HTTP dan meneruskan request PHP ke container app (container dari image yang nanti di build terlebih dahulu dan di push ke Docker Hub).

### docker-compose.yml

```yaml
version: '3.7'
services:
  app:
    image: ricardo08/ajk-penugasan3:latest
    container_name: app
    restart: unless-stopped
    tty: true
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    working_dir: /var/www/html
    volumes:
      - dbdata:/var/www/html
    networks:
      - rics-net

  webserver:
    image: nginx:stable-alpine
    container_name: webserver
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - dbdata:/var/www/html
      - ./nginx/conf.d/:/etc/nginx/conf.d/
    networks:
      - rics-net

  db:
    image: mysql:5.7.22
    container_name: mysql
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: homestead
      MYSQL_USER: homestead
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: secret
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    networks:
      - rics-net

networks:
  rics-net:
    driver: bridge

volumes:
  dbdata:
    driver: local
```

File ini mendefinisikan tiga service: app (container aplikasi yang berasal dari pull docker image yang di build lalu di push pada Docker Hub menggunakan file [Dockerfile](./Dockerfile)), webserver (Nginx dengan konfigurasi file [Default.conf](./nginx/conf.d/default.conf)), dan db (MySQL). Service-service ini saling berhubungan melalui jaringan Docker rics-net dan berbagi volume dbdata untuk persistent storage.

### ci-cd.yml

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'

      - name: Install dependencies
        run: composer install --no-progress --no-suggest --prefer-dist --optimize-autoloader

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/ajk-penugasan3:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Copy docker-compose.yml to remote server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "./docker-compose.yml, ./nginx/conf.d/default.conf"
          target: "."
          overwrite: true

      - name: Deploy to Azure
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/${{ secrets.SERVER_USERNAME }}
            docker pull ${{ secrets.DOCKERHUB_USERNAME }}/ajk-penugasan3:latest
            docker compose down
            docker compose up -d
            docker exec -i app /bin/sh -c "php artisan key:generate"
            docker exec -i app /bin/sh -c "yarn"
            docker exec -i app /bin/sh -c "yarn build"
            docker exec -i app /bin/sh -c "php artisan migrate --force"
            docker exec -i app /bin/sh -c "php artisan db:seed --force"
```

> Workflow ini terdiri dari dua job: build dan deploy. Pada job build, kode di-checkout, dependencies di-install, dan image Docker dibuild dan dipush ke Docker Hub. Job deploy dijalankan jika job build berhasil dan akan menyalin file konfigurasi ke server dan menjalankan container Docker baru di server.

#### Alur Kerja CI/CD
1. Push ke Branch main: Setiap kali ada perubahan yang dipush ke branch main, GitHub Actions akan memicu workflow CI/CD.
2. Job build:
    - Checkout kode.
    - Setup environment PHP.
    - Install dependencies menggunakan Composer.
    - Setup QEMU dan Docker Buildx.
    - Login ke Docker Hub.
    - Build dan push image Docker ke Docker Hub.
3. Job deploy (dijalankan setelah build berhasil):
    - Checkout kode.
    - Salin file konfigurasi ke server menggunakan SCP.
    - Deploy ke server dengan menjalankan beberapa perintah Docker dan konfigurasi aplikasi.

Dengan konfigurasi ini, setiap perubahan pada kode backend atau frontend akan melakukan build dan deploy otomatis, memastikan aplikasi selalu up-to-date dan aman dari kebocoran data penting.

**Kendala:**
- Masih belum sempat untuk mengoptimasi build time dengan memanfaatkan cache (waktu total untuk build dan deploy sekitar 3-5 menit).
- Menemui error saat melakukan uji coba github action di bagian langkah-langkah build dan deploy, lalu memanfaatkan dari tutorial saat zoom menggunakan appleboy dan lainnya.

Sekian dari saya, terima kasih. Jika ingin mengakses tutorial untuk menjalankan repository secara manual bisa mengakses [README.md manual run](./manual-run/README.md)