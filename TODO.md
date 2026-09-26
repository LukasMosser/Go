# TODO

## Current experiment: compact Jev input

- [x] Keep the complete legal point list plus `pass`.
- [x] Use coordinate names with null Choice descriptions instead of
      per-move tactical annotations.
- [x] Compress the state to board, captures, recent moves, komi, and pass
      count; put the column/row coordinate legend at the end.
- [x] Remove the misleading mid-game territory estimate and derived
      group-threat list.
- [x] Rerun the same three seeded autoplay games and record the result in
      DESIGN.md.

## Next steps

- [ ] Try a compact pass-specific warning while keeping coordinate move
      criteria null; Jev chose pass on 228 of 230 requests in this run.

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
