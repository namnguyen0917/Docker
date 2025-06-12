### Cấu trúc thư mục
```
cakephp-docker/
├── app/                   # Source code của CakePHP (đặt mã nguồn ở đây)
├── docker/
│   └── web/
│       ├── Dockerfile
│       └── httpd.conf
├── docker-compose.yml
```
### 1. docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: ./docker/web
    container_name: cakephp-app
    volumes:
      - ./app:/var/www/html
    ports:
      - "8080:80"
    environment:
      - TZ=Asia/Ho_Chi_Minh
    depends_on:
      - db

  db:
    image: mysql:8.0
    container_name: mysql-db
    restart: always
    environment:
      MYSQL_DATABASE: cakephp
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: cakeuser
      MYSQL_PASSWORD: secret
    ports:
      - "3306:3306"
    volumes:
      - db-data:/var/lib/mysql

volumes:
  db-data:

### 2. docker/web/Dockerfile

FROM amazonlinux:2023

RUN yum update -y

# Cài Apache + PHP và các extension
RUN yum install -y httpd php php-mbstring php-xml php-json php-gd \
    php-intl php-mysqli php-pdo php-curl php-zip php-bcmath php-cli php-pear

# Tạo thư mục chạy PHP
RUN mkdir -p /run/php-fpm && chmod -R 777 /run/php-fpm

# Cài Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Cấu hình Apache (file riêng)
COPY httpd.conf /etc/httpd/conf/httpd.conf

# Mở port 80
EXPOSE 80

CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]


FROM php:8.2-apache

# Cài các gói hệ thống bắt buộc để build ext-intl và zip
RUN apt-get update && apt-get install -y \
    libicu-dev \
    libonig-dev \
    libzip-dev \
    zip unzip \
    && docker-php-ext-install intl pdo pdo_mysql mbstring zip

# Bật rewrite cho Apache
RUN a2enmod rewrite

# Copy Composer vào container
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www/html

# Copy mã nguồn vào container
COPY . /var/www/html


📄 3. docker/web/httpd.conf

ServerRoot "/etc/httpd"
Listen 80

Include conf.modules.d/*.conf
User apache
Group apache

ServerAdmin you@example.com
DocumentRoot "/var/www/html"

<Directory "/var/www/html">
    AllowOverride All
    Require all granted
</Directory>

ErrorLog /var/log/httpd/error.log
CustomLog /var/log/httpd/access.log combined

<IfModule dir_module>
    DirectoryIndex index.php index.html
</IfModule>

# MIME types
AddType application/x-httpd-php .php
AddType application/x-httpd-php-source .phps

# PHP config
<FilesMatch \.php$>
    SetHandler application/x-httpd-php
</FilesMatch>
