# Utilisation de PHP 8.1.2 FPM Alpine
ARG PHP_VERSION=8.1.2
FROM php:${PHP_VERSION}-fpm-alpine AS app_php

# Désinstallation de Composer 1 si nécessaire
RUN rm -f /usr/local/bin/composer

# Installation de Composer 2
RUN set -eux; \
    wget https://getcomposer.org/installer -O composer-setup.php; \
    php composer-setup.php --install-dir=/usr/local/bin --filename=composer; \
    rm composer-setup.php

# Définition du répertoire de travail
ARG WORKDIR=/app
WORKDIR ${WORKDIR}

# Configuration PHP
RUN docker-php-source extract \
    && apk add --update --virtual .build-deps autoconf g++ make pcre-dev icu-dev openssl-dev libxml2-dev libmcrypt-dev git libpng-dev \
    && apk add postgresql-dev \
    && docker-php-ext-install pgsql pdo_pgsql \
    && apk del postgresql-libs libsasl db \
    && pecl install apcu \
    && docker-php-ext-enable apcu opcache \
    && apk add icu-libs icu \
    && docker-php-ext-install intl \
    && runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )" \
    && apk add --no-cache --virtual .app-phpexts-rundeps $runDeps \
    && pecl clear-cache \
    && docker-php-source delete \
    && apk del --purge .build-deps \
    && rm -rf /tmp/pear \
    && rm -rf /var/cache/apk/*

# Installation Composer 1 (optionnel si nécessaire pour des raisons spécifiques)
 COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# Copie des fichiers de configuration
COPY docker/php/php.ini $PHP_INI_DIR/conf.d/php.ini
COPY docker/php/php-cli.ini $PHP_INI_DIR/conf.d/php-cli.ini

# Installation des dépendances PHP avec Composer
COPY composer.json composer.lock symfony.lock ./
RUN set -eux; \
    composer install --prefer-dist --no-autoloader --no-scripts --no-progress --no-suggest; \
    composer clear-cache

# Copie des fichiers de l'application
COPY .env ./
COPY bin bin/
COPY config config/
COPY src src/
COPY public public/

# Création des répertoires de cache et de log
RUN set -eux \
    && mkdir -p var/cache var/log \
    && composer dump-autoload --classmap-authoritative 

# Configuration de l'utilisateur et du point d'entrée
COPY docker/php/docker-entrypoint.sh /usr/local/bin/docker-entrypoint
RUN chmod +x /usr/local/bin/docker-entrypoint


ENTRYPOINT ["docker-entrypoint"]
CMD ["php-fpm"]
