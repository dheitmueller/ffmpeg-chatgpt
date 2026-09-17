# FFmpeg ChatGPT Engineering Notebook

This repository is a persistent engineering knowledge base for future FFmpeg
development. It records durable findings that are expensive to rediscover:
project conventions, useful exemplars, assembly and dispatch patterns,
benchmarking practices, validation workflows, reviewer feedback, design
decisions, and current work state.

It is not a mirror of the FFmpeg source tree and should not become a dump of
chat transcripts. Entries should be concise, evidence-based, and useful in a
future task with no access to the conversation that produced them.

## Instructions for ChatGPT

When Devin asks ChatGPT to consult or use this notebook for FFmpeg work, also
maintain it with durable new findings discovered during that work. Update or
remove stale conclusions when stronger evidence supersedes them. Keep temporary
implementation details in `SESSION_STATE.md`; consolidate lasting lessons into
the topic files.

Before preparing FFmpeg code for upstream submission:

1. Consult `personal-review-feedback.md` so reviewers do not need to repeat
   earlier feedback.
2. Find and study the closest current upstream implementation.
3. Run correctness and portability tests proportional to the changed paths.
4. Benchmark speed-critical changes reproducibly.
5. Inspect the final submitted diff after any rebase or history rewrite.

## Repository-operation authority

- Local Git operations are allowed when they are part of the requested task.
- Never push FFmpeg source branches, tags, or commits without Devin's explicit
  instruction for that push. Available credentials are not authorization.
- A request to maintain this notebook authorizes committing and pushing the
  notebook changes to `dheitmueller/ffmpeg-chatgpt`; it does not authorize
  changes to any FFmpeg source remote.
- Never create a pull request, post review comments, or respond on Devin's
  behalf unless he explicitly requests that external action.
- Generate patches mechanically from Git history or actual before/after files.
  Validate that a generated series applies to its intended base.

## Evidence rules

- Treat current FFmpeg source and documentation as authoritative.
- Give accepted upstream code and human maintainer feedback more weight than
  speculative review output.
- Record provenance for review-derived rules and distinguish verified behavior
  from modeling or static inspection.
- Do not report emulated, cross-assembled, or modeled results as native runtime
  results.
- Preserve exact commands, CPU model, compiler/assembler versions, and timing
  units for benchmark claims.

## Files

- `assembly-conventions.md` — SIMD/assembly structure, style, and portability.
- `architecture-and-dispatch.md` — DSP context and runtime dispatch patterns.
- `benchmarking-and-validation.md` — checkasm, FATE, benchmarking, and QEMU.
- `submission-and-review.md` — patch construction and upstream presentation.
- `personal-review-feedback.md` — cumulative feedback on Devin's FFmpeg work.
- `decisions.md` — important choices, rejected alternatives, and rationale.
- `SESSION_STATE.md` — current branch, PR, validation, and next actions.

## Maintenance

Prefer a few high-signal entries over exhaustive notes. A future contributor
should be able to understand what FFmpeg expects, why a decision was made, and
how to reproduce important results without reading old conversations.
