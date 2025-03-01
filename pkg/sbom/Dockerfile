# syntax=docker/dockerfile-upstream:1

# Copyright 2022 Docker Packaging authors
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

ARG XX_VERSION="1.2.1"
ARG DEBIAN_FRONTEND="noninteractive"

# common args
ARG GO_IMAGE="golang"
ARG GO_VERSION="1.18"
ARG GO_IMAGE_VARIANT="bullseye"
ARG PKG_RELEASE="debian11"
ARG PKG_TYPE="deb"
ARG PKG_DISTRO="debian"
ARG PKG_SUITE="bullseye"
ARG PKG_BASE_IMAGE="debian:bullseye"

# stage used as named context that mounts common/scripts
# see pkg target in docker-bake.hcl
FROM scratch AS common-scripts

# cross compilation helper
FROM --platform=$BUILDPLATFORM tonistiigi/xx:${XX_VERSION} AS xx

# go base image to retrieve /usr/local/go
FROM ${GO_IMAGE}:${GO_VERSION}-${GO_IMAGE_VARIANT} AS go
FROM --platform=$BUILDPLATFORM ${GO_IMAGE}:${GO_VERSION}-${GO_IMAGE_VARIANT} AS gocross

# src
FROM --platform=$BUILDPLATFORM alpine AS src-base
RUN apk add --no-cache bash curl file git zip tar

