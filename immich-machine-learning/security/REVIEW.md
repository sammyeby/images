# ML dependency review — 21 November 2026

The current base is official Immich v3.4.2, amd64 manifest
`72db2089878746176c1a5083272cd2d73e4270f839596b42bdf6b12887bbbb89`.
Local ARM64 compatibility checks use its matching `0cd97ded…` manifest.
The complete installed inventory is regenerated in `runtime.lock`.

Upstream now supplies Pillow 13, newer certifi, requests and python-dotenv;
these overlays are retired to avoid downgrades. The remaining hashed wheel
patches retain their fixes and add OpenTelemetry API 1.45.2 for
[CVE-2026-126360](https://github.com/advisories/GHSA-mf5v-w3w3-f6cg), found by the
independent Python audit. Debian packages receive all available Trixie updates,
including the apt and zlib fixes that caused the scheduled CI failure.
Scheduled/manual builds bypass the cache so new fixes reach the pinned base.

The current independent Python audit reports no known vulnerabilities. Runtime
inference, exact inventory and restrictions are checked with the current image;
native amd64 CI remains the publication gate. Full reports retain unfixed Debian
findings. Review again by 28 November 2026 and before expanding the pilot.
The historical evidence below describes 3.3.5 only.

## Historical 3.3.5 review — 1 November 2026


The official [v3.3.5 application](https://github.com/immich-app/immich/tree/v3.3.5/machine-learning)
and CPU ONNX runtime are retained. The base is pinned to amd64 manifest
`sha256:1270449e47e1fcd703661c4d728b7904d89aacfb715662c26116cc9dd73f266b`;
the corresponding arm64 fixture uses `3719420e…`. Source commit is
`8303cb117e197c700332b33b61b45314c9cc5bb4`.

`patches.lock` records exact versions and SHA256 hashes. `--no-deps --no-build
--require-hashes` installs only reviewed binary wheels. The entire virtual
environment replaces the original, avoiding stale package copies/metadata.
Runtime pip is uninstalled, including its bundled vulnerable setuptools,
msgpack, certifi and urllib3 copies. No advisory metadata is hidden or ignored.

The container scan selected fixes for anyio, certifi, fsspec, msgpack, Pillow,
protobuf, Starlette and urllib3. The independent Python audit found additional
filelock, idna, Pygments, python-dotenv and requests advisories; their published
fixes are included. All other app dependencies stay at upstream versions.
`verify_environment.py` validates every active installed requirement, including
upstream's explicit `rapidocr` GUI→headless OpenCV substitution (both expose cv2).

The mandatory Pillow 12→13 change crosses upstream's `<13` bound. Its runtime
uses Image.open/convert/resize/resampling and ImageOps.exif_transpose; real JPEG
inference, preprocessing and vector tests cover this path. The required Starlette
0.50→1.3.1 change adds parser/path fixes; the upstream FastAPI 0.136.3 requirement
permits it. Actual multipart image/text and malformed-request tests cover this
interface. The filelock 3→4 fix affects cache locking; offline load/preparation,
startup and idle reload are checked. Remaining fixes preserve
the interfaces used here; external model download by the worker stays disabled.
Primary sources: [Pillow 13 release](https://github.com/python-pillow/Pillow/releases/tag/13.0.0),
[Starlette notes](https://github.com/Kludex/starlette/blob/1.3.1/docs/release-notes.md),
[upstream dependency policy](https://github.com/immich-app/immich/blob/v3.3.5/machine-learning/pyproject.toml),
[published package metadata](https://pypi.org/project/filelock/4.2.1/).

Local ARM64 full filesystem scan: zero fixable HIGH/CRITICAL, zero CRITICAL,
44 HIGH without published Debian fixes. These affect util-linux/libmount/libuuid,
libsystemd/libudev, ncurses, perl-base and related Debian packages. Native amd64
CI captures a separate authoritative full report/SBOM and applies the same gate.
No CVE ignore file entries are used. The complete installed Python inventory is
committed in `runtime.lock`; pip-audit 2.10.1 reports no known vulnerabilities.
The audit explicitly cannot match `immich-model==0.2.0` to PyPI: this is upstream's
[git-pinned model runtime](https://github.com/immich-app/ml-models/tree/8bf3fd1b44b0c8608f52fb6e50ddff52826576d8),
retained by the immutable base, not an unreviewed registry substitution. Preserve
that skip in the audit artifact rather than claiming every component is covered.

Limited-pilot deferral for the unfixed native findings: only authenticated
own-camera sample previews reach this CPU worker; no shell/package-manager API,
public port, external network, media/DB/socket mount, privileged device or
capability is provided. Read-only root, encrypted regenerable model cache and
resource limits reduce impact. The separate inference network excludes DB/cache.
These controls do not make a malicious image/parser harmless. Reassess by
**7 November 2026**, before processing unknown-source/WhatsApp media, and whenever
fixes become available. Weekly CI rescans do not automatically publish or deploy.
