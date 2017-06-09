#FROM debian:jessie
FROM ubuntu:14.04

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
#RUN groupadd -r www-data && useradd -r --create-home -g www-data www-data

ENV HTTPD_PREFIX /usr/local/apache2
ENV PATH $PATH:$HTTPD_PREFIX/bin
RUN mkdir -p "$HTTPD_PREFIX" \
	&& chown www-data:www-data "$HTTPD_PREFIX"
WORKDIR $HTTPD_PREFIX

# install httpd runtime dependencies
# https://httpd.apache.org/docs/2.4/install.html#requirements
# ckretler: had to add ldap lib for switching to ubuntu 14.04.
RUN apt-get update \
	&& apt-get install -y --no-install-recommends \
		libapr1 \
		libaprutil1 \
		libapr1-dev \
		libaprutil1-dev \
		libpcre++0 \
		libssl1.0.0 \
		libldap-2.4-2 \
	&& rm -r /var/lib/apt/lists/*

# see https://httpd.apache.org/download.cgi#verify
RUN gpg --keyserver ha.pool.sks-keyservers.net --recv-keys A93D62ECC3C8EA12DB220EC934EA76E6791485A8

ENV HTTPD_VERSION 2.4.25
ENV HTTPD_BZ2_URL https://www.apache.org/dist/httpd/httpd-$HTTPD_VERSION.tar.bz2

###### BUILD APACHE #####
# install build dependencies, build, then purge the dependencies
RUN buildDeps=' \
		ca-certificates \
		curl \
		bzip2 \
		gcc \
		libpcre++-dev \
		libssl-dev \
		make \
		wget \
	' \
	set -x \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends $buildDeps \
	&& rm -r /var/lib/apt/lists/* \
	&& curl -SL "$HTTPD_BZ2_URL" -o httpd.tar.bz2 \
	&& curl -SL "$HTTPD_BZ2_URL.asc" -o httpd.tar.bz2.asc \
	&& gpg --verify httpd.tar.bz2.asc \
	&& mkdir -p src/httpd \
	&& tar -xvf httpd.tar.bz2 -C src/httpd --strip-components=1 \
	&& rm httpd.tar.bz2* \
	&& cd src/httpd \
	&& ./configure --enable-so --enable-ssl --prefix=$HTTPD_PREFIX --enable-mods-shared=most \
	&& make -j"$(nproc)" \
	&& make install


###### BUILD COSIGN #####
WORKDIR $HTTPD_PREFIX
ENV COSIGN_VERSION 3.2.0
ENV COSIGN_URL http://downloads.sourceforge.net/project/cosign/cosign/cosign-3.2.0/cosign-3.2.0.tar.gz
ENV CPPFLAGS="-I/usr/kerberos/include"

RUN wget "$COSIGN_URL" \
	&& mkdir -p src/cosign \
	&& tar -xvf cosign-3.2.0.tar.gz -C src/cosign --strip-components=1 \
	&& rm cosign-3.2.0.tar.gz \
	&& cd src/cosign \
	&& ./configure --enable-apache2=/usr/local/apache2/bin/apxs \
	&& sed -i 's/remote_ip/client_ip/g' ./filters/apache2/mod_cosign.c \
	&& make \
	&& make install \
	&& cd ../../ \
	&& rm -r src/cosign

##### BUILD mod_jk #######
#WORKDIR $HTTPD_PREFIX
# Not needed unless communicating with tomcat via AJP.
#RUN wget http://mirrors.koehn.com/apache/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.42-src.tar.gz \
#	&& mkdir -p src/mod_jk \
#	&& tar -xvf tomcat-connectors-1.2.42-src.tar.gz -C src/mod_jk --strip-components=1 \
#	&& rm tomcat-connectors-1.2.42-src.tar.gz \
#	&& cd src/mod_jk/native \
#	&& ./configure -with-apxs=/usr/local/apache2/bin/apxs \
#	&& make \
#	&& make install \
#	&& rm -r src/mod_jk \
#	&& apt-get purge -y --auto-remove $buildDeps
	
#COPY start.sh /usr/local/apache2/bin/

EXPOSE 443
EXPOSE 80
#CMD ./start.sh

RUN mkdir -p /var/cosign/filter

CMD rm /usr/local/apache2/conf/httpd.conf; ln -s /usr/local/apache2/local/conf/httpd.conf /usr/local/apache2/conf/httpd.conf; ln -s /usr/local/apache2/local/conf/httpd-cosign.conf /usr/local/apache2/conf/extra/httpd-cosign.conf; /usr/local/apache2/bin/httpd -DFOREGROUND
# CMD ifconfig > /tmp/ifconfig.txt; df -h > /tmp/df.txt; while [ "0" = "0" ]; do sleep 60; done 

