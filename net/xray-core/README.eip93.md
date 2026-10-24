# AN7581 EIP93 hybrid AES-GCM

Ported from `helloworld/xray-core` patch 190. The ARM64 build uses a private
Go source overlay for Xray; it does not modify the shared Go installation.
The source overlay expects Go 1.27's internal AES-GCM implementation and fails
the build if the source fragments it patches no longer match.

The backend binds `gcm_base(ctr(aes-eip93),ghash-generic)` through Linux AF_ALG.
EIP93 performs AES-CTR; GHASH runs on the CPU. This is hybrid AES-GCM, not native
hardware GCM. Users of Go AES-GCM, including Go TLS, can benefit; other ciphers
and TLS implementations outside Go do not use this overlay.

## Selection and fallback

- Enable only on AN7581/EN7581 without CPU AES instructions, based on the root
  device tree `compatible` property. AN7583/EN7583 and FIPS mode are excluded.
- Accelerate 4–32 KiB payloads with 12-byte nonces, 16-byte tags and at most
  256 bytes of additional authenticated data.
- Verify each newly opened key against Go's generic implementation, including
  corrupted-tag rejection, before handling application data.
- Limit operation sockets to 128, serialize operations per key and set socket
  timeouts. Unsupported requests and operational failures use Go's original
  implementation. Authentication failures remain failures.
- Set `XRAY_EIP93=off` in the Xray process environment to disable acceleration.

The first successful activation writes this message to stderr:

```text
Xray EIP93: AN7581 AES-GCM hybrid backend enabled (AF_ALG CTR + CPU GHASH)
```

On the AN7581 target the package selects `kmod-crypto-hw-eip93`,
`kmod-crypto-gcm` (including GHASH/CTR dependencies) and `kmod-crypto-user`.
A working EIP93 device tree and kernel driver are also required. Missing or
unusable backends fall back to software; socket timeouts cannot recover a
hardware DMA engine that has hung.

## Validation

Before this port, an independent AF_ALG diagnostic on Nokia XG-040G-MD (UBI),
AN7581, PonWrt Linux 6.18.55 passed AES-256 CTR and AES-128/192/256 GCM
encryption/decryption comparisons against unmodified Go, plus corrupted-tag
rejection for 257-, 4096-, 16384- and 32768-byte payloads. EIP93 IRQ counts
increased from 0 to 35. That verifies the kernel path, not this Xray build's
concurrency, long-term stability or throughput.

The patch includes `TestEIP93DTGate` and `TestEIP93Hardware`. After generating
the overlay with Go 1.27.2, cross-compile the test binaries from the patched
Xray source directory:

```sh
python3 tools/eip93/overlay.py --go go --out "$PWD/.eip93-overlay"
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go test \
  -overlay="$PWD/.eip93-overlay/overlay.json" -c -o gcm.test \
  crypto/internal/fips140/aes/gcm
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go test \
  -overlay="$PWD/.eip93-overlay/overlay.json" -c -o cipher.test crypto/cipher
```

On AN7581, run `./gcm.test -test.run TestEIP93 -test.v`, then
`./cipher.test -test.short`. Repeat the cipher suite with `XRAY_EIP93=off`.
The full cipher short suite includes a CTR test holding three 32 MiB buffers;
`-test.short` does not exclude it. Account for its memory use, including any
test binaries and archives stored in tmpfs, when running on a router.

Port checks completed with Go 1.27.2:

- The patch applies to the hash-verified 26.10.21 archive with `--fuzz=0`.
- ARM64 Xray and both test binaries compile. The existing `golang-build.sh`
  configure/build flow also succeeds with the overlay and CGO disabled.
- Host `crypto/cipher` short regressions pass with normal CPU detection and
  with `GODEBUG=cpu.all=off`.
- Xray `common/crypto` and `infra/conf` short tests pass. The latter uses empty
  GeoIP/GeoSite fixtures for its file-existence checks, not geodata parsing.
- The shared GOROOT files remain unchanged.

On-device validation of the migrated Go 1.27.2 overlay on the same Nokia
AN7581 / Linux 6.18.55 passed:

- The static Xray binary reports version 26.10.21 and starts its version command.
- `TestEIP93DTGate` and `TestEIP93Hardware` pass, including payload boundaries,
  in-place operations, corrupted tags, concurrent requests, closed-socket
  fallback and nonstandard nonce/tag exclusion. IRQ counts rise 8838 to 11775.
- The full `crypto/cipher` short regression passes with acceleration enabled;
  IRQ counts rise 11775 to 13106.
- The same cipher regression passes with `XRAY_EIP93=off`; IRQ counts stay
  at 13106.

An earlier cipher run was killed without an OOM log being collected; the full
rerun from `/opt` passed. A full OpenWrt package build with the target C toolchain,
Xray proxy interoperability, sustained-load stability and throughput remain to
be verified.

Confirm real Xray traffic with deterministic data hashes and compare throughput
and CPU use with `XRAY_EIP93=off`. Check the `1fb70000.crypto` interrupt line
before and after each run; an activation message alone is not hardware or
performance evidence. The `.go.in` templates are only added to the Go standard
library through the overlay, so they are not standalone Xray Go packages.
