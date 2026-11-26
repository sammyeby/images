# Keep the supported v3.4.2 application/ONNX runtime; patch only affected wheels.
ARG UPSTREAM=ghcr.io/immich-app/immich-machine-learning:v3.4.2@sha256:72db2089878746176c1a5083272cd2d73e4270f839596b42bdf6b12887bbbb89
FROM ${UPSTREAM} AS patches
COPY --from=ghcr.io/astral-sh/uv:0.8.15@sha256:a5727064a0de127bdb7c9d3c1383f3a9ac307d9f2d8a391edc7896c54289ced0 /uv /usr/local/bin/uv
COPY security/patches.lock /tmp/patches.lock
COPY verify_environment.py /tmp/verify_environment.py
RUN uv pip install --python /opt/venv/bin/python --no-deps --no-build --require-hashes --no-cache -r /tmp/patches.lock && /opt/venv/bin/python /tmp/verify_environment.py

FROM ${UPSTREAM} AS runtime
LABEL org.opencontainers.image.title="SamSoft Technologies Immich Machine Learning" \
    org.opencontainers.image.vendor="SamSoft Technologies" \
    org.opencontainers.image.source="https://github.com/sammyeby/images"
# Refresh the pinned Debian filesystem: CVE-2026-109518 (apt) and
# CVE-2026-76844 / CVE-2026-85091 (zlib) have Trixie security backports.
RUN apt-get update && apt-get upgrade -y && rm -rf /var/lib/apt/lists/*
# Replace the environment completely; merging leaves old vulnerable dist-info.
RUN rm -rf /opt/venv
COPY --from=patches /opt/venv /opt/venv
# The runtime has no installer use. Remove pip and its vulnerable vendored copies,
# including setuptools70, msgpack, certifi and urllib3; do not hide their metadata.
RUN /usr/local/bin/python3 -m pip uninstall -y pip && \
    groupadd --gid 1000 immich && useradd --uid 1000 --gid 1000 --no-create-home --shell /usr/sbin/nologin immich
USER 1000:1000
ENV IMMICH_HOST=0.0.0.0 \
    MACHINE_LEARNING_WORKERS=1 \
    MACHINE_LEARNING_REQUEST_THREADS=1 \
    MACHINE_LEARNING_MODEL_INTER_OP_THREADS=1 \
    MACHINE_LEARNING_MODEL_INTRA_OP_THREADS=1 \
    MACHINE_LEARNING_MODEL_TTL=60
