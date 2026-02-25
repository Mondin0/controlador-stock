FROM php:8.4-fpm

RUN apt-get update && apt-get install -y \
    libpq-dev postgresql-client git unzip curl \
    && rm -rf /var/lib/apt/lists/*

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
RUN docker-php-ext-install pdo_pgsql

WORKDIR /var/www/html

# php-fpm YA funciona como PID 1, NO tocar
EXPOSE 9000
