# Retain upstream web/CLI interface, media tools and plugins; rebuild Node code
# from the exact release source with the committed pnpm security patch.
# Preserve the current FUTO fork, including its disabled self-update contract.
# Source: https://github.com/futo-org/restic/blob/520db2dd422dfb223e974d75866d7e200f2ac0be/docker/Dockerfile.futo
FROM golang:1.27.4-bookworm@sha256:4fa7c968ba93f1635670522dbb01a2c78150443e1385746924566079f36570ea AS restic-builder
ENV GOTOOLCHAIN=local GOFLAGS=-mod=readonly
WORKDIR /source
ADD https://codeload.github.com/futo-org/restic/tar.gz/520db2dd422dfb223e974d75866d7e200f2ac0be /tmp/restic-source.tar.gz
RUN echo 'cd9717015a57987fda8531b1fba0335952b7ff66fd1ed0061ece7c1bdfae945c  /tmp/restic-source.tar.gz' | sha256sum -c - && \
    tar -xzf /tmp/restic-source.tar.gz --strip-components=1 -C /source
COPY patches/restic-dependencies.patch /tmp/restic-dependencies.patch
# CVE-2026-63209, CVE-2026-56854, CVE-2026-78661, CVE-2026-78669,
# CVE-2026-56851 / CVE-2026-56852 and CVE-2026-84304 / CVE-2026-84445.
RUN apt-get update && apt-get install -y --no-install-recommends patch python-is-python3 && \
    rm -rf /var/lib/apt/lists/* && \
    patch -p1 < /tmp/restic-dependencies.patch && go mod download && go mod verify
# Upstream permission/error-path tests require an unprivileged process and
# invoke Python. Match that test environment rather than running them as root.
RUN useradd --uid 1000 --create-home restic-build && mkdir /out && \
    chown -R 1000:1000 /source /go /out
ENV HOME=/home/restic-build
USER 1000:1000
RUN RESTIC_TEST_FUSE=0 go test -timeout 180s ./internal/archiver/... ./cmd/restic/... && \
    CGO_ENABLED=0 go build -trimpath -ldflags '-s -w' -tags disable_grpc_modules -o /out/restic ./cmd/restic

FROM ghcr.io/immich-app/base-server-dev:202610291416@sha256:5f4f9f77059e1cd0182994419a19f4ee045c3fd48e611f5e77623ed63f7aa1e4 AS builder
ENV CI=1 PNPM_HOME=/opt/pnpm PATH="/opt/pnpm/bin:$PATH"
# pnpm 12 ships native executables. Pin both supported builder architectures;
# do not let its JavaScript launcher download an unchecked binary at build time.
ADD https://registry.npmjs.org/@pnpm/exe.linux-x64/-/exe.linux-x64-12.15.0.tgz /tmp/pnpm-amd64.tgz
ADD https://registry.npmjs.org/@pnpm/exe.linux-arm64/-/exe.linux-arm64-12.15.0.tgz /tmp/pnpm-arm64.tgz
RUN echo 'c9cf9095947ef7e9ebdbd2787d1ce13286a98314c10e776affea5eca8ab55d4b9f9d01677db5d4305b9d0c82b31e2e78e658a71aa98e9977e7d0da2e7cd81c3a  /tmp/pnpm-amd64.tgz' | sha512sum -c - && \
    echo 'a8c2fff1a097b2587cb5dff0e14f8724da94719dbe9ba11f4db592708cd9466c8fb1114dfb10782cfd54fe1ca6c5cd6248988ab46e7a1c9c767ef504ad61481b  /tmp/pnpm-arm64.tgz' | sha512sum -c - && \
    mkdir -p /opt/pnpm/bin && \
    tar -xzf "/tmp/pnpm-$(dpkg --print-architecture).tgz" --strip-components=1 -C /opt/pnpm/bin && \
    chmod 755 /opt/pnpm/bin/pnpm
ADD https://codeload.github.com/immich-app/immich/tar.gz/refs/tags/v3.4.2 /tmp/source.tar.gz
RUN echo '035151c34a9fec72b40779bb8122bf4c2f9c420c11d1cc0f4e54801f300e6200  /tmp/source.tar.gz' | sha256sum -c - && \
    mkdir -p /usr/src/app && tar -xzf /tmp/source.tar.gz --strip-components=1 -C /usr/src/app
WORKDIR /usr/src/app
COPY patches/dependencies.patch /tmp/dependencies.patch
COPY patches/nestjs-zod-5.5.0.patch /usr/src/app/patches/nestjs-zod@5.5.0.patch
RUN patch -p1 < /tmp/dependencies.patch && \
    pnpm --filter immich... --filter @immich/cli... install --frozen-lockfile --ignore-scripts
# The source policy allows only bcrypt/sharp native builds for these packages.
# Keep scripts disabled during initial resolution; the two builds are explicit.
RUN pnpm --filter immich rebuild bcrypt sharp && \
    pnpm --filter @immich/sdk build && pnpm --filter @immich/plugin-sdk build && \
    pnpm --filter immich build && pnpm --filter @immich/cli build && \
    pnpm --filter immich --prod --no-optional deploy /output/server-pruned && \
    pnpm --filter @immich/cli --prod --no-optional deploy /output/cli-pruned
RUN SHARP_FORCE_GLOBAL_LIBVIPS=true pnpm --config.verify-deps-before-run=false \
    --dir /output/server-pruned/node_modules/sharp exec npm run build
# Audit the deployed lockfiles: unrelated docs/web development dependencies are
# not part of either deployed workspace. HIGH/CRITICAL still fail this build.
RUN mkdir -p /output/audits && \
    (cd /output/server-pruned && pnpm audit --prod --audit-level high --json > /output/audits/server.json) && \
    (cd /output/cli-pruned && pnpm audit --prod --audit-level high --json > /output/audits/cli.json)

FROM ghcr.io/immich-app/immich-server:v3.4.2@sha256:fc880c1bf665b4aab1e74c15e56ab871225989cd546536e6b77e771869af06bd AS runtime
USER root
LABEL org.opencontainers.image.title="SamSoft Technologies Immich" \
    org.opencontainers.image.vendor="SamSoft Technologies" \
    org.opencontainers.image.source="https://github.com/sammyeby/images" \
    org.opencontainers.image.version="3.4.2-r1" \
    org.opencontainers.image.licenses="AGPL-3.0-only"
# CVE-2026-111497: native image decoders use Debian Brotli, distinct from Node's
# bundled decoder (already fixed in upstream Node 24.22.1).
RUN apt-get update && apt-get upgrade -y && \
    dpkg --compare-versions "$(dpkg-query -W -f='${Version}' libbrotli1)" ge '1.1.0-2+deb13u1' && \
    rm -rf /var/lib/apt/lists/* /usr/src/app/server /usr/src/app/cli && \
    rm -rf /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/corepack && \
    rm -f /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack /usr/local/bin/pnpm /usr/local/bin/pnpx
COPY --from=builder /output/server-pruned /usr/src/app/server
COPY --from=builder /output/cli-pruned /usr/src/app/cli
COPY --from=builder /output/audits /usr/share/bysamio/audits
# Replace the vulnerable bundled executable; retain all FUTO backup features.
COPY --from=restic-builder /out/restic /usr/local/bin/restic
RUN ln -s ../../cli/bin/immich /usr/src/app/server/bin/immich
ENV HOME=/tmp
USER 1000:1000
