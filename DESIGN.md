# Joseon Duel — Design Document

한국어 버전: [DESIGN.ko.md](DESIGN.ko.md)

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

1. **Plan phase.** The enemies' plans are shown on the board (ghost tokens, firing lines, badges; see §5). You spend your action points (AP) on an ordered plan.
2. **Battle phase.** The plans resolve step by step. Each step: guards, then moves, then attacks, then damage, then knock-back.
3. **Next round** until all enemies are defeated (win) or you reach 0 HP (lose). A simultaneous double knock-out is a draw.

---

## 3. Rules (as implemented)

### 3.1 Board and units

| Item | Value |
|---|---|
| Map | 3×3, 4×4 or 5×5 (squares named A1…). Pick it in the Tuning menu; the page opens on 3×3. |
| HP | 8 for every character, player and enemy |
| AP per round | **Per character**, and set by map size: 3×3 → 3, 4×4 → 4, 5×5 → 5 (tunable). Nothing is shared across a team. |
| Damage per hit | 2 |

**Who fights on each map**

| Map | Your party | Enemies |
|---|---|---|
| 3×3 | 1 character ("You") | 2 — Melee Chaser + Ranged Sniper |
| 4×4 | 2 characters | 3 — Melee Chaser + Hybrid Fighter + Ranged Sniper |
| 5×5 | 3 characters | 4 or 5 — one Ranged Sniper plus 3–4 Melee Chasers / Hybrid Fighters (random mix, never all the same type) |

The line-up is drawn each game by `enemyStyles(N)`. Hybrid Fighters first appear on 4×4.

**Start positions.** Your party starts spread out along the **left edge**. The enemies start on **different random squares of the right side** (the right edge on 3×3, the two right-most columns on larger maps).

Units may share a square, but **at most 4 units (any mix of sides) fit in one square** (`CAP` in the script). There is no friendly fire.

- **Blocked moves.** Moves resolve together. If more units try to enter a square than it has room for, the ones already there stay, and the arrivals get in in a random order; the late ones are blocked and stay in the square they came from ("can't enter — it's full"). While planning, the move buttons show "full" on squares that are expected to be full at that step.
- **Knock-back** also avoids full squares; if no neighbouring square has room, the unit isn't pushed.
- Icons are a fixed, small size. Each square is a two-column grid, **allies on the left and enemies on the right** (real tokens first, then ghosts); a lone side is centred. Badges are a separate layer around the icons and never move them (§5).

### 3.2 Actions

| Action | AP | Effect |
|---|---|---|
| **Move** | 1 per square | Any reachable square in one go, as far as your remaining AP allows (diagonals count as 1 square); hover shows the path and cost, like ranged. |
| **Melee** | 1 | Hits an opponent in **your own square**. |
| **Ranged** | 1 + distance (min 2) | Aimed at a **square**; hits whoever stands there when it fires. Can never target your own square. |
| **Guard** | 1 | Blocks **one** hit landing on the same step it is played. Lasts only that step. A guard that really blocks a hit also earns the **guard charge** (below). |

**Guard charge.** A successful guard (it actually blocked a hit) gives that character **+1 AP in the next round only**. It is per character, applies to enemies too, never exceeds +1 and is **repeatable**: use the bonus, guard successfully again, and you have the extra AP again. Unused bonus AP is lost at the end of that round. The UI shows the bonus as a **gold dot** after the normal AP dots: a `+●` pops in on the card the moment the guard blocks, and the dot **blinks** during the next planning phase as a reminder to use it. A bonus round has one extra step. A clash cancels the strikes inside it, so a guard doesn't earn anything there. Buttons whose action the remaining AP can't pay for (Ranged under 2 AP, anything at 0 AP) are disabled.

**Long moves.** A move costs 1 AP per square travelled and covers up to the remaining AP in one action (resolved in a single step, like a ranged attack). Every reachable square shows its cost while you plan. If a knock-back moves a unit before its planned move, it covers as many squares as the move's cost allows and stops short. Enemies plan long moves too.

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

