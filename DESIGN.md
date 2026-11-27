# Joseon Duel — Design Document

Status: working prototype (single-file browser game, `index.html`). This document records what is implemented today, why the rules are the way they are, and what is planned. Items marked **(planned)** or **(open)** are not in the game yet.

---

## 1. Vision

A small, readable, turn-based tactics duel played on a tiny grid. Each round the enemies **commit and reveal** their plans first, then you plan a reaction with the same action budget, and then everything resolves **simultaneously**. The fun is in reading the enemy plan, predicting how it collides with yours, and winning the exchanges — plus a short hands-on **sword-clash minigame** when two melee strikes meet.

(The grid is 3×3 only because it began as a "tic-tac-toe-shaped" board. There is no tic-tac-toe in the rules.)

### Pillars

1. **Perfect information on the enemy side.** Enemy plans are fully shown, as in *Into the Breach*: enemies commit first, the player reacts. Uncertainty comes from simultaneous resolution and knock-back, not from hidden enemy actions.
2. **Simultaneous, step-based resolution.** Everyone's Nth action happens together on step N. No "my turn / your turn" inside a round.
3. **Position matters at small scale.** Few squares, many interactions: moving, sharing squares, ranged lines, melee clashes, knock-back.
4. **Enemies with distinct tactics.** Each enemy style behaves differently and visibly, not just with different numbers.
5. **Short, tactile moments.** Animations for moves, sword swings, arrows, and a click-based clash minigame.

---

## 2. Core loop

1. **Plan phase.** The enemies' plans are shown (action list + badges on the board). You spend your action points (AP) on an ordered plan.
2. **Battle phase.** The plans resolve step by step. Each step: guards, then moves, then attacks, then damage, then knock-back.
3. **Next round** until all enemies are defeated (win) or you reach 0 HP (lose). A simultaneous double knock-out is a draw.

---

## 3. Rules (as implemented)

### 3.1 Board and units

| Item | Value |
|---|---|
| Board | 3×3 (squares named A1–C3). Larger sizes exist in code but are closed for now (see §8). |
| Player ("You") | 8 HP |
| Enemy A — *Melee Chaser* | 8 HP |
| Enemy B — *Ranged Sniper* | 8 HP |
| AP per round | Depends on map size: 3×3 → 3, 4×4 → 4, 5×5 → 5 (tunable) |
| Damage per hit | 2 |

Start positions: you start on the **left edge, middle row**. The enemies start on **different random squares of the right edge**.

Everyone may share a square. There is no friendly fire.

### 3.2 Actions

| Action | AP | Effect |
|---|---|---|
| **Move** | 1 | One square in any direction, diagonals included. |
| **Melee** | 1 | Hits an opponent in **your own square**. |
| **Ranged** | 1 + distance (min 2) | Aimed at a **square**; hits whoever stands there when it fires. Can never target your own square. |
| **Guard** | 1 | Blocks **one** hit landing on the same step it is played. Lasts only that step. |

Distance is Chebyshev (diagonals count as 1). Ranged cost is measured from where you will be at that point of your plan. Hovering a ranged target highlights the track and shows `dist n`.

Details that were decided deliberately:

- **A shot is locked in at planning.** If someone steps onto the shooter's square afterwards, the shot still goes to its target square.
- **You can't shoot the unit sharing your square**, but you can always shoot a *different* square, even while sharing. To hit a unit in your square, melee it or step away and shoot.
- **One attack hits one unit.** If several enemies share the target square, you choose which one when planning (a prompt appears) — except when a melee clash applies (see below).
- **Guard blocks only one hit per step.** Two simultaneous hits: one still lands. **(open — see §9)**

### 3.3 Step resolution

Each unit's Nth planned action resolves on step N, all together. Within a step:

1. Guards go up.
2. Moves happen. (A planned move is aimed at a square. If a knock-back has left the unit too far from it, it takes one step toward it instead of teleporting.)
3. Melee clash check (§3.5).
4. Attacks resolve; damage is applied after all attacks, with guard blocking one hit per unit.
5. Knock-back (§3.4).
6. All guards drop (guard only covers the step it was played in).

AP is **not** time. An earlier experiment that treated AP as time was reverted because it was unintuitive.

### 3.4 Knock-back

A unit that takes **melee damage** is pushed to an adjacent square. The landing square is **random** for everyone. A melee hit fully absorbed by a guard does not push. Ranged hits do not push.

The landing choice lives in one function (`pushDest`) so smarter AI logic can replace the randomness later. **(planned)**

### 3.5 Melee clash

