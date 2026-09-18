# Design Decisions

## Bitpacked decoder SIMD decomposition

**Decision:** Extract packed YUV422P10 unpacking into an internal function
pointer with a C default and architecture-specific initialization.

**Rationale:** This creates a small checkasm target, keeps runtime dispatch out
of the row loop, and lets AArch64 and x86 implementations share one contract.

## Exact-width source access

**Decision:** Keep exact-width loads in the submitted implementations.

**Rationale:** The function contract states that the source contains exactly
`width * 5 / 2` bytes. Wider legal-overread alternatives should only replace
this after confirming the decoder's padding guarantee and benchmarking the
change with guarded-buffer coverage.

## AArch64 endianness scope

**Decision:** Dispatch the NEON implementation only when `!HAVE_BIGENDIAN` and
retain the C fallback on big-endian AArch64.

**Rejected alternative:** Carry an endian-neutral assembly variant using
element-aware loads/stores and a big-endian tail reversal.

**Rationale:** The endian-neutral variant was functionally validated under
QEMU, but Martin Storsjö clarified that big-endian AArch64 is not a supported
target worth expanding this patch for; Linux was concurrently removing its
big-endian AArch64 support. The guard keeps the rare configuration correct
without adding assembly that upstream does not expect contributors to maintain
or benchmark. The little-endian path can retain the simpler `ldr`/`str` form.

## AArch64 load and bit-alignment forms

**Decision:** Load each 40-byte block with a non-writeback `ld1`, an independent
`ldr` at offset 32, and one explicit pointer `add`. Keep the per-lane
`mul {4,64}` followed by the common right shift.

**Rejected alternatives:** Two chained post-index loads; `ldp` for the first
32 bytes; and replacing the multiplies with per-lane `ushl` operations.

**Rationale:** Removing the address dependency helps modeled in-order cores and
does not materially change M4 performance. `ldp` was modeled worse on A55 and
slightly worse on A76/N1. Native timings did not establish an `ushl` win, while
models predict lower `ushl` throughput on several newer cores. The multiply
form also uses 16-bit overflow to trim unwanted bits without another operation;
comments now make that non-obvious transform explicit.

## AVX-512 feature level

**Decision:** Require AVX-512ICL/VBMI rather than baseline AVX-512.

**Rationale:** The compact implementation depends on `vpermb`. Adding a
different baseline AVX-512 implementation would require a separate design and
benchmark; AVX2 remains the fallback.

## No bitpacked encoder work

**Decision:** Decoder optimization is in scope; a bitpacked encoder is not.

**Rationale:** Devin has no current use for an encoder, and it should not expand
the submission scope.
