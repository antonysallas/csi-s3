# Build stage for Golang
FROM golang:1.21.3-bullseye as gobuild

WORKDIR /build

# Copy the Go module files and download dependencies
COPY go.mod go.sum ./
RUN go mod tidy && \
    go mod download && \
    go get -d -v ./...

# Copy the rest of the source code
COPY . .

# Build the application
RUN CGO_ENABLED=0 GOOS=linux go build -a -ldflags '-extldflags "-static"' -o ./s3driver ./cmd/s3driver

# Build stage for s3backer
FROM debian:bullseye-slim as s3backer
ARG S3BACKER_VERSION=2.1.1

RUN apt-get clean && rm -rf /var/lib/apt/lists/* && \
    apt-get update && \
    apt-get update --fix-missing && \
    apt-get install -y \
    build-essential \
    autoconf \
    libcurl4-openssl-dev \
    libfuse-dev \
    libexpat1-dev \
    libssl-dev \
    zlib1g-dev \
    psmisc \
    pkg-config \
    git && \
    git clone https://github.com/archiecobbs/s3backer.git /src/s3backer && \
    cd /src/s3backer && \
    git checkout tags/${S3BACKER_VERSION} && \
    ./autogen.sh && \
    ./configure && \
    make && \
    make install && \
    apt-get purge -y \
    build-essential \
    autoconf \
    pkg-config \
    git && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/* /src/s3backer && \
    apt-get clean

# Final stage
FROM debian:bullseye-slim

COPY --from=s3backer /usr/bin/s3backer /usr/bin/s3backer

# Install s3fs, rclone, and other dependencies
RUN apt-get clean && rm -rf /var/lib/apt/lists/* && \
    apt-get update && \
    apt-get update --fix-missing && \
    apt-get install -y \
    libfuse2 gcc sqlite3 libsqlite3-dev \
    s3fs psmisc procps libcurl4 xfsprogs curl unzip && \
    apt-get autoremove -y

ARG RCLONE_VERSION=v1.64.2
RUN curl -O https://downloads.rclone.org/${RCLONE_VERSION}/rclone-${RCLONE_VERSION}-linux-amd64.zip && \
    unzip rclone-${RCLONE_VERSION}-linux-amd64.zip && \
    mv rclone-*-linux-amd64/rclone /usr/bin/ && \
    rm -r rclone*

COPY --from=gobuild /build/s3driver /s3driver
ENTRYPOINT ["/s3driver"]
