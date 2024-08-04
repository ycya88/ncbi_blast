# syntax=docker/dockerfile:1

# renovate: datasource=docker depName=ycya88/ncbi_edirect
ARG EDIRECT_VERSION=22.4.20240802-ubuntu3.20-root

# renovate: datasource=docker depName=ubuntu
ARG UBUNTU_VERSION=3.20.2

FROM ycya88/ncbi_edirect:${EDIRECT_VERSION} as edirect

FROM ubuntu:${UBUNTU_VERSION} AS base

LABEL \
    org.opencontainers.image.description="NCBI BLAST+ tool" \
    org.opencontainers.image.authors="ycya88@gmail.com" \
    org.opencontainers.image.url="https://hub.docker.com/r/ycya88/ncbi_edirect" \
    org.opencontainers.image.vendor="NCBI/NLM/NIH" \
    org.opencontainers.image.source="https://github.com/ycya88/ncbi_edirect" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.os="ubuntu"

RUN apt-get -y -m update && apt-get install -y build-essential curl libidn12 libnet-perl perl-doc liblmdb-dev wget libsqlite3-dev cmake \
    libgomp1 libxml-simple-perl libjson-perl parallel vmtouch cpanminus python3 python3-openssl \
    && rm -fr /var/lib/apt/lists/* \
    && cpanm HTML::Entities && rm -fr .cpanm

FROM base AS build

# renovate: datasource=docker depName=ycya88/ncbi_blast
ARG BLAST_VERSION=2.16.0

ARG VDB_VERSION=3.1.1
ARG CLOUD_SDK_VERSION=472.0.0

WORKDIR /root

RUN mkdir -p /blast/bin /blast/lib

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

## COPY BLAST

RUN mkdir -p /blast/blastdb /blast/blastdb_custom
RUN sed -i '$ a BLASTDB=/blast/blastdb:/blast/blastdb_custom' /etc/environment
ENV BLASTDB /blast/blastdb:/blast/blastdb_custom
ENV BLAST_DOCKER true
ENV PATH="/root/edirect:/blast/bin:${PATH}"

# RUN cp /blast/bin/blastp /blast/bin/blastp.REAL \
#     && cp /blast/bin/blastn /blast/bin/blastn.REAL \
#     && cp /blast/bin/blastx /blast/bin/blastx.REAL \
#     && cp /blast/bin/tblastn /blast/bin/tblastn.REAL \
#     && cp /blast/bin/tblastx /blast/bin/tblastx.REAL

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
