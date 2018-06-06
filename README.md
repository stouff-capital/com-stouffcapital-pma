# com-stouffcapital-pma

https://hub.docker.com/r/phpmyadmin/phpmyadmin/

## Environment variables summary

- `PMA_ARBITRARY` when set to 1 connection to the arbitrary server will be allowed
- `PMA_HOST` define address/host name of the MySQL server
- `PMA_VERBOSE` define verbose name of the MySQL server
- `PMA_PORT` define port of the MySQL server
- `PMA_HOSTS` define comma separated list of address/host names of the MySQL servers
- `PMA_VERBOSES` define comma separated list of verbose names of the MySQL servers
- `PMA_PORTS` define comma separated list of ports of the MySQL servers
- `PMA_USER` and `PMA_PASSWORD` define username to use for config authentication method
- `PMA_ABSOLUTE_URI` define user-facing URI


## Dockerfile

```
FROM php:7.2-fpm-alpine RUN apk add --no-cache \
    nginx \
    supervisor
# Install dependencies RUN set -ex; \
    \
    apk add --no-cache --virtual .build-deps \
        bzip2-dev \
        freetype-dev \
        libjpeg-turbo-dev \
        libpng-dev \
        libwebp-dev \
        libxpm-dev \
    ; \
    \
    docker-php-ext-configure gd --with-freetype-dir=/usr --with-jpeg-dir=/usr --with-webp-dir=/usr --with-png-dir=/usr --with-xpm-dir=/usr; \
    docker-php-ext-install bz2 gd mysqli opcache zip; \
    \
    runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )"; \
    apk add --virtual .phpmyadmin-phpexts-rundeps $runDeps; \
    apk del .build-deps
# Copy configuration COPY etc /etc/
# Copy main script COPY run.sh /run.sh
RUN chmod u+rwx /run.sh
# Calculate download URL ENV VERSION 4.8.1 ENV URL https://files.phpmyadmin.net/phpMyAdmin/${VERSION}/phpMyAdmin-${VERSION}-all-languages.tar.gz LABEL version=$VERSION
# Download tarball, verify it using gpg and extract RUN set -ex; \
    apk add --no-cache --virtual .fetch-deps \
        gnupg \
    ; \
    \
    export GNUPGHOME="$(mktemp -d)"; \
    export GPGKEY="3D06A59ECE730EB71B511C17CE752F178259BD92"; \
    curl --output phpMyAdmin.tar.gz --location $URL; \
    curl --output phpMyAdmin.tar.gz.asc --location $URL.asc; \
    gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$GPGKEY" \
        || gpg --keyserver ipv4.pool.sks-keyservers.net --recv-keys "$GPGKEY" \
        || gpg --keyserver keys.gnupg.net --recv-keys "$GPGKEY" \
        || gpg --keyserver pgp.mit.edu --recv-keys "$GPGKEY" \
        || gpg --keyserver keyserver.pgp.com --recv-keys "$GPGKEY"; \
    gpg --batch --verify phpMyAdmin.tar.gz.asc phpMyAdmin.tar.gz; \
    rm -rf "$GNUPGHOME"; \
    tar xzf phpMyAdmin.tar.gz; \
    rm -f phpMyAdmin.tar.gz phpMyAdmin.tar.gz.asc; \
    mv phpMyAdmin-$VERSION-all-languages /www; \
    rm -rf /www/setup/ /www/examples/ /www/test/ /www/po/ /www/composer.json /www/RELEASE-DATE-$VERSION; \
    sed -i "s@define('CONFIG_DIR'.*@define('CONFIG_DIR', '/etc/phpmyadmin/');@" /www/libraries/vendor_config.php; \
    chown -R root:nobody /www; \
    find /www -type d -exec chmod 750 {} \; ; \
    find /www -type f -exec chmod 640 {} \; ; \
    apk del .fetch-deps
# Add directory for sessions to allow session persistence RUN mkdir /sessions \
    && mkdir -p /www/tmp \
    && chmod -R 777 /www/tmp
# We expose phpMyAdmin on port 80 EXPOSE 80 ENTRYPOINT [ "/run.sh" ]
CMD ["phpmyadmin"]
```