After a **clash**, only the **losing side** is thrown clear to adjacent squares (allies and enemies that both lost land on different squares; each side lands together 60% of the time per unit, `CLASH_SHARE_PCT`); the winners keep the square. Outside clashes, a unit that takes **melee damage** is pushed to an adjacent square. The landing square is **random** for everyone. A melee hit fully absorbed by a guard does not push. Ranged hits do not push.

The landing choice lives in one function (`pushDest`) so smarter AI logic can replace the randomness later. **(planned)**

### 3.5 Melee clash

On a square where player melees and enemy melees fire on the same step, the strikes **clash** instead of dealing damage. A clash takes priority over target choice. Ranged attacks never clash.

**The bigger side is split evenly over the smaller side, and each share is its own clash.**

| Melee on the square | Clashes |
|---|---|
| 1 vs 1 | one 1-vs-1 |
| 2 vs 2 | two 1-vs-1 |
| 1 vs 3 | one 1-vs-3 (the lone character is outnumbered — hard) |
| 2 vs 3 | a 1-vs-1 and a 1-vs-2 |
| 2 vs 1, 3 vs 1 | **one** clash: the allies attack the enemy together |
| 3 vs 2 | a 2-vs-1 and a 1-vs-1 |

The extra unit when it doesn't divide evenly goes to a random share. Every melee on the square belongs to some clash, so none are left to strike on their own. This rewards standing together and helping allies, and punishes a lone character who gets surrounded.

During a clash the battle pauses and a **click minigame** runs:

- Two crossed swords shake on the clash square, like a power struggle.
- A row of prompts appears. Each prompt is a ⚔️ (**left click**) or a 🛡️ (**right click**), with a mouse icon showing which button.
- You click the matching button for each prompt in order.
- **Two wrong clicks total = you lose the clash** (`CLASH_MAX_FAILS = 2`). Finishing the row with fewer than two misses = you win.
- **Win:** every ally's strike lands, so each enemy in that clash takes **one hit per ally** (3 allies = 3 hits = 6 damage) and is knocked back.
- **Lose:** each enemy in that clash hits one of the clashing player characters (chosen at random). The other allies are not hurt.
- **After a clash, only the losing side is thrown clear** to adjacent squares (all of them, hit or not); the winners keep the square. Allies and enemies that both lost (separate clashes on one square) land on **different** squares; each side tends to land together (`CLASH_SHARE_PCT`, 60% per unit) but not always. If no clean square has room, any adjacent square with room is used.
- Clashes in the same step play one after the other (the panel names who is clashing with whom). The human always does the clicking.

**Row length.** Every enemy has a clash range `{min, max}` (default 3–6) in the `UNITS` table. The row is the **sum** of the rolls of the enemies in that clash. The miss limit stays at two. Bosses or special enemies can use bigger ranges (e.g. 6–10).

Allies do not shorten the row; their advantage is the payoff (more hits on a win, only one ally hurt on a loss). Tune with the miss limit; a future timer will add pressure on top.

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
- **Several player characters:** each candidate plan is simulated against guessed plans for *every* player character. Enemies aim at the nearest character most of the time, sometimes at another one. A kill bonus applies per player character downed, and a bigger one for finishing off the whole party. In bigger fights the AI uses fewer samples per enemy so planning stays fast (under about 0.3 s total at 5×5).

### 4.2 Styles

| Style | Enemy | Behaviour |
|---|---|---|
| **Melee Chaser** | 👾 | Chases its chosen target. Actions: move, melee, guard only (no ranged). |
| **Ranged Sniper** | 👹 | Keeps its distance and shoots. When you close in, it prefers to run away, otherwise guards or melees. Closes in if you are out of its shooting reach. Never wants to end a round out of reach. |
| **Hybrid Fighter** | 🧟 | Does everything (move/melee/ranged/guard) but prefers melee. Flexible: melee, or back off and shoot, or guard; makes distance when hurt and cornered. |

