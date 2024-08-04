# syntax=docker/dockerfile:1

# renovate: datasource=docker depName=ubuntu
ARG UBUNTU_VERSION=22.04

FROM ubuntu:${UBUNTU_VERSION} AS blast

RUN apt-get update -y -m && apt-get install -y build-essential curl libidn12 libnet-perl perl-doc liblmdb-dev wget libsqlite3-dev cmake \
    libgomp1 libxml-simple-perl libjson-perl parallel vmtouch cpanminus python3 python3-openssl \
    && rm -fr /var/lib/apt/lists/*

# renovate: datasource=docker depName=ycya88/ncbi_edirect
ARG EDIRECT_VERSION=22.4.20240802-alpine3.20-root

FROM ycya88/ncbi_edirect:${EDIRECT_VERSION} as edirect

# renovate: datasource=docker depName=alpine
ARG ALPINE_VERSION=3.20.2

FROM alpine:${ALPINE_VERSION} AS base

LABEL \
    org.opencontainers.image.description="NCBI BLAST+ tool" \
    org.opencontainers.image.authors="ycya88@gmail.com" \
    org.opencontainers.image.url="https://hub.docker.com/r/ycya88/ncbi_edirect" \
    org.opencontainers.image.vendor="NCBI/NLM/NIH" \
    org.opencontainers.image.source="https://github.com/ycya88/ncbi_edirect" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.os="alpine"

RUN echo "http://dl-cdn.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories

RUN apk add --no-cache build-base curl libidn perl perl-doc perl-app-cpanminus perl-dev perl-xml-simple perl-libwww perl-net-mpd \
    lmdb-dev wget sqlite-dev cmake gmp libxml2 json-c parallel vmtouch py3-pip py3-openssl coreutils bash

RUN cpanm --notest HTML::Entities

RUN rm -fr /root/.cpanm

FROM blast

# renovate: datasource=custom.ncbi depName=blast
ARG BLAST_VERSION=2.16.0
ARG NUM_PROCS=2

RUN mkdir -p /blast/bin /blast/lib

RUN curl -sS -H 'Cache-Control: no-cache, no-store' https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/${BLAST_VERSION}/ncbi-blast-${BLAST_VERSION}+-src.tar.gz | \
    tar xzf - \
    && cd ncbi-blast-${BLAST_VERSION}+-src/c++ \
    && ./configure --with-mt --with-strip --with-optimization --with-dll --with-experimental=Int8GI --with-flat-makefile --with-openmp --with-sqlite3 --with-vdb=${VDB} --without-gnutls --without-gcrypt --without-zstd --without-lzo --prefix=/blast \
    && cd ReleaseMT/build \
    && timeout 600 make -j ${NUM_PROCS} -f Makefile.flat blastdb_aliastool.exe blastdbcheck.exe blastdbcmd.exe blast_formatter.exe blastn.exe blastp.exe blastx.exe convert2blastmask.exe deltablast.exe dustmasker.exe makeblastdb.exe makembindex.exe makeprofiledb.exe psiblast.exe rpsblast.exe rpstblastn.exe segmasker.exe tblastn.exe tblastx.exe windowmasker.exe blastn_vdb.exe tblastn_vdb.exe blast_formatter_vdb.exe blast_vdb_cmd.exe blastdb_path.exe \
    && find /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/bin/*blast* /blast/bin/ \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/bin/*masker /blast/bin/ \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/bin/makembindex /blast/bin/ \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/bin/makeprofiledb /blast/bin/ \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/lib/* /blast/lib/ \
    && rm -fr /root/ncbi-blast-${BLAST_VERSION}+-src /root/ncbi-outdir

FROM base AS build

ARG VDB_VERSION=3.1.1
ARG CLOUD_SDK_VERSION=472.0.0

WORKDIR /root

RUN wget -q https://github.com/ncbi/ncbi-vdb/archive/refs/tags/${VDB_VERSION}.tar.gz \
    && tar xfz ${VDB_VERSION}.tar.gz \
    && rm ${VDB_VERSION}.tar.gz \
    && mv ncbi-vdb-${VDB_VERSION} ncbi-vdb \
    && cd ncbi-vdb && ./configure --prefix /root/vdb \
    && make install -j ${num_proc} \
    && rm -fr ncbi-outdir

ARG VDB=/root/VDB
RUN mkdir -p ${VDB}/linux/release/x86_64/lib \
    && cp -R /root/vdb/lib64/* ${VDB}/linux/release/x86_64/lib \
    && cp -R /root/vdb/lib64/* /blast/lib/ \
    && mkdir -p ${VDB}/interfaces \
    && cp -R /root/vdb/include/* ${VDB}/interfaces

RUN mkdir -p /blast/bin /blast/lib

## COPY BLAST

RUN mkdir -p /blast/blastdb /blast/blastdb_custom
RUN sed -i '$ a BLASTDB=/blast/blastdb:/blast/blastdb_custom' /etc/environment
ENV BLASTDB /blast/blastdb:/blast/blastdb_custom
ENV BLAST_DOCKER true
ENV PATH="/root/edirect:/blast/bin:${PATH}"

RUN cp /blast/bin/blastp /blast/bin/blastp.REAL \
    && cp /blast/bin/blastn /blast/bin/blastn.REAL \
    && cp /blast/bin/blastx /blast/bin/blastx.REAL \
    && cp /blast/bin/tblastn /blast/bin/tblastn.REAL \
    && cp /blast/bin/tblastx /blast/bin/tblastx.REAL

COPY blastn.sh /blast/bin/blastn
COPY blastp.sh /blast/bin/blastp
COPY blastx.sh /blast/bin/blastx
COPY tblastn.sh /blast/bin/tblastn
COPY tblastx.sh /blast/bin/tblastx

RUN chmod +x /blast/bin/blastn \
    && chmod +x /blast/bin/blastp \
    && chmod +x /blast/bin/blastx \
    && chmod +x /blast/bin/tblastn \
    && chmod +x /blast/bin/tblastx

ENV CLOUD_SDK_VERSION=$CLOUD_SDK_VERSION
ENV PATH /root/google-cloud-sdk/bin:$PATH
ENV CLOUDSDK_PYTHON=/usr/bin/python3

RUN ARCH=$(uname -m) && \
    if [ "$ARCH" = "aarch64" ]; then \
        export ARCH=arm; \
    else \
        export ARCH=x86_64; \
    fi \
    && echo $ARCH > /tmp/ARCH

RUN export ARCH=$(cat /tmp/ARCH) \
    && curl -s -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-${CLOUD_SDK_VERSION}-linux-${ARCH}.tar.gz \
    && tar xzf google-cloud-cli-${CLOUD_SDK_VERSION}-linux-${ARCH}.tar.gz \
    && rm google-cloud-cli-${CLOUD_SDK_VERSION}-linux-${ARCH}.tar.gz \
    && rm -rf google-cloud-sdk/platform/bundledpythonunix \
    && gcloud components remove -q bq \
    && rm -fr $(find google-cloud-sdk/ -regex ".*/__pycache__") \
    && rm -rf google-cloud-sdk/.install/.backup \
    && rm -fr google-cloud-sdk/bin/anthoscli \
    && rm -fr google-cloud-sdk/lib/third_party/{botocore,kubernetes} \
    && rm -fr google-cloud-sdk/lib/googlecloudsdk/generated_clients/apis/compute/{alpha,beta} \
    && rm -fr google-cloud-sdk/lib/googlecloudsdk/generated_clients/apis/{healthcare,gkehub,cloudbuild,containeranalysis,dialogflow,aiplatform} \
    && rm -fr google-cloud-sdk/lib/surface \
    && rm -fr $(find google-cloud-sdk/ -name doc -o -name docs) \
    && gsutil --version \
    && gsutil ls gs://blast-db

ENV PATH /root/google-cloud-sdk/bin:$PATH
ENV CLOUDSDK_PYTHON=/usr/bin/python3

COPY --from=edirect /root/edirect /root/edirect
COPY --from=edirect /root/installconfirm /root/edirect
ENV PATH="/root/edirect:${PATH}"

WORKDIR /blast

USER root
