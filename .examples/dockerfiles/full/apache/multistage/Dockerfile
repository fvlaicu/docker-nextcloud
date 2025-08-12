#
# Use a temporary image to compile and test the libraries
#
FROM nextcloud:30-apache as builder

# Build and install dlib on builder
RUN apt-get update ; \
    apt-get install -y build-essential wget cmake libx11-dev libopenblas-dev

ARG DLIB_BRANCH=v19.24.8
RUN wget -c -q https://github.com/davisking/dlib/archive/$DLIB_BRANCH.tar.gz \
    && tar xf $DLIB_BRANCH.tar.gz \
    && mv dlib-* dlib \
    && cd dlib/dlib \
    && mkdir build \
    && cd build \
    && cmake -DBUILD_SHARED_LIBS=ON -DCMAKE_BUILD_TYPE=Release .. \
    && make \
    && make install

# Build and install PDLib on builder
ARG PDLIB_BRANCH=master
RUN apt-get install unzip
RUN wget -c -q https://github.com/goodspb/pdlib/archive/$PDLIB_BRANCH.zip \
    && unzip $PDLIB_BRANCH \
    && mv pdlib-* pdlib \
    && cd pdlib \
    && phpize \
    && ./configure \
    && make \
    && make install

# Enable PDlib on builder

# If necesary take the php settings folder uncommenting the next line
# RUN php -i | grep "Scan this dir for additional .ini files"
RUN echo "extension=pdlib.so" > /usr/local/etc/php/conf.d/pdlib.ini

# Install bzip2 needed to extract models

RUN apt-get install -y libbz2-dev
RUN docker-php-ext-install bz2

# Test PDlib instalation on builder

RUN apt-get install -y git
RUN git clone https://github.com/matiasdelellis/pdlib-min-test-suite.git \
    && cd pdlib-min-test-suite \
    && make

#
# If pass the tests, we are able to create the final image.
#
FROM nextcloud:30-apache


RUN set -ex; \
    mkdir -p /usr/share/man/man1;\
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ffmpeg \
        ghostscript \
        libmagickcore-6.q16-6-extra \
        procps \
        smbclient \
        supervisor \
        libreoffice \
        wget \
        gnupg2 \
        unzip \
    ; \
    rm -rf /var/lib/apt/lists/*
RUN set -ex; \
    \
    savedAptMark="$(apt-mark showmanual)"; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libbz2-dev \
        libc-client-dev \
        libkrb5-dev \
        libsmbclient-dev \
    ; \
    \
    docker-php-ext-configure imap --with-kerberos --with-imap-ssl; \
    docker-php-ext-install \
        bz2 \
        imap \
    ; \
    pecl install smbclient; \
    docker-php-ext-enable smbclient; \
    \
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
    apt-mark auto '.*' > /dev/null; \
    apt-mark manual $savedAptMark; \
    ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
        | awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); print so }' \
        | sort -u \
        | xargs -r dpkg-query --search \
        | cut -d: -f1 \
        | sort -u \
        | xargs -rt apt-mark manual; \
    \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    rm -rf /var/lib/apt/lists/*

RUN echo '*/30 * * * * php -f /var/www/html/occ face:background_job -t 900' >> /var/spool/cron/crontabs/www-data
RUN echo '*/45 * * * * php -f /var/www/html/occ preview:pre-generate' >> /var/spool/cron/crontabs/www-data
RUN mkdir -p \
    /var/log/supervisord \
    /var/run/supervisord \
;

# Install dependencies to image

RUN apt-get update ; \
    apt-get install -y libopenblas-dev

# Install dlib and PDlib to image

COPY --from=builder /usr/local/lib/libdlib.so* /usr/local/lib

# If is necesary take the php extention folder (/usr/local/lib/php/extensions/no-debug-non-zts-20180731) uncommenting the next line
RUN php -i | grep extension_dir
COPY --from=builder /usr/local/lib/php/extensions/no-debug-non-zts-20230831/pdlib.so /usr/local/lib/php/extensions/no-debug-non-zts-20230831/

# Enable PDlib on final image
RUN echo "extension=pdlib.so" > /usr/local/etc/php/conf.d/pdlib.ini

# Increse memory limits
RUN echo memory_limit=1024M > /usr/local/etc/php/conf.d/memory-limit.ini

# Pdlib is already installed now on final image without all build dependencies.
# You could test again if everything is correct, uncommenting the next lines.
#
# RUN apt-get install -y git wget
# RUN git clone https://github.com/matiasdelellis/pdlib-min-test-suite.git \
#    && cd pdlib-min-test-suite \
#    && make

COPY supervisord.conf /

ENV NEXTCLOUD_UPDATE=1

CMD ["/usr/bin/supervisord", "-c", "/supervisord.conf"]
