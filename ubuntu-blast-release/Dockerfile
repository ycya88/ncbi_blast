# syntax=docker/dockerfile:1

# renovate: datasource=docker depName=ubuntu
ARG UBUNTU_VERSION=22.04

FROM ubuntu:${UBUNTU_VERSION} AS blast

RUN apt-get update -y -m && apt-get install -y build-essential curl libidn12 libnet-perl perl-doc liblmdb-dev wget libsqlite3-dev cmake \
    libgomp1 libxml-simple-perl libjson-perl parallel vmtouch cpanminus python3 python3-openssl \
    && rm -fr /var/lib/apt/lists/*

FROM blast

# renovate: datasource=custom.ncbi depName=blast
ARG BLAST_VERSION=2.16.0

RUN mkdir -p /blast/bin /blast/lib

RUN curl -sS -H 'Cache-Control: no-cache, no-store' https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/${BLAST_VERSION}/ncbi-blast-${BLAST_VERSION}+-src.tar.gz | \
    tar xzf - \
    && cd ncbi-blast-${BLAST_VERSION}+-src/c++ \
    && ./configure --with-mt --with-strip --with-optimization --with-dll --with-experimental=Int8GI --with-flat-makefile --with-openmp --with-sqlite3 --with-vdb=${VDB} --without-gnutls --without-gcrypt --without-zstd --without-lzo --prefix=/blast \
    && cd ReleaseMT/build \
    && make -j ${NUM_PROCS} -f Makefile.flat blastdb_aliastool.exe blastdbcheck.exe blastdbcmd.exe blast_formatter.exe blastn.exe blastp.exe blastx.exe convert2blastmask.exe deltablast.exe dustmasker.exe makeblastdb.exe makembindex.exe makeprofiledb.exe psiblast.exe rpsblast.exe rpstblastn.exe segmasker.exe tblastn.exe tblastx.exe windowmasker.exe blastn_vdb.exe tblastn_vdb.exe blast_formatter_vdb.exe blast_vdb_cmd.exe blastdb_path.exe \
    && find /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/bin/*blast* /blast/bin/ \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/bin/*masker /blast/bin/ \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/bin/makembindex /blast/bin/ \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/bin/makeprofiledb /blast/bin/ \
    && cp -pv /root/ncbi-blast-${BLAST_VERSION}+-src/c++/ReleaseMT/lib/* /blast/lib/ \
    rm -fr /root/ncbi-blast-${BLAST_VERSION}+-src /root/ncbi-outdir

# FROM scratch AS final
