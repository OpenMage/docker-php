FROM openmage/php-base:stretch-latest AS php

COPY FS /

ARG PHP_RUNTIME_REQUIREMENTS="curl libcurl3-gnutls ca-certificates libedit2 libfreetype6 libltdl7 libmcrypt4  libpng16-16 libzip4 libfcgi-bin libtidy5 cron vim.tiny"
ARG PHP_RUNTIME_REQUIREMENTS_EXTRA=""
ARG PHP_BUILD_REQUIREMENTS="jq build-essential dpkg-dev automake autoconf libtool file g++ gcc libc-dev make pkg-config re2c libxml2-dev libcurl4-gnutls-dev libtidy-dev libsqlite3-dev libssl1.0-dev libxml2-dev zlib1g-dev libpng-dev libreadline6-dev libfreetype6-dev libmcrypt-dev libedit-dev  libltdl-dev libxslt-dev libzip-dev"
ARG PHP_BUILD_REQUIREMENTS_EXTRA=""
ARG PHP_PECL_REQUIREMENTS="imagick-3.4.4 xdebug-2.8.0 redis-5.1.1 igbinary-3.1.2 APCu-5.1.18"
ARG PHP_CONFIGURE_ARGS="--enable-soap --enable-intl"
ARG PHP_CONFIGURE_ARGS_EXTRA=""
ARG PHP_CONFIGURE_ARGS_FPM="--enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data"
ARG PHP_VERSION="7.1"

ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2" \
    PHP_CPPFLAGS="-fstack-protector-strong -fpic -fpie -O2" \
    PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie" \
    PHP_VERSION="${PHP_VERSION}" \
    PHP_INI_DIR="/usr/local/etc/php" \
    PHP_SRC_DIR="/usr/src/php" \
    PHP_SRC_FILE="/usr/src/php.tar.gz" \
    PHPIZE_BUILD_REQUIREMENTS="build-essential automake autoconf libtool file g++ gcc libc-dev make pkg-config re2c libltdl-dev" \
    DOCUMENT_ROOT="/var/www/html"

### configure php
RUN set -xe; \
    \
    /usr/local/bin/docker-install-requirements php; \
    ln -fs /etc/alternatives/vi /usr/bin/vim; \
    mkdir -p /tmp/mhsendmail; \
    cd /tmp/mhsendmail; \
    docker-package-download -o /usr/local/bin/mhsendmail "https://github.com/mailhog/mhsendmail/releases/download/v0.2.0/mhsendmail_linux_amd64"; \
    chmod +x /usr/local/bin/mhsendmail; \
    mkdir -p "${PHP_INI_DIR}/conf.d"; \
    docker-php-source extract; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
	if [[ ! -d "/usr/include/curl" ]]; then \
	    ln -sT "/usr/include/${debMultiarch}/curl" /usr/local/include/curl; \
	fi; \
    cd /usr/src/php; \
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
        --with-mcrypt \
        --with-mhash \
        --enable-ftp \
        --enable-mbstring \
        --with-curl \
        --with-libedit \
        --with-openssl \
        --with-zlib \
        --enable-zip \
        --enable-exif \
        --enable-bcmath \
        --enable-sockets \
        --with-xsl \
        --with-gd \
        --with-tidy \
        --with-freetype-dir=/usr/include/ \
        --with-jpeg-dir=shared,/usr/include/ \
        --with-webp-dir=/usr/include/ \
        --with-png-dir=/usr/include/ \
        ${PHP_CONFIGURE_ARGS:-} \
        ${PHP_CONFIGURE_ARGS_EXTRA:-} \
        ${PHP_CONFIGURE_ARGS_FPM:-} \
            ; \
    make -j "$(nproc)"; \
    make install; \
    docker-php-pecl-install "${PHP_PECL_REQUIREMENTS}"; \
    docker-php-ext-enable opcache igbinary; \
    docker-php-source delete; \
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
    } | tee /usr/local/etc/php-fpm.d/zz-docker.conf; \
    /usr/local/bin/docker-layer-clean

ARG PHP_EXT_BLACKFIRE_VERSION="1.31.0"
ARG PHP_EXT_NEWRELIC_VERSION="9.9.0.260"
ARG PHP_EXT_TIDEWAYS_VERSION="5.0.58"

