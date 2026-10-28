# Copyright 2020-2020 Amazon.com, Inc. or its affiliates. All Rights Reserved.
#
# Licensed under the Amazon Software License (the "License"). You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/asl/
#
# or in the "license" file accompanying this file.
# This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied.
# See the License for the specific language governing permissions and limitations under the License.

FROM public.ecr.aws/amazonlinux/amazonlinux:2023 AS core

RUN set -ex \
    && yum install -yq openssh-clients \
    && mkdir ~/.ssh \
    && touch ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H github.com >> ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H bitbucket.org >> ~/.ssh/known_hosts \
    && chmod 600 ~/.ssh/known_hosts \
    && yum groupinstall -yq "Development tools" \
    && yum install -yq --allowerasing \
           bzip2 bzip2-devel automake \
           e2fsprogs expat-devel expect fakeroot \
           glib2-devel groff gzip icu jq libargon2-devel \
           libcurl-devel libdb-devel libedit-devel libevent-devel libffi-devel \
           libicu-devel libjpeg-devel libpng-devel zlib-devel \
           libtidy libtidy-devel libunwind libwebp-devel libxml2-devel libxslt libxslt-devel \
           libyaml-devel libzip-devel \
           ncurses-devel oniguruma-devel openssl openssl-devel \
           perl perl-FindBin \
           procps-ng python-configobj readline-devel rsync \
           patch pkg-config procps python3-configobj rsync sqlite-devel \
           tar time tzdata vim wget which xfsprogs xz-devel \
           git git-lfs acl unzip \
           automake binutils bison brotli file findutils flex gnupg2 libtool \
           lz4 m4 make net-tools pigz pkg-config xz zip

RUN useradd codebuild-user

#=======================End of layer: core  =================

FROM core AS tools

# Configure SSM
RUN set -ex \
    && yum install -yq https://s3.amazonaws.com/amazon-ssm-us-east-1/latest/linux_arm64/amazon-ssm-agent.rpm

# Install env tools for runtimes

##nodejs
ARG SRC_DIR="/usr/src"
ENV N_SRC_DIR="$SRC_DIR/n"
RUN git clone https://github.com/tj/n $N_SRC_DIR \
    && cd $N_SRC_DIR && make install

##php
RUN curl -L https://raw.githubusercontent.com/phpenv/phpenv-installer/master/bin/phpenv-installer | bash
ENV PATH="/root/.phpenv/shims:/root/.phpenv/bin:$PATH"

##awscliv2
RUN curl https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip -o /tmp/awscliv2.zip \
     && unzip /tmp/awscliv2.zip -d /opt \
     && /opt/aws/install --update -i /usr/local/aws-cli -b /usr/local/bin \
     && rm /tmp/awscliv2.zip \
     && rm -rf /opt/aws \
     && aws --version
#=======================End of layer: tools  =================

FROM tools AS runtimes_1

#****************      NODEJS     ****************************************************

ENV NODE_18_VERSION="18.20.6"

RUN n --no-preserve $NODE_18_VERSION \
    && npm install --save-dev -g -f yarn \
    && dnf install -y -v libuv-1.44* \
    && cd / && rm -rf $N_SRC_DIR \
    && rm -rf /tmp/* && rm -rf ~/.npm/_logs/
#****************      END NODEJS     ****************************************************

#****************      PHP     ****************************************************

# Installed in packages
ENV PHP_85_VERSION="8.5.12"

# Set environment variables for PHP configure options
ENV PHP_BUILD_CONFIGURE_OPTS="--with-curl --with-password-argon2 --with-libedit"
# Set make arguments to use 4 parallel jobs.
ENV PHP_BUILD_EXTRA_MAKE_ARGUMENTS="-j4"

RUN phpenv update \
    && PHP_BUILD_EXTRA_MAKE_ARGUMENTS="-j$(nproc)" phpenv install $PHP_85_VERSION \
    && phpenv global $PHP_85_VERSION \
    && php -v \
    && echo "memory_limit = -1;" >> "/root/.phpenv/versions/$PHP_85_VERSION/etc/conf.d/memory.ini" \
    && curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer \
    && rm -rf /tmp/*

#****************      END PHP     ****************************************************


#****************      INSTALL FROM BUILDSPEC.YML     ****************************************************
#  in order to avoid running these commands during built time.
# Built from the source tarballs instead of pecl: the PEAR client in the OS
# repos can't read the chunked responses pecl.php.net now sends.
RUN set -ex \
    && for ext in mongodb-2.5.5 redis-6.3.0; do \
         curl -fsSL "https://pecl.php.net/get/${ext}.tgz" -o /tmp/${ext}.tgz \
         && mkdir /tmp/${ext} && tar xzf /tmp/${ext}.tgz -C /tmp/${ext} --strip-components=1 \
         && cd /tmp/${ext} \
         && /root/.phpenv/versions/$PHP_85_VERSION/bin/phpize \
         && ./configure --with-php-config=/root/.phpenv/versions/$PHP_85_VERSION/bin/php-config \
         && make -j"$(nproc)" && make install && cd / || exit 1; \
       done \
    && echo "extension=mongodb.so" >> "/root/.phpenv/versions/$PHP_85_VERSION/etc/php.ini" \
    && echo "extension=redis.so" >> "/root/.phpenv/versions/$PHP_85_VERSION/etc/php.ini" \
    && rm -rf /tmp/*
#****************      END INSTALL FROM BUILDSPEC.YML     ****************************************************

#=======================End of layer: runtimes_1  =================
FROM runtimes_1 AS runtimes_3

#===================END of runtimes_3 ==============
FROM runtimes_3 AS aarch64_v3

# Activate runtime versions specific to image version.
RUN n --preserve $NODE_18_VERSION
RUN phpenv global $PHP_85_VERSION

# Configure SSH
COPY ssh_config /root/.ssh/config
COPY runtimes.yml /codebuild/image/config/runtimes.yml
COPY legal/THIRD_PARTY_LICENSES.txt /usr/share/doc
COPY legal/bill_of_material.txt /usr/share/doc
COPY amazon-ssm-agent.json /etc/amazon/ssm/

#=======================End of layer: aarch64_v3  =================