If you melee on the same step as an enemy who also melees in your square, the two strikes **clash** instead of dealing damage. A clash takes priority over target choice. Ranged attacks never clash.

During a clash the battle pauses and a **click minigame** runs:

- Two crossed swords shake on the clash square, like a power struggle.
- A row of prompts appears. Each prompt is a ⚔️ (**left click**) or a 🛡️ (**right click**), with a mouse icon showing which button.
- You click the matching button for each prompt in order.
- **Two wrong clicks total = you lose the clash** (`CLASH_MAX_FAILS = 2`). Finishing the row with fewer than two misses = you win.
- **Win:** you hit every clashing enemy for normal damage and knock them back.
- **Lose:** every clashing enemy hits you for normal damage and you are knocked back once.

**Row length.** Every enemy has a clash range `{min, max}` (default 3–6) in the `UNITS` table. Each clashing enemy rolls its own number of prompts and the row is the **sum**, so two clashing enemies make one longer row while the miss limit stays at two. Bosses or special enemies can use bigger ranges (e.g. 6–10).

Input is ignored for the first half second so the player can get ready. There is currently **no timer**. **(planned — see §7)**

### 3.6 Win / lose

Defeat both enemies to win. You lose at 0 HP. If both happen on the same step it is a draw.

---

## 4. Enemy AI

Enemies plan before you, so they cannot see your plan. Enemy A plans first; Enemy B sees A's committed plan.

### 4.1 Method

