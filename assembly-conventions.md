# FFmpeg Assembly Conventions

## Start from project exemplars

For SIMD work, study the closest current implementation before designing the
loop. For packed 10-bit video, `libavcodec/x86/v210dec.asm` is a particularly
useful exemplar. Also read Lessons 2 and 3 from
<https://github.com/FFmpeg/asm-lessons>; they were written by the developer
responsible for the v210 SIMD implementation and explain FFmpeg's preferred
loop and register idioms.

## Loop construction

- Prefer FFmpeg/x86inc idioms over conventional standalone-assembly style.
- Minimize loop scaffolding and instruction count.
- Consider the negative-width/output-offset pattern used by v210: convert the
  loop count to an offset and let the arithmetic that advances the loop also
  set FLAGS for the branch.
- Avoid a separate `cmp` when an existing `add`, `sub`, or `inc` can set the
  required flags.
- If an `int` argument becomes a 64-bit index, explicitly sign-extend it, or
  determine whether a `ptrdiff_t` interface is more appropriate.
- Benchmark plausible loop structures. Readability intuition alone does not
  establish which form is best on modern cores.

## Loads, stores, and overread

- Do not assume an exact-width load sequence is automatically fastest or most
  idiomatic. Check FFmpeg's padding guarantees and nearby assembly first.
- Conversely, do not rely on overread merely because buffers are often padded.
  Document the actual caller contract and validate guarded/exact buffers when
  exact access is part of the function contract.
- Use aligned loads/stores only when every caller and allocator establishes the
  required alignment. Otherwise use the project's unaligned idiom.
- For packed bitstreams, keep byte-stream order distinct from native-endian
  sample representation.

## AArch64 endianness

FFmpeg does not generally disable ARM SIMD on big-endian hosts. Most routines
are endian-neutral by construction, and the few explicit cases adapt tables or
instructions rather than withholding dispatch.

Repository audit on 2026-09-17:

- 67 AArch64 `.S` files and 73 32-bit ARM `.S` files were inspected by search.
- No established AArch64 initializer used `!HAVE_BIGENDIAN && have_neon(...)`.
- `libavcodec/arm/sbcdsp_init_arm.c` explicitly changes permutation tables for
  host endianness while retaining its optimized implementation.
- `libavcodec/aarch64/huffyuvdsp_neon.S` and
  `libavcodec/aarch64/vp9mc_16bpp_neon.S` use element-sized `.8h` loads and
  stores for native 16-bit values.

Important AArch64 distinction:

- `ld1 {vN.16b}` and `ld1 {vN.8b}` describe ordered byte elements. A byte has
  no internal endianness.
- `ld1/st1` with `.8h` or `.4h` describes native 16-bit elements and preserves
  their numeric value across host endianness.
- Whole-register `ldr/str` treats the register as a scalar-sized object. Mixing
  whole-register operations with byte-element loads and halfword arithmetic can
  silently assume little-endian register layout.

The bitpacked NEON decoder therefore loads packed input as byte elements,
loads arithmetic constants and stores output as halfword elements, and uses a
big-endian `rev32` after the exact four-byte scalar tail load. This preserves
the packed stream's byte order while producing native-endian `uint16_t` planes.

## x86 feature tiers

- SSSE3 is the practical baseline for the bitpacked byte-shuffle algorithm.
- AVX2 can process 16 pixels per main iteration and retain an internal
  eight-pixel tail.
- The AVX-512 implementation requires AVX-512ICL/VBMI because it uses `vpermb`.
  Baseline AVX-512 systems without VBMI should continue to use AVX2.
- x86-32 has fewer SIMD registers. Keep register pressure within the supported
  x86inc register set; the bitpacked AVX-512ICL implementation was reduced to
  seven SIMD registers by reusing an output register.
- Assemble new x86 code for ELF, Windows COFF, and Mach-O in both 32- and
  64-bit modes when the source is intended to support those configurations.

## Instruction transformations

Favor unconventional but correct transformations when they reduce instruction
count and match project practice. In the bitpacked decoder, multiplying selected
16-bit lanes by `{1, 4, 16, 64}` and applying a common right shift avoids a
larger collection of per-field shifts and masks.

Any such transformation needs a scalar reference, randomized comparisons, and
coverage of main-loop plus remainder combinations.
