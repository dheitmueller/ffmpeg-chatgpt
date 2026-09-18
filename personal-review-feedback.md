# Personal Upstream Review Feedback

This file records substantive feedback on Devin Heitmueller's FFmpeg
contributions. Its purpose is practical: an upstream reviewer should not need
to give Devin the same feedback twice.

Before preparing code for upstream submission, apply every relevant check
below. Preserve provenance and distinguish project-wide conventions from
context-specific advice.

## Current checklist

### Preserve experienced maintainers' tacit knowledge

**Provenance:** Standing workbook practice requested by Devin after Martin
Storsjö's AArch64 review of PR #24544.

**Feedback:** Long-time assembly maintainers often contribute microarchitecture
insights that are difficult to recover from instruction references or source
inspection alone. Do not leave those observations buried only in a pull-request
thread.

**Classification:** standing developer-notebook practice.

**Future check:** Capture reusable observations with attribution and context.
Separate measured results from analytical-model output and experienced judgment;
record all three when available. Note whether advice is universal, applies only
to some cores, or is a low-risk scheduling preference.

### Credit maintainers whose feedback shapes a patch

**Provenance:** Submission practice requested by Devin after PR #24544 review.

**Feedback:** When a maintainer takes the time to provide substantive feedback,
credit them in the relevant patch description after incorporating it. Besides
showing appreciation, this tells other reviewers that the revision reflects
guidance from someone with established project expertise.

**Classification:** standing personal submission practice.

**Future check:** Before finalizing a reroll, identify maintainers whose review
materially changed the implementation and include a concise acknowledgment in
the affected patch description.

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
existing AArch64 assembly. After the optional implementation was demonstrated,
Martin requested that the patch stop spending effort on it; Linux was also
removing big-endian AArch64 support.

**Classification:** confirmed maintainer scope guidance.

**Future check:** Separate optional portability improvements from configurations
the project actually promises to support. Prefer a correct scalar fallback when
a maintainer says an exotic SIMD configuration is outside project scope.

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

### Avoid chained post-index loads on in-order AArch64 cores

**Provenance:** PR #24544 review, Martin Storsjö.

**Feedback:** A second post-index load using a base updated by the first load
creates a serial address dependency. Prefer two loads from the original base
followed by one explicit pointer update. `ldp q0, q1` may be faster on some
cores, but is not universally preferable to `ld1`.

**Resolution:** Changed the 40-byte load to `ld1` without writeback, `ldr` at
offset 32, then one `add x0, x0, #40`. Native M4 timings were within noise.
LLVM scheduling models favored eliminating the dependency on A53/A55/A72 and
N2; `ld1` was safer across the examined cores than `ldp`.

**Future check:** Treat post-index addressing as both an instruction-count tool
and a dependency choice. Model in-order targets, not only the development CPU.

### Schedule independent SIMD operations between dependencies

**Provenance:** PR #24544 review, Martin Storsjö.

**Feedback:** Move the independent V `shl` after the Y `mul` operations so an
in-order core does not immediately consume the preceding `tbl` result.

**Resolution:** Applied to both the main macro and eight-pixel tail.

**Future check:** Even when an out-of-order desktop core hides latency, arrange
independent work between producer and consumer instructions where it costs
nothing.

### Explain non-obvious SIMD constants and test instruction alternatives

**Provenance:** PR #24544 review, Martin Storsjö.

**Feedback:** The `mul` transformation and table numbers were surprising. A
per-lane shift is more literal, although the reviewer explicitly left relative
throughput open for measurement.

**Resolution:** Added comments identifying the Y/U/V shuffle indices and
per-lane multipliers. Native M4 measurements did not show a meaningful `ushl`
advantage. LLVM models were mixed and predicted worse `ushl` reciprocal
throughput on A76, N1, and N2, so the multiply-plus-common-shift form remains;
its 16-bit overflow performs the necessary trim without another instruction.

**Future check:** Preserve compact, unconventional transforms when measured or
modeled performance supports them, but document the mathematical purpose next
to their constants.

### Validate every enabled architecture configuration

**Provenance:** forgejo_fairy combined review of FFmpeg PR #24544, 2026-09.

**Feedback:** The first AArch64 initializer selected NEON on big-endian builds,
but the routine mixed byte-element and whole-register loads/stores and produced
corrupt native-endian `YUV422P10` output. Little-endian testing did not establish
big-endian correctness.

**Resolution:** An endian-neutral version passed 6,600 scalar comparisons under
QEMU big-endian EL1 data mode. After maintainer clarification, the submitted
version instead restores `!HAVE_BIGENDIAN` dispatch and removes the extra
big-endian assembly handling, so big-endian hosts use the C implementation.

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
