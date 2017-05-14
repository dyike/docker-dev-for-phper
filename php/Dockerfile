FROM  php:7.1.4-fpm

MAINTAINER ityike<yuanfeng634@gmail.com>

ADD php.ini    /usr/local/etc/php/php.ini

# Must install dependencies for your extensions, if need.
RUN apt-get update && apt-get install -y \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libmcrypt-dev \
    libpng12-dev \
    vim \
    git \
    libxml2-dev \
    build-essential \
    openssl \
    libssl-dev \
    make \
    curl \
    libcurl4-gnutls-dev \
    libjpeg-dev \
    libreadline6 \
    libreadline6-dev \

    && docker-php-ext-install -j$(nproc) iconv mcrypt \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ --with-mcrypt=/usr/include \
    && docker-php-ext-install gd \

# no dependency extension

    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-install sockets \
    && docker-php-ext-install gettext \
    && docker-php-ext-install opcache \

# Install PECL extensions

RUN apt-get install -y \

# for memcached
    libmemcached-dev \

#    && pecl install gearman && docker-php-ext-enable gearman \
#    && pecl install yaf && docker-php-ext-enable yaf \

    && pecl install xdebug && docker-php-ext-enable xdebug \
    && pecl install redis && docker-php-ext-enable redis \
    && pecl install xhprof && docker-php-ext-enable xhprof \

    && apt-get clean; rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/* \
    && echo 'PHP 7.1 installed.'

# Other extensions
# RUN curl -fsSL 'https://xcache.lighttpd.net/pub/Releases/3.2.0/xcache-3.2.0.tar.gz' -o xcache.tar.gz \
#     && mkdir -p xcache \
#     && tar -xf xcache.tar.gz -C xcache --strip-components=1 \
#     && rm xcache.tar.gz \
#     && ( \
#         cd xcache \
#         && phpize \
#         && ./configure --enable-xcache \
#         && make -j$(nproc) \
#         && make install \
#     ) \
#     && rm -r xcache \
#     && docker-php-ext-enable xcache


# open pid file
# RUN sed -i '/^;pid\s*=\s*/s/\;//g' /usr/local/etc/php-fpm.d/www.conf \
RUN sed -i '/^;pid\s*=\s*/s/\;//g' /usr/local/etc/php-fpm.conf \

# add php-fpm to service

    && cp /usr/src/php/sapi/fpm/init.d.php-fpm.in /etc/init.d/php-fpm && chmod +x /etc/init.d/php-fpm

# && chkconfig --add php-fpm

# composer tool
# download from https://getcomposer.org/download/   and then add to php-tools/

ADD composer.phar /usr/local/bin/composer
RUN chmod 755 /usr/local/bin/composer

WORKDIR "/var/www"

# Volumes

VOLUME ["/var/www"]

# extends from parent
# EXPOSE 9000
# CMD ["php-fpm"]
