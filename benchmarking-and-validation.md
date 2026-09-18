# Benchmarking and Validation

## checkasm correctness

For a new optimized function, checkasm should compare the C reference and each
runtime-selected implementation using identical randomized input. Compare every
output plane and verify that input is unchanged when the function is read-only.

For a loop with a 32-pixel main body and eight-/sixteen-pixel subpaths, exercise
both isolated and combined paths. The bitpacked width set is:

```c
{ 8, 16, 24, 32, 40, 48, 56, 3840 }
```

Widths 40, 48, and 56 matter because they combine a completed 32-pixel loop
with nonzero remainders; testing only 8/16/24/32 and a large benchmark width can
miss those transitions.

Submission-style correctness command:

```sh
./tests/checkasm/checkasm --test=bitpackeddec --repeat=300 93000
make fate-checkasm-bitpackeddec
```

Use fixed starting seeds in reports so failures can be reproduced.

## checkasm benchmarks

Use checkasm's built-in benchmark mode for reports comparable to upstream SIMD
submissions:

```sh
./tests/checkasm/checkasm --bench --test=bitpackeddec \
    --function=bitpacked_unpack_yuv422p10 --duration=1000000 95000
```

Report:

- CPU model.
- Compiler and assembler versions.
- Exact command and seed.
- Timing source/unit shown by checkasm.
- C and optimized values plus ratio.
- Whether the compiler auto-vectorized the C reference.

Run several paired/interleaved processes when comparing two assembly variants;
single runs are vulnerable to thermal and system-state noise. Treat small
differences conservatively.

## Bitpacked benchmark results

### AArch64

Apple M4 Mac mini, Apple clang 17.0.0, AArch64 `cntvct` timing:

```text
bitpacked_unpack_yuv422p10_c:       876.8 ns
bitpacked_unpack_yuv422p10_neon:    227.2 ns (3.86x)
```

Apple clang auto-vectorizes the C reference with NEON. This is therefore a
comparison against compiler-generated SIMD, not a purely scalar loop.

Earlier single-threaded decoding of 100 UHD frames measured roughly 1.22 ms
versus 2.17 ms user CPU per frame and 343.6 versus 319.5 frames/s wall-clock
throughput. Microbenchmark speedup should not be presented as equivalent to
end-to-end decoder speedup.

### x86

Intel Xeon E-2356G, GCC 8.5, NASM 2.15.03:

```text
bitpacked_unpack_yuv422p10_c:          13502.8
bitpacked_unpack_yuv422p10_ssse3:       3436.6 ( 3.93x)
bitpacked_unpack_yuv422p10_avx2:        2222.8 ( 6.07x)
bitpacked_unpack_yuv422p10_avx512icl:   1131.5 (11.93x)
```

Preserve the units printed by the tested checkasm build; do not relabel cycle or
TSC measurements as nanoseconds.

## Memory traffic interpretation

For every two output pixels, the unpacker reads five packed bytes and writes
four bytes of Y plus two bytes each of U and V: thirteen bytes of explicit
traffic. Scalar and SIMD implementations move essentially the same payload;
SIMD primarily reduces instruction/dependency overhead rather than memory
volume. Large end-to-end gains may therefore be limited once other decoder work
or memory/cache behavior dominates.

## Historical big-endian QEMU validation

The experimental endian-neutral bitpacked NEON implementation was validated on
2026-09-17 using a freestanding 64-bit MSB ELF under `qemu-system-aarch64`:

- QEMU `virt` machine with a Cortex-A72 CPU model.
- Execution at EL1 with `SCTLR_EL1.EE` and `E0E` enabled.
- FP/NEON enabled through `CPACR_EL1.FPEN`.
- The actual FFmpeg assembly object was linked into the harness.
- The scalar reference and test harness were compiled for big-endian AArch64.
- Every width from 8 through 256 in steps of eight, plus 3840, was tested with
  200 inputs per width: 6,600 comparisons total.
- The first input at each width was sequential bytes, reproducing the reviewer
  example; remaining inputs were deterministic pseudorandom data.
- Y/U/V planes, source preservation, and output canaries were checked.

Result:

```text
BITPACKED BIG-ENDIAN PASS
```

QEMU establishes functional behavior for the emulated architecture. It does
not establish performance on real big-endian ARM hardware. The experiment was
later removed from the submitted patch at maintainer request; the final dispatch
keeps big-endian AArch64 on the C implementation.

## Portability validation

- Cross-assemble x86 assembly in all supported object formats and word sizes.
- Compile architecture initializers in feature-disabled configurations so no
  unavailable symbol remains referenced.
- Run `git diff --check`, relevant FATE targets, and source/header checks before
  submission.
- After history rewriting, apply the generated patch series to the intended
  base in a clean worktree and compare resulting tree IDs.
