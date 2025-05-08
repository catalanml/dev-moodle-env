ARG PHP_VERSION=8.1
FROM php:${PHP_VERSION}-fpm

# Install system dependencies for Moodle PHP extensions
RUN apt-get update && apt-get install -y --no-install-recommends \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    libzip-dev \
    libxml2-dev \
    libicu-dev \
    libxslt1-dev \
    default-libmysqlclient-dev \
    libonig-dev \
    libsodium-dev \
    git \
    unzip \
    # Dependencies for PECL packages if any, e.g., re2c for some
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install xmlrpc via PECL as it's removed from core in PHP 8+
# Specify the version explicitly as the stable one might not be available
RUN pecl install channel://pecl.php.net/xmlrpc-1.0.0RC3 \
    && docker-php-ext-enable xmlrpc

# Configure and install Moodle required PHP extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg && \
    docker-php-ext-install -j$(nproc) \
    gd \
    intl \
    opcache \
    pdo \
    pdo_mysql \
    mysqli \
    soap \
    zip \
    xsl \
    exif \
    bcmath \
    sodium \
    mbstring

# You might need to set a working directory or copy Moodle code if building a fully custom image
# WORKDIR /var/www/html