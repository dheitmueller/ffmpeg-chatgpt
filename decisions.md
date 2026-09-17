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

## AArch64 support on both endian modes

**Decision:** Use element-aware loads/stores and dispatch NEON on both little-
and big-endian AArch64.

**Rejected alternative:** Keep `!HAVE_BIGENDIAN` in the initializer.

**Rationale:** The required support is small, matches neighboring ARM assembly
practice, and passed QEMU execution. A guard was the correct temporary safety
fix before validation, but is unnecessary in the final implementation.

## Unconditional element-aware main path

**Decision:** Use `ld1/st1` element operations unconditionally for the main
AArch64 paths, with only the scalar four-byte tail reversal conditional on host
endianness.

**Rationale:** This expresses the real byte/halfword contract and resembles
other FFmpeg ARM routines. Exploratory paired timings suggested a possible small
M4 difference, but later one-second checkasm measured 225.1 ns and 3.82x versus
the auto-vectorized C reference. Avoid claiming either equivalence or regression
without broader measurements.

## AVX-512 feature level

**Decision:** Require AVX-512ICL/VBMI rather than baseline AVX-512.

**Rationale:** The compact implementation depends on `vpermb`. Adding a
different baseline AVX-512 implementation would require a separate design and
benchmark; AVX2 remains the fallback.

## No bitpacked encoder work

**Decision:** Decoder optimization is in scope; a bitpacked encoder is not.

**Rationale:** Devin has no current use for an encoder, and it should not expand
the submission scope.