## configure php extensions
RUN set -xe; \
    \
    ################################################
    ## install blackfire
    ################################################
    mkdir -p /tmp/blackfire; \
    cd /tmp/blackfire; \
    docker-package-download -o blackfire.so -A "Docker" -D - -L -s "https://packages.blackfire.io/binaries/blackfire-php/${PHP_EXT_BLACKFIRE_VERSION}/blackfire-php-linux_amd64-php-$(php -r \@phpinfo\(\)\; | grep 'PHP Version' -m 1 | grep -oE '([0-9\.]+)' | cut -d. -f1,2 | tr -d '.').so"; \
    cp blackfire.so "$(/usr/local/bin/php-config --extension-dir)/blackfire.so"; \
    ################################################
    ## install newrelic
    ################################################
    mkdir -p /tmp/newrelic; \
    cd /tmp/newrelic; \
	docker-package-download -o newrelic.tar.gz -s "https://download.newrelic.com/php_agent/archive/${PHP_EXT_NEWRELIC_VERSION}/newrelic-php5-${PHP_EXT_NEWRELIC_VERSION}-linux.tar.gz"; \
    tar --strip 1 -xzf newrelic.tar.gz; \
    cp "agent/$(uname -m | grep -oE '(x?[0-9]+)' | head -1)/newrelic-$(phpize -v | grep Zend\ M | grep -oE '([0-9]+)').so" "$(/usr/local/bin/php-config --extension-dir)/newrelic.so"; \
    ls -lisah "$(/usr/local/bin/php-config --extension-dir)/newrelic.so"; \
    ################################################
    ## install tideways
    ################################################
    mkdir -p /tmp/tideways; \
    cd /tmp/tideways; \
 	docker-package-download -o tideways.tar.gz -s https://s3-eu-west-1.amazonaws.com/tideways/extension/${PHP_EXT_TIDEWAYS_VERSION}/tideways-php-${PHP_EXT_TIDEWAYS_VERSION}-x86_64.tar.gz; \
    tar --strip 2 -xzf tideways.tar.gz; \
    ls -lisah; \
    cp "tideways-php-$(php -r \@phpinfo\(\)\; | grep 'PHP Version' -m 1 | grep -oE '([0-9\.]+)' | cut -d. -f1,2).so" "$(/usr/local/bin/php-config --extension-dir)/tideways.so"; \
    ls -lisah "$(/usr/local/bin/php-config --extension-dir)/tideways.so"; \
    docker-php-source delete; \
    ################################################
    ## install Composer
    ################################################
    COMPOSER_DOWNLOAD_LATEST=$(curl -s curl -s https://api.github.com/repos/composer/composer/releases/latest | grep "browser_download_url.*phar" | cut -d '"' -f 4) ; \
    docker-package-download -o /usr/local/bin/composer ${COMPOSER_DOWNLOAD_LATEST}; \
    chmod +x /usr/local/bin/composer; \
    ################################################
    ## install magerun
    ################################################
    curl -s -f -L -o /usr/local/bin/magerun https://files.magerun.net/n98-magerun.phar; \
    chmod +x /usr/local/bin/magerun; \
    ################################################
    ## install magerun2
    ################################################
    curl -s -f -L -o /usr/local/bin/magerun2 https://files.magerun.net/n98-magerun2.phar; \
    chmod +x /usr/local/bin/magerun2; \
    /usr/local/bin/docker-layer-clean

VOLUME ["/var/www/"]
WORKDIR /var/www/html
ENTRYPOINT ["docker-entrypoint"]
CMD ["php-fpm"]
EXPOSE 9000

FROM php AS toolbox

ARG TOOLBOX_RUNTIME_REQUIREMENTS="curl git redis-tools mariadb-client percona-toolkit rsync openssh-client vim.tiny python3"
ARG TOOLBOX_RUNTIME_REQUIREMENTS_EXTRA=""
ARG TOOLBOX_BUILD_REQUIREMENTS=""
ARG TOOLBOX_BUILD_REQUIREMENTS_EXTRA=""

RUN set -xe; \
    \
    /usr/local/bin/docker-install-requirements toolbox; \
    update-alternatives --install /usr/bin/python python /usr/bin/python3 0; \
    cd /tmp; \
    curl -s https://bootstrap.pypa.io/get-pip.py -o get-pip.py; \
    python get-pip.py; \
    pip install awscli --upgrade; \
    rm get-pip.py; \
    curl https://rclone.org/install.sh | bash;\
    /usr/local/bin/docker-layer-clean

CMD ["php", "-a"]