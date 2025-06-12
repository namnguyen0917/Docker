# Docker PHP
## Basic directory structure
```
cakephp-docker/
├── app/                   # Source code của CakePHP (đặt mã nguồn ở đây)
├── docker/
│   └── web/
│       ├── Dockerfile
│       └── httpd.conf
├── docker-compose.yml
```

## 1. docker-compose.yml
```
version: '3.8'  # Phiên bản Docker Compose.

services:
  app:
    build:
      context: ./docker/web  # Tham Chiếu đến thư mục hiện tại chứa Dockerfile.
    container_name: cakephp_app  # Tên container, bạn có thể đặt tùy ý nếu k có mục này nó lấy tên service Ex: app-1
    volumes:
      - ./app:/var/www/html  # Map mã nguồn local vào container để code cập nhật tự động
    ports:
      - "8080:80"  # Map cổng 8080 (local) sang 80 (container), vào trình duyệt dùng http://localhost:8080
    environment:
      - CAKEPHP_DEBUG=true  # Biến môi trường tuỳ chỉnh, bạn có thể thêm biến APP_NAME, ENV... nếu muốn
    depends_on:
      - db  # Chờ service "db" (MySQL) khởi động trước rồi mới khởi động app

  db:
    image: mysql:8.0  # Image MySQL phiên bản 8.0 từ Docker Hub
    container_name: cakephp_db  # Tên container MySQL
    restart: always  # Tự động restart nếu MySQL bị dừng hoặc hệ thống khởi động lại
    environment:
      MYSQL_DATABASE: cakephp        # Tên database mặc định
      MYSQL_USER: cakeuser           # Tên user
      MYSQL_PASSWORD: secret         # Mật khẩu của user
      MYSQL_ROOT_PASSWORD: rootpass  # Mật khẩu root
    volumes:
      - mysql_data:/var/lib/mysql  # Tạo volume để lưu dữ liệu vĩnh viễn
    ports:
      - "3306:3306"  # Map cổng MySQL nếu muốn kết nối từ máy host
    networks:
      - cakephp_network  # Kết nối vào cùng network với app
```

## 2. docker/web/Dockerfile
```
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
```

docker-compose build         # Build image
docker-compose up -d         # Khởi động container nền (detached mode)
docker-compose ps            # Kiểm tra container đang chạy


# Set working directory
WORKDIR /var/www/html

# Copy mã nguồn vào container
COPY . /var/www/html
