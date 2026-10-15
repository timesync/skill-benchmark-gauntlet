---
name: skill-benchmark-gauntlet
description: Run a model through eval lanes (the benchmark rig) as serial three-run segments with per-segment emails to Kurt. Use whenever Kurt asks to run/sweep a model through any lanes (L2/L3/L5/L6/L7/L8/L9) — covers kickoff-plan emails, segment launches, watching, folding verdicts into MATRIX.md, and results-queue check-offs.
---

# Model eval gauntlet (serial segments, per-segment emails)

Standing cadence for this task class (12+ gauntlets): one gauntlet = the requested lanes, each lane run as a segment of **3 serial runs**; segments run **one at a time in Kurt's stated order**; an email goes to Kurt after **each segment** completes; when he asks for confirmation before starting, send a kickoff email listing the intended segments first.

*(2026-10-13, Kurt: this workspace copy supersedes the sandbox-image copy — edits land here.)*

## Steps

1. **Row check:** confirm the model's row exists in `skills/skill-benchmark-routing/runner/manifest.json`. For a new model: query the Requesty `/v1/models` endpoint for the exact id and `input_price`/`output_price` (per-token price × 1e6 = per-million), probe `chat/completions` once, then append a row mirroring an existing row's schema (id/provider/model/price/_note; effort/temperature overrides only where the vendor requires them). A missing row kills every sweep at the "row ok" check.
2. **Launch one segment** (chaining the email first if one is owed): exec background with `python3 ops/scripts/send-reply.py` '<verified-id>' <email-file> && { export REQUESTY_BASE_URL=... REQUESTY_API_KEY=...; for i in 1 2 3; do bash ops/eval-sweep.sh <row> --lanes <LANE>; done; echo "=== <ROW> <LANE> SEGMENT COMPLETE $(date -u +%Y-%m-%dT%H:%M:%SZ) ==="; } > /tmp/<row>-<lane>-segment.log 2>&1`. The `&&` makes a bad message-id fail visibly before the sweep spends tokens. **Never run two segments of the same gauntlet concurrently** — serial pacing is part of the standard; a duplicate orphaned chain once produced a double L6 trio and a matrix repair.
3. **Watch:** use `ops/eval-sweep-status.sh "<log-glob>" --verdicts` — one call reads chain state (DONE/RUNNING per log) AND per-run verdicts from the newest results dirs. **Never the session-registry snapshot** (it lags live sessions — see skill `verify-run-state` and `patterns/exec-session-registry-lag.md`). **Narrate only at milestones** (launch, each segment's completion, final verdict), never per poll — at most one state-check per turn. *(2026-10-13: twelve poll-narrations on one gauntlet taught this the hard way; the status script replaces the poll loop entirely.)*
4. **Fold idempotently:** read the verdicts from the queue; before inserting rows into `skills/skill-benchmark-routing/MATRIX.md`, grep for the run-id first — anchor-based inserts are not idempotent, and an aborted turn that already folded will otherwise duplicate the rows (verified twice, 2026-10-12). Insert the three run rows plus a dated verdict paragraph.
5. **Check off the queue entries** with done-notes that describe only actions that have actually happened — never write "email sent; next segment launched" before those are true (see `verify-run-state` step 5).
6. **Email the segment verdicts** (reply to Kurt's request message using a VERIFIED id — see skill `email-reply-full-message-id`), then launch the next segment per step 2. The final segment gets a closing email with the full model picture across all lanes.

## Ground rules

- **Non-determinism principle (Kurt, 2026-10-14):** LLMs are not deterministic; no benchmark result is a guarantee of a specific outcome on any given run. "Frozen" means consistent-enough-to-route (stable across sampled runs), never reproducible-by-guarantee. A single unexpected failure on a frozen row is within expected behavior and does not invalidate the segment; a pattern shift triggers a re-segment. Write all notes/verdicts in expectation language ("strong at X, expect it to hold most runs"), never guarantee language.
- Three runs per lane is the variance standard: identical failure sets across runs = "frozen" (a routing answer); case-level flapping = flag the model as unstable.
- Before a lane's FIRST sweep, check the lane directory for stray non-pack subdirectories — the pack loader dies on any subdir without prompt.md (see the stray-dir class in skill `benchmark-pack-review`).
- Effort-level rows: probe each level individually before running it; effort is a per-model, per-lane dial — never assume monotonic.
- **Coverage first (2026-10-13, Kurt):** before queueing any gauntlet, run `skills/skill-benchmark-routing/scripts/coverage.py` to see what's actually missing — never answer "what's missing" from memory or session context (the DeepSeek-coverage answer from memory was wrong on disk).
- **L4 runs** are NOT gauntlet sweeps: they go through `skills/skill-benchmark-routing/packs/L4/event-capture-daemon/grade/run-draft.py` (multi-call A→B→C + harness), one run at a time, with the wall-clock guard protocol (uniform 131k budgets; see SKILL.md of skill-benchmark-routing).
- **Judgment-pass items go to the backlog:** when folding a segment, queue any judgment/scoring questions into `model-evals/review-backlog.md` instead of leaving them in TODO notes (the data-quality gate of skill-benchmark-routing).
