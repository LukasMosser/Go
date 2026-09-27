<img width="790" height="764" alt="go" src="https://github.com/user-attachments/assets/135e689e-840d-4afd-97c7-015f69b51834" />

# Jev Go

A small 9x9 Go game where the White stones are played by
[Jev](https://www.typesafe.ai), TypeSafe AI's "System One" decision
model, when an API key is available. Without a key, White falls back to
a built-in local heuristic AI. You play Black. In autoplay mode, the
local heuristic drives Black against Jev's White (or against itself if
no key is set).

Jev is a general-purpose decision model, not a dedicated Go engine. The
current bot wraps Jev in a small Monte Carlo tree search (MCTS): Jev supplies
a move policy, a position value, and a pass judgment at each evaluated node.
The search is shallow and uses no random playouts. In the first paired test,
MCTS won 4/10 games against the previous Jev score-based policy and 2/10
against KataGo's human rank-5k profile. It won no games as Black in either
set, so this experiment is not yet an improvement. See the approach and
benchmark summary below, and [DESIGN.md](DESIGN.md) for the full experiment
history and results.

A short recap of the rules of Go, with links for learning more, is in
[GO_RULES.md](GO_RULES.md).

The game is also served from GitHub Pages:
**https://dagfinndybvig.github.io/Go/** — Jev needs the local proxy
server and an API key (see Running below). On Pages (or when opening
`jev-go.html` directly without a server) the game tries the TypeSafe
API directly with your browser key, but the API sends no CORS headers,
so the browser blocks the call. To play against Jev, run `node server.js`
locally. Without a key (on Pages, file://, or localhost without a key),
White is played by the local heuristic AI instead — the game still works,
just without Jev.

## Rules

Full Go rules on a 9x9 board: captures, suicide prevention, and simple ko.
Two consecutive passes end the game; area scoring (stones + surrounded
territory) with komi 5.5 for White. Territory is marked on the board at
game end.

## Controls

| Action | Input |
| --- | --- |
| Place a stone | Click an intersection |
| Pass | Pass button (two passes end the game) |
| Undo | Undo button (returns to your turn) |
| New game | New game button |
| Set Jev API key | `J` |
| Toggle Jev log panel | `L` |
| Toggle autoplay (Jev vs local AI) | `0` |

All of these are also visible as buttons above the board: **Autoplay:
off/on (0)**, **API key (J)**, and **Jev log (L)**.

## Running

**Without a server (local AI plays White):** open `jev-go.html` directly in
a browser. No build step, no external assets. Without an API key, White
is played by the local heuristic AI — the game works, just without Jev.

**With Jev AI:** the TypeSafe API does not send CORS headers, so
browser-to-API calls are blocked. A zero-dependency Node.js proxy server
is included. Run it locally:

```
node server.js
```

Then open **http://localhost:3000** in your browser. The server reads
`TYPESAFE_API_KEY` or `TYPESAFEAI_API_KEY` from its environment or the
game folder's `.env` file (check `GET /jevstatus`); you can also press
**J** in-game and paste a key from
[console.typesafe.ai](https://console.typesafe.ai). A browser key always
takes precedence. The key is stored in `localStorage`.

**Environment variable:**

```
# macOS / Linux
TYPESAFE_API_KEY=yourkey node server.js

# Windows (cmd.exe)
set TYPESAFE_API_KEY=yourkey && node server.js

# Windows (PowerShell)
$env:TYPESAFE_API_KEY="yourkey"; node server.js
```

For a local `.env` file, put either `TYPESAFE_API_KEY=yourkey` or
`TYPESAFEAI_API_KEY=yourkey` in the game folder. Hidden files are not
served by the local web server.

The HUD shows who is playing at all times:

- A yellow **matchup line** under the title with stone glyphs, e.g.
  `● You (Black)  vs  ○ Jev (White)`,
  `● You (Black)  vs  ○ Local AI (White)` (no key),
  `● Local AI (Black)  vs  ○ Jev (White)` (autoplay with key), or
  `● Local AI 1 (Black)  vs  ○ Local AI 2 (White)` (autoplay without key),
  naming the actual driver of each colour.
- A bordered **player combinations** panel listing the possible
  matchups and how to switch between them.
- The indicator in the bottom-right corner:

- **green WHITE: JEV** — Jev is active and choosing White's moves
- **red WHITE: LOCAL AI** — no API key set; the local heuristic is
  playing White. Press J to enter a key (Jev needs `node server.js` on
  localhost).

### Starting, stopping, restarting the server

**Start** — from the game folder:

```
cd C:\Users\dybvig\Arcade\Go
node server.js
```

It prints a banner, the game URL, and whether a server-side key was
found. The game is then at **http://localhost:3000**.

**Stop** — press `Ctrl+C` in the terminal running it. If it runs in the
background with no terminal, kill the process holding port 3000:

```
# Windows (cmd.exe / PowerShell)
netstat -ano | findstr :3000
taskkill /F /PID <pid>

# macOS / Linux
lsof -ti :3000 | xargs kill
```

**Restart** — stop it, then start it again. Two things worth knowing:

- Changes to `jev-go.html` do **not** need a restart — static files are
  read from disk on every request, so a browser refresh picks them up.
- Changes to `server.js` **do** need a restart.

**Port already in use** — if startup fails with
`Error: listen EADDRINUSE: address already in use :::3000`, a previous
instance is still running. Stop it with the commands above, then start
again.

**If the server stops mid-game** — Jev polls fail and White stops
moving (the HUD turns red and shows "NO KEY"). The game retries up to
3 times before showing an error. Once the server is running again, Jev
resumes automatically on White's next turn — no page reload needed, as
long as the server had a key when the page was loaded. If the page was
loaded while the server was down, either reload the page after starting
the server, or press `J` and enter a key.

## How it works

On every Jev turn, the game sends the current position and every legal
coordinate (up to 81 points plus `pass`) to the TypeSafe System One API
(`jev-latest`) through the local proxy. State text includes the compact
board (`O` black, `X` white, `.` empty), captures, pass count, last move,
komi recipient, legal actions, and a coordinate legend. Columns are
`A B C D E F G H J` (Go omits I); rows are numbered 1–9 from bottom to
top. Each Choice option is a coordinate or `pass` with a `null` description.

The browser performs MCTS around these model outputs:

1. It evaluates the root position with three typed questions: a `Choice`
   distribution over all legal actions, a `Score` for the current position,
   and a `Noul` probability that passing is sound.
2. It selects up to eight leaves using PUCT. Jev's Choice probabilities are
   the action priors. The Score rubric has nine ordered levels (0–8), from
   an expected area margin of at most −30 points to at least +30; the
   expected score is mapped to `[-1, 1]` for tree backup. The pass prior is
   multiplied by Jev's Noul probability and the action priors are normalized.
3. It evaluates leaves in batches of four, with each board labeled as its
   own `POSITION` and its own Choice, Score, and Noul questions. The search
   value changes sign at each turn. If two passes end the game, the leaf
   value instead comes from the rules engine's exact area score.
4. It plays the root action with the most visits (ties use mean value, then
   prior). The default is eight simulations, batches of four, and `c_puct`
   1.4. Jev evaluates leaves directly; the search does not use random
   rollouts.

The root request has this shape; later requests include up to four separately
labeled board positions and repeat the three questions for each one:

```json
{
  "model": "jev-latest",
  "state": "POSITION 0 — White (X) to play; ... board, komi, legal actions, coordinate legend ...",
  "questions": {
    "position_0_policy": {
      "type": "choice",
      "instructions": "Choose White’s strongest legal action for POSITION 0.",
      "criteria": { "A1": null, "B2": null, "pass": null }
    },
    "position_0_value": {
      "type": "score",
      "instructions": "Estimate White’s eventual area-score margin in POSITION 0.",
      "criteria": ["~−30 or worse", "~−20", "~−10", "~−3", "~0", "~+3", "~+10", "~+20", "+30 or better"]
    },
    "position_0_pass": {
      "type": "noul",
      "instructions": "Is passing now strategically sound for White?",
      "criteria": { "true": "The position is settled.", "false": "A useful move remains." }
    }
  }
}
```

For an empty board, Choice contains at most 82 actions (81 points and pass);
the request asks for three outputs per evaluated position rather than one
Score per move. The old candidate-scoring policy remains available in the
benchmark as `--bot scores` / `jev-scores`. Without an API key, White uses the
local heuristic.

### Replay

This historical candidate-scoring replay displays the board and the
per-point Jev score heat map side by side. It predates MCTS. Placed stones
are shown on the board and set their heat-map positions to zero; open points
show the score-plus-log-prior value with interpolation between
intersections. The pass probability is shown above the boards.

<video controls preload="metadata" width="100%">
  <source src="./jev-game-replay.mp4" type="video/mp4">
  Your browser does not support embedded video. [Open the MP4](jev-game-replay.mp4).
</video>

### MCTS benchmark

The selected soft-prior MCTS run played five paired seeds (10 games per
opponent), using eight simulations per move, batch size four, and
`jev-1.13.0`:

| Opponent | Jev W–D–L | Score rate | Elo Δ (approx. 95% range) | Mean margin | Jev wins as B/W |
| --- | ---: | ---: | ---: | ---: | ---: |
| Previous Jev candidate scorer | 4–0–6 | 40% | −70 (−278 to +137) | −9.5 | 0/5, 4/5 |
| KataGo human-SL `rank_5k` | 2–0–8 | 20% | −241 (−488 to +7) | −19.8 | 0/5, 2/5 |

Against the score-based baseline, Jev used 657 API calls (2,685,667 input
tokens; 985,745 output tokens). Against KataGo it used 660 calls (2,388,582
input tokens; 1,022,429 output tokens). Every game was color-swapped, but
MCTS won no game as Black; this strong color asymmetry and the small sample
make the result an initial diagnostic, not a stable rating. The search did
not improve on the old Jev policy in this run. Full paired results and
limitations are in [DESIGN.md](DESIGN.md#jev-guided-mcts-experiment); runner
options and KataGo setup are in [BENCHMARK.md](BENCHMARK.md).

I also tested a hard pass gate that removed `pass` from a node whenever
Noul was below 0.5. It reduced Jev's passes and produced longer games, but
scored worse: 3–7 (mean margin −18.3) against the score baseline and 0–10
(−50.1) against KataGo. The soft-prior version remains the default because
it performed better in these paired runs. Both variants lost every game as
Black, which remains an open issue.

The earlier ten-game candidate-scoring comparison is historical context: it
won 9/10 against the deliberately weak local greedy baseline, including one
tactical loss where Black captured 41 stones. That result is not directly
comparable to the stronger color-swapped anchors above.

### Benchmark against KataGo

On macOS, install KataGo and download its human-SL model once:

```sh
brew install katago
mkdir -p ~/.local/share/katago/models
curl -fL 'https://github.com/lightvector/KataGo/releases/download/v1.15.0/b18c384nbt-humanv0.bin.gz' \
  -o ~/.local/share/katago/models/b18c384nbt-humanv0.bin.gz
```

With `TYPESAFE_API_KEY` in the environment or the repository's `.env`, run five
color-swapped pairs (10 games) against KataGo's `rank_5k` human-SL profile:

```sh
node benchmark.js --bot mcts --mcts-simulations 8 --mcts-batch-size 4 \
  --pairs 5 --seed 1 --opponents katago-5k
```

The runner uses the installed Homebrew model/config, 9×9, 5.5 komi, Chinese
rules, one visit, and temperature 1. Results go to an ignored JSONL file and
include model hashes and per-game color/results. See [the benchmark guide](BENCHMARK.md)
for alternate opponents, path overrides, and rating caveats.

Press **L** in-game to watch the decisions live. In the browser console,
`window.jevLog()` returns the last 200 decisions and `window.jevClear()`
empties the log.

## Autoplay mode

Press **0** to toggle autoplay: Jev (White) plays against the local
heuristic AI (Black), with no human input. Each side moves on a ~700ms
cadence, and when the game ends the result appears in large red letters
across the board for a few seconds before a new game starts
automatically. The score line and game-over message name the AIs
instead of "you" — **Local AI** (Black) vs **Jev** (White) — so you
can watch Jev's best moves against the greedy heuristic's
captures-and-liberties play. Without an API key, autoplay is local AI vs
local AI — both sides use the heuristic.

Toggling autoplay off mid-game returns control: you play Black from
whatever position the board is in. Pass and Undo are disabled while
autoplay runs.

## Architecture

```
jev-go.html   — entire game (single file, no dependencies)
server.js     — local Node.js server + Jev CORS proxy (run: node server.js)
index.html    — redirect to jev-go.html, so GitHub Pages serves the game
```

The game logic (groups, liberties, captures, ko, scoring) is pure
functions over a 9x9 array; the Jev integration mirrors the pattern used
in [Fight](https://github.com/dagfinndybvig/Fight).
