# Ultimate Tic-Tac-Toe — Implementation Plan

## Architecture

Single-page application delivered as one HTML file. All logic lives in plain JavaScript modules inlined into the file. The AI runs in an inline Web Worker (constructed from a Blob URL) to keep the UI thread free during MCTS.

```
index.html
├── <style>          CSS — layout, theming, animations
├── <script id="worker-src" type="text/plain">
│   ├── game.js      Pure game logic (state, legal moves, win detection)
│   └── mcts.js      MCTS engine (UCT node, selection, expansion, rollout, backprop)
└── <script>         Main thread
    ├── render.js    DOM construction and updates
    ├── controller.js Game flow, click handling, mode management
    └── worker-bridge.js  Spawns worker, posts/receives messages
```

---

## Module Breakdown

### 1. `game.js` — Game Logic (shared by main thread and worker)

**Data shape**

```js
{
  boards: Array(9).fill(null).map(() => Array(9).fill(null)), // 'X' | 'O' | null
  macroStatus: Array(9).fill(null),  // 'X' | 'O' | 'draw' | null
  currentPlayer: 'X',
  activeBoard: null  // 0-8 or null (free choice)
}
```

**Functions**

| Function | Signature | Description |
|---|---|---|
| `getLegalMoves` | `(state) → [{board, cell}]` | All valid (board, cell) pairs |
| `applyMove` | `(state, board, cell) → newState` | Returns new state; no mutation |
| `checkLocalWin` | `(cells, player) → bool` | 3-in-a-row check on 9-cell array |
| `checkMacroWin` | `(macroStatus, player) → bool` | 3-in-a-row on macro statuses |
| `getWinner` | `(state) → 'X' \| 'O' \| 'draw' \| null` | Terminal state check |
| `isTerminal` | `(state) → bool` | Shorthand for `getWinner !== null` |

Win patterns (stored as constant): `[[0,1,2],[3,4,5],[6,7,8],[0,3,6],[1,4,7],[2,5,8],[0,4,8],[2,4,6]]`

---

### 2. `mcts.js` — Monte Carlo Tree Search Engine

**Node structure**

```js
{
  state,          // game state at this node
  move,           // {board, cell} that led here (null for root)
  parent,         // reference to parent node
  children: [],
  wins: 0,
  visits: 0,
  untriedMoves: [] // moves not yet expanded
}
```

**Algorithm loop** (runs for `maxIterations`)

```
1. SELECTION   — from root, repeatedly pick child maximizing UCB1 until
                 a node with untried moves or a terminal node is reached
2. EXPANSION   — if non-terminal and has untried moves, pick one at random,
                 create child node, add to tree
3. ROLLOUT     — from new node, pick random legal moves until terminal state
4. BACKPROP    — walk up to root; at each node: visits++
                 wins += (result favors node.state.currentPlayer ? 1 :
                          result is draw ? 0 : -1)
```

**UCB1 formula**

```js
const C = Math.SQRT2;
const ucb1 = (node) =>
  node.visits === 0
    ? Infinity
    : node.wins / node.visits + C * Math.sqrt(Math.log(node.parent.visits) / node.visits);
```

**Best-move selection** after iterations: child of root with max `wins/visits`.

**Worker message protocol**

```
→ POST  { type: 'search', state, maxIterations }
← POST  { type: 'result', move: {board, cell}, stats: {iterations, timeMs} }
```

---

### 3. `render.js` — DOM Rendering

- Build the 9 macro-cells at init time; each contains a 3×3 grid of 9 button elements (81 buttons total).
- Each button: `data-board`, `data-cell` attributes; ARIA label `"Board {b+1}, cell {c+1}"`.
- On state change, `render(state)` diffs only what changed:
  - Add `active` class to legal macro-boards; remove from others.
  - Set cell text content (`X` / `O` / empty); add `taken` class.
  - Overlay won macro-boards with a `<div class="macro-winner">X</div>` or `O`.
  - Dim drawn macro-boards.
- Status bar: current player, or end-game result, or "AI thinking…".

---

### 4. `controller.js` — Game Flow

```
init()
  → read mode + difficulty from controls
  → create initial state
  → render(state)
  → if AI goes first, triggerAI()

onCellClick(board, cell)
  → if not human's turn or game over: return
  → if move illegal: return
  → state = applyMove(state, board, cell)
  → render(state)
  → if game over: showResult(); return
  → if AI's turn: triggerAI()

triggerAI()
  → disable board, show spinner
  → workerBridge.search(state, maxIterations)
  → on result: state = applyMove(state, move)
              enable board, render(state)
              if game over: showResult()
```

---

## File & Delivery

| Item | Detail |
|---|---|
| Output | Single `index.html` file |
| Styling | CSS custom properties for theming; clean sans-serif font |
| Colors | Neutral background; blue highlight for active boards; red/blue for X/O |
| No dependencies | Everything inline; no CDN required |

---

## Implementation Phases

### Phase 1 — Game Engine
- Implement `game.js` functions
- Unit-test win detection, legal moves, `applyMove` in browser console
- Verify terminal-state detection covers all edge cases (free-choice rule, drawn macro-boards)

### Phase 2 — Basic UI (Human vs. Human)
- Build HTML skeleton and CSS grid layout
- Wire `render.js` and `controller.js`
- Play through several games manually; confirm all rule enforcement works

### Phase 3 — MCTS AI
- Implement `mcts.js` standalone; test with `console.log` on known positions
- Wrap in Web Worker; confirm async message passing works
- Add difficulty slider (iterations: 500 / 2000 / 5000 / 10000)
- Smoke-test AI in Human vs. AI mode

### Phase 4 — Polish
- Animate cell placement (CSS transition)
- Add "New Game" and mode-select controls
- Tune default iteration count for 1–2 s think time
- Accessibility pass (ARIA labels, keyboard focus)
- Test on tablet viewport

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| Single HTML file | Zero build tooling; easy to share and deploy |
| Web Worker for MCTS | Keeps UI responsive during AI computation |
| Copy-on-write state | Safe to pass to worker via `structuredClone`; no shared-memory bugs |
| UCB1 C = √2 | Theoretical optimum for game trees; can be tuned later |
| Random rollouts | Sufficient for UTTT; RAVE or heuristic-guided rollouts are a stretch goal |
| Iterations via slider | Easy to tune; exposes the exploration/speed tradeoff to the user |
