# AGENTS.md

Notes for coding agents working in this repo. Read this before editing.

## What this is

A 9x9 Go game (`jev-go.html`, single file, no dependencies) whose White
moves use Jev-guided MCTS when an API key is available; without a key,
White uses the local heuristic. API errors do not trigger a heuristic
fallback. A local greedy heuristic drives Black in autoplay mode.
`server.js` is a zero-dependency Node proxy that makes Jev work locally.
`benchmark.js` runs paired, color-balanced rating matches against fixed
opponent anchors. See DESIGN.md for architecture and README.md for usage.

## Gotchas

### CRLF vs LF (the big one)

The working copy is LF, but any `git checkout` / `git rebase` / `git
stash pop` converts files to CRLF (Windows `core.autocrlf`). After that,
edit-tool `old_string` matching silently fails with "not found" because
the file now has `\r\n` while your string has `\n`.

Fix: normalize before editing after any git operation that touches files:

```
node -e "const fs=require('fs');for(const f of ['jev-go.html','README.md','DESIGN.md']){fs.writeFileSync(f,fs.readFileSync(f,'utf8').replace(/\r\n/g,'\n'));}"
```

Expect this after every rebase — it has happened repeatedly in this repo.

### Board indexing

The board is `board[y][x]` — row first. Test fixtures that assume
`board[x][y]` will pass syntax checks and fail mysteriously. Coordinates
in the UI and Jev state text are `A-J` columns (no I, Go convention) and
1-9 rows, mapped by `coordName(x, y) = COLS[x] + (y + 1)`.

### Testing headlessly

There is no test framework. Tests are throwaway Node scripts using
`vm.runInContext` over the extracted `<script>` block. Known traps:

- Top-level `const`/`let` in the script do **not** become sandbox
  properties (only `function` declarations do). Append an export shim:
  `script + '\n;globalThis.__x = { humanPlay, getBoard: () => board };'`
  (getters for anything reassigned, like `board`).
- The DOM stub does not parse HTML. Static markup (e.g. the modes panel
  text) is invisible to tests — only assert on what JS writes via
  `textContent`.
- Timing: `/jevstatus` resolves asynchronously (sleep ~50ms before
  asserting `Jev.isEnabled()`), and White's move fires after a 350ms
  `setTimeout` (sleep ~600ms+ after a Black move). Jev's fetch uses a
  10s `AbortController` timeout — polyfill `AbortController` in the vm
  context when testing `Jev.chooseMove`.
- The heuristic has random tie-breaking (`Math.random() * 2` in the
  score). Never assert a specific move choice — assert stone counts.
- Jev mocks: any probabilities work — the game plays the argmax over
  legal options (deterministic, no temperature sampling).
- Jev receives the complete legal move list. On a 9x9 board there are at
  most 81 legal points, plus `pass`. `Jev.chooseMove` is the retained
  candidate-scoring baseline; `Jev.chooseMctsMove` is the current game policy.
  MCTS sends one root position, then batches up to four separately labeled
  leaf positions. Each position asks for Choice, Score, and Noul outputs.
- `labels()` takes no parameters — White is always Jev.
- Delete test scripts when done; they are not committed.

### Shell quirks (Git Bash on Windows)

- The bash tool runs Git Bash (MINGW64), not cmd.exe/PowerShell. Windows
  paths fail: `cd C:\Users\...` errors with "No such file or directory".
  Use POSIX paths: `cd /c/Users/dybvig/Arcade/Go`.
- `$1`/`$2` inside double-quoted `node -e "..."` strings are expanded by
  bash to empty strings — regex replacements silently produce
  `getB()[][]`-style garbage. Use the edit tool for source changes, not
  shell one-liners.
- Unquoted URLs with parentheses (`Go_(game)`) are shell syntax errors —
  quote them.
- Do not print `TYPESAFE_API_KEY`; it is set in this environment.

### Server lifecycle

- `node server.js` serves on port 3000. A second instance fails with
  `EADDRINUSE` — check `netstat -ano | findstr :3000` and kill the
  holder (`taskkill /F /PID <pid>`) before starting.
- Static files are read per request: `jev-go.html` changes need no
  restart; `server.js` changes do.
- When testing the live API through the proxy, start the server with
  `tools.process.start` (background), not a foreground bash call — a
  foreground call blocks until timeout.

### GitHub Pages

- `index.html` is a redirect to `jev-go.html`. Without it, Pages renders
  README.md instead of the game. Do not delete it.
- Jev never runs on Pages (the API sends no CORS headers, so the
  browser blocks direct calls even with a browser key) — White does not
  move there. Don't "fix" this by pointing the browser at the API
  directly; CORS blocks it. The endpoint logic uses the proxy
  (`/jev`) only on `localhost`/`127.0.0.1`; everywhere else it goes
  direct to `https://api.typesafe.ai`, which the browser blocks. The
  on-screen text reflects this: the modes panel says Jev needs
  `node server.js` + API key, and error/status messages on non-localhost
  say to run the server locally.
- The user pushes from the web UI and other sessions concurrently.
  Expect push rejections; `git fetch` + `git rebase origin/main`, then
  push. Never force-push without asking.

### Jev integration invariants

- `lastMove` holds a full board snapshot (for the ko check), not a
  coordinate. `lastCoord` is the display/state-text coordinate. Keep
  both updated in `applyMove` and `doPass`.
- White is Jev when an API key is available (browser key or server
  key). Without a key, White falls back to the local heuristic — the
  game keeps playing. There is no fallback on low confidence or errors:
  `jevMove` retries up to 3 times (10s timeout per attempt); if all
  retries fail it shows an error message and does not play a move. The
  HUD (`setHud`), score line, and matchup line show "Jev" or "Local AI"
  for White depending on `Jev.isEnabled()`. `labels()` takes no
  parameters — it checks `Jev.isEnabled()` internally.
- `chooseMctsMove` is the production Jev policy. It evaluates the root and
  up to eight search leaves by default, batches four leaves per request,
  uses PUCT with `c_puct = 1.4`, and picks the most-visited root action.
  Keep its pass prior soft: multiply Jev's Choice probability by the Noul
  pass probability and renormalize. A hard Noul threshold was benchmarked
  and performed worse.
- MCTS Choice criteria contain every legal coordinate plus `pass`, each
  with a null description. Each labeled position carries the compact board,
  captures, pass count, previous move, komi owner, legal list, and coordinate
  legend. The nine-level Score is indexed 0–8 and maps to
  `clamp(score / 4 - 1, -1, 1)`; value backup flips perspective each turn.
  Two-pass leaves use the exact area scorer. `chooseMove` remains a separate
  candidate-score baseline with move annotations and the existing hard Noul
  pass threshold; don't conflate its prompt or gating behavior with MCTS.

- The benchmark runner keeps Jev player-relative: when Jev plays actual
  Black, the benchmark swaps board colors and capture counts before asking
  the same White-oriented prompt. It must also swap the komi owner in the
  prompt and terminal value. Keep color-swapped games paired by seed, and
  record the resolved model version and anchor name for every game.

- For the optional KataGo anchor, `genmove` advances KataGo's GTP board itself;
  only send GTP `play` commands for Jev's moves. Reset board size, komi, rules,
  and `rank_5k` parameters at each paired game. KataGo Chinese rules match this
  app's simple ko, area scoring, and suicide-illegal behavior.
