# TODO

## Current experiment: Jev multi-output move scoring

- [x] Keep the complete legal point list plus `pass` in a Choice output.
- [x] Add a same-rubric Score output for each legal move and pass.
- [x] Add a Noul pass gate and use Choice probabilities as a small policy
      prior when ranking the per-move Scores.
- [x] Send exact candidate deltas and benchmark three seeded autoplay
      games; record results and limitations in DESIGN.md.

## Next steps

- [ ] Repeat with more paired seeds using a retained benchmark runner so
      the random streams are exactly reproducible.
- [ ] Save position-level traces and terminal area margins before trying
      DSPy/ReAnchor calibration.

## Done (pushed)

- [x] Autoplay status line lists players Black-first (commit `d7d65c2`)
- [x] Canvas flipped to Go orientation: row 1 at bottom, matching Jev's text
      board (commit `4cbca62`)
- [x] `jevLastChoice` updated after heuristic fallback (commit `4cbca62`)
- [x] `heuristicPick` guard against empty/null moves (commit `c72ef84`)
- [x] Confidence floor lowered from 0.3 to 0.1 (commit `f588128`)
- [x] Removed all fallbacks: Jev always plays White, retries on error instead
      of substituting heuristic. No `fallbackToHeuristic`, no `whiteIsJev`,
      no `CONFIDENCE_FLOOR`. `labels()` simplified. Timeout 3s -> 10s.
      (commit `014223f`)
