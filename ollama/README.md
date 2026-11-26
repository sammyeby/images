# Ollama CPU image

Prepare `ghcr.io/sammyeby/ollama:0.40.16-cpu-r1-<run>-<attempt>` for the home-server's
local generation and embeddings. The image is not published or approved yet.

The reviewed contract is Linux/amd64 CPU inference, Ollama's existing API,
upstream stable source and native CPU payload provenance, Go dependency
remediation without suppressions, non-root UID/GID 1000, and startup under a
read-only root with a writable private `.ollama` directory. Cloud functions are
disabled. The consumer must provide encrypted state and restrict networking.

The executable is rebuilt from Ollama v0.40.16, commit
`d63b702609c4300a2ca07be2bdb0adb9834725e5`, with Go 1.27.4 and the generated
`go.mod`/`go.sum`. The upstream graph includes klauspost/compress 1.20.3;
the remaining x/net security fix is applied at 0.62.0. Go 1.27.4 fixes the
standard-library findings that blocked the scheduled audit. All content hashes
are regenerated with Go tooling and verified before tests/build.
Explicit release version/mode linker flags preserve model-version checks.

The complete flat native CPU executable/library payload and native license files are retained from the
exact official AMD64 manifest. GPU/MLX subdirectories and their unused dangling header-directory link are excluded. Native parsing
coverage is limited: a clean OS/Go scan does not prove every C/C++ component safe.
Use only deliberately selected trusted model artifacts and verify their digests.

Acceptance requires focused upstream tests, actual CPU generation and embeddings,
correct `/api/version`, fresh state creation under runtime restrictions, invalid
API input rejection, complete image/SBOM scans and the collection's HIGH/CRITICAL
gate. CI must publish the exact tested artifact under a unique tag, then consumers
pin its immutable digest. No floating/latest tag or vulnerability ignore applies.

Sources:
- [Official Dockerfile](https://github.com/ollama/ollama/blob/d63b702609c4300a2ca07be2bdb0adb9834725e5/Dockerfile)
- [Official release build flags](https://github.com/ollama/ollama/blob/d63b702609c4300a2ca07be2bdb0adb9834725e5/.github/workflows/release.yaml)
- [Startup keypair](https://github.com/ollama/ollama/blob/d63b702609c4300a2ca07be2bdb0adb9834725e5/cmd/cmd.go)

Build: `make build-local`; tests: `make test`; full scan: `make scan-report`;
publication gate: `make scan`. The builder uses its native platform and cross-compiles when necessary; the
runtime target remains Linux/amd64.

## Security evidence and updates

[Historical 0.40.3 candidate review](security/review.json) records the earlier
candidate's binary, scan counts and report hashes. Those results do not accept
the current 0.40.16 image. The current build uses Go 1.27.4, upstream's patched
compressor and the x/net 0.62.0 overlay; the focused upstream tests run during
its build.

`test/audit/check.py` verifies Govulncheck's findings against the exact tested
runtime binary, including its compiled package/function closure. It permits only
the previously reviewed deprecated OpenPGP module-metadata finding when OpenPGP
packages and functions are absent; every other advisory fails. No vulnerability
ignore is added. Native audit, inference and the full image scan are repeated
for every candidate.

The native AMD64 workflow repeats restricted real inference, records a full scan,
SBOM and Go audit, and blocks all HIGH/CRITICAL findings including those without
fixes before publication. A separate permission-limited job loads and publishes
the exact tested image. Weekly Monday checks rebuild and scan but never publish.
Review failures promptly; recheck residual findings by **28 November 2026**. Rebuild when fixed packages become available, rerun the same gates, merge
an image PR, and update consumers only to the resulting verified registry digest.

Native CPU libraries remain the pinned upstream payload (the official 0.40.16 CPU payload).
The OS/Go scans do not cover all native parser behavior. Use trusted, deliberately
selected model artifacts, record their immutable manifests, and isolate inference
from document/database credentials and outbound access. Local results used AMD64
emulation on a Mac; CI must pass on a native AMD64 runner before release.
