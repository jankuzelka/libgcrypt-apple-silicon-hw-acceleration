# Benchmark Comparison (Apple Silicon M2, 8GB, macOS 26.4.1)

## Libgcrypt 1.12.2 vs OpenSSL 3.6.2

| Algorithm     | Libgcrypt (Homebrew) | Libgcrypt (GnuPG build) | Libgcrypt (Patched build) | OpenSSL | Patched vs Homebrew | Patched vs GnuPG | Patched vs OpenSSL |
|---------------|----------------:|------------------:|--------------:|------------:|----------------:|------------:|---------------:|
| SHA1          | 724             | 1033              | 2549          | 2508        | 3.5×            | 2.5×        | +2%            |
| SHA256        | 352             | 352               | 2520          | 2540        | 7.2×            | 7.2×        | -1%            |
| SHA512        | 601             | 598               | 1505          | 1485        | 2.5×            | 2.5×        | +1%            |
| AES-128       | 345             | 343               | 17825         | 16177       | 51×             | 52×         | +10%           |
| AES-256       | 245             | 247               | 13329         | 11893       | 54×             | 54×         | +12%           |

| Algorithm | Libgcrypt (Homebrew) | Libgcrypt (GnuPG build) | Libgcrypt (Patched build) | Patched vs Homebrew | Patched vs GnuPG |
|---------------|----------------:|------------------:|--------------:|--------------------:|--------------------:|
| CRC32     | 1207                | 1208              | 32580         | 27×                 | 27×                 |

> All throughput values are in **MiB/s**.

---

## Notes

- OpenSSL values are taken from the **16384-byte column** and converted from “1000s of bytes/sec” to MiB/s.
- Libgcrypt values are taken directly from `bench-slope`.
- “Homebrew” and “GnuPG” builds represent **software fallback paths (no hardware acceleration)**.
- “Patched” build represents **ARMv8 Crypto Extensions enabled build**.
- AES values correspond to **ECB mode** for comparability with OpenSSL.

---

## Interpretation

- Hardware acceleration provides **massive gains for AES (~50×+)**, consistent with the ARMv8 AES accelerated path being active.
- SHA256 shows the most significant hash improvement (~7×), consistent with the ARMv8 SHA2 accelerated path being active.
- SHA1 and SHA512 show ~2.5× gains, consistent with expected hardware acceleration characteristics.
- In this test set, accelerated Libgcrypt is **on par with or slightly faster than OpenSSL** across the compared algorithms.
- The tested Homebrew and GnuPG builds perform similarly and represent **software fallback paths** in this comparison.

---

## Key Takeaways

- Libgcrypt on macOS can fully utilize ARMv8 Crypto Extensions with proper build adjustments.
- Performance parity with OpenSSL is achievable.
- The tested Homebrew and GnuPG builds did **not** use the available hardware-accelerated paths.
