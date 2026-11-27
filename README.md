# Joseon Duel

A small turn-based tactics game for the browser. Open `index.html` — no build step.

You (8 HP) fight two enemies (8 HP each) on a 3×3 grid. Each round everyone plans 3 AP of actions (bigger maps give more AP); the enemies plan first and **show their whole plan**, then all actions resolve step by step.

| Action | Cost | Effect |
|---|---|---|
| Move | 1 | One square, diagonals allowed |
| Melee | 1 | Hits one opponent in your own square |
| Ranged | 1 + distance (min 2) | Hits one opponent in a *different* square; hover a target to see the track |
| Guard | 1 | Blocks one hit on the step it is played |

- Melee clashes (you and an enemy both meleeing the same square) cancel out.
- Enemy A is a **Melee Chaser**; Enemy B is a **Ranged Sniper**. Both go all-in on an attack that could finish you.

`snake.html` is an early throwaway prototype.
