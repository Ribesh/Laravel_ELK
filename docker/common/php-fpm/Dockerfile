# Stage 1: Build Frontend Assets

FROM node:20-alpine AS frontend-builder

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# 🏗 Stage 2: Build PHP Dependencies
FROM php:8.2-fpm-alpine AS builder

# Install system and PHP build dependencies
RUN apk add --no-cache --virtual .build-deps \
        $PHPIZE_DEPS \
        curl \
	fcgi \
        unzip \
        libxml2-dev \
        icu-dev \
        oniguruma-dev \
        freetype-dev \
        libjpeg-turbo-dev \
        libpng-dev \
        libwebp-dev \
        libxpm-dev \
        postgresql-dev \
        zlib-dev \
        libzip-dev \
    && docker-php-ext-install -j$(nproc) \
        pdo_mysql \
        pdo_pgsql \
        pgsql \
        intl \
        bcmath \
        zip \
        soap \
        opcache \
    && pecl install redis \
    && docker-php-ext-enable redis

# Set working directory
WORKDIR /var/www

# Copy only composer files first for layer caching
COPY composer.json composer.lock ./

# Install Composer (with cache mount for faster rebuilds)
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
    && composer install --no-scripts --no-dev --no-interaction --no-progress --prefer-dist


# Now copy full application (after dependencies are installed)
COPY . /var/www

# Copy built frontend assets from frontend-builder
COPY --from=frontend-builder /app/public/build /var/www/public/build

# Run post-install optimization
RUN composer run-script post-autoload-dump

#  Stage 2: Production Image
FROM php:8.2-fpm-alpine AS production

# Install only runtime dependencies (fcgi included)
RUN apk add --no-cache \
        libzip \
        icu-libs \
        postgresql-libs \
        libpng \
        libjpeg-turbo \
        libwebp \
        freetype \
        libxpm \
        procps \
        curl \
        fcgi

# Install PHP-FPM healthcheck
RUN curl -o /usr/local/bin/php-fpm-healthcheck \
    https://raw.githubusercontent.com/renatomefi/php-fpm-healthcheck/master/php-fpm-healthcheck \
    && chmod +x /usr/local/bin/php-fpm-healthcheck

# Copy entrypoint script
COPY ./docker/production/php-fpm/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

# Copy initial storage structure
COPY ./storage /var/www/storage-init

# Copy compiled extensions and config from builder
COPY --from=builder /usr/local/lib/php/extensions/ /usr/local/lib/php/extensions/
COPY --from=builder /usr/local/etc/php/conf.d/ /usr/local/etc/php/conf.d/
COPY --from=builder /usr/local/bin/docker-php-ext-* /usr/local/bin/

# Copy application code + vendor dependencies
COPY --from=builder /var/www /var/www

# Use production PHP settings
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

# Enable PHP-FPM status page
RUN sed -i '/\[www\]/a pm.status_path = /status' /usr/local/etc/php-fpm.d/zz-docker.conf

# Set working directory
WORKDIR /var/www

# Fix permissions
RUN chown -R www-data:www-data /var/www

# Switch to non-root user
USER www-data

# Entrypoint and default command
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
EXPOSE 9000
CMD ["php-fpm"]
