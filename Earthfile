VERSION 0.6

IMPORT github.com/kairos-io/kairos

FROM alpine
ARG VARIANT=kairos # core, lite, framework
ARG FLAVOR=opensuse

## Versioning
ARG K3S_VERSION
RUN apk add git
COPY . ./
RUN echo $(git describe --exact-match --tags || echo "v0.0.0-$(git log --oneline -n 1 | cut -d" " -f1)") > VERSION
ARG CORE_VERSION=$(cat CORE_VERSION || echo "latest")
ARG VERSION=$(cat VERSION)
RUN echo "version ${VERSION}"
ARG K3S_VERSION_TAG=$(echo $K3S_VERSION | sed s/+/-/)
ARG TAG=${VERSION}-k3s${K3S_VERSION_TAG}
ARG IMAGE=quay.io/kairos/${VARIANT}-${FLAVOR}:$TAG
ARG BASE_IMAGE=quay.io/kairos/core-${FLAVOR}:${CORE_VERSION}
ARG ISO_NAME=${VARIANT}-${FLAVOR}-${VERSION}-k3s${K3S_VERSION}
ARG OSBUILDER_IMAGE=quay.io/kairos/osbuilder-tools

## External deps pinned versions
ARG LUET_VERSION=0.32.4
ARG ELEMENTAL_IMAGE=quay.io/costoolkit/elemental-cli:v0.0.15-8a78e6b
ARG GOLINT_VERSION=1.47.3
ARG GO_VERSION=1.18

ARG OS_ID=kairos
ARG CGO_ENABLED=0

RELEASEVERSION:
    COMMAND
    RUN echo "$IMAGE" > IMAGE
    RUN echo "$VERSION" > VERSION
    SAVE ARTIFACT VERSION AS LOCAL build/VERSION
    SAVE ARTIFACT IMAGE AS LOCAL build/IMAGE

all:
  BUILD +docker
  BUILD +iso
  BUILD +netboot
  BUILD +ipxe-iso
  DO +RELEASEVERSION

all-arm:
  BUILD --platform=linux/arm64 +docker
  BUILD +arm-image
  DO +RELEASEVERSION

go-deps:
    ARG GO_VERSION
    FROM golang:$GO_VERSION
    WORKDIR /build
    COPY go.mod go.sum ./
    RUN go mod download
    RUN apt-get update && apt-get install -y upx
    SAVE ARTIFACT go.mod AS LOCAL go.mod
    SAVE ARTIFACT go.sum AS LOCAL go.sum

test:
    FROM +go-deps
    WORKDIR /build
    RUN go get github.com/onsi/gomega/...
    RUN go get github.com/onsi/ginkgo/v2/ginkgo/internal@v2.1.4
    RUN go get github.com/onsi/ginkgo/v2/ginkgo/generators@v2.1.4
    RUN go get github.com/onsi/ginkgo/v2/ginkgo/labels@v2.1.4

    COPY (kairos+luet/luet) /usr/bin/luet
    COPY . .
    RUN go install github.com/onsi/ginkgo/v2/ginkgo
    RUN ginkgo run --fail-fast --slow-spec-threshold 30s --covermode=atomic --coverprofile=coverage.out -p -r ./internal
    SAVE ARTIFACT coverage.out AS LOCAL coverage.out

BUILD_GOLANG:
    COMMAND
    WORKDIR /build
    COPY . ./
    ARG CGO_ENABLED
    ARG BIN
    ARG SRC
    ENV CGO_ENABLED=${CGO_ENABLED}

    RUN go build -ldflags "-s -w" -o ${BIN} ${SRC} && upx ${BIN}
    SAVE ARTIFACT ${BIN} ${BIN} AS LOCAL build/${BIN}

build-kairos-agent-provider:
    FROM +go-deps
    DO +BUILD_GOLANG --BIN=agent-provider-kairos --SRC=./ --CGO_ENABLED=$CGO_ENABLED

build:
    BUILD +build-kairos-agent-provider

