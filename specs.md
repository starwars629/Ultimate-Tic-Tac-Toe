# Ultimate Tic-Tac-Toe — Specifications

## Overview

A web-based implementation of Ultimate Tic-Tac-Toe (UTTT) supporting human vs. human, human vs. AI, and (optionally) AI vs. AI play modes. The AI opponent uses Monte Carlo Tree Search (MCTS) with the UCB1 selection policy.

---

## Game Rules

1. The board is a 3×3 grid of smaller 3×3 tic-tac-toe boards (81 cells total).
2. X always moves first and may start on any of the 81 cells.
3. The local cell position chosen by a player (0–8) dictates which macro-board the opponent must play in next.
4. If the target macro-board is already won or fully drawn, the opponent may choose any active (unfinished) macro-board.
5. A macro-board is won when a player achieves three in a row in that board; it is then closed to further play.
6. A macro-board with no winner and no remaining moves is a draw; it counts for neither player on the large board.
7. The game ends when one player wins the macro game (three macro-boards in a row), or no win is possible (draw).

---

## Functional Requirements

### Board & State

- **F1** — Represent state as: 9 local boards (each 9 cells: `null | 'X' | 'O'`), 9 macro-board statuses (`null | 'X' | 'O' | 'draw'`), current player (`'X' | 'O'`), and active macro-board constraint (`0–8 | null` for free choice).
- **F2** — Enforce legal-move rules: only allow moves in the constrained macro-board (or any active board if unconstrained).
- **F3** — Detect local board wins and draws after every move.
- **F4** — Detect macro game wins and draws after every macro-board status change.
- **F5** — Emit a clean copy-on-write game state (no mutation) for safe use by the MCTS engine.

### UI / UX

- **F6** — Render the 9×9 grid with clear visual separation between macro and local boards.
- **F7** — Highlight the currently active macro-board(s) so the human player always knows where they can move.
- **F8** — Mark won macro-boards with a large X or O overlay; dim or cross out drawn boards.
- **F9** — Show whose turn it is and, when the game ends, display the result (X wins / O wins / Draw).
- **F10** — Provide a "New Game" button to reset all state.
- **F11** — Provide mode selection: Human vs. Human, Human vs. AI (X or O side), with a difficulty slider controlling MCTS iteration count.
- **F12** — Disable cell clicks during AI thinking; optionally show a spinner or "Thinking…" indicator.

### AI — Monte Carlo Tree Search

- **F13** — Implement MCTS as a self-contained module, operating entirely on serializable game-state objects.
- **F14 (Selection)** — Descend the tree using UCB1:  
  `UCB1 = w/n + C * sqrt(ln(N) / n)`  
  where `C = sqrt(2)` (default), `w` = wins for this node's mover, `n` = node visits, `N` = parent visits.
- **F15 (Expansion)** — When a node is first visited and is non-terminal, add all legal children.
- **F16 (Rollout)** — From the expanded node, simulate to a terminal state by selecting uniformly random legal moves at every step.
- **F17 (Backpropagation)** — Propagate result back up the tree: `+1` for a win for the node's mover, `-1` for a loss, `0` for a draw, from the perspective of each ancestor's mover.
- **F18 (Move selection)** — After exhausting iterations, choose the root child with the highest `w/n` (exploitation only, no exploration bonus).
- **F19** — Run MCTS in a Web Worker to avoid blocking the main thread.
- **F20** — Configurable `maxIterations` setting exposed via UI slider (suggested range: 500–10 000).

---

## Non-Functional Requirements

- **NF1** — Single-file or minimal-file frontend; no backend required; deployable as a static page.
- **NF2** — No build step required for development (plain HTML + JS or a single-file React/JSX bundle via CDN is acceptable).
- **NF3** — AI move must complete in ≤ 3 seconds at the default iteration setting on a modern laptop.
- **NF4** — Responsive layout; playable on tablet and desktop screens (minimum 768 px wide).
- **NF5** — Accessible: keyboard navigation optional but board cells must have ARIA labels describing their position and state.

---

## Out of Scope

- Multiplayer over a network
- User accounts / persistence
- Mobile (< 768 px) optimization in v1
- Opening book or endgame tablebase for MCTS
