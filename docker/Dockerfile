FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt update \
    && apt install -y gnupg tzdata \
    && echo "CET" > /etc/timezone \
    && dpkg-reconfigure -f noninteractive tzdata

RUN apt update \
    && apt install -y curl zip unzip git supervisor sqlite3 \
       nginx php-fpm php-cli \
       php-pgsql php-sqlite3 php-gd \
       php-curl php-memcached \
       php-imap php-mysql php-mbstring \
       php-xml php-zip php-bcmath php-soap \
       php-intl php-readline php-xdebug \
       php-msgpack php-igbinary \
    && php -r "readfile('http://getcomposer.org/installer');" | php -- --install-dir=/usr/bin/ --filename=composer \
    && apt -y autoremove \
    && apt clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    && echo "daemon off;" >> /etc/nginx/nginx.conf


RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log

RUN ln -sf /etc/nginx/sites-available/* /etc/nginx/sites-enabled/*

COPY app.conf /etc/nginx/sites-available/default

COPY nginx.conf /etc/nginx

COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

COPY php-fpm.conf /etc/php/7.4/fpm/php-fpm.conf

COPY script.sh /usr/bin/script

RUN chmod +x /usr/bin/script

EXPOSE 80

ENTRYPOINT ["script"]