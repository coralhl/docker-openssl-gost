FROM debian:bullseye-slim

ARG NGINX_VERSION=1.22.0
ARG NGINX_SHA256=b33d569a6f11a01433a57ce17e83935e953ad4dc77cdd4d40f896c88ac26eb53
ARG OPENSSL_VERSION=1_1_1q
ARG OPENSSL_SHA256=df86e6adcff1c91a85cef139dd061ea40b7e49005e8be16522cf4864bfcf5eb8

RUN apt-get update \
  && apt-get install wget build-essential libpcre++-dev libz-dev ca-certificates unzip cmake --no-install-recommends -y \
  && mkdir -p /usr/local/src \
  && cd /usr/local/src \
  && wget "http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" -O "nginx-${NGINX_VERSION}.tar.gz" \
  && echo "$NGINX_SHA256" "nginx-${NGINX_VERSION}.tar.gz" | sha256sum -c - \
  && tar -zxvf "nginx-${NGINX_VERSION}.tar.gz" \
  && wget "https://github.com/openssl/openssl/archive/${OPENSSL_VERSION}.zip" -O "${OPENSSL_VERSION}.zip" \
  && echo "$OPENSSL_SHA256" "${OPENSSL_VERSION}.zip" | sha256sum -c - \
  && unzip "${OPENSSL_VERSION}.zip" -d ./ \
  && cd "nginx-${NGINX_VERSION}" \
  && sed -i 's|--prefix=$ngx_prefix no-shared|--prefix=$ngx_prefix|' auto/lib/openssl/make \
  && ./configure \
  --prefix=/etc/nginx \
  --sbin-path=/usr/sbin/nginx \
  --modules-path=/usr/lib/nginx/modules \
  --conf-path=/etc/nginx/nginx.conf \
  --error-log-path=/var/log/nginx/error.log \
  --http-log-path=/var/log/nginx/access.log \
  --pid-path=/var/run/nginx.pid \
  --lock-path=/var/run/nginx.lock \
  --http-client-body-temp-path=/var/cache/nginx/client_temp \
  --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
  --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
  --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
  --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
  --user=www-data \
  --group=www-data \
  --with-compat \
  --with-file-aio \
  --with-threads \
  --with-http_addition_module \
  --with-http_auth_request_module \
  --with-http_dav_module \
  --with-http_flv_module \
  --with-http_gunzip_module \
  --with-http_gzip_static_module \
  --with-http_mp4_module \
  --with-http_random_index_module \
  --with-http_realip_module \
  --with-http_secure_link_module \
  --with-http_slice_module \
  --with-http_ssl_module \
  --with-http_stub_status_module \
  --with-http_sub_module \
  --with-http_v2_module \
  --with-mail \
  --with-mail_ssl_module \
  --with-stream \
  --with-stream_realip_module \
  --with-stream_ssl_module \
  --with-stream_ssl_preread_module \
  --with-openssl="/usr/local/src/openssl-${OPENSSL_VERSION}" \
  && make \
  && make install \
  && echo "/usr/local/src/openssl-${OPENSSL_VERSION}/.openssl/lib" >>/etc/ld.so.conf.d/ssl.conf && ldconfig \
  && cp "/usr/local/src/openssl-${OPENSSL_VERSION}/.openssl/bin/openssl" /usr/bin/openssl \
  && mkdir -p /var/cache/nginx/

ENV OPENSSL_DIR="/usr/local/src/openssl-${OPENSSL_VERSION}/.openssl"

# Build GOST-engine for OpenSSL
ARG GOST_ENGINE_VERSION=58a46b289d6b8df06072fc9c0304f4b2d3f4b051
ARG GOST_ENGINE_SHA256=6b47e24ee1ce619557c039fc0c1201500963f8f8dea83cad6d05d05b3dcc2255
RUN cd /usr/local/src \
  && wget "https://github.com/gost-engine/engine/archive/${GOST_ENGINE_VERSION}.zip" -O gost-engine.zip \
  && echo "$GOST_ENGINE_SHA256" gost-engine.zip | sha256sum -c - \
  && unzip gost-engine.zip -d ./ \
  && cd "engine-${GOST_ENGINE_VERSION}" \
  && sed -i 's|printf("GOST engine already loaded\\n");|goto end;|' gost_eng.c \
  && mkdir build \
  && cd build \
  && cmake -DCMAKE_BUILD_TYPE=Release \
   -DOPENSSL_ROOT_DIR="${OPENSSL_DIR}" \
   -DOPENSSL_INCLUDE_DIR="${OPENSSL_DIR}/include" \
   -DOPENSSL_LIBRARIES="${OPENSSL_DIR}/lib" .. \
   -DOPENSSL_ENGINES_DIR="${OPENSSL_DIR}/lib/engines-1.1" \
  && cmake --build . --config Release \
  && cmake --build . --target install --config Release \
  && cp ./bin/gost.so "${OPENSSL_DIR}/lib/engines-1.1" \
  && cp -r "${OPENSSL_DIR}/lib/engines-1.1" /usr/lib/x86_64-linux-gnu/ \
  && rm -rf "/usr/local/src/gost-engine.zip" "/usr/local/src/engine-${GOST_ENGINE_VERSION}" 

# Enable engine
ENV OPENSSL_CONF=/etc/ssl/openssl.cnf
RUN sed -i '6i openssl_conf=openssl_def' "${OPENSSL_CONF}" \
  && echo "" >> "${OPENSSL_CONF}" \
  && echo "# OpenSSL default section" >> "${OPENSSL_CONF}" \
  && echo "[openssl_def]" >> "${OPENSSL_CONF}" \
  && echo "engines = engine_section" >> "${OPENSSL_CONF}" \
  && echo "" >> "${OPENSSL_CONF}" \
  && echo "# Engine scetion" >> "${OPENSSL_CONF}" \
  && echo "[engine_section]" >> "${OPENSSL_CONF}" \
  && echo "gost = gost_section" >> "${OPENSSL_CONF}" \
  && echo "" >> "${OPENSSL_CONF}" \
  && echo "# Engine gost section" >> "${OPENSSL_CONF}" \
  && echo "[gost_section]" >> "${OPENSSL_CONF}" \
  && echo "engine_id = gost" >> "${OPENSSL_CONF}" \
  && echo "dynamic_path = ${OPENSSL_DIR}/lib/engines-1.1/gost.so" >> "${OPENSSL_CONF}f" \
  && echo "default_algorithms = ALL" >> "${OPENSSL_CONF}" \
  && echo "CRYPT_PARAMS = id-Gost28147-89-CryptoPro-A-ParamSet" >> "${OPENSSL_CONF}"

# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80

STOPSIGNAL SIGTERM

CMD ["nginx", "-g", "daemon off;"]
