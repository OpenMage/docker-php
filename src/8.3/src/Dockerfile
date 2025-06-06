FROM openmage/php-base:bookworm-latest AS php

COPY FS /

ARG PHP_RUNTIME_REQUIREMENTS="jq curl libcurl3-gnutls ca-certificates libedit2 libgmp-dev libfreetype6 libltdl7 libmcrypt4 sqlite3 libpng16-16 libzip4 libfcgi-bin libtidy5deb1 libsodium23 libonig5 cron vim.tiny php-xdebug zip"
ARG PHP_RUNTIME_REQUIREMENTS_EXTRA=""
ARG PHP_BUILD_REQUIREMENTS="build-essential curl dpkg-dev automake autoconf libtool file g++ gcc libc-dev make pkg-config re2c libxml2-dev libcurl4-gnutls-dev libtidy-dev libsqlite3-dev libssl-dev libxml2-dev zlib1g-dev libpng-dev liblzf-dev libreadline6-dev libfreetype6-dev libmcrypt-dev libedit-dev  libltdl-dev libxslt-dev libzip-dev libsodium-dev libonig-dev gnupg2 argon2 libargon2-0 libargon2-dev libzip-dev"
ARG PHP_BUILD_REQUIREMENTS_EXTRA=""
ARG PHP_PECL_REQUIREMENTS="redis igbinary"
ARG PHP_CONFIGURE_ARGS="--enable-soap --enable-intl"
ARG PHP_CONFIGURE_ARGS_EXTRA=""
ARG PHP_CONFIGURE_ARGS_FPM="--enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data"
ARG PHP_VERSION="8.3.21"

ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2" \
    PHP_CPPFLAGS="-fstack-protector-strong -fpic -fpie -O2" \
    PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie" \
    PHP_VERSION="${PHP_VERSION}" \
    PHP_INI_DIR="/usr/local/etc/php" \
    PHP_SRC_DIR="/usr/src/php" \
    PHP_SRC_FILE="/usr/src/php.tar.gz" \
    PHPIZE_BUILD_REQUIREMENTS="build-essential automake autoconf libtool file g++ gcc libc-dev make pkg-config re2c libltdl-dev" \
    DOCUMENT_ROOT="/var/www/html" \
    PHP_URL="https://www.php.net/distributions/php-${PHP_VERSION}.tar.xz" \
    IMAGICK_VERSION="3.7.0"

### configure php
# hadolint ignore=DL4006,SC2039,DL3003
SHELL ["/bin/bash", "-eo", "pipefail", "-c"]
RUN set -xe; \
    \
    chmod 755 /usr/local/bin/docker-entrypoint; \
    /usr/local/bin/docker-install-requirements php; \
    ln -fs /etc/alternatives/vi /usr/bin/vim; \
    mkdir -p /tmp/mhsendmail; \
    mkdir -p "${PHP_INI_DIR}/conf.d"; \
    mkdir -p /usr/src;
WORKDIR /tmp/mhsendmail
RUN set -xe; \
    \
    curl -LkSso /usr/local/bin/mhsendmail 'https://github.com/mailhog/mhsendmail/releases/download/v0.2.0/mhsendmail_linux_amd64'&& \
    chmod 0755 /usr/local/bin/mhsendmail; \
    chmod +x /usr/local/bin/mhsendmail;
WORKDIR /usr/src
RUN set -xe; \
    \
    curl -fsSL -o php.tar.xz "$PHP_URL"; \
    docker-php-source extract;
WORKDIR /usr/src/php
RUN set -xe; \
    \
    gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
    debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
    if [ ! -d "/usr/include/curl" ]; then \
        ln -sT "/usr/include/${debMultiarch}/curl" /usr/local/include/curl; \
    fi; \
    export \
        CFLAGS="$PHP_CFLAGS" \
        CPPFLAGS="$PHP_CPPFLAGS" \
        LDFLAGS="$PHP_LDFLAGS" \
    ; \
    ./configure \
        --build="$gnuArch" \
        --with-config-file-path="$PHP_INI_DIR" \
        --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
        --enable-option-checking=fatal \
        --disable-cgi \
        --with-pdo-mysql=mysqlnd \
        --enable-mysqlnd \
        --with-mhash \
        --enable-ftp \
        --enable-mbstring \
        --with-curl \
        --with-libedit \
        --with-openssl \
        --with-zlib \
        --enable-exif \
        --enable-bcmath \
        --enable-sockets \
        --with-xsl \
        --enable-pcntl \
        --with-sodium \
        --with-tidy \
        --enable-gd \
        --with-jpeg \
        --with-freetype \
        ${PHP_CONFIGURE_ARGS:-} \
        ${PHP_CONFIGURE_ARGS_EXTRA:-} \
        ${PHP_CONFIGURE_ARGS_FPM:-} \
            ; \
    make -j "$(nproc)"; \
    make install;
