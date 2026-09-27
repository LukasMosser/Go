# 9x9 Jev rating benchmark

`benchmark.js` runs paired games from the normal empty-board start using the
same rules and Jev move policy as `jev-go.html`. Each pair reuses a seed for
two games and swaps which side Jev plays. The seed controls local-opponent
randomness; API outputs and the resolved Jev model are recorded because the
remote service may change independently.

## Opponent anchors

The initial pool is deliberately small and reproducible:

| CLI name | Anchor | Role |
| --- | --- | --- |
| `greedy` | Current in-game one-ply heuristic | Reference anchor, assigned 1000 by convention |
| `jev-scores` | Previous per-legal-move Jev candidate scorer | Direct baseline for MCTS; both variants use the live API model |
| `choice-only` | Compact Choice-only policy from commit `6322135` | Historical Jev baseline; uses the same live API model and is version-stamped per game |
| `noise25` | Current heuristic replaced by a random legal move on 25% of turns | Matchup sensitivity check |
| `noise50` | Current heuristic replaced by a random legal move on 50% of turns | Matchup sensitivity check |
| `random` | Uniform random legal move | Lower-bound sanity check |
| `katago-5k` | KataGo human-SL `rank_5k` profile, one visit and temperature 1 | External labeled anchor; profile is not a calibrated Elo |

Jev is always represented as `White` in the prompt. When it plays actual
Black, the runner swaps the board colors, captures, and candidate result
boards before asking Jev to move, then translates its coordinate back to the
real game. The game itself retains the normal Black-first turn order, area
scoring, simple ko, and 5.5 komi for actual White.

## Run

The runner reads `TYPESAFE_API_KEY` or `TYPESAFEAI_API_KEY` from the
environment or local `.env` without printing it. It requires Node.js with
global `fetch` support.

```sh
node benchmark.js --pairs 10
node benchmark.js --pairs 10 --opponents greedy,choice-only
node benchmark.js --pairs 20 --opponents greedy,noise25,noise50,random
node benchmark.js --pairs 10 --opponents katago-5k
node benchmark.js --bot mcts --mcts-simulations 8 --mcts-batch-size 4 \
  --pairs 5 --seed 1 --opponents jev-scores,katago-5k
```

The default bot is MCTS. Use `--bot scores` to select the older
candidate-scoring policy. By default, `--pairs 10` means 20 games against
`local-greedy`: ten Jev-Black
games and ten Jev-White games, paired by seed. Add opponents explicitly because
each Jev turn can require many API calls. Results print as the run proceeds
and are written to a unique `benchmark-results-*.jsonl` file (ignored by Git). Use
`--output PATH` to choose a different file. Each record includes the seed,
color, result, score margin, captures, passes, API calls, token usage, and
resolved model name. MCTS records also include visits, evaluated positions,
pass probability, and root value summaries.

### KataGo setup

Install KataGo and place its official human-SL model at
`~/.local/share/katago/models/b18c384nbt-humanv0.bin.gz`:

```sh
brew install katago
mkdir -p ~/.local/share/katago/models
curl -fL 'https://github.com/lightvector/KataGo/releases/download/v1.15.0/b18c384nbt-humanv0.bin.gz' \
  -o ~/.local/share/katago/models/b18c384nbt-humanv0.bin.gz
```

The runner discovers the Homebrew `b18c384nbt` main model and
`gtp_human5k_example.cfg`. Override paths with `--katago-bin`,
`--katago-model`, `--katago-human-model`, `--katago-config`, or the matching
`KATAGO_*` environment variables. For each game it selects `rank_5k`, one visit,
temperature 1, 9x9, 5.5 komi, and Chinese rules (simple ko, area scoring,
suicide illegal). Per-game JSONL records include both model SHA-256 hashes and
the resolved KataGo version. KataGo and the downloaded weights are local
dependencies and are not stored in this repository.

The initial MCTS benchmark used five paired seeds (10 games per opponent),
eight simulations per move, and batches of four. Against the previous Jev
candidate scorer, MCTS scored 4–6 (mean margin −9.5; Elo difference −70,
approximate 95% range −278 to +137). Against KataGo rank_5k it scored 2–8
(mean margin −19.8; Elo difference −241, range −488 to +7). MCTS won no games
as Black in either set. See README.md and DESIGN.md for the hard-pass-gate
ablation, full token counts, and interpretation. These are small-sample
head-to-head estimates, not calibrated Go ratings.

On this Mac the runner overrides `metalDeviceToUseThread0=100`, which selects
the Apple Neural Engine / CoreML backend. KataGo's packaged default GPU
backend could not create a Metal device in this environment.

## Interpreting the rating

For each opponent, the runner reports the score rate and its Elo difference
using `400 * log10(p / (1 - p))`, where `p` counts a win as 1, a draw as 0.5,
and a loss as 0. The 95% range is a Wilson interval transformed through the
same formula. With only a few games this interval will be wide. A perfect
record displays an infinite point estimate and a finite confidence range where
possible; it does not mean the strength is known to be infinite.

The number `1000` for `local-greedy` is an arbitrary local anchor. The direct
Elo differences against other opponents are separate head-to-head estimates;
they should not be averaged into one rating unless those opponents are also
calibrated against the same pool. These results are specific to this 9x9
ruleset and opponent set, not human or 19x19 Go ratings. The random/noisy
anchors are diagnostics, not independently rated Go engines. The KataGo
human-SL anchor is a model-labeled opponent, not a ground-truth Elo: KataGo
recommends one visit with full-temperature sampling to most closely imitate a
rank profile, and warns that the human model can have biases and pathologies.
See the [official human-SL guide](https://github.com/lightvector/KataGo/blob/master/docs/Analysis_Engine.md#human-sl-analysis-guide)
and [GTP rules reference](https://github.com/lightvector/KataGo/blob/master/docs/GTP_Extensions.md#kata-set-rules).
