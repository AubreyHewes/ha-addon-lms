ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:9.3.0
FROM ${BUILD_FROM} AS base

ENV DEBIAN_FRONTEND="noninteractive"

RUN mv /init /s6init

# Upgrading the operating system inside the container
RUN apt-get update && apt-get dist-upgrade -y -o Dpkg::Options::="--force-confold" && apt-get --no-install-recommends --no-install-suggests -qq -y install wget git rsync patch && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

FROM base AS builder
ARG BUILD_ARCH

RUN apt-get -qq -y update && \
    apt-get --no-install-recommends --no-install-suggests -qq -y install \
    patch rsync nasm make gcc g++ libc-bin zlib1g-dev libgd-dev libmodule-install-perl bzip2 build-essential pkg-config file libjpeg-dev && \
    mkdir -p /tmp/sources && \
    cd /tmp/sources && \
    git clone --depth 1 https://github.com/LMS-Community/slimserver-vendor -b public/9.1 && \
    mkdir -p tmp && \
    cd slimserver-vendor/CPAN && \
    mkdir -p build/include build/lib && \
    if [ "${BUILD_ARCH}" = "i386" ];then \
    ln -s /usr/include/jpeglib.h build/include/jpeglib.h && \
    ln -s /usr/include/*/jconfig.h build/include/jconfig.h && \
    ln -s /usr/lib/*/libjpeg.a build/lib/libjpeg.a \
    ; fi && \
    \
    sed -i '/cd icu\/source/a wget -O config.guess "https://raw.githubusercontent.com/gcc-mirror/gcc/master/config.guess" && wget -O config.sub "https://raw.githubusercontent.com/gcc-mirror/gcc/master/config.sub"' buildme.sh && \
    cat ./buildme.sh && \
    \
    TMPDIR=/tmp/sources/tmp ./buildme.sh Media::Scan && \
    TMPDIR=/tmp/sources/tmp ./buildme.sh && \
    mkdir -p /usr/share/squeezeboxserver/CPAN/arch/$(basename build/arch/*)/ && \
    cp -vr build/arch/*/*/ /usr/share/squeezeboxserver/CPAN/arch/$(basename build/arch/*)/

#ln -s /usr/include/libavformat.h build/include/libavformat.h && \
FROM base

# Bump cpan build to same version if possible
#  ARG url="http://www.mysqueezebox.com/update/?version=8.0&revision=1&geturl=1&os=deb"
# xlatest_lms=$(wget -q -O - "${url}" | sed 's/_all\.deb/_'$(arch | sed 's/arm.*/arm/' | sed 's/x86_64/amd64/' | sed 's/aarch64/arm/')'\.deb/') && \
ARG purl="https://downloads.lms-community.org/LyrionMusicServer_v9.1.0/lyrionmusicserver_9.1.0_all.deb"
ARG debug=true
ARG target_uid=1000
ARG BUILD_ARCH

ENV S6_SERVICES_GRACETIME=119000
ENV DISABLE_HA_AUTHENTICATION=true

# LMS
ENV LMS_ACFG=/config/lms
ENV LMS_OCFG=/homeassistant/lms
ENV LMS_CFG=${LMS_ACFG}
ENV DEB_PKG_DIR=${LMS_CFG}/packages
ENV LMS_PREFS=${LMS_CFG}/prefs
ENV LMS_PLUGS=${LMS_CFG}/plugins
ENV LMS_CACHE=${LMS_CFG}/cache
ENV LMS_LOGDIR=${LMS_CFG}/logs
ENV LMS_HACFGDIR=${LMS_CFG}/addon
#ENV LMS_LOGFILE stdout
ENV LMS_OPTIONS=" "
ENV LMS_USER="squeezeboxserver"
ENV LMS_GROUP="nogroup"
ENV HA_USER="squeezeboxserver"
ENV debug=${debug}

## (moved) rootfs and nginx artifacts will be copied into the final stage after the nginx-builder

RUN apt-get -qq -y update && \
    apt-get -qq -y dist-upgrade && \
    ls -la /tmp/ && \
    apt-get -qq -y install --no-install-recommends --no-install-suggests \
        icedax \
        faad \
        flac \
        lame \
        sox \
        libio-socket-ssl-perl \
        nfs-common \
        awscli \
        lvm2 \
        exfat-fuse \
        exfatprogs \
        davfs2 \
        sshfs \
        cifs-utils \
        libssl-dev \
        mosquitto-clients \
        psmisc \
        tzdata \
        wget curl perl libwww-perl libfont-freetype-perl liblinux-inotify2-perl libdata-dump-perl libio-socket-ssl-perl libnet-ssleay-perl libcrypt-ssleay-perl libcrypt-openssl-rsa-perl && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Debug packages
RUN if [ "${debug}" = "true " ];then touch /debug && apt-get update && apt-get upgrade -y -o Dpkg::Options::="--force-confold" && apt-get --no-install-recommends --no-install-suggests -qq -y install vim-tiny procps && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*;fi

RUN echo "Download url: ${purl}" && \
	cd /tmp && \
        latest_lms=$(echo "${purl}" | sed 's/_all\.deb/_'$(echo ${BUILD_ARCH} | sed 's/arm.*/arm/' | sed 's/x86_64/amd64/' | sed 's/aarch64/arm/')'\.deb/') && \
	if [ $latest_lms = $latest_lms ];then  echo $latest_lms != $latest_lms ;sleep 600;fi && \
	wget -nv $latest_lms && \
	lms_deb=${latest_lms##*/} && \
	dpkg -i $lms_deb && \
        if [[ $(id -u squeezeboxserver) != ${target_uid} ]];then usermod -u $target_uid squeezeboxserver ;fi && \
	id squeezeboxserver && \
        rm -rf /var/lib/squeezeboxserver && \
	rm /usr/share/squeezeboxserver/Plugins && \
	ln -s ${LMS_PLUGS} /usr/share/squeezeboxserver/Plugins && \
	rm $lms_deb && \
	ls -la /usr/share/squeezeboxserver/CPAN/arch/* && \
	rm -rf /usr/share/squeezeboxserver/CPAN/arch/* && \
	touch /usr/share/squeezeboxserver/Bin/deleteme && \
	cd /usr/share/squeezeboxserver/Bin/ && rm -r $(ls -1 /usr/share/squeezeboxserver/Bin/| grep -v $(arch|sed 's/arm.*/arm/'))

COPY --from=builder /usr/share/squeezeboxserver/CPAN/arch/ /usr/share/squeezeboxserver/CPAN/arch/
RUN ls -la /usr/share/squeezeboxserver/CPAN/arch/*

#eg. http://downloads.slimdevices.com/LogitechMediaServer_v7.9.3/logitechmediaserver_7.9.3_arm.deb
# Copy script and run Script?
#RUN echo "${purl}" && \
#        latest_lms=$(echo $purl | sed 's/_all\.deb/_'$(arch | sed 's/arm.*/arm/')'\.deb/') && \
#	cd /sources/lms/packages && \
#	wget $latest_lms && \
#	lms_deb=${latest_lms##*/} && \
#	dpkg -i $lms_deb
#        latest_lms=$(wget -q -O - "${surl}") && \
#        echo $latest_lms && \
#	cd /sources && \
#        wget $latest_lms && \

# Nginx Ingress support
# Use the current Debian 13 nginx-full source release so the tarball is still available.
ARG NGINX_VERSION=1.26.3
ARG NGINX_PAGE_SIZE=16384
FROM base AS nginx-builder
ARG NGINX_VERSION=1.26.3
ARG NGINX_PAGE_SIZE=16384
RUN apt-get -qq -y update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        dpkg-dev \
        git \
        libexpat-dev \
        libgd-dev \
        libluajit-5.1-dev \
        luajit \
        libssl-dev \
        libxslt1-dev \
        libxml2-dev \
        zlib1g-dev \
        luarocks \
        wget

RUN set +x && cd /tmp && \
    wget -q https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz && \
    tar xzf nginx-${NGINX_VERSION}.tar.gz && \
    wget -q https://downloads.sourceforge.net/project/pcre/pcre/8.45/pcre-8.45.tar.gz && \
    tar xzf pcre-8.45.tar.gz && \
    cd pcre-8.45 && \
    ./configure --prefix=/usr/local --enable-jit --enable-utf --enable-unicode-properties --enable-shared && \
    make -j$(nproc) && make install && cd .. && \
    git clone --depth 1 https://github.com/simpl/ngx_devel_kit.git /tmp/ngx_devel_kit && \
    git clone --depth 1 https://github.com/openresty/lua-nginx-module.git /tmp/lua-nginx-module && \
    export LUAJIT_LIB=/usr/lib/$(dpkg-architecture -qDEB_HOST_MULTIARCH) && \
    export LUAJIT_INC=/usr/include/luajit-2.1 && \
    export CPPFLAGS="-I/usr/local/include -fPIC" && \
    export LDFLAGS="-L/usr/local/lib $(dpkg-buildflags --get LDFLAGS)" && \
    cd nginx-${NGINX_VERSION} && \
    ./configure \
        --sbin-path=/usr/sbin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --pid-path=/run/nginx.pid \
        --lock-path=/var/lock/nginx.lock \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --modules-path=/usr/lib/nginx/modules \
        --http-client-body-temp-path=/var/lib/nginx/body \
        --http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
        --http-proxy-temp-path=/var/lib/nginx/proxy \
        --http-scgi-temp-path=/var/lib/nginx/scgi \
        --http-uwsgi-temp-path=/var/lib/nginx/uwsgi \
        --with-compat \
        --with-cc-opt="$(dpkg-buildflags --get CFLAGS) -fPIC -DNGX_PAGESIZE=${NGINX_PAGE_SIZE}" \
        --with-ld-opt="-Wl,-E -lpcre $(dpkg-buildflags --get LDFLAGS)" \
        --with-debug \
        --with-pcre=../pcre-8.45 \
        --with-pcre-jit \
        --with-ipv6 \
        --with-http_ssl_module \
        --with-http_stub_status_module \
        --with-http_realip_module \
        --with-http_auth_request_module \
        --with-http_v2_module \
        --with-http_dav_module \
        --with-http_slice_module \
        --with-threads \
        --with-http_addition_module \
        --with-http_gunzip_module \
        --with-http_gzip_static_module \
        --with-http_image_filter_module=dynamic \
        --with-http_sub_module \
        --with-http_xslt_module=dynamic \
        --with-stream=dynamic \
        --with-stream_ssl_module \
        --with-mail=dynamic \
        --with-mail_ssl_module \
        --add-dynamic-module=/tmp/ngx_devel_kit \
        --add-dynamic-module=/tmp/lua-nginx-module && \
    make -j$(nproc) && \
    make install && \
    mkdir -p /nginx-built/modules && \
    cp -v /usr/lib/nginx/modules/*.so /nginx-built/modules/ 2>/dev/null || true && \
    mkdir -p /nginx-built/libs && \
    cp -v /usr/local/lib/libpcre* /nginx-built/libs/ 2>/dev/null || true && \
    rm -rf /tmp/nginx-${NGINX_VERSION} /tmp/pcre-8.45 /tmp/ngx_devel_kit /tmp/lua-nginx-module /tmp/nginx-${NGINX_VERSION}.tar.gz /tmp/pcre-8.45.tar.gz

FROM base

# Final stage: copy runtime files and nginx artifacts from builder
COPY rootfs/ /
COPY --from=nginx-builder /usr/sbin/nginx /usr/sbin/nginx
COPY --from=nginx-builder /nginx-built/modules/ /usr/lib/nginx/modules/
COPY --from=nginx-builder /nginx-built/libs/ /usr/local/lib/
RUN echo "/usr/local/lib" > /etc/ld.so.conf.d/nginx-pcre.conf && ldconfig || true
RUN ls -la /usr/lib/nginx/modules || true

RUN set +x && \
    apt-get -qq -y update && \
    apt-get install -y --no-install-recommends luarocks libluajit-5.1-2 && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
    luarocks install lua-resty-http 0.15-0

# Ensure nginx log dir exists so nginx can open error log
RUN mkdir -p /var/log/nginx || true

COPY rootfs-ingress/ /

# osync
RUN cd /tmp && git clone --depth 1 -b "stable" https://github.com/deajan/osync && cd osync && bash install.sh --no-stats ; cd /tmp && rm -r /tmp/osync

# hass-cli
#RUN apt-get -qq -y update && \
#    apt-get install -y --no-install-recommends python3-pip python3-setuptools python3-dev python3-wheel gcc && \
#    apt-get install -y --no-install-recommends python3-jsonpath-rw python3-regex python3-requests python3-aiohttp python3-tabulate python3-netdisco python3-jinja2 python3-dateparser python3-ruamel.yaml python3-click python3-click-log && \
#    pip3 install click==8.0.4 && \
#    pip3 install homeassistant-cli && \
#    apt-get autoremove && \
#    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
#pip3 show homeassistant-cli | grep Requires | sed 's/.*: /python3-/' | sed 's/, / python3-/g'
# apt-get remove python3-dev gcc \

# squeezy
RUN apt-get -qq -y update && \
    apt-get install -y --no-install-recommends libjson-rpc-perl && \
    cd /tmp && git clone --depth 1 -b "0.28" https://github.com/pssc/squeezy && cd squeezy && cp squeezy /usr/local/bin/ && cd /tmp && rm -r /tmp/squeezy && \
    apt-get autoremove && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Patching (run only if patch files are present)
RUN if ls /patch/*.patch 1> /dev/null 2>&1; then \
            for i in /patch/*.patch; do patch -d / -p1 -i "$i"; done; \
        fi && rm -rf /patch || true

#Cleanup
RUN if [ -z "${debug}" -o "${debug}" = "false" ];then apt-get -y remove \
        git \
        wget && \
	apt-get -y autoremove && \
        apt-get clean && \
	rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
        rm -fr /sources \
    ;fi

#VOLUME /data
#VOLUME /share
#VOLUME /backup
#VOLUME /config
#VOLUME /ssl
#VOLUME /media

# Build arguments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_REF
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="lms" \
    io.hass.description="LMS, Lyrion Music Server, Logitech Media Server, SqueezeBox Server / UPNP Media Server" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Phillip Camp <phillip.camp+ha<removeme>@gmail.com>" \
    org.label-schema.description="LMS,Lyrion Music Server, Logitech Media Server, SqueezeBox Server / UPNP Media Server" \
    org.label-schema.build-date=${BUILD_DATE} \
    org.label-schema.name="lms" \
    org.label-schema.schema-version="1.0" \
    org.label-schema.usage="https://github.com/pssc/ha-addon-lms/tree/master/README.md" \
    org.label-schema.vcs-ref=${BUILD_REF} \
    org.label-schema.vcs-url="https://github.com/pssc/ha-addon-lms" \
    org.label-schema.vendor="pssc"

#    org.label-schema.url="https://community.home-assistant.io/t/home-assistant-community-add-on-portainer/68836?u=frenck" \
