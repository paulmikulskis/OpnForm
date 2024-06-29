# Use ARG for reusable variables
ARG PHP_VERSION=8.3
ARG NODE_VERSION=20
ARG POSTGRES_VERSION=15
ARG UBUNTU_VERSION=latest

# Base PHP image
FROM ubuntu:${UBUNTU_VERSION} AS php-base
ENV DEBIAN_FRONTEND=noninteractive
ARG PHP_VERSION

# Install common dependencies
RUN apt-get update && apt-get install -y \
    software-properties-common \
    wget gnupg2 lsb-release \
    && add-apt-repository -y ppa:ondrej/php \
    && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - \
    && echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list \
    && apt-get update && apt-get upgrade -y \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install PHP and extensions
RUN apt-get update && apt-get install -y \
    php${PHP_VERSION} \
    php${PHP_VERSION}-common \
    php${PHP_VERSION}-pgsql \
    php${PHP_VERSION}-redis \
    php${PHP_VERSION}-mbstring \
    php${PHP_VERSION}-xml \
    php${PHP_VERSION}-bcmath \
    php${PHP_VERSION}-gd \
    php${PHP_VERSION}-curl \
    php${PHP_VERSION}-zip \
    php${PHP_VERSION}-imagick \
    php${PHP_VERSION}-bz2 \
    php${PHP_VERSION}-gmp \
    php${PHP_VERSION}-intl \
    php${PHP_VERSION}-pcov \
    php${PHP_VERSION}-soap \
    php${PHP_VERSION}-xsl \
    php${PHP_VERSION}-fpm \
    php-curl \
    composer \
    && phpenmod curl xml pgsql \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Node.js builder stage
FROM node:${NODE_VERSION}-alpine AS javascript-builder
WORKDIR /app
COPY client/package*.json ./
RUN npm ci
COPY client .
COPY client/.env.docker .env
RUN npm run build

# PHP dependencies stage
FROM php-base AS php-dependency-installer
WORKDIR /app
COPY composer.* artisan ./
COPY app/helpers.php app/helpers.php
RUN composer install --no-scripts --no-autoloader
COPY . .
RUN composer dump-autoload --optimize && composer run-script post-autoload-dump

# Final stage
FROM php-base
ARG PHP_VERSION
ARG POSTGRES_VERSION

# Install additional packages
RUN apt-get update && apt-get install -y \
    supervisor \
    nginx \
    sudo \
    redis-server \
    postgresql-${POSTGRES_VERSION} \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Setup Node.js
RUN useradd -m nuxt \
    && su nuxt -c "curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash" \
    && su nuxt -c ". ~/.nvm/nvm.sh && nvm install ${NODE_VERSION}"

# Copy configuration files
COPY docker/postgres-wrapper.sh docker/php-fpm-wrapper.sh docker/redis-wrapper.sh \
    docker/nuxt-wrapper.sh docker/generate-api-secret.sh /usr/local/bin/
COPY docker/php-fpm.conf /etc/php/${PHP_VERSION}/fpm/pool.d/
COPY docker/nginx.conf /etc/nginx/sites-enabled/default
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Copy application files
WORKDIR /app
COPY --from=javascript-builder /app/.output/ ./nuxt/
COPY --from=php-dependency-installer /app .
COPY .env.docker .env
COPY client/.env.docker client/.env

# Set permissions and configurations
RUN cp -r nuxt/public . \
    && chmod -R 777 /app/client /var/run /app/.env \
    && echo 'variables_order = "EGPCS"' >> /etc/php/${PHP_VERSION}/cli/php.ini \
    && echo 'variables_order = "EGPCS"' >> /etc/php/${PHP_VERSION}/fpm/php.ini \
    && chmod a+x /usr/local/bin/*.sh /app/artisan \
    && ln -s /app/artisan /usr/local/bin/artisan \
    && useradd opnform \
    && echo "daemon off;" >> /etc/nginx/nginx.conf \
    && echo "daemonize no" >> /etc/redis/redis.conf \
    && echo "appendonly yes" >> /etc/redis/redis.conf \
    && echo "dir /persist/redis/data" >> /etc/redis/redis.conf

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
