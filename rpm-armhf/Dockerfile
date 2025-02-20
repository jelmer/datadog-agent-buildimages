ARG BASE_IMAGE=arm32v7/centos:7

FROM ubuntu as CERT_GETTER
ENV CACERT_BUNDLE_VERSION=2022-04-26
ENV CACERT_BUNDLE_SHA256="08df40e8f528ed283b0e480ba4bcdbfdd2fdcf695a7ada1668243072d80f8b6f"
RUN apt-get update && apt-get install -y wget
RUN wget https://curl.se/ca/cacert-${CACERT_BUNDLE_VERSION}.pem -O /cacert.pem
RUN echo "${CACERT_BUNDLE_SHA256}  /cacert.pem" | sha256sum --check

FROM ${BASE_IMAGE}

# Build Args
ARG GIMME_GO_VERSION=1.17.6
ARG DD_TARGET_ARCH=armhf

# Environment
ENV GOPATH /go
ENV GIMME_GO_VERSION $GIMME_GO_VERSION
ENV DD_TARGET_ARCH $DD_TARGET_ARCH

# configure yum and rpm for running on non-armv7l architectures
RUN echo "armhfp" > /etc/yum/vars/basearch && \
    echo "armv7hl" > /etc/yum/vars/arch && \
    echo "armv7hl-redhat-linux-gnu" > /etc/rpm/platform && \
    mkdir -p /usr/local/etc/rpm && \
    echo "armv7hl-redhat-linux-gnu" > /usr/local/etc/rpm/platform

# Resolves a nasty NOKEY warning that appears when using yum
RUN rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7 && \
    rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-AltArch-Arm32

# The last two lines contain dependencies for build of newer rpm
RUN yum -y install @development which perl-ExtUtils-MakeMaker ncurses-compat-libs git procps \
    curl-devel expat-devel gettext-devel openssl-devel systemd-devel zlib-devel bzip2 glibc-static tar pkgconfig  \
    libtool autoconf policycoreutils-python \
    bzip2-devel e2fsprogs-devel file-devel libacl-devel libarchive-devel libattr-devel \
    libxml2-devel lzo-devel nss nss-devel popt-devel sharutils xz-devel

COPY --from=CERT_GETTER /cacert.pem /etc/pki/tls/certs/ca-bundle.crt

# Build new rpm
COPY patches/rpm-4.15.1-fix-rpmbuild-segfault.patch /tmp
# Cannot use HTTPS here: cert name is invalid
RUN curl -sL -o /tmp/rpm-4.15.1.tar.bz2 http://ftp.rpm.org/releases/rpm-4.15.x/rpm-4.15.1.tar.bz2 \
    && echo "ddef45f9601cd12042edfc9b6e37efcca32814e1e0f4bb8682d08144a3e2d230  /tmp/rpm-4.15.1.tar.bz2" | sha256sum --check \
    && cd /tmp \
    && tar -xjf /tmp/rpm-4.15.1.tar.bz2 \
    && cd rpm-4.15.1 \
    && cat /tmp/rpm-4.15.1-fix-rpmbuild-segfault.patch | patch -p1 \
    && ./configure --without-lua --without-audit \
    && make \
    && make install \
    && cd / \
    && rm -rf /tmp/rpm-4.15.1-fix-rpmbuild-segfault.patch /tmp/rpm-4.15.1.tar.bz2 /tmp/rpm-4.15.1

# Rebuild RPM database with the new rpm
RUN mkdir -p /usr/local/var/lib/rpm \
    && cp /var/lib/rpm/Packages /usr/local/var/lib/rpm/Packages \
    && /usr/local/bin/rpm --rebuilddb

# This trick preloads a dynamic library that will replace the uname syscall to pretend the machine is armv7l
# The fakearmv7l library gets installed in "/usr/local/lib/libfakearmv7l.so"
RUN git clone --depth 1 https://github.com/DataDog/fakearmv7l ; \
        cd fakearmv7l ; \
        make ; \
        make install ; \
        cd .. ; \
        rm -rf ./fakearmv7l ; \
        export LD_PRELOAD="/usr/local/lib/libfakearmv7l.so"

# RVM
COPY ./rvm/gpg-keys /gpg-keys
RUN gpg --import /gpg-keys/*
RUN rm -rf /gpg-keys
RUN curl -sSL -o get-rvm.sh https://raw.githubusercontent.com/rvm/rvm/1.29.12/binscripts/rvm-installer \
    && echo "fea24461e98d41528d6e28684aa4c216dbe903869bc3fcdb3493b6518fae2e7e  get-rvm.sh" | sha256sum --check \
    && bash get-rvm.sh stable --version 1.29.12 \
    && echo "d2de0b610ee321489e5c673fe749e13be8fb34c0aa08a74446d87f95a17de730  /usr/local/rvm/bin/rvm" | sha256sum --check \
    && rm get-rvm.sh
RUN /bin/bash -l -c "rvm requirements"
RUN /bin/bash -l -c "rvm install --with-arch='armv7-a' -C '--build' -C 'arm-linux-gnueabihf' 2.7 && rvm cleanup all"
RUN /bin/bash -l -c "gem install bundler --no-document"

# Pip & Invoke
COPY ./python-packages-versions.txt /python-packages-versions.txt
COPY ./setup_python.sh /setup_python.sh
COPY ./requirements.txt ./requirements-py2.txt /
RUN ./setup_python.sh

# Gimme
RUN curl -sL -o /bin/gimme https://raw.githubusercontent.com/travis-ci/gimme/v1.5.4/gimme
RUN echo "03b295636d4e22870b6f6e9bc06a71d65311ae90d3d48cbc7071f82dd5837fbc  /bin/gimme" | sha256sum --check
RUN chmod +x /bin/gimme

# GIMME_ARCH = GOARCH, so must be a valid entry from `goarchlist` here:
# https://github.com/golang/go/blob/master/src/go/build/syslist.go
# Also see https://github.com/travis-ci/gimme/blob/master/gimme#L880
RUN GIMME_ARCH=arm gimme $GIMME_GO_VERSION

COPY ./gobin.sh /etc/profile.d/

# Entrypoint
COPY ./entrypoint.sh /
RUN chmod +x /entrypoint.sh

# create the agent build folder within $GOPATH
RUN mkdir -p /go/src/github.com/DataDog/datadog-agent

# Force umask to 0022
RUN echo "umask 0022" >> /root/.bashrc

ENTRYPOINT ["/entrypoint.sh"]
