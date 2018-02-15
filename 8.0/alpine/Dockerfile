FROM alpine:3.7

RUN addgroup -S mysql && adduser -S -G mysql mysql

RUN apk add --no-cache \
# docker-entrypoint.sh
		bash pwgen su-exec tzdata \
# mysql_ssl_rsa_setup
		libressl

ENV PATH /opt/mysql/bin:$PATH

ENV MYSQL_MAJOR 8.0
ENV MYSQL_VERSION 8.0.4-rc

RUN set -eux; \
	\
	apk add --no-cache --virtual .fetch-deps \
		gnupg \
		ca-certificates \
	; \
	\
	wget -O mysql.tar.gz "https://cdn.mysql.com/Downloads/MySQL-${MYSQL_MAJOR}/mysql-${MYSQL_VERSION}.tar.gz"; \
	wget -O mysql.tar.gz.asc "https://cdn.mysql.com/Downloads/MySQL-${MYSQL_MAJOR}/mysql-${MYSQL_VERSION}.tar.gz.asc"; \
	\
	export GNUPGHOME="$(mktemp -d)"; \
# pub   dsa1024 2003-02-03 [SCA] [expires: 2019-02-17]
#       A4A9 4068 76FC BD3C 4567  70C8 8C71 8D3B 5072 E1F5
# uid           [ unknown] MySQL Release Engineering <mysql-build@oss.oracle.com>
	gpg --keyserver ha.pool.sks-keyservers.net --recv-keys A4A9406876FCBD3C456770C88C718D3B5072E1F5; \
	gpg --batch --verify mysql.tar.gz.asc mysql.tar.gz; \
	rm -r "$GNUPGHOME" mysql.tar.gz.asc; \
	\
	mkdir /mysql-src; \
	tar -xzf mysql.tar.gz -C /mysql-src --strip-components=1; \
	rm mysql.tar.gz; \
	\
# https://dev.mysql.com/doc/refman/8.0/en/source-installation.html
	apk add --no-cache --virtual .build-deps \
		bsd-compat-headers \
		cmake \
		g++ \
		gcc \
		libaio-dev \
		linux-headers \
		make \
		ncurses-dev \
		portablexdr-dev \
		rpcgen \
	; \
	apk del .fetch-deps; \
# https://dev.mysql.com/doc/refman/8.0/en/installing-source-distribution.html
	mkdir /mysql-src/bld; \
	cd /mysql-src/bld; \
	cmake .. \
# https://dev.mysql.com/doc/refman/8.0/en/source-configuration-options.html
# https://anonscm.debian.org/cgit/pkg-mysql/mysql.git/tree/debian/rules?h=mysql-8.0/debian/master&id=a464e2b137db93bd9e2cafd28ac3c4008c18c341
		-DBUILD_CONFIG=mysql_release \
		-DDOWNLOAD_BOOST=ON -DWITH_BOOST=boost \
		-DINSTALL_LAYOUT=DEB \
		-DCMAKE_INSTALL_PREFIX=/opt/mysql \
		-DSYSCONFDIR=/etc/mysql \
		-DMYSQL_DATADIR=/var/lib/mysql \
		-DINSTALL_PLUGINDIR=plugin \
		-DINSTALL_BINDIR=all-bin \
		-DINSTALL_SBINDIR=all-bin \
		-DINSTALL_STATIC_LIBRARIES=OFF \
		-DWITH_UNIT_TESTS=OFF \
		-DINSTALL_MYSQLTESTDIR= \
	; \
	make -j "$(nproc)"; \
	make install/strip; \
	\
	cd /; \
	rm -rf /mysql-src; \
	\
# copy over just the binaries we officially support (sourced from Debian's list):
# $ docker run --rm mysql:8 dpkg -L mysql-community-client-core mysql-community-server-core | grep -E '/s?bin/' | xargs -n1 basename | sort
	mkdir /opt/mysql/bin; \
	for bin in \
		ibd2sdi \
		innochecksum \
		lz4_decompress \
		myisamchk \
		myisamlog \
		myisampack \
		my_print_defaults \
		mysql \
		mysqlbinlog \
		mysqld \
		mysqld_multi \
		mysqld_safe \
		mysqldump \
		mysqlpump \
		mysql_secure_installation \
		mysql_ssl_rsa_setup \
		mysql_tzinfo_to_sql \
		mysql_upgrade \
		perror \
		resolveip \
		resolve_stack_dump \
		zlib_decompress \
	; do \
		mv "/opt/mysql/all-bin/$bin" /opt/mysql/bin; \
	done; \
	\
# remove a few bits that aren't necessary for our Docker image
	rm -rf \
		/opt/mysql/docs \
		/opt/mysql/all-bin \
		/opt/mysql/include \
		/opt/mysql/lib \
		/opt/mysql/man \
		/opt/mysql/support-files \
	; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /opt/mysql \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /opt/mysql/lib/" $1 " ]") == 0 { next } $1 { print "so:" $1 }' \
	)"; \
	apk add --no-cache --virtual .mysql-rundeps $runDeps; \
	apk del .build-deps; \
	mysql --version

RUN set -eux; \
	mkdir -p /var/lib/mysql /var/run/mysqld; \
	chown -R mysql:mysql /var/lib/mysql /var/run/mysqld
VOLUME /var/lib/mysql
# Config files
COPY config/ /etc/mysql/
COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 3306
CMD ["mysqld"]