Rules shared by all styles:

- **Lethal priority:** a plan that could finish you off gets a large bonus that beats any style preference. This prevents the player from winning by cornering enemies.
- **Repeated melee penalty:** each melee beyond the first at the same square costs score, so enemies don't swing several times at a square you will have left. (This was a recurring complaint at higher AP.)
- **Sniper can't shoot your square**, so it avoids ending on it.
- **Outnumbering (not a must):** a melee style (Chaser, Hybrid) gets a small score bonus (`GANG_BONUS`, 0.4 per extra enemy, up to 2) for striking the same square on the same step as another enemy's committed melee, and a few "join that strike" candidate plans are generated, so 2-vs-1 and 3-vs-1 clashes against you happen more often. In a 40-round test the share of rounds with such a gang-up rose from about 50% to about 78%.
- **Long moves:** the plan generators pick any square within the unit's remaining AP (cost = squares travelled), usually the one closest to the target half the time, so enemies close in with one action like a player can.
- **Guard charge in the AI:** each simulated run where a guard really blocks earns a score bonus (Chaser 0.5, Hybrid 0.6, Sniper 0.9), and 60 guard variants of the best-looking plans are tested so guards are chosen on purpose. A sniper with a player in its face prefers to move away first, then chooses guard or melee. Extra guards in one plan are penalised slightly (the charge caps at +1).

Known weaknesses: enemies never plan around clash outcomes beyond a coin flip, and push destinations are random, not tactical.

---

## 5. Interface

The game is **visual first**: there are no plan-list text panels and no per-step action text. Everything you need is on the board.