For each enemy the AI samples a few hundred **candidate plans** and scores each by **simulating it against many guessed player plans**, then picks randomly among plans within 0.35 of the best score (so it doesn't always do the same thing).

- **Candidate generators:** random plan, chase plan, snipe plan (shoot from here), kite plan (back away, then shoot), approach plan (close in just enough to get you into shooting reach), and coverage plans that deliberately cover each square.
- **Guessed player plans:** random, charging at the enemy, holding position, a player who reads the enemy plan (`smartPlan`), and a player who **steps away** each step (`evadePlan`).
- **Look-ahead for clashes** treats a clash as a coin flip.

### 4.2 Styles

| Style | Enemy | Behaviour |
|---|---|---|
| **Melee Chaser** | A 👾 | Prefers melee and chasing you onto your square. May shoot, but favours close range. |
| **Ranged Sniper** | B 👹 | Keeps its distance and shoots. When you close in, it prefers to run away, otherwise guards or melees. Closes in if you are out of its shooting reach. Never wants to end a round out of reach. |

Rules shared by both:

- **Lethal priority:** a plan that could finish you off gets a large bonus that beats any style preference. This prevents the player from winning by cornering enemies.
- **Repeated melee penalty:** each melee beyond the first at the same square costs score, so enemies don't swing several times at a square you will have left. (This was a recurring complaint at higher AP.)
- **Sniper can't shoot your square**, so it avoids ending on it.

Known weaknesses: enemies never plan around clash outcomes beyond a coin flip, and push destinations are random, not tactical.

---

## 5. Interface

The game is **visual first**: there are no plan-list text panels and no per-step action text. Everything you need is on the board.

- **Layout:** the board is the biggest square that fits the screen; animation and text sizes scale with it. The **action buttons sit in a bar at the bottom** (Move, Melee, Ranged, Guard, Undo, Clear, **Fight!**) with the AP pips above them.
- **Unit cards** along the top show each unit's HP; enemy style labels and descriptions.
- **Board:** units are tokens. **Numbered action badges** on squares show the plans — yours and the enemies' (colour-coded per enemy). Hover a ranged target to see the track and distance; valid squares show their AP cost.
- **Below the first screen:** a collapsible **Battle log** and the **Rules** section.
- **Tuning** (collapsed at the top): AP, Your HP, Enemy HP, Damage, with Apply/Reset. Settings travel in the page address (e.g. `?ap=4&hp=8`). Permanent defaults live in the `VERSIONS` table in the script.

### Animations

- Tokens **slide** between squares on moves and knock-back.
- **Sword swing:** an SVG sword held out from the attacker, blade outward, sweeping across the target.
- **Ranged:** a bow turned toward the target and an arrow flying along the line.
- **Hit feedback:** a square blinks red **only** when the unit on it actually takes damage. Misses, clashes and blocked hits do not blink.
- **Clash:** the board dims and the clashing square **zooms in** to the middle of the map, showing the clashing units and shaking crossed swords. The **click prompts float above the magnified square**, inside the battle map. The square then shrinks back and the clash ends in 💥.

> The UI has been spot-checked in the preview browser (layout, a double clash with 11–14 prompts). It has not had a full visual review, so expect rough edges.

---

## 6. Balance notes

- Everyone has **8 HP** at 2 damage per hit (four hits to defeat anyone). You still face two enemies, so you are outnumbered; enemy count and HP are the main knobs.
- **AP follows the map size:** 3×3 → 3 AP, 4×4 → 4 AP, 5×5 → 5 AP.
- Ranged reach is `AP − 1` squares (cost is 1 + distance), so reach grows with the map and a ranged shot can cover most of the board.
- Losing a **double clash** costs 4 of your 8 HP; winning it costs each enemy 2 of their 8. It is a big gamble for the player. Tune with HP, damage and prompt ranges.
- Higher AP lengthens plans and made repeated enemy melee more common until the repeat penalty was added.

---

## 7. Roadmap

### Near term
- **Clash pressure (planned):** a timer / click window so that not pressing in time counts as one failure.
- **Smarter push destinations** (and AI that plans around knock-back and clash outcomes).
- **Optional melee range (planned):** currently melee hits only your own square. Making melee reach adjacent squares would change clash, target choice and the AI, so it needs its own design pass.

### Scaling the battle
The long-term scale is up to **9 characters** on one map. Board size is chosen **per encounter by total unit count** (aiming for about 2.3–3.6 squares per unit):

| Total units | Board |
|---|---|
| 3–4 | 3×3 |
| 5–6 | 4×4 |
| 7–9 | **5×5 (maximum)** |

Party size (1, 2, or 3 player characters) is a separate encounter setting that follows progression, e.g. 1 vs 2 on 3×3, 2 vs 3 on 4×4, 3 vs 4 on 5×5.

Things that will need work for larger scale:
- Spawn layouts for each side (multiple columns).
- Enemy AI cost grows with units and squares; planning needs a lighter version.
- Readability of many tokens and badges per square.
- Ranged reach versus board size (raise AP or change the distance cost).
- The 4×4 version is still in the code but closed; 5×5 would be added the same way. Board sizes above 5 are not supported (square labels run A–E).

### Multiple player characters
Open questions:
- How the player selects which character they are ordering, and how a 9-action plan stays readable.
- **Which character's clicks** a clash uses, and whether simultaneous clashes play one after another.
- How enemies choose which player character to target.

### Special enemies
- **Bosses** with bigger clash ranges (6–10).
- **Hidden-information enemies:** a boss may hide part or all of its plan while normal enemies stay fully revealed. This is the intended home for the hidden-information idea (see §10).

---

## 8. Technical notes

- Single file `index.html`, vanilla HTML/CSS/JS, no build step. `snake.html` is the earlier prototype.
- `simulateStep(state, plans, i, clashFn)` resolves one step and is shared by the AI look-ahead and the real battle. `simulate` loops it. The battle plays steps one at a time so a clash can be decided by the player's clicks: it first runs a preview of the step (moves, clash detection), plays the move animation, runs the clash minigame if needed, then runs the real step with the outcome.
- Constants and defaults: `VERSIONS` table (AP, HP, enemy HP, damage per board size), `UNITS[k].clash` (prompt range), `CLASH_MAX_FAILS`.
- Hosting: GitHub Pages from `main`, root. Repository: `jkj1362/joseon-duel`.

---

## 9. Open design questions

1. **Guard strength.** Should guard block all hits in its step instead of one? Currently two simultaneous hits mean one still lands.
2. **Clash fairness.** Win-all / lose-all for multi-enemy clashes is strong; the alternatives were partial results (win knocks back only one) or fewer allowed misses per extra enemy.
3. **Ranged versus melee balance.** Ranged can't hit your own square, which makes melee the counter to a sniper in your face; does that hold at 4 AP and on larger boards?
4. **Difficulty curve** across party sizes and board tiers.

---

## 10. Decisions and experiments (history)

| Idea | Outcome |
|---|---|
| Treat AP as time (an action of cost 2 takes two ticks) | Tried, **reverted** — unintuitive. |
| Merge melee and ranged into one "attack" | Rejected; kept separate (melee 1 AP own square, ranged 2+ AP). |
| Show enemy **targets only** instead of full plans | Tried, **reverted** — hiding movement turns the game into guessing, and a sniper could run away indefinitely. With full reveal the player can see and cut off its retreat. |
| "Engaged units can't be shot at" | Tried and **removed** — shots are locked at planning; only "can't target your own square" remains. |
| Melee clash depends on chosen target | Tried, **reverted** — clash takes priority over target choice. |
| 4×4 board | Built and then **closed** for now; board scaling is planned per encounter (§7). |
