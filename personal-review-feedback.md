# Personal Upstream Review Feedback

This file records substantive feedback on Devin Heitmueller's FFmpeg
contributions. Its purpose is practical: an upstream reviewer should not need
to give Devin the same feedback twice.

Before preparing code for upstream submission, apply every relevant check
below. Preserve provenance and distinguish project-wide conventions from
context-specific advice.

## Current checklist

### Keep jointly loaded assembly constants in one block

**Provenance:** PR #24544 review, Martin Storsjö.

**Feedback:** Five separately declared 16-byte constants were always loaded
together. They should be one `const ... endconst` block; internal labels are
only needed if individual offsets are referenced.

**Classification:** confirmed AArch64 assembly style guidance.

**Future check:** Are adjacent constants consumed as one unit? If so, declare
them as one unit rather than relying on several sections remaining consecutive.

### Avoid unnecessary AArch64 NEON preprocessor guards

**Provenance:** PR #24544 review, Martin Storsjö.

**Feedback:** The `#if HAVE_NEON` guards around both the assembly declaration
and initializer body are not normally used on AArch64. `have_neon()` includes
`HAVE_NEON`, and AArch64 toolchains support NEON.

**Classification:** confirmed maintainer guidance.

**Future check:** Compare architecture initializer structure with neighboring
AArch64 initializers. Do not add redundant feature ifdefs around declarations
or code already handled by the feature predicate.

### Do not infer big-endian AArch64 requirements from existing assembly

**Provenance:** PR #24544 discussion, Martin Storsjö.

**Feedback:** FFmpeg explicitly does not care about big-endian AArch64 in its
existing AArch64 assembly. The bitpacked implementation may support it, but
review should not present that as a project-wide requirement.

**Classification:** confirmed maintainer scope guidance.

**Future check:** Separate optional portability improvements from configurations
the project actually promises to support. Do not generalize from an absence of
guards without maintainer or documentation evidence.

### Keep historical sponsorship attribution at its original scope

**Provenance:** PR #24544 review, Martin Storsjö; discussion with Devin.

**Feedback:** Duplicating `Development sponsored by CBC/Radio-Canada` into a new
internal header looked unusual. Once its origin was explained, Martin had no
strong objection, but suggested keeping it in the original source file may be
enough for the small moved function.

**Classification:** personal cleanup check; reviewer expressed no strong
requirement.

**Future check:** Preserve historical attribution in its original file. Copy it
to a new file only when the moved work and provenance make that scope accurate
and useful.

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
