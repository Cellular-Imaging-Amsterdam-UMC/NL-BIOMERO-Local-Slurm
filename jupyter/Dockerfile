# FROM rockylinux:8
FROM nvidia/cuda:11.7.0-devel-rockylinux8

# Image information
LABEL org.opencontainers.image.source="https://github.com/TorecLuik/slurm-docker-cluster" \
      org.opencontainers.image.title="slurm-docker-cluster" \
      org.opencontainers.image.description="Slurm Docker cluster on cuda" \
      org.label-schema.docker.cmd="docker-compose up -d" \
      maintainer="Torec Luik"

ARG SLURM_TAG=slurm-21-08-6-1
ARG GOSU_VERSION=1.11

# Install basic packages (e.g. Python, MUNGE)
RUN set -ex \
    && yum makecache \
    && yum -y update \
    && yum -y install dnf-plugins-core \
    && yum config-manager --set-enabled powertools \
    && yum -y install \
       wget \
       bzip2 \
       perl \
       gcc \
       gcc-c++\
       git \
       gnupg \
       make \
       munge \
       munge-devel \
    #    python3.9-dev \
       python3-pip \
       python3.9 \
       mariadb-server \
       mariadb-devel \
       psmisc \
       bash-completion \
       vim-enhanced \
    && yum clean all \
    && rm -rf /var/cache/yum

# Set python version
RUN alternatives --set python /usr/bin/python3.9

# Install 
RUN pip3 install Cython nose

# Install GOSU 
RUN set -ex \
    && wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-amd64" \
    && wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-amd64.asc" \
    && export GNUPGHOME="$(mktemp -d)" \
    && gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
    && gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
    && rm -rf "${GNUPGHOME}" /usr/local/bin/gosu.asc \
    && chmod +x /usr/local/bin/gosu \
    && gosu nobody true

## Install prerequisites for REST API
RUN yum -y install autoconf libtool cmake jansson
# http parser
RUN git clone --depth 1 --single-branch -b v2.9.4 https://github.com/nodejs/http-parser.git http_parser \
    && cd http_parser \
    && make \
    && make install
# yaml parser
RUN git clone --depth 1 --single-branch -b 0.2.5 https://github.com/yaml/libyaml libyaml \
    && cd libyaml \
    && ./bootstrap \
    && ./configure \
    && make \
    && make install
# json parser
RUN git clone --depth 1 --single-branch -b json-c-0.15-20200726 https://github.com/json-c/json-c.git json-c \
    && mkdir json-c-build \
    && cd json-c-build \
    && cmake ../json-c \
    && make \
    && make install \
    && export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig/:$PKG_CONFIG_PATH
# # JWT Auth library (w/ jansson)
# RUN git clone --depth 1 --single-branch -b 2.13 https://github.com/akheron/jansson jansson \
#     && cd jansson \
#     && autoreconf --force --install \
#     && ./configure --prefix=/usr/local/lib/pkgconfig/ \
#     && make \
#     && make install \
#     && export PKG_CONFIG_PATH=/usr/local/:/usr/lib/x86_64-linux-gnu/:$PKG_CONFIG_PATH

# RUN git clone --depth 1 --single-branch -b v1.12.0 https://github.com/benmcollins/libjwt.git libjwt \
#     && cd libjwt \
#     && autoreconf --force --install \
#     && ./configure --prefix=/usr/local \
#     && make -j \
#     && make install

## Perhaps even install slurm-web?
# http://edf-hpc.github.io/slurm-web/installation.html

## Install SLURM with plugins
RUN set -x \
    && git clone -b ${SLURM_TAG} --single-branch --depth=1 https://github.com/SchedMD/slurm.git \
    && pushd slurm \
    && ./configure --enable-debug --prefix=/usr --sysconfdir=/etc/slurm \
        --with-mysql_config=/usr/bin  --libdir=/usr/lib64 \
        --with-http-parser=/usr/local/ \
        --with-json=/usr/local/ \
        --with-yaml=/usr/local/ \
        #--with-jwt=/usr/local/ \
    && make install \
    && install -D -m644 etc/cgroup.conf.example /etc/slurm/cgroup.conf.example \
    && install -D -m644 etc/slurm.conf.example /etc/slurm/slurm.conf.example \
    && install -D -m644 etc/slurmdbd.conf.example /etc/slurm/slurmdbd.conf.example \
    && install -D -m644 contribs/slurm_completion_help/slurm_completion.sh /etc/profile.d/slurm_completion.sh \
    && popd \
    && rm -rf slurm \
    && groupadd -r --gid=990 slurm \
    && useradd -r -g slurm --uid=990 slurm \
    && mkdir /etc/sysconfig/slurm \
        /var/spool/slurmd \
        /var/run/slurmd \
        /var/run/slurmdbd \
        /var/lib/slurmd \
        /var/log/slurm \
        /data \
    && touch /var/lib/slurmd/node_state \
        /var/lib/slurmd/front_end_state \
        /var/lib/slurmd/job_state \
        /var/lib/slurmd/resv_state \
        /var/lib/slurmd/trigger_state \
        /var/lib/slurmd/assoc_mgr_state \
        /var/lib/slurmd/assoc_usage \
        /var/lib/slurmd/qos_usage \
        /var/lib/slurmd/fed_mgr_state \
    && chown -R slurm:slurm /var/*/slurm* \
    && /sbin/create-munge-key

# Install JupyterLab + Slurm ext.
RUN yum install sudo -y \
    # && yum install python3.9 python3-pip -y \
    && yum update -y \
    && yum install nodejs -y \
    && npm install -g configurable-http-proxy \
    && python -m pip install jupyterlab
WORKDIR /home/slurm
EXPOSE 8888

ENV USER slurm
ENV SHELL bash

RUN yum install openmpi-devel -y \
    # && export CC=/usr/lib64/openmpi/bin/mpicc \
    # && python -m pip install mpi4py \
    && python -m pip install jupyterlab_slurm


COPY slurm.conf /etc/slurm/slurm.conf
# COPY gres.conf /etc/slurm/gres.conf
# COPY cgroup.conf /etc/slurm/cgroup.conf
# COPY slurmdbd.conf /etc/slurm/slurmdbd.conf
# RUN set -x \
    # && chown slurm:slurm /etc/slurm/slurmdbd.conf \
    # && chmod 600 /etc/slurm/slurmdbd.conf

COPY docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

CMD ["slurmjupyter"]
