# Joseon Duel

A small turn-based tactics game for the browser. Open `index.html` — no build step.

Every character has 8 HP and a per-character AP budget that follows the map size (3×3 → 3 AP, 4×4 → 4, 5×5 → 5). Each round the enemies plan first and **show their whole plan**, then you plan, then all actions resolve step by step, simultaneously.

| Map | You | Enemies |
|---|---|---|
| 3×3 | 1 character | Melee Chaser + Ranged Sniper |
| 4×4 | 2 characters | Melee Chaser + Hybrid Fighter + Ranged Sniper |
| 5×5 | 3 characters | 1 Sniper + 3–4 Chasers / Hybrids |

| Action | Cost | Effect |
|---|---|---|
| Move | 1 per square | Up to as far as your AP allows in one go, diagonals allowed |
| Melee | 1 | Hits an opponent in your own square |
| Ranged | 1 + distance (min 2) | Hits whoever is in a *different* square when it fires; shown as an arc |
| Guard | 1 | Blocks one hit on the step it is played; a block gives +1 AP next round |

- At most 4 units fit in one square.
- When melee strikes meet on a square they **clash**: a click minigame (left click ⚔️, right click 🛡️) decides who wins; the losing side is thrown clear.
- On the board, moves show as numbered ghost tokens, melee / guard / ranged as numbered badges beside the unit, and shots as thin arcs. Hover or click a character to see its full route; Shift toggles "all steps".

Design documents: [DESIGN.md](DESIGN.md) (English), [DESIGN.ko.md](DESIGN.ko.md) (한국어).

`snake.html` is an early throwaway prototype.