FROM src-base AS src
WORKDIR /src
ARG SBOM_REPO
RUN git init . && git remote add origin "${SBOM_REPO}"
ARG SBOM_REF
RUN git fetch origin "${SBOM_REF}" +refs/heads/*:refs/remotes/origin/* +refs/tags/*:refs/tags/* && git checkout -q FETCH_HEAD

FROM src-base AS src-tgz
RUN --mount=from=src,source=/src,target=/sbom \
    mkdir /out && tar -C / -zcf /out/sbom.tgz --exclude .git sbom

# metadata
FROM src-base AS metadata-builder
ARG SBOM_REPO
ARG SBOM_REF
ARG NIGHTLY_BUILD
RUN --mount=type=bind,from=common-scripts,source=gen-ver.sh,target=/usr/local/bin/gen-ver \
    --mount=type=bind,from=src,source=/src,target=/src <<EOT
  set -e
  mkdir -p /out
  for l in $(gen-ver "/src"); do
    export "${l?}"
  done
  cat > "/out/metadata.env" <<-EOF
REPO=${SBOM_REPO%.*}
REF=${SBOM_REF}
VERSION=${GENVER_VERSION}
COMMIT=${GENVER_COMMIT}
EOF
EOT

FROM scratch AS metadata
COPY --from=metadata-builder /out /

# deb
FROM ${PKG_BASE_IMAGE} AS builder-deb
COPY --from=xx / /
ARG DEBIAN_FRONTEND
ENV GOPROXY="https://proxy.golang.org|direct"
ENV GOPATH="/go"
ENV PATH="$PATH:/usr/local/go/bin:$GOPATH/bin"
ENV GO111MODULE="on"
ENV CGO_ENABLED="0"
ARG PKG_RELEASE
RUN apt-get update && apt-get install -y --no-install-recommends apt-utils bash ca-certificates curl devscripts equivs git
COPY deb /root/package/debian
ARG TARGETPLATFORM
RUN mk-build-deps -t "xx-apt-get -o Debug::pkgProblemResolver=yes --no-install-recommends -y" -i /root/package/debian/control
WORKDIR /root/package
ARG SBOM_REF
ARG NIGHTLY_BUILD
ARG PKG_NAME
ARG PKG_DISTRO
ARG PKG_SUITE
ARG PKG_PACKAGER
ARG PKG_DEB_BUILDFLAGS
ARG PKG_DEB_REVISION
ARG PKG_DEB_EPOCH
ARG SOURCE_DATE_EPOCH
RUN --mount=type=bind,source=scripts/pkg-deb-build.sh,target=/usr/local/bin/pkg-deb-build \
    --mount=type=bind,from=common-scripts,source=gen-ver.sh,target=/usr/local/bin/gen-ver \
    --mount=type=bind,from=common-scripts,source=fix-cc.sh,target=/usr/local/bin/fix-cc \
    --mount=type=bind,from=src,source=/src,target=/root/package/sbom,rw \
    --mount=type=bind,from=go,source=/usr/local/go,target=/usr/local/go \
    OUTDIR=/out SRCDIR=./sbom pkg-deb-build

# rpm
FROM ${PKG_BASE_IMAGE} AS builder-rpm
COPY --from=xx / /
ENV GOPROXY="https://proxy.golang.org|direct"
ENV GOPATH="/go"
ENV PATH="$PATH:/usr/local/go/bin:$GOPATH/bin"
ENV GO111MODULE="on"
ENV CGO_ENABLED="0"
ARG PKG_RELEASE
RUN --mount=type=bind,from=common-scripts,source=rpm-init.sh,target=/usr/local/bin/rpm-init \
  rpm-init $PKG_RELEASE
COPY rpm /root/rpmbuild/SPECS
ARG TARGETPLATFORM
RUN <<EOT
  set -e
  builddepCmd=""
  if command -v dnf &> /dev/null; then
    builddepCmd="setarch $(xx-info rhel-arch) dnf builddep"
  elif command -v yum-builddep &> /dev/null; then
    builddepCmd="yum-builddep --target $(xx-info rhel-arch)"
  else
    echo >&2 "unable to detect package manager"
    exit 1
  fi
  set -x
  $builddepCmd -y /root/rpmbuild/SPECS/*.spec
EOT
WORKDIR /root/rpmbuild
ARG SBOM_REF
ARG NIGHTLY_BUILD
ARG PKG_NAME
ARG PKG_DISTRO
ARG PKG_SUITE
ARG PKG_PACKAGER
ARG PKG_RPM_BUILDFLAGS
ARG PKG_RPM_RELEASE
ARG SOURCE_DATE_EPOCH
RUN --mount=type=bind,source=scripts/pkg-rpm-build.sh,target=/usr/local/bin/pkg-rpm-build \
    --mount=type=bind,from=common-scripts,source=gen-ver.sh,target=/usr/local/bin/gen-ver \
    --mount=type=bind,from=common-scripts,source=fix-cc.sh,target=/usr/local/bin/fix-cc \
    --mount=type=bind,from=src-tgz,source=/out/sbom.tgz,target=/root/rpmbuild/SOURCES/sbom.tgz \
    --mount=type=bind,from=src,source=/src,target=/usr/local/src/sbom \
    --mount=type=bind,from=go,source=/usr/local/go,target=/usr/local/go \
    OUTDIR=/out SRCDIR=/usr/local/src/sbom pkg-rpm-build

# static
FROM --platform=$BUILDPLATFORM ${PKG_BASE_IMAGE} AS builder-static
COPY --from=xx / /
ARG DEBIAN_FRONTEND
RUN apt-get update && apt-get install -y --no-install-recommends bash ca-certificates file git zip tar
ENV GOPROXY="https://proxy.golang.org|direct"
ENV GOPATH="/go"
ENV PATH="$PATH:/usr/local/go/bin:$GOPATH/bin"
ENV GO111MODULE="on"
ENV CGO_ENABLED="0"
ARG PKG_NAME
ARG SBOM_REF
ARG NIGHTLY_BUILD
WORKDIR /build
ARG TARGETPLATFORM
RUN --mount=type=bind,source=scripts/pkg-static-build.sh,target=/usr/local/bin/pkg-static-build \
    --mount=type=bind,from=common-scripts,source=gen-ver.sh,target=/usr/local/bin/gen-ver \
    --mount=type=bind,from=common-scripts,source=fix-cc.sh,target=/usr/local/bin/fix-cc \
    --mount=type=bind,from=src,source=/src,target=/usr/local/src/sbom \
    --mount=type=bind,from=gocross,source=/usr/local/go,target=/usr/local/go,rw \
    OUTDIR=/out BUILDDIR=/build SRCDIR=/usr/local/src/sbom pkg-static-build

FROM builder-${PKG_TYPE} AS build-pkg
ARG BUILDKIT_SBOM_SCAN_STAGE=true

FROM scratch AS pkg
COPY --from=build-pkg /out /
