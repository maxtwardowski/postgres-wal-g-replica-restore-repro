ARG PG_VERSION=14.5
ARG GO_VERSION=1.19.2
ARG ALPINE_VERSION=3.16
ARG WALG_VERSION=e7ffb57e4c5ba385ed596e45295ad8b60eb1ae76

# build wal-g
FROM golang:${GO_VERSION}-alpine${ALPINE_VERSION} AS wal-g

ARG WALG_VERSION

RUN apk add --update --no-cache lzo-dev cmake make git bash gcc musl-dev

RUN git clone https://github.com/google/brotli.git \
    && cd brotli \
    && git checkout v1.0.9 \
    && mkdir out && cd out \
    && echo "a" \
    && CMAKE_C_COMPILER=gcc ../configure-cmake \
    && echo "b" \
    && CMAKE_C_COMPILER=gcc make \
    && echo "c" \
    && make install

RUN git clone https://github.com/wal-g/wal-g.git \
    && cd wal-g \
    && git checkout ${WALG_VERSION} \
    && make install \
    && make deps \
    && make pg_build

# assemble the final image
FROM postgres:${PG_VERSION}-alpine${ALPINE_VERSION}

COPY --from=wal-g /go/wal-g/main/pg/wal-g /usr/local/bin/wal-g