dist:
    ARG GO_VERSION
    FROM golang:$GO_VERSION
    RUN echo 'deb [trusted=yes] https://repo.goreleaser.com/apt/ /' | tee /etc/apt/sources.list.d/goreleaser.list
    RUN apt update
    RUN apt install -y goreleaser
    WORKDIR /build
    COPY . .
    RUN goreleaser build --rm-dist --skip-validate --snapshot
    SAVE ARTIFACT /build/dist/* AS LOCAL dist/

lint:
    ARG GO_VERSION
    FROM golang:$GO_VERSION
    ARG GOLINT_VERSION
    RUN wget -O- -nv https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s v$GOLINT_VERSION
    WORKDIR /build
    COPY . .
    RUN golangci-lint run --timeout 120s

docker:
    ARG FLAVOR
    ARG VARIANT

    FROM $BASE_IMAGE

    IF [ "$K3S_VERSION" = "latest" ]
    ELSE
        ENV INSTALL_K3S_VERSION=${K3S_VERSION}
    END
    
    COPY repository.yaml /etc/luet/luet.yaml

    ENV INSTALL_K3S_BIN_DIR="/usr/bin"
    RUN curl -sfL https://get.k3s.io > installer.sh \
        && INSTALL_K3S_SKIP_START="true" INSTALL_K3S_SKIP_ENABLE="true" bash installer.sh \
        && INSTALL_K3S_SKIP_START="true" INSTALL_K3S_SKIP_ENABLE="true" bash installer.sh agent \
        && rm -rf installer.sh
    RUN luet install -y utils/edgevpn utils/k9s utils/nerdctl container/kubectl && luet cleanup
    # Drop env files from k3s as we will generate them
    IF [ -e "/etc/rancher/k3s/k3s.env" ]
        RUN rm -rf /etc/rancher/k3s/k3s.env /etc/rancher/k3s/k3s-agent.env && touch /etc/rancher/k3s/.keep
    END

    COPY +build-kairos-agent-provider/agent-provider-kairos /system/providers/agent-provider-kairos
    RUN ln -s /system/providers/agent-provider-kairos /usr/bin/kairos

    ARG KAIROS_VERSION
    IF [ "$KAIROS_VERSION" = "" ]
        ARG OS_VERSION=${VERSION}
    ELSE 
        ARG OS_VERSION=${KAIROS_VERSION}
    END
    
    ARG OS_ID
    ARG OS_NAME=${OS_ID}-${FLAVOR}
    ARG OS_REPO=quay.io/kairos/${VARIANT}-${FLAVOR}
    ARG OS_LABEL=latest

    DO kairos+OSRELEASE --BUG_REPORT_URL="https://github.com/kairos-io/kairos/issues/new/choose" --HOME_URL="https://github.com/kairos-io/provider-kairos" --OS_ID=${OS_ID} --OS_LABEL=${OS_LABEL} --OS_NAME=${OS_NAME} --OS_REPO=${OS_REPO} --OS_VERSION=${OS_VERSION}-k3s${K3S_VERSION} --GITHUB_REPO="kairos-io/provider-kairos"

    SAVE IMAGE $IMAGE

docker-rootfs:
    FROM +docker
    SAVE ARTIFACT /. rootfs

elemental:
    ARG ELEMENTAL_IMAGE
    FROM ${ELEMENTAL_IMAGE}
    SAVE ARTIFACT /usr/bin/elemental elemental

kairos:
   ARG KAIROS_VERSION=master
   FROM alpine
   RUN apk add git
   WORKDIR /kairos
   RUN git clone https://github.com/kairos-io/kairos /kairos && cd /kairos && git checkout "$KAIROS_VERSION"
   SAVE ARTIFACT /kairos/

get-kairos-scripts:
    FROM alpine
    WORKDIR /build
    COPY +kairos/kairos/ ./
    SAVE ARTIFACT /build/scripts AS LOCAL scripts

iso:
    ARG OSBUILDER_IMAGE
    ARG ISO_NAME=${OS_ID}
    ARG IMG=docker:$IMAGE
    ARG overlay=overlay/files-iso
    FROM $OSBUILDER_IMAGE
    RUN zypper in -y jq docker
    WORKDIR /build
    COPY . ./
    RUN mkdir -p overlay/files-iso
    COPY +kairos/kairos/overlay/files-iso/ ./overlay/files-iso/
    WITH DOCKER --allow-privileged --load $IMAGE=(+docker)
        RUN /entrypoint.sh --name $ISO_NAME --debug build-iso --date=false --local --overlay-iso /build/${overlay} $IMAGE --output /build/
    END
    # See: https://github.com/rancher/elemental-cli/issues/228
    RUN sha256sum $ISO_NAME.iso > $ISO_NAME.iso.sha256
    SAVE ARTIFACT /build/$ISO_NAME.iso kairos.iso AS LOCAL build/$ISO_NAME.iso
    SAVE ARTIFACT /build/$ISO_NAME.iso.sha256 kairos.iso.sha256 AS LOCAL build/$ISO_NAME.iso.sha256

netboot:
   FROM opensuse/leap
   ARG VERSION
   ARG ISO_NAME
   WORKDIR /build
   COPY +iso/kairos.iso kairos.iso
   COPY . .
   RUN zypper in -y cdrtools

   COPY +kairos/kairos/scripts/netboot.sh ./
   RUN sh netboot.sh kairos.iso $ISO_NAME $VERSION

   SAVE ARTIFACT /build/$ISO_NAME.squashfs squashfs AS LOCAL build/$ISO_NAME.squashfs
   SAVE ARTIFACT /build/$ISO_NAME-kernel kernel AS LOCAL build/$ISO_NAME-kernel
   SAVE ARTIFACT /build/$ISO_NAME-initrd initrd AS LOCAL build/$ISO_NAME-initrd
   SAVE ARTIFACT /build/$ISO_NAME.ipxe ipxe AS LOCAL build/$ISO_NAME.ipxe

arm-image:
  ARG ELEMENTAL_IMAGE
  FROM $ELEMENTAL_IMAGE
  ARG MODEL=rpi64
  ARG IMAGE_NAME=${VARIANT}-${FLAVOR}-${VERSION}-k3s${K3S_VERSION}.img
  RUN zypper in -y jq docker git curl gptfdisk kpartx sudo
  #COPY +luet/luet /usr/bin/luet
  WORKDIR /build
  RUN git clone https://github.com/rancher/elemental-toolkit && mkdir elemental-toolkit/build
  RUN curl https://luet.io/install.sh | sh
  ENV STATE_SIZE="6200"
  ENV RECOVERY_SIZE="4200"
  ENV SIZE="15200"
  ENV DEFAULT_ACTIVE_SIZE="2000"
  COPY --platform=linux/arm64 +docker-rootfs/rootfs /build/image
  # With docker is required for loop devices
  WITH DOCKER --allow-privileged
    RUN cd elemental-toolkit && \
          ./images/arm-img-builder.sh --model $MODEL --directory "/build/image" build/$IMAGE_NAME && mv build ../
  END
  RUN xz -v /build/build/$IMAGE_NAME
  SAVE ARTIFACT /build/build/$IMAGE_NAME.xz img AS LOCAL build/$IMAGE_NAME
  SAVE ARTIFACT /build/build/$IMAGE_NAME.sha256 img-sha256 AS LOCAL build/$IMAGE_NAME.sha256

ipxe-iso:
    FROM ubuntu
    ARG ipxe_script
    RUN apt update
    RUN apt install -y -o Acquire::Retries=50 \
                           mtools syslinux isolinux gcc-arm-none-eabi git make gcc liblzma-dev mkisofs xorriso
                           # jq docker
    WORKDIR /build
    ARG ISO_NAME=${OS_ID}
    RUN git clone https://github.com/ipxe/ipxe
    IF [ "$ipxe_script" = "" ]
        COPY +netboot/ipxe /build/ipxe/script.ipxe
    ELSE
        COPY $ipxe_script /build/ipxe/script.ipxe
    END
    RUN cd ipxe/src && make EMBED=/build/ipxe/script.ipxe
    SAVE ARTIFACT /build/ipxe/src/bin/ipxe.iso iso AS LOCAL build/${ISO_NAME}-ipxe.iso.ipxe
    SAVE ARTIFACT /build/ipxe/src/bin/ipxe.usb usb AS LOCAL build/${ISO_NAME}-ipxe-usb.img.ipxe


## Security targets
trivy:
    FROM aquasec/trivy
    SAVE ARTIFACT /usr/local/bin/trivy /trivy

trivy-scan:
    ARG SEVERITY=CRITICAL
    FROM +docker
    COPY +trivy/trivy /trivy
    RUN /trivy filesystem --severity $SEVERITY --exit-code 1 --no-progress /

linux-bench:
    ARG GO_VERSION
    FROM golang:$GO_VERSION
    GIT CLONE https://github.com/aquasecurity/linux-bench /linux-bench-src
    RUN cd /linux-bench-src && CGO_ENABLED=0 go build -o linux-bench . && mv linux-bench /
    SAVE ARTIFACT /linux-bench /linux-bench

# The target below should run on a live host instead. 
# However, some checks are relevant as well at container level.
# It is good enough for a quick assessment.
linux-bench-scan:
    FROM +docker
    GIT CLONE https://github.com/aquasecurity/linux-bench /build/linux-bench
    WORKDIR /build/linux-bench
    COPY +linux-bench/linux-bench /build/linux-bench/linux-bench
    RUN /build/linux-bench/linux-bench

# Generic targets
# usage e.g. ./earthly.sh +datasource-iso --CLOUD_CONFIG=tests/assets/qrcode.yaml
datasource-iso:
  ARG ELEMENTAL_IMAGE
  ARG CLOUD_CONFIG
  FROM $ELEMENTAL_IMAGE
  RUN zypper in -y mkisofs
  WORKDIR /build
  RUN touch meta-data
  COPY ./${CLOUD_CONFIG} user-data
  RUN cat user-data
  RUN mkisofs -output ci.iso -volid cidata -joliet -rock user-data meta-data
  SAVE ARTIFACT /build/ci.iso iso.iso AS LOCAL build/datasource.iso

# usage e.g. ./earthly.sh +run-qemu-tests --FLAVOR=alpine --FROM_ARTIFACTS=true
run-qemu-tests:
    FROM opensuse/leap
    WORKDIR /test
    RUN zypper in -y qemu-x86 qemu-arm qemu-tools go
    ARG FLAVOR
    ARG TEST_SUITE=autoinstall-test
    ARG FROM_ARTIFACTS
    ENV FLAVOR=$FLAVOR
    ENV SSH_PORT=60022
    ENV CREATE_VM=true
    ARG CLOUD_CONFIG="/tests/tests/assets/autoinstall.yaml"
    ENV USE_QEMU=true

    ENV GOPATH="/go"

    ENV CLOUD_CONFIG=$CLOUD_CONFIG

    IF [ "$FROM_ARTIFACTS" = "true" ]
        COPY . .
        ENV ISO=/test/build/kairos.iso
        ENV DATASOURCE=/test/build/datasource.iso
    ELSE
        COPY ./tests .
        COPY +iso/kairos.iso kairos.iso
        COPY ( +datasource-iso/iso.iso --CLOUD_CONFIG=$CLOUD_CONFIG) datasource.iso
        ENV ISO=/test/kairos.iso
        ENV DATASOURCE=/test/datasource.iso
    END

    RUN go install github.com/onsi/ginkgo/v2/ginkgo

    ENV CLOUD_INIT=$CLOUD_CONFIG

    RUN PATH=$PATH:$GOPATH/bin ginkgo --label-filter "$TEST_SUITE" --fail-fast -r ./tests/

test-create-config:
    FROM alpine
    ARG WITH_DNS
    COPY +build-kairos-agent-provider/agent-provider-kairos agent-provider-kairos
    COPY . .
    RUN ./agent-provider-kairos create-config > config.yaml
    RUN cat tests/assets/config.yaml >> config.yaml 
    IF [ "$WITH_DNS" == "true" ]
        RUN apk add yq
        RUN yq -i '.kairos.dns = true' 'config.yaml'
    END
    SAVE ARTIFACT config.yaml AS LOCAL config.yaml
