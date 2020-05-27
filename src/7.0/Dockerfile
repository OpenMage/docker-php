FROM openmage/php-base:stretch-latest

COPY FS /

ARG PHP_RUNTIME_REQUIREMENTS="curl libcurl3-gnutls ca-certificates libedit2 libfreetype6 libltdl7 libmcrypt4  libpng16-16 libzip4 libfcgi-bin libtidy5"
ARG PHP_RUNTIME_REQUIREMENTS_EXTRA=""
ARG PHP_BUILD_REQUIREMENTS="jq build-essential dpkg-dev autoconf file g++ gcc libc-dev make pkg-config re2c libxml2-dev libcurl4-gnutls-dev libtidy-dev libsqlite3-dev libssl1.0-dev libxml2-dev zlib1g-dev libpng-dev libreadline6-dev libfreetype6-dev libmcrypt-dev libedit-dev  libltdl-dev libxslt-dev libzip-dev"
ARG PHP_BUILD_REQUIREMENTS_EXTRA=""
ARG PHP_PECL_REQUIREMENTS="imagick-3.4.4 xdebug-2.9.2 redis-5.1.1 igbinary-3.1.2 APCu-5.1.18"
ARG PHP_CONFIGURE_ARGS="--enable-soap --enable-intl"
ARG PHP_CONFIGURE_ARGS_EXTRA=""
ARG PHP_CONFIGURE_ARGS_FPM="--enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data"
ARG PHP_VERSION="7.0"

ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2" \
    PHP_CPPFLAGS="-fstack-protector-strong -fpic -fpie -O2" \
    PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie" \
    PHP_VERSION="${PHP_VERSION}" \
    PHP_INI_DIR="/usr/local/etc/php" \
    PHP_SRC_DIR="/usr/src/php" \
    PHP_SRC_FILE="/usr/src/php.tar.gz" \
    PHPIZE_BUILD_REQUIREMENTS="build-essential autoconf file g++ gcc libc-dev make pkg-config re2c libltdl-dev"

### configure php
RUN set -xe; \
    \
    /usr/local/bin/docker-install-requirements php; \
    /usr/local/bin/docker-php-source-prepare; \
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

## configure php extensions
RUN set -xe; \
    \
    ################################################
    ## install blackfire
    ################################################
    mkdir -p /tmp/blackfire; \
    cd /tmp/blackfire; \
    docker-package-download -o blackfire-probe.tar.gz -A "Docker" -D - -L -s "https://blackfire.io/api/v1/releases/probe/php/linux/amd64/$(php -r \@phpinfo\(\)\; | grep 'PHP Version' -m 1 | grep -oE '([0-9\.]+)' | cut -d. -f1,2 | tr -d '.')"; \
    tar -xzf blackfire-probe.tar.gz; \
    cp blackfire-*.so "$(/usr/local/bin/php-config --extension-dir)/blackfire.so"; \
    ls -lisah "$(/usr/local/bin/php-config --extension-dir)/blackfire.so"; \
    ################################################
    ## install tideways
    ################################################
#     mkdir -p /tmp/tideways; \
#     cd /tmp/tideways; \
# 	docker-package-download -o tideways.tar.gz -s https://s3-eu-west-1.amazonaws.com/tideways/extension/${PHP_EXT_TIDEWAYS_VERSION:-5.0.34}/tideways-php-${PHP_EXT_TIDEWAYS_VERSION:-5.0.34}-x86_64.tar.gz; \
#     tar --strip 1 -xzf tideways.tar.gz; \
#     cp "tideways-${PHP_EXT_TIDEWAYS_VERSION:-5.0.34}/tideways-php-$(php -r \@phpinfo\(\)\; | grep 'PHP Version' -m 1 | grep -oE '([0-9\.]+)' | cut -d. -f1,2).so" "$(/usr/local/bin/php-config --extension-dir)/tideways.so"; \
#     ls -lisah "$(/usr/local/bin/php-config --extension-dir)/tideways.so"; \
    ################################################
    ## install ioncube
    ################################################
    mkdir -p /tmp/ioncube; \
    cd /tmp/ioncube; \
	docker-package-download -o ioncube.tar.gz -s https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz; \
    tar --strip 1 -xzf ioncube.tar.gz; \
    cp "ioncube_loader_lin_$(php -r \@phpinfo\(\)\; | grep 'PHP Version' -m 1 | grep -oE '([0-9\.]+)' | cut -d. -f1,2).so" "$(/usr/local/bin/php-config --extension-dir)/ioncube.so"; \
    ls -lisah "$(/usr/local/bin/php-config --extension-dir)/ioncube.so"; \
    ################################################
    ## install newrelic
    ################################################
    mkdir -p /tmp/newrelic; \
    cd /tmp/newrelic; \
	docker-package-download -o newrelic.tar.gz -s https://download.newrelic.com/php_agent/archive/${PHP_EXT_NEWRELIC_VERSION:-8.5.0.235}/newrelic-php5-${PHP_EXT_NEWRELIC_VERSION:-8.5.0.235}-linux.tar.gz; \
    tar --strip 1 -xzf newrelic.tar.gz; \
    cp "agent/$(uname -m | grep -oE '(x?[0-9]+)' | head -1)/newrelic-$(phpize -v | grep Zend\ M | grep -oE '([0-9]+)').so" "$(/usr/local/bin/php-config --extension-dir)/newrelic.so"; \
    ls -lisah "$(/usr/local/bin/php-config --extension-dir)/newrelic.so"; \
    docker-php-source delete; \
    /usr/local/bin/docker-layer-clean

VOLUME ["/var/www/"]
WORKDIR /var/www/html
ENTRYPOINT ["docker-entrypoint"]
EXPOSE 9000