- **Layout:** three columns. **Your party on the left**, the **board in the middle** (the biggest square that fits; animation and text sizes scale with it) and the **enemies on the right**. The **action buttons sit in a bar at the bottom** (Move, Melee, Ranged, Guard, **👁 All steps**, Undo, Clear, **Fight!**; buttons the remaining AP can't pay for are disabled).
- **Unit cards** group each unit's **HP and AP together**: name, HP bar, HP number and AP dots. Filled dots are AP still unplanned; an enemy's dots show what its revealed plan spends. Click any card (or the character on the map) to pick it; the picked card is highlighted. On narrow screens the columns move above and below the board.
- **Board:** icons (real tokens first, then ghosts) sit on **fixed positions that depend only on how many allies and enemies are on the square**, never on badges: allies left of the centre, enemies right, each team a column whose spacing grows with the tallest badge column (so badges don't pile onto each other) but never beyond keeping the icons inside the square, centred on the square's true centre (a lone side is centred; 1 vs 1 is a pair; **2 vs 2 is an exact 2×2 grid**; 1 vs 3 and 3 vs 1 are a single icon facing a column of three). At most 4 + 4 = 8 icons, because the cap of 4 holds both now and at the end of the round. Badges are **overlays** around the icons and are allowed to overflow the square; they never move an icon. The square's name label (A1, B2…) is an overlay too. Plans are drawn like this:
  - **Move → ghost token** where the unit will end up, a faded copy of its icon with a dashed ring, plus a **`n👣` badge = the step it arrives**. No path is drawn. Only the **final destination** is shown all the time; **earlier stops** (fainter, numbered) appear for focused units.
  - **Ranged → a thin, dashed, translucent arc bending upward like a parabola** (a symbol for "shot through the air", not always an arrow; it also keeps the line from covering the character at the landing square; never heavy, even when brightened) from the **centre of the shooter's icon** (its real token, or the ghost for the stop it fires from; it starts just outside the icon) to the **centre of the icon of the opponent expected on the target square at that step** (or the square centre if nobody is expected). It ends in an arrowhead and a **crosshair ring centred on that icon**; landing squares get a faint red tint. The shot's step badge (`n🏹`) is **not** drawn on the line: it sits in the shooter's own **badge column** like every other action. A line **brightens** when its shooter, the line itself or its landing square is focused or hovered. Lines never block clicks. A stop that fires a shot always shows its ghost, so every line starts at a visible icon.
  - **Badge columns (move arrival `n👣`, melee `n⚔️`, guard `n🛡️`, ranged `n🏹`):** one badge style for everything (coloured pill, step number, then icon), attached **beside the unit's icon**: a column on the **left of allies** and on the **right of enemies**, stacked **top to bottom in turn order**. They sit by its ghost token, or by its real token when it acts before moving. A stop that carries an action always shows its ghost, even an earlier one. **Cap warning:** if more units are planned to arrive at a square on one step than it has room for (cap 4), those arriving ghosts get a **red dashed outline** with a glow (the arrivals compete for the free places in a random order, so any of them may be the one that is blocked).
  - **Focus and selection:** hovering a character (token, ghost or card) shows its full route while hovered. **Clicking** any character (yours or an enemy's, on its card, token or ghost) **picks** it: its route and highlight stay until you click it again or pick someone else, and only one character is picked at a time. Picking one of your characters also makes it the one you are ordering; picking an enemy leaves your ordering character as it was (it keeps a faint border), so you can inspect an enemy and still give orders. Nothing is picked automatically except your ordering character at the start of a round. The **👁 All steps** button (or the **Shift** key, a toggle) shows every intermediate ghost of every unit.
  - Hover a ranged target while planning to see the yellow track.
- **Below the first screen:** a collapsible **Battle log** and the **Rules** section.
- **Tuning** (collapsed at the top): AP, Your HP, Enemy HP, Damage, with Apply/Reset. Settings travel in the page address (e.g. `?ap=4&hp=8`). Permanent defaults live in the `VERSIONS` table in the script.

### Animations

- Tokens **slide** between squares on moves and knock-back.
- **Sword swing:** an SVG sword held out from the attacker, blade outward, sweeping across the target.
- **Ranged:** a bow turned toward the target and an arrow flying along the line.
- **Hit feedback:** a square blinks red **only** when the unit on it actually takes damage. Misses, clashes and blocked hits do not blink.
- **Guard success:** when a guard really blocks a hit, a **blue clash burst** pops partly over the unit's **🛡️, which shakes** (the red 💥 is for clashes).
- **Clash:** the board dims and the clashing square **zooms in** to the middle of the map, showing the clashing units and shaking crossed swords. The **click prompts float above the magnified square**, inside the battle map. The square then shrinks back and the clash ends in 💥.

> The UI has been spot-checked in the preview browser (layout, a double clash with 11–14 prompts). It has not had a full visual review, so expect rough edges.

---

## 6. Balance notes

- Everyone has **8 HP** at 2 damage per hit (four hits to defeat anyone). You are usually outnumbered (1 vs 2, 2 vs 3, 3 vs 4–5); enemy count and HP are the main knobs.
- **AP follows the map size:** 3×3 → 3 AP, 4×4 → 4 AP, 5×5 → 5 AP.
- Ranged reach is `AP − 1` squares (cost is 1 + distance), so reach grows with the map and a ranged shot can cover most of the board.
- Losing a **double clash** costs 4 HP in total (two hits, taken by whichever of your clashing characters the enemies pick); winning it costs each enemy 2 of their 8. It is a big gamble for the player. Tune with HP, damage and prompt ranges.
- Higher AP lengthens plans and made repeated enemy melee more common until the repeat penalty was added.

---

## 7. Roadmap

### Near term
- **Plan readability (built; user feedback: better, but needs more improvement).** Ghost tokens for moves, thin lines for ranged shots, focus/pin/toggle for the full route (§5). Still on the shelf: a **step scrubber** (a 1–N control showing where everyone stands after each step) and a **plan strip on each card**. Add them only if this isn't enough on 5×5.
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
- 4×4 and 5×5 are implemented (switch in the Tuning menu). Board sizes above 5 are not supported (square labels run A–E).

### Multiple player characters
Implemented first pass: click a unit card (or a character on the map) to select who you are ordering; the plan moves on to the next character when one runs out of AP; each character has its own colour. Enemies pick targets among the whole party.

Open questions:
- How a 9-action plan (3 characters × 3+ AP) stays readable on a crowded board.
- **Crowd cap (currently 4 per square).** It keeps squares readable, limits a lone character to at most 3 enemies in one clash, and makes occupying a square a way to block movement. The costs are blocked moves (a "failed" move can feel unfair), random tie-breaks when units race for the last spot, and it only matters at larger scales (on 3×3 with 3 units it never triggers). Without a cap, up to 8 units could stack on one square and the enemies could dogpile a single character. Open: should the cap scale with the map, or be per side (e.g. max 3 each)?
- Smarter enemy targeting (focus-fire the weakest character, protect their own, split up).

### Special enemies
- **Bosses** with bigger clash ranges (6–10).
- **Hidden-information enemies:** a boss may hide part or all of its plan while normal enemies stay fully revealed. This is the intended home for the hidden-information idea (see §10).

---

## 8. Technical notes

- Single file `index.html`, vanilla HTML/CSS/JS, no build step. `snake.html` is the earlier prototype.
- `simulateStep(state, plans, i, clashFn)` resolves one step and is shared by the AI look-ahead and the real battle. `simulate` loops it. The battle plays steps one at a time so a clash can be decided by the player's clicks: it first runs a preview of the step (moves, clash detection), plays the move animation, runs the clash minigame if needed, then runs the real step with the outcome.
- Constants and defaults: `VERSIONS` table (AP, HP, enemy HP, damage per board size), `ENCOUNTERS` / `enemyStyles(N)` (who fights on each map), `UNITS[k].clash` (prompt range), `CLASH_MAX_FAILS`, `CAP` (units per square), `CLASH_SHARE_PCT` (losers landing together), and for the AI `STYLE_AI` (self-HP weight and guard-charge value per style), `GUARD_VARIANTS`, `GANG_BONUS`.
- Plan display: `drawPlans()` redraws the ghosts, badge columns, icon positions, cap warnings and the SVG firing arcs from the current plans; `render()` rebuilds the squares and calls it. Focus state is `hoverU`, `picked`, `hoverCell`, `hoverLine` and `showAll`. `S[k]` carries `hp`, `pos`, `g` (guard up), `bonus` (guard-charge AP this round) and `earn` (charge earned this round).
- Hosting: GitHub Pages from `main`, root. Repository: `jkj1362/joseon-duel`.

---

## 9. Open design questions

1. **Guard strength.** Should guard block all hits in its step instead of one? Currently two simultaneous hits mean one still lands. The guard charge (+1 AP next round) is the current reward for a good read; other ideas (riposte damage, an extra allowed miss in a clash) are parked.
2. **Clash fairness.** Each player character now clashes separately with an equal share of the enemies. Open: should allies standing in the same square give any extra help (fewer prompts, a higher miss limit) beyond splitting the enemies?
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
| Allies shorten the clash row (-1 prompt per extra ally) | Tried, **removed** — unintuitive and needs extra explanation. Allies keep the bigger win payoff only. |
| Loss in a 3-vs-1 / 2-vs-1 clash hits every ally | Considered, **not chosen for now** — a lost clash hurts one random ally (rule A); to be re-tested if grouping up proves too strong. |
| Plan overlay with path arrows for every unit, ghost tokens and focus-only highlighting | Tried, **reverted** — far harder to read: all paths at once crossed each other and fading non-focused plans hid information needed for timing. Replaced by ghost tokens for moves, thin lines for ranged shots, numbered melee/guard badges and hover/click focus (§5). |
| Showing an enemy already moved (preview of its end position) instead of ghosts | Discussed, **not chosen** — mid-round positions matter because attacks resolve at their own step. |
| 4×4 board | Built, briefly closed, then **re-opened** with 2 vs 3 (and 5×5 with 3 vs 4–5). |
