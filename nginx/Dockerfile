# syntax=docker/dockerfile:1

ARG NGINX_IMAGE=nginx:1.30.4

# ==================================================
# Stage 1: сборка GeoIP2-модуля
# ==================================================
FROM ${NGINX_IMAGE} AS builder

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        curl \
        git \
        libmaxminddb-dev \
        libpcre2-dev \
        zlib1g-dev \
        libssl-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

RUN git clone --depth 1 \
    https://github.com/leev/ngx_http_geoip2_module.git

RUN curl -fsSL \
    https://nginx.org/download/nginx-1.30.4.tar.gz \
    -o nginx.tar.gz \
    && tar -xzf nginx.tar.gz

WORKDIR /build/nginx-1.30.4

RUN ./configure \
    --with-compat \
    --add-dynamic-module=/build/ngx_http_geoip2_module \
    && make modules

# ==================================================
# Stage 2: финальный Nginx
# ==================================================
FROM ${NGINX_IMAGE}

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libmaxminddb0 \
        mmdb-bin \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder \
    /build/nginx-1.30.4/objs/ngx_http_geoip2_module.so \
    /usr/lib/nginx/modules/ngx_http_geoip2_module.so