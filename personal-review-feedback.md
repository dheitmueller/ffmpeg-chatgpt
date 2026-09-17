# Personal Upstream Review Feedback

This file records substantive feedback on Devin Heitmueller's FFmpeg
contributions. Its purpose is practical: an upstream reviewer should not need
to give Devin the same feedback twice.

Before preparing code for upstream submission, apply every relevant check
below. Preserve provenance and distinguish project-wide conventions from
context-specific advice.

## Current checklist

### Validate every enabled architecture configuration

**Provenance:** forgejo_fairy combined review of FFmpeg PR #24544, 2026-09.

**Feedback:** The first AArch64 initializer selected NEON on big-endian builds,
but the routine mixed byte-element and whole-register loads/stores and produced
corrupt native-endian `YUV422P10` output. Little-endian testing did not establish
big-endian correctness.

**Resolution:** The assembly was made endian-neutral using element-aware loads
and stores plus a big-endian tail reversal. It passed 6,600 scalar comparisons
under QEMU big-endian EL1 data mode.

**Classification:** confirmed correctness requirement.

**Future check:** For every architecture dispatch condition, list the build and
runtime configurations it enables. Has each been executed, or safely excluded
until it can be validated? For SIMD with multi-byte elements, inspect every
load/store for host-endian assumptions.

### Cover completed main loops followed by every remainder class

**Provenance:** forgejo_fairy combined review of FFmpeg PR #24544.

**Feedback:** Widths exercising the main loop and remainders separately did not
cover a completed 32-pixel iteration followed by nonzero remainders.

**Resolution:** Added widths 40, 48, and 56 to checkasm.

**Classification:** confirmed test-design lesson.

**Future check:** Derive test sizes from the loop state machine. Include zero
main loops, exactly one main loop, multiple main loops, each tail independently,
and main-loop-plus-tail combinations.

### Document internal buffer and table contracts

**Provenance:** forgejo_fairy/GLM review of FFmpeg PR #24544.

**Feedback:** The assembly function's width/source-size contract and the
requirement that five shuffle tables remain consecutive were implicit.

**Resolution:** Added concise comments at the function pointer and table block.

**Classification:** likely project convention for non-obvious assembly
dependencies.

**Future check:** Does a future caller or table editor need an unstated fact for
correctness? Document exact width, byte-count, alignment, adjacency, or overread
requirements at the narrowest useful location.

### Make benchmark claims reproducible

**Provenance:** review feedback supplied while preparing FFmpeg SIMD changes;
consistent with FFmpeg's patch-submission checklist.

**Feedback:** Performance claims need the CPU model, compiler/assembler,
benchmark command, and before/after values. A microbenchmark does not establish
an end-to-end decoder improvement or cross-processor identity.

**Classification:** confirmed submission practice.

**Future check:** Does every performance-sensitive commit provide enough detail
to rerun the measurement and understand exactly what the ratio represents?

### Avoid unnecessary style and provenance changes

**Provenance:** Devin's review of the bitpacked series.

**Feedback:** Do not replace explicit `struct` spellings with typedefs, move
prototypes, copy historical sponsorship notices, or add includes unless the
implementation requires those changes.

**Classification:** personal pre-submission check.

**Future check:** For every changed line outside the optimized routine and its
dispatch/tests, can the change be directly justified by the feature?

## Maintenance

Add future human or automated upstream feedback when it produces a durable
pre-submission check. If later maintainer guidance contradicts an entry, update
the rule and explain the stronger evidence.
