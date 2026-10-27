# Copyright 2020-2022 Amazon.com, Inc. or its affiliates. All Rights Reserved.
#
# Licensed under the Amazon Software License (the "License"). You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/asl/
#
# or in the "license" file accompanying this file.
# This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied.
# See the License for the specific language governing permissions and limitations under the License.
FROM public.ecr.aws/ubuntu/ubuntu:22.04 AS core

ARG DEBIAN_FRONTEND="noninteractive"

# Install SSH, and other utilities
RUN set -ex \
    && echo 'Acquire::CompressionTypes::Order:: "gz";' > /etc/apt/apt.conf.d/99use-gzip-compression \
    && apt-get update \
    && apt install -y -qq apt-transport-https gnupg ca-certificates sudo \
    && apt-get install software-properties-common -y -qq --no-install-recommends \
    && apt-get install -y -qq --no-install-recommends openssh-client \
    && mkdir ~/.ssh \
    && mkdir -p /codebuild/image/config \
    && touch ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H github.com >> ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H bitbucket.org >> ~/.ssh/known_hosts \
    && chmod 600 ~/.ssh/known_hosts \
    && apt-get install -y -qq --no-install-recommends \
        acl apt-utils aria2 asciidoc autoconf automake \
        binutils bison brotli build-essential bzip2 bzr \
        coreutils curl dirmngr dnsutils docbook-xml docbook-xsl dpkg dpkg-dev \
        e2fsprogs expect fakeroot file findutils flex fonts-noto-color-emoji ftp \
        g++ gcc git git-lfs gettext gettext-base gnupg2 groff gzip \
        haveged imagemagick iproute2 jq less \
        lib32z1 libapr1 libaprutil1 libargon2-0-dev libbz2-dev \
        libc++-dev libc++abi-dev libc6-dev libcurl4-openssl-dev \
        libdb-dev libdbd-sqlite3-perl libdbi-perl libdpkg-perl \
        libedit-dev liberror-perl libevent-dev libffi-dev \
        libgeoip-dev libgbm-dev libgconf-2-4 libglib2.0-dev libgsl-dev libgtk-3-0 \
        libhttp-date-perl libio-pty-perl libjpeg-dev libkrb5-dev liblzma-dev \
        libmagic-dev libmagickcore-dev libmagickwand-dev \
        libncurses5-dev libncursesw5-dev libonig-dev libreadline-dev \
        libsecret-1-dev libserf-1-1 libsodium-dev libsqlite3-dev libssl-dev \
        libsvn1 libsvn-perl libtcl8.6 libtidy-dev libtimedate-perl libtool libunwind8 \
        libwebp-dev libxkbfile-dev libxml2-dev libxml2-utils libxslt1-dev libxss1 \
        libyaml-dev libyaml-perl libzip-dev llvm locales lz4 \
        m4 make mediainfo mercurial mlocate net-tools netbase netcat \
        openssl patch p7zip-full p7zip-rar parallel pass patchelf pigz pkg-config pollinate procps \
        python-is-python3 python3-configobj python3-openssl rpm rsync \
        sgml-base sgml-data shellcheck sphinxsearch sqlite3 ssh sshpass subversion sudo swig systemd-coredump \
        tar tcl tcl8.6 telnet texinfo time tk tk-dev tzdata \
        unzip upx vim wget xfsprogs xml-core xmlto xorriso xsltproc \
        xvfb xz-utils zip zlib1g-dev zsync \
    && rm -rf /var/lib/apt/lists/*

RUN apt-get update && \
    apt-get install -y unattended-upgrades && \
    unattended-upgrade -v --dry-run && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

ENV LC_CTYPE="C.UTF-8"

# Set locale
RUN locale-gen en_US.UTF-8 \
    && update-locale LANG=en_US.UTF-8

RUN useradd codebuild-user

#=======================End of layer: core  =================

FROM core AS tools

# Install AWS CLI v2
# https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
RUN curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip \
    && unzip -q /tmp/awscliv2.zip -d /opt/awscli \
    && /opt/awscli/aws/install --update -i /usr/local/aws-cli -b /usr/local/bin \
    && rm -rf /opt/awscli /tmp/* \
    && aws --version

# Configure SSM
RUN set -ex \
    && mkdir /tmp/ssm \
    && cd /tmp/ssm \
    && wget -q https://s3.amazonaws.com/amazon-ssm-us-east-1/latest/debian_amd64/amazon-ssm-agent.deb \
    && dpkg -i amazon-ssm-agent.deb

#=======================End of stage: tools  =================
FROM tools AS runtimes


#****************      PHP     ****************************************************

#phpenv
RUN curl -L https://raw.githubusercontent.com/phpenv/phpenv-installer/master/bin/phpenv-installer | bash
ENV PATH="/root/.phpenv/shims:/root/.phpenv/bin:$PATH"

ENV PHP_83_VERSION="8.3.36"
# Set environment variables for PHP configure options
ENV PHP_BUILD_CONFIGURE_OPTS="--with-curl --with-password-argon2 --with-libedit"

RUN phpenv update \
    && PHP_BUILD_EXTRA_MAKE_ARGUMENTS="-j$(nproc)" phpenv install $PHP_83_VERSION \
    && phpenv global $PHP_83_VERSION \
    && php -v \
    && echo "memory_limit = 1G;" >> "/root/.phpenv/versions/$PHP_83_VERSION/etc/conf.d/memory.ini" \
    && curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer \
    && rm -rf /tmp/*
#****************      END PHP     ****************************************************

#****************      INSTALL FROM BUILDSPEC.YML     ****************************************************
#  in order to avoid running these commands during built time.
RUN apt-get install -y php-pear php-dev
RUN pecl install mongodb-1.21.0
RUN echo "extension=mongodb.so" >> "/root/.phpenv/versions/$PHP_83_VERSION/etc/php.ini"
RUN pecl install redis
RUN echo "extension=redis.so" >> "/root/.phpenv/versions/$PHP_83_VERSION/etc/php.ini"

#****************      END INSTALL FROM BUILDSPEC.YML     ****************************************************

#=======================End of stage: runtimes  =================

FROM runtimes AS std_v7

# Activate runtime versions specific to image version.
RUN phpenv global $PHP_83_VERSION

# Configure SSH
COPY ssh_config /root/.ssh/config
COPY runtimes.yml /codebuild/image/config/runtimes.yml
COPY legal/bill_of_material.txt /usr/share/doc/bill_of_material.txt
COPY amazon-ssm-agent.json /etc/amazon/ssm/amazon-ssm-agent.json

#=======================END of STD:7.0  =================