RUN curl -L -o /tmp/imagick.tar.gz https://github.com/Imagick/imagick/archive/refs/tags/3.7.0.tar.gz \
    && tar --strip-components=1 -xf /tmp/imagick.tar.gz \
    && phpize \
    && ./configure \
    && make \
    && make install \
    && echo "extension=imagick.so" > /usr/local/etc/php/conf.d/ext-imagick.ini \
    && rm -rf /tmp/*
RUN set -xe; \
    \
    docker-php-ext-install gd zip; \
    curl -fsSL  https://github.com/FriendsOfPHP/pickle/releases/latest/download/pickle.phar -o pickle.phar; \
    chmod +x pickle.phar; \
    mv pickle.phar /usr/local/bin/pickle; \
    pickle install redis; \
    pickle install igbinary; \
    pickle install xdebug; \
    docker-php-ext-enable imagick opcache redis imagick igbinary gd; \
    docker-php-source delete;
RUN set -xe; \
    \
    # for some reason, upstream's php-fpm.conf.default has "include=NONE/etc/php-fpm.d/*.conf"
    sed 's!=NONE/!=!g' /usr/local/etc/php-fpm.conf.default | tee /usr/local/etc/php-fpm.conf > /dev/null; \
    cp /usr/local/etc/php-fpm.d/www.conf.default /usr/local/etc/php-fpm.d/www.conf; \
    { \
        echo '[global]'; \
        echo 'error_log = /proc/self/fd/2'; \
        echo; \
        echo '[www]'; \
        echo '; if we send this to /proc/self/fd/1, it never appears'; \
        echo 'access.log = /proc/self/fd/2'; \
        echo; \
        echo 'clear_env = no'; \
        echo; \
        echo '; Ensure worker stdout and stderr are sent to the main error log.'; \
        echo 'catch_workers_output = yes'; \
    } | tee /usr/local/etc/php-fpm.d/docker.conf; \
    { \
        echo '[global]'; \
        echo 'daemonize = no'; \
        echo; \
        echo '[www]'; \
        echo 'listen = 9000'; \
        echo 'pm.max_children = 10'; \
        echo 'pm.min_spare_servers = 2'; \
        echo 'pm.max_spare_servers = 5'; \
        echo 'pm.status_path = /status'; \
    } | tee /usr/local/etc/php-fpm.d/zz-docker.conf;

## configure php extensions
# hadolint ignore=DL4006,SC2086,DL3003
RUN set -xe; \
    \
    ################################################
    ## install Composer 2 as default
    ################################################
    COMPOSER_DOWNLOAD_LATEST=$(curl -s https://api.github.com/repos/composer/composer/releases/latest | jq --raw-output '.assets[] | .browser_download_url' | head -n 1) ; \
    docker-package-download -o /usr/local/bin/composer ${COMPOSER_DOWNLOAD_LATEST}; \
    chmod +x /usr/local/bin/composer; \
    composer selfupdate;
RUN set -xe; \
    \
    ################################################
    ## install Composer 1
    ################################################
    docker-package-download -o /usr/local/bin/composer1 https://getcomposer.org/composer-1.phar; \
    chmod +x /usr/local/bin/composer1; \
    composer1 selfupdate;
RUN set -xe; \
    \
    ################################################
    ## install magerun
    ################################################
    curl -s -f -L -o /usr/local/bin/magerun https://files.magerun.net/n98-magerun.phar; \
    chmod +x /usr/local/bin/magerun;
RUN set -xe; \
    \
    ################################################
    ## install magerun2
    ################################################
    curl -s -f -L -o /usr/local/bin/magerun2 https://files.magerun.net/n98-magerun2.phar; \
    chmod +x /usr/local/bin/magerun2; \
    /usr/local/bin/docker-layer-clean;

WORKDIR /var/www/html
ENTRYPOINT ["docker-entrypoint"]
CMD ["php-fpm"]
EXPOSE 9000

FROM php AS toolbox

ARG TOOLBOX_RUNTIME_REQUIREMENTS="curl git redis-tools mariadb-client percona-toolkit rsync openssh-client vim.tiny php-xdebug"
ARG TOOLBOX_RUNTIME_REQUIREMENTS_EXTRA=""
ARG TOOLBOX_BUILD_REQUIREMENTS=""
ARG TOOLBOX_BUILD_REQUIREMENTS_EXTRA=""

SHELL ["/bin/bash", "-eo", "pipefail", "-c"]

RUN /usr/local/bin/docker-install-requirements toolbox; \
    curl https://rclone.org/install.sh | bash; \
    composer require deployer/deployer; \
    composer update; \
    echo "export PATH=$PATH:/var/www/html/vendor/bin" >> /etc/profile; \
    echo "xdebug.mode=coverage" > /usr/local/etc/php/conf.d/xdebug-coverage.ini; \
    /usr/local/bin/docker-layer-clean

CMD ["php", "-a"]
