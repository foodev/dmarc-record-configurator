ARG ARCHITECTURE

FROM ${ARCHITECTURE}node:buster AS build/css

COPY public/css /usr/src/

RUN npm install -g csso-cli \
    && \
    csso --source-map file --input /usr/src/app.css --output /usr/src/app.css

ARG ARCHITECTURE

FROM ${ARCHITECTURE}debian:buster AS build/webserver

ARG ARCHITECTURE

#
# Install dependencies
#
RUN apt-get update && apt-get install --assume-yes \
    wget \
    build-essential \
    unzip \
    zlib1g-dev \
    libssl-dev \
    libpcre3-dev \
    ca-certificates

ARG NGINX_VERSION=1.19.6

#
# Download and compile nginx (dynamically linked)
# https://nginx.org/en/docs/configure.html
#
RUN wget "https://nginx.org/download/nginx-$NGINX_VERSION.tar.gz" --output-document=- | \
        tar --extract --gzip --file=- --directory /

WORKDIR /nginx-$NGINX_VERSION

RUN ./configure \
        --conf-path=/etc/nginx/nginx.conf \
        --sbin-path=/bin/nginx \
        --pid-path=/dev/null \
        --http-client-body-temp-path=/tmp/client_body_temp \
        --http-fastcgi-temp-path=/tmp/fastcgi_temp \
        --http-log-path=/dev/stdout \
        --error-log-path=/dev/stderr \
        --user=www-data \
        --group=www-data \
        --with-http_ssl_module \
        --with-http_v2_module \
        --with-pcre \
        --without-http_ssi_module \
        --without-http_userid_module \
        --without-http_auth_basic_module \
        --without-http_mirror_module \
        --without-http_autoindex_module \
        --without-http_geo_module \
        --without-http_map_module \
        --without-http_split_clients_module \
        --without-http_referer_module \
        --without-http_proxy_module \
        --without-http_uwsgi_module \
        --without-http_scgi_module \
        --without-http_grpc_module \
        --without-http_memcached_module \
        --without-http_limit_conn_module \
        --without-http_limit_req_module \
        --without-http_empty_gif_module \
        --without-http_browser_module \
        --without-http_upstream_hash_module \
        --without-http_upstream_ip_hash_module \
        --without-http_upstream_least_conn_module \
        --without-http_upstream_keepalive_module \
        --without-http_upstream_zone_module && \
    make

#
# Create necessary folders for the build
#
RUN mkdir --parents \
    /build/bin \
    /build/etc/nginx \
    /build/etc/ssl \
    /build/usr/lib/ssl \
    /build/usr/src/app

#
# Copy all necessary libraries
#
RUN if [ "$ARCHITECTURE" = "arm32v7" ]; then \
        ld_linux="/lib/ld-linux-armhf.so.3"; \
        lib_root="/lib/arm-linux-gnueabihf"; \
    else \
        ld_linux="/lib64/ld-linux-x86-64.so.2"; \
        lib_root="/lib/x86_64-linux-gnu"; \
    fi; \
    \
    for library in \
        $(ldd /nginx-$NGINX_VERSION/objs/nginx | grep '=> ' | awk '{print $3}') \
        \
        "$ld_linux" \
        \
        # Needed to run nginx as different user
        "$lib_root/libnss_files.so.2" \
        \
        # Needed to resolve any hostnames
        "$lib_root/libresolv.so.2" \
        "$lib_root/libnss_dns.so.2"; \
    do \
        mkdir --parents "/build$(dirname $library)"; \
        cp --archive $library "/build$(dirname $library)"; \
        \
        if [ -L $library ]; then \
            mkdir --parents "/build$(dirname `realpath $library`)"; \
            cp --archive $(realpath $library) "/build$(dirname `realpath $library`)"; \
        fi \
    done

#
# Copy openssl configuration and certificates
#
RUN cp --archive /usr/lib/ssl /build/usr/lib/ && \
    cp --archive /etc/ssl /build/etc/

#
# Copy nginx binary
#
RUN cp --archive /nginx-$NGINX_VERSION/objs/nginx /build/bin/nginx

#
# Copy configuration files
#
COPY container/etc /build/etc

#
# Copy source code
#
COPY public /build/usr/src/app/
COPY --from=build/css /usr/src/ /build/usr/src/app/css/

#
# Everything below this line will be the actual content of the image
#
FROM scratch

COPY --from=build/webserver /build /

WORKDIR /usr/src/app

EXPOSE 8080

ENTRYPOINT [ "/bin/nginx", "-g", "daemon off;" ]
