# Submission and Review

## Patch structure

- Keep logically separate architecture implementations in separate commits
  when each is independently reviewable.
- Place shared refactoring before architecture implementations that require it.
- Keep fixups out of the submitted history.
- Rebase onto the intended upstream base and inspect the final diff after the
  rewrite.
- Generate patches with `git format-patch`; validate them by applying them to a
  clean worktree at the target base.

## Commit messages for performance changes

A useful SIMD commit message records:

- What the loop processes and its block size.
- CPU feature requirement and fallback behavior.
- Important buffer-access contracts, especially exact access versus overread.
- Correctness tests and number/range of seeds.
- Benchmark command, CPU, toolchain, timings, units, and speedup.
- Limitations, such as compiler auto-vectorization of the reference or lack of
  runtime access to a feature tier.

Do not claim performance is "identical" based on one processor or a noisy
measurement. State the observed values and scope.

## Scope discipline

- Avoid unrelated style changes. For example, do not change
  `struct BitpackedContext *` to a typedef spelling merely because a typedef is
  available.
- Do not move declarations between files unless the architecture split needs
  it.
- Preserve historical copyright and sponsorship text only on files whose
  history it actually describes. A newly created file gets a copyright header
  consistent with nearby files; moving one prototype does not transfer an old
  sponsorship statement to a new file.
- Include standard headers directly when the file uses their public types or
  functions, but verify whether a proposed include is genuinely necessary and
  compare with neighboring implementations.

## Review handling

- Treat a verified dispatch-to-incorrect-SIMD issue as blocking, even on a rare
  architecture configuration.
- Convert reviewer observations into regression coverage when possible.
- Distinguish independently reproduced results from review-draft claims.
- When a maintainer's feedback materially shapes a revision, thank them by name
  in the relevant patch description. This acknowledges their contribution and
  records for later reviewers that the submitted design already reflects
  informed upstream feedback.
- Preserve useful maintainer observations in the developer notebook, especially
  architecture-specific details that are not obvious from manuals or existing
  code. Record the original context, which CPU classes may be affected, the
  resulting implementation decision, and whether supporting evidence came from
  hardware measurements, analytical models, or maintainer experience.
- Update the existing pull request by rewriting its topic branch when requested;
  do not create replacement PRs that lose discussion history.
- Never push a rewritten source branch without explicit authorization. When
  authorized, fetch first and use an explicit `--force-with-lease` value.

## Current exemplar

FFmpeg PR #24544 adds bitpacked decoder SIMD for AArch64 and x86:
<https://code.ffmpeg.org/FFmpeg/FFmpeg/pulls/24544>

Its AArch64 commit documents reproducible native benchmarking and preserves the
C fallback on big-endian hosts; the x86 commits document feature-tier fallback
and cross-object-format assembly checks. The discarded big-endian assembly
variant remains a useful validation case in this notebook, not a submission
claim.
