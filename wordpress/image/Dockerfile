#Dockerfile Wordpress 
#Apache and PHP 
FROM php:7.4-apache

# persistent dependencies
RUN set -eux; \
  apt-get update; \
  apt-get install -y --no-install-recommends \
# Ghostscript is required for rendering PDF previews
        ghostscript \
### msmtp und Hilfsprogramme für Mailversand
        msmtp \
        msmtp-mta \
        bsd-mailx \
        mailutils \
    ; \
    rm -rf /var/lib/apt/lists/*

# Install PHP-ZIP
RUN set -ex; \
    \
    savedAptMark="$(apt-mark showmanual)"; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
            libfreetype6-dev \
            libjpeg-dev \
            libpng-dev \
            libzip-dev \
            libmagickwand-dev \
        ; \
        \
        docker-php-ext-configure gd \
                --with-freetype \
                --with-jpeg \
        ; \
        \
        docker-php-ext-install -j "$(nproc)" \
            bcmath \
            exif \
            gd \
            mysqli \
            zip \
            pdo_mysql \
        ; \
        \
        pecl install imagick; \
        docker-php-ext-enable \
              imagick \
              opcache \
        ; \
  \
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
        apt-mark auto '.*' > /dev/null; \
        apt-mark manual $savedAptMark; \
        ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
        | awk '/=>/ { print $3 }' \
        | sort -u \
        | xargs -r dpkg-query -S \
        | cut -d: -f1 \
        | sort -u \
        | xargs -rt apt-mark manual; \
        \
        apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    rm -rf /var/lib/apt/lists/*
#
# https://secure.php.net/manual/en/opcache.installation.php
# https://kinsta.com/blog/php-benchmarks/
RUN { \
    echo 'opcache.memory_consumption=128'; \
    echo 'opcache.interned_strings_buffer=8'; \
    echo 'opcache.max_accelerated_files=50000'; \
    echo 'opcache.revalidate_freq=60'; \
    echo 'opcache.fast_shutdown=1'; \
    echo 'opcache.enable_cli=1'; \
  } > /usr/local/etc/php/conf.d/opcache-recommended.ini
#
# https://wordpress.org/support/article/editing-wp-config-php/#configure-error-logging
# https://www.php.net/manual/en/errorfunc.constants.php
# https://github.com/docker-library/wordpress/issues/420#issuecomment-517839670
RUN { \
    echo 'error_reporting = E_ERROR | E_WARNING | E_PARSE | E_CORE_ERROR | E_CORE_WARNING | E_COMPILE_ERROR | E_COMPILE_WARNING | E_RECOVERABLE_ERROR'; \
    echo 'display_errors = Off'; \
    echo 'display_startup_errors = Off'; \
    echo 'log_errors = On'; \
    echo 'error_log = /dev/stderr'; \
    echo 'log_errors_max_len = 1024'; \
    echo 'ignore_repeated_errors = On'; \
    echo 'ignore_repeated_source = Off'; \
    echo 'html_errors = Off'; \
  } > /usr/local/etc/php/conf.d/error-logging.ini
#
### File Upload auf 2G und Memory-Limits auf 512 MB hochschieben. Execution Timeout auf 30m (1800s)
RUN { \
    echo 'file_uploads = On'; \
    echo 'upload_max_filesize = 2G'; \
    echo 'post_max_size = 2G'; \
    echo 'memory_limit = 512M'; \
    echo 'max_execution_time = 1800'; \
    echo 'default_socket_timeout = 1800'; \
  } > /usr/local/etc/php/conf.d/upload-limits.ini
#

RUN set -eux; \
    a2enmod rewrite expires headers; \
        \
# https://httpd.apache.org/docs/2.4/mod/mod_remoteip.html
    a2enmod remoteip; \
    { \
        echo 'RemoteIPHeader X-Forwarded-For'; \
# these IP ranges are reserved for "private" use and should thus *usually* be safe inside Docker
        echo 'RemoteIPTrustedProxy 10.0.0.0/8'; \
        echo 'RemoteIPTrustedProxy 172.16.0.0/12'; \
        echo 'RemoteIPTrustedProxy 192.168.0.0/16'; \
        echo 'RemoteIPTrustedProxy 169.254.0.0/16'; \
        echo 'RemoteIPTrustedProxy 127.0.0.0/8'; \
    } > /etc/apache2/conf-available/remoteip.conf; \
    a2enconf remoteip; \
#
# https://github.com/docker-library/wordpress/issues/383#issuecomment-507886512
# (replace all instances of "%h" with "%a" in LogFormat)
find /etc/apache2 -type f -name '*.conf' -exec sed -ri 's/([[:space:]]*LogFormat[[:space:]]+"[^"]*)%h([^"]*")/\1%a\2/g' '{}' +

ENV WORDPRESS_VERSION 5.6
ENV WORDPRESS_SHA1 e5631f812232fbd45d3431783d3db2e0d5670d2d

RUN set -ex; \
        curl -o wordpress.tar.gz -fSL "https://wordpress.org/wordpress-${WORDPRESS_VERSION}.tar.gz"; \
        echo "$WORDPRESS_SHA1 *wordpress.tar.gz" | sha1sum -c -; \
# upstream tarballs include ./wordpress/ so this gives us /usr/src/wordpress
        tar -xzf wordpress.tar.gz -C /usr/src/; \
        rm wordpress.tar.gz; \
        chown -R www-data:www-data /usr/src/wordpress; \
# pre-create wp-content (and single-level children) for folks who want to bind-mount themes, etc so permissions are pre-created properly instead of root:root
        mkdir wp-content; \
        for dir in /usr/src/wordpress/wp-content/*/ cache; do \
            dir="$(basename "${dir%/}")"; \
            mkdir "wp-content/$dir"; \
        done; \
    chown -R www-data:www-data wp-content; \
    chmod -R 777 wp-content

VOLUME /var/www/html

COPY docker-entrypoint.sh /usr/local/bin/

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["apache2-foreground"]

