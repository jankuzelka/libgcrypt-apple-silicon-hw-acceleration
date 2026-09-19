# libgcrypt AArch64 hardware acceleration on macOS

Patch series for **libgcrypt 1.12.2** that enables its existing AArch64
hardware-accelerated crypto paths on Apple Silicon / macOS.

The upstream AArch64 implementation is optimized for ELF/GAS. On macOS the
library could build while the accelerated assembly paths remained disabled or
failed later because of Mach-O and Apple Clang assembler differences. This
patchset fixes those compatibility layers without replacing the cryptographic
algorithms themselves.

## Result

On an Apple M2 running macOS 26.4.1, the patched libgcrypt build:

- enables ARM NEON and ARMv8 Crypto Extensions during `configure`;
- builds the AArch64 assembly implementations for AES, SHA, GCM/GHASH,
  ChaCha20, Poly1305, CRC, Camellia, Twofish, SM3 and SM4;
- passes all **39 executed libgcrypt tests**; 2 large-data tests are skipped
  by the standard test run;
- reaches hardware-accelerated throughput comparable to OpenSSL for the tested
  algorithms.

Selected `bench-slope` results:

| Algorithm | Homebrew build | Patched build | Speedup |
|---|---:|---:|---:|
| SHA-256 | 352 MiB/s | 2520 MiB/s | 7.2x |
| AES-128 ECB | 345 MiB/s | 17825 MiB/s | 51x |
| AES-256 ECB | 245 MiB/s | 13329 MiB/s | 54x |
| CRC32 | 1207 MiB/s | 32580 MiB/s | 27x |

Full benchmark data and methodology are in
[`docs/benchmarks.md`](docs/benchmarks.md).

## What was broken

The work identified several independent incompatibilities between the existing
AArch64 code and the macOS toolchain:

1. `AC_LINK_IFELSE` assembly probes used ELF-style symbol names, so feature
   detection failed on Mach-O.
2. `.section .rodata` is not valid Mach-O assembly syntax.
3. The Apple data-pointer macro used unsuitable GOT relocations.
4. On Apple Clang AArch64, `;` starts a comment instead of separating
   instructions; multi-instruction CPP macros therefore silently lost all
   instructions after the first one.
5. CPP macro expansion cannot emit the newlines needed to solve that issue
   directly in the existing macros.
6. C-visible assembly symbols require Mach-O underscore handling.
7. Apple Clang rejects some CFA-tracking CFI directives in very large assembly
   functions.
8. Mach-O does not support the label-difference `add` immediates used by the
   CRC implementation.

The detailed investigation and implementation notes are in
[`docs/technical-report.md`](docs/technical-report.md).

## Patch series

The patches are intentionally kept as a small ordered series instead of a fork
of the complete libgcrypt repository:

1. **Configure detection** — fix Mach-O assembly labels in feature probes.
2. **Common AArch64 macros** — adapt sections, data addressing and
   multi-instruction macros to Apple assembly syntax.
3. **Build pipeline** — add the Darwin CPP -> awk -> assembler path and handle
   incompatible CFI metadata.
4. **Symbol naming** — apply a Mach-O-aware `FUNC_NAME` convention across the
   AArch64 assembly implementations.
5. **CRC relocation** — replace unsupported Mach-O label-difference fixups.

## Applying the patches

Start from a clean libgcrypt **1.12.2** source tree and apply the series in
order:

```bash
git am /path/to/patches/*.patch
```

If the source tree is not a Git checkout, the patches can also be applied with
`patch`, but `git am` preserves the original patch metadata and ordering.

Then use the normal libgcrypt build flow, for example:

```bash
autoreconf -fi
./configure
make -j$(sysctl -n hw.ncpu)
make check
```

## Validation

### Native validation

The final patchset was built and tested on Apple Silicon / macOS against
libgcrypt 1.12.2. All 39 executed tests passed; 2 large-data tests were
skipped by the standard test run.

Performance measurements use libgcrypt's own `bench-slope` utility. See
[`docs/benchmarks.md`](docs/benchmarks.md).

### Independent patch/cross-toolchain validation

The published five-patch series was also rechecked from a clean libgcrypt
1.12.2 source tree:

- all five patches apply cleanly in order;
- `git diff --check` succeeds;
- `autoreconf -fi` succeeds;
- all 15 affected AArch64 `.S` files preprocess and assemble successfully as
  **arm64 Mach-O** objects with Clang's Apple target;
- the central Apple-specific assumptions (`;` comment behavior, `.const`,
  `@PAGE/@PAGEOFF`) were reproduced independently.

A full native runtime test still requires an Apple Silicon host.

## Design trade-offs

The most visible compromise is the Darwin assembly preprocessing pipeline. The
existing libgcrypt sources make extensive use of multi-instruction CPP macros
with `;` separators. GAS treats `;` as a separator, while Apple Clang treats it
as a comment. Since CPP cannot emit replacement newlines from those macro
expansions, the Darwin build preprocesses the assembly and splits the
instructions before assembling it.

Some detailed CFA-tracking CFI directives are also suppressed on macOS because
Apple Clang rejects them in large generated functions. This reduces unwind
metadata quality for debugging/profiling but does not change the cryptographic
operation itself. See the technical report for the exact scope and rationale.

## Scope

This repository does **not** contain a fork of libgcrypt and does not implement
new cryptographic primitives. It is a compatibility patchset that enables the
AArch64 implementations already present upstream to build and run on Apple
Silicon macOS.

## Author

Jan Kuželka — [https://kuzelka.dev](https://kuzelka.dev)

## License

The patches modify libgcrypt source files and follow the licensing terms of the
corresponding upstream files. See [`LICENSE.md`](LICENSE.md), `COPYING`,
`COPYING.LIB`, and `LICENSES`.
