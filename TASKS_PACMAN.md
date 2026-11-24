# Pac-Man project — TASKS

Reviewed: 2026-11-24 on **`main` @ `3792ffb`** (= `origin/main`, working tree clean).
All `file:line` = that commit. **(tested)** = reproduced headless (SDL dummy driver, fake clock, screenshots).
Each task = **Problem → Files → Suggested fix** (to discuss, not final).

> ⚠ This file and `en.subject.pdf` are **not in git**. This file was lost once already →
> `git add TASKS_PACMAN.md` (or move it to `docs/project_management/`, which the subject asks for anyway).

---

## 1. WHERE WE ARE

| Area | State |
|------|-------|
| Game starts, Menu → play → levels → WINNER / LOSER → name → Menu | ✅ (tested) |
| Lives / score / timer / level win / win+death same frame | ✅ (tested) |
| Spawn kill (eaten ghost respawns on Pac-Man) | ✅ fixed (guard ≥ 3 cells) |
| Instructions screen | ✅ starts, fits, BACK works, comma fixed — text polish left (step 6) |
| Cheat mode keys P A C M N W L | ✅ all work (tested) — 2 bugs: HUD overflow + invincible stays (step 1) |
| HUD | 🔴 cheat ON pushes **Score off the screen** (tested) (step 1) |
| Super-gum countdown | 🔴 missing (step 2) |
| `__main__.py` (no traceback, 1 arg) | 🔴 **pushed commented** on `origin/main` → traceback on 0 args / bad file (step 0) |
| Ghost AI | 🟡 works but ignores walls → BFS inside `Ghost` (step 3) |
| Colors | 🟡 mixed: dark menus, **white** HUD / Winner / Loser / Highscores (step 4) |
| Highscores | 🟡 crashes on bad file, reads file every frame (step 5) |
| Config loader | 🔴 crashes on reviewer edits (step 7) |
| Pause | ⏸ parked (section 4) |
| `pac-man.py`, packaging, README, docs, lint | 🔴 not started (step 9) |

---

## 2. ROADMAP

| Step | Task | Size | Files |
|------|------|------|-------|
| 0 | Restore `__main__.py` protection | 5 min | `__main__.py`, `game.py` |
| 1 | HUD: cheat block hides data + invincible bug | 30 min | `hud_menu.py`, `menu_widget.py`, `text_widget.py`, `game.py` |
| 2 | Super-gum countdown + blinking ghosts | 20 min | `game.py`, `hud_menu.py`, `ghost.py` |
| 3 | Ghost BFS **inside `Ghost`** | 30 min | `ghost.py` |
| 4 | Colors: one palette | 30 min | new `colors.py` + screens |
| 5 | Highscores safe | 40 min | `save_scores.py`, `highscores.py`, `input_widget.py`, `game.py` |
| 6 | Instructions text polish | 10 min | `instructions.py` |
| 7 | Config loader safe | 45 min | `load_config.py` |
| 8 | Small bugs + cleanup | 45 min | several |
| 9 | Delivery: `pac-man.py`, `config.json`, Makefile, packaging, lint, README, docs | 3 h | root |
| — | Pause polish | parked | section 4 |

**Before every push:** `make run` → Start → play → P (cheat) → ESC → Main menu → Instructions → BACK →
Highscores → ESC → Exit. **Also** `uv run -m src` (no arg) → must print usage, no traceback.

---

## 3. TASKS

### Step 0 — `__main__.py` protection pushed commented **(tested)**

**Problem:** `origin/main` has the argc check and the try/except commented:
```
$ uv run -m src             → IndexError: list index out of range   (traceback)
$ uv run -m src nofile.json → ConfigError traceback
```
Subject V.1 + III.1: "never a Python traceback", "if your program crashes … considered non-functional".
Also `print(f"ERR0R: {e}")` after `sys.exit(0)` → dead code with undefined `e`.
`Game.__init__` still calls `self.run()` (`game.py:52`) → `run()` called twice.

**Files:** `src/__main__.py:12-22`, `src/logic/game.py:52`

**Suggested fix:** restore the `c4169db` version (`git checkout c4169db -- src/__main__.py`), print errors to
`sys.stderr`, delete `game.py:52`. Debug with `make debug`, never by commenting the try.

---

### Step 1 — HUD: cheat block hides Score + invincible stays ON **(tested)**

**Problem 1 — overflow:** cheat ON adds 8 lines (`hud_menu.py:63-71`). `calculate_layout()` then gives
**every** element the HUD font = 10 % of HUD width (`menu_widget.py:17-20`, ~45 px) → the `update_size(14)`
on the cheat lines is overwritten. 16 lines × (45 px + 36 px gap) ≈ 1300 px > 1000 px window.
Measured: **Score top = -34 px (hidden)**, last cheat line bottom = 1034 px (cut). Screenshot: Score missing,
"L : Lose the game" missing.

**Problem 2 — add/remove logic:** `if self.cheat_mode != cheated` is true on ON **and** on OFF → turning OFF
appends 8 more lines (tested: 16 → 24 elements), then the `elif` pops 1 line per frame until clean
(≈ 16 frames). Works by luck, hard to follow.

**Problem 3 — invincible stays:** press P, A, P → cheat OFF but `GameState.invincible = True` (tested).
Player looks normal but can't die.

**Problem 4 — separators:** `"________________" * 30` = 480 underscores → a ~10 000 px wide text surface
× 4, color `(0, 0, 15)` on white.

**Files:** `src/graphic/menus/hud_menu.py`, `src/graphic/menus/menu_widget.py:17-20`,
`src/graphic/widgets/text_widget.py`, `src/logic/game.py:189` (P key)

**Suggested fix:**

a) `TextWidget` gets a `scale` (also fixes the title `font_size = 100` hack in `main_menu.py:13`):
```python
def __init__(self, text, color=(0, 0, 0), scale: float = 1.0) -> None:
    self.raw_text = text
    self.color = color
    self.scale = scale
    self.font = None
    self.surface = None
    self.rect = None

def update_size(self, new_size, fixed_width=None) -> None:
    self.font = pygame.font.Font(None, max(10, int(new_size * self.scale)))
    self.render_text()
```
Title: `TextWidget("PAC-MAN", YELLOW, scale=2.5)`. Cheat lines: `TextWidget(item, GREY, scale=0.5)`.

b) HUD font limited by height too (`menu_widget.py` HUD branch):
```python
menu_width, menu_height = self.surface.get_size()
dynamic_font_size = min(int(menu_width * 0.10), menu_height // 12)
self.gap = int(dynamic_font_size * 0.4)
```

c) HUD: rebuild the list from flags instead of append / pop:
```python
def build(self) -> None:
    """Choose which lines the HUD shows, then lay them out."""
    self.elements = [self.score_text, self.lives_text,
                     self.level_text, self.time_text]
    if self.power_left > 0:                       # step 2
        self.elements.append(self.power_text)
    if self.cheat_mode:
        self.elements.append(self.cheat_mode_txt)
        self.elements += self.cheat_lines         # built once in __init__, scale=0.5
    self.calculate_layout()

# in update_values
if cheated != self.cheat_mode:
    self.cheat_mode = cheated
    self.build()
```
Separators: drop them (gap is enough) or a tiny `LineWidget` drawing a 2 px line.

d) `game.py:189` P key: turning cheat OFF also turns invincible OFF:
```python
elif event.key == pygame.K_p:
    GameState.cheat_mode = not GameState.cheat_mode
    if not GameState.cheat_mode:
        GameState.invincible = False
```
Show `"INVINCIBLE"` in the HUD cheat block when on, so the reviewer sees it.

**To discuss:** `C` and `N` both = next level → free one key (idea: `F` freeze ghosts, suggested by the
subject VI.5; or `V` speed x2, also suggested).

---

### Step 2 — Super-gum countdown ("time left before ghosts are back to normal")

**Problem:** after a super pacgum, nothing shows how long the ghosts stay blue. Player can't plan.

**Files:** `src/logic/game.py:90-96` (HUD call), `src/graphic/menus/hud_menu.py`, `src/graphic/characters/ghost.py:41-58` (draw)

**Suggested fix:**

a) Time left = the latest `afraid_until` of the blue ghosts (already stored in each ghost):
```python
# game.py, before hud_menu.update_values(...)
now = pygame.time.get_ticks()
power_ms = max((g.afraid_until - now for g in self.ghosts if g.is_afraid),
               default=0)
power_s = (power_ms + 999) // 1000          # round up: 6.2 s → "7"
```
Pass `power_s` to `update_values`; HUD line `"Ghosts blue: 7 s"` in blue, shown only when `power_s > 0`
(rebuild with `build()` when it goes 0 → >0 or >0 → 0).

b) Classic warning: ghosts blink white in the last 2 s (`ghost.py` `draw`):
```python
color = self.color
if self.is_afraid:
    left = self.afraid_until - pygame.time.get_ticks()
    blink = left < 2000 and (left // 250) % 2 == 0
    color = (255, 255, 255) if blink else (33, 33, 255)
```

**Note:** while P3 pause timers are parked, this countdown keeps running during pause too (same clock).

---

### Step 3 — Ghost BFS inside the `Ghost` class **(code tested)**

**Problem:** `ghost.py:121` measures **straight-line** distance → walls ignored → dead ends, circles.
Pinky / Inky targets can be outside the maze or inside a closed `15` cell.

**Why inside `Ghost`:** only ghosts use it, `MazeEnv.maze` is already imported there, and
`self.pacman` gives the fallback target. 3 small methods + new `next_direction` replace l.82-129.
`ghosts_move.py` stays empty → delete it (step 8).

#### 3-1. Neighbors — one table drives everything

Each cell is a number, each bit = one wall:

| Move | Wall bit | Blocked if | Next cell |
|------|----------|-----------|-----------|
| UP | `1` | `cell & 1` | `(y - 1, x)` |
| RIGHT | `2` | `cell & 2` | `(y, x + 1)` |
| DOWN | `4` | `cell & 4` | `(y + 1, x)` |
| LEFT | `8` | `cell & 8` | `(y, x - 1)` |

`9 = 8 + 1` → walls LEFT + UP → open RIGHT, DOWN. `15` → closed everywhere ("42" logo).

#### 3-2. Example (3x3, checked with code)

```
      x=0 x=1 x=2
     +---+---+---+
y=0  | A   B | C |       A=9   B=7   C=11
     +   +---+   +
y=1  | D   E   F |       D=12  E=1   F=6
     +---+   +---+
y=2  | G   H |###|       G=13  H=6   I=15
     +---+---+---+
```
BFS from target **C**: step 0 C → 1 F → 2 E → 3 D, H → 4 A, G → 5 B. Grid (`-1` = unreachable):
```
 4  5  0
 3  2  1
 4  3 -1
```
Ghost on **A** wants **C**: RIGHT → B (5 steps), DOWN → D (3 steps) → **DOWN** ✅.
Straight-line would pick B (1 cell from C) = dead end ❌.

#### 3-3. Code — inside `class Ghost` (`ghost.py`)

```python
class Ghost(Character, ABC):
    # direction: (wall bit, dy, dx)
    MOVES = {
        Direction.UP: (1, -1, 0),
        Direction.RIGHT: (2, 0, 1),
        Direction.DOWN: (4, 1, 0),
        Direction.LEFT: (8, 0, -1),
    }

    def open_neighbors(self, y: int, x: int) -> list[tuple[int, int]]:
        """Cells reachable in one step from (y, x)."""
        maze = MazeEnv.maze
        result = []
        for bit, dy, dx in self.MOVES.values():
            ny, nx = y + dy, x + dx
            if maze[y][x] & bit:                 # wall on this side
                continue
            if 0 <= ny < len(maze) and 0 <= nx < len(maze[0]):
                result.append((ny, nx))
        return result

    def bfs_from(self, start: tuple[int, int]) -> list[list[int]]:
        """Steps from start to every cell (-1 = unreachable)."""
        maze = MazeEnv.maze
        steps = [[-1] * len(maze[0]) for _ in range(len(maze))]
        steps[start[0]][start[1]] = 0
        queue = [start]
        i = 0                                    # read position in the list
        while i < len(queue):
            y, x = queue[i]
            i += 1
            for ny, nx in self.open_neighbors(y, x):
                if steps[ny][nx] == -1:          # first visit = shortest path
                    steps[ny][nx] = steps[y][x] + 1
                    queue.append((ny, nx))
        return steps

    def safe_target(self, target: tuple[int, int]) -> tuple[int, int]:
        """Clamp target into the maze; closed cell -> Pac-Man's cell."""
        maze = MazeEnv.maze
        y = min(max(target[0], 0), len(maze) - 1)
        x = min(max(target[1], 0), len(maze[0]) - 1)
        if maze[y][x] == 15:
            return self.pacman.current_cell
        return (y, x)

    def next_direction(self) -> Direction:
        """Move with the fewest corridor steps to the target (BFS)."""
        self.update_mode()
        if self.is_afraid:
            target = self.pacman.current_cell
        elif self.is_scatter_mode:
            target = self.get_start_pos()
        else:
            target = self.define_target()
        steps = self.bfs_from(self.safe_target(target))

        opposite = self.get_opposite_direction(self.direction)
        y, x = self.current_cell
        best_dir, best_val = opposite, None      # dead end -> go back
        for d, (bit, dy, dx) in self.MOVES.items():
            if d == opposite or MazeEnv.maze[y][x] & bit:
                continue                         # no U-turn, no wall
            val = steps[y + dy][x + dx]
            if val == -1:
                val = 9999                       # unreachable = very far
            if self.is_afraid:
                val = -val                       # afraid: go FAR away
            if best_val is None or val < best_val:
                best_val, best_dir = val, d
        return best_dir
```
Add `assert MazeEnv.maze is not None` at the top of each method for mypy.

**Tested** (real `Blinky/Pinky/Inky/Clyde`, 30 mazes 20x20, 2 min limit, never on a `15` cell):

| Scenario | Now (straight-line) | This code (BFS) |
|---|---|---|
| Blinky alone, Pac-Man still | caught **18/30**, avg 13.9 s | **30/30**, avg 8.0 s, max 10.3 s |
| 4 ghosts, Pac-Man still | 29/30, **max 83.5 s** | 30/30, max 7.9 s |
| 4 ghosts, Pac-Man random walk | 30/30, avg 10.4 s | 30/30, avg 7.9 s |

Cost: one `bfs_from` = **0.34 ms** (20x20), **1.4 ms** (40x40), only when a ghost reaches a cell center.
Defense sentence: *"Each ghost picks a target cell (its personality), then takes the move with the fewest
steps through the corridors, computed by BFS."*

**To discuss later:** afraid = random direction (easier to catch); ghost speed + per level.

---

### Step 4 — Colors **(screenshots)**

**Problem:** two styles mixed.

| Screen | Now | Problem |
|--------|-----|---------|
| Main menu, Instructions, Pause | dark navy `(0,0,10)` / `(0,0,20)` | ✅ good |
| HUD | **white** bg, black text | bright white block next to the black maze |
| Winner, Loser, Highscores | **white** bg, black text | looks like a different game |
| Buttons | grey `(200,200,200)` + black text; hover **`(50,50,50)` + black text** | hover = black on dark grey → unreadable |
| Name input | black text, transparent box | invisible text once bg is dark |
| Walls | white | OK, classic is blue |
| Pacgum + super pacgum | white, same as walls | super pacgum looks like a wall dot |
| Afraid ghost | `(33,33,255)` | OK on black; would clash if walls turn blue |
| Cheat title / lines | magenta / grey `(150,150,150)` on white | grey on white = low contrast |
| Highscores | `"1 -       got"` | leading spaces saved in name (step 5) |

**Suggested palette** — new `src/graphic/colors.py`, used everywhere:
```python
BG = (0, 0, 20)             # every screen + HUD
TEXT = (230, 230, 230)
TITLE = (255, 220, 0)       # titles, PAC-MAN, labels
DIM = (150, 150, 150)       # hints, cheat lines (fine on dark)
CHEAT = (255, 0, 255)
WALL = (33, 33, 222)        # classic blue walls (or keep white)
PACGUM = (255, 184, 151)    # classic peach dots
AFRAID = (60, 60, 255)
BLINK = (255, 255, 255)
BUTTON = (200, 200, 200)
BUTTON_HOVER = (255, 220, 0)    # yellow bg, black text stays readable
```
Apply: `MenuWidget.bg_color = BG` default (fixes HUD, Winner, Loser, Highscores at once);
`TextWidget` default color `TEXT`; input text `TEXT`; button hover `BUTTON_HOVER`.

---

### Step 5 — Highscores safe **(tested)**

**Problem:**
- Bad file → crash (screen AND save): `{}`, `null`, `[1, 2]`, entry without `score`, `"score": "10"`.
- `{broken` → screen overwrites the file with `[]` (scores lost); save crashes.
- File read + fonts rebuilt **every frame** on Highscores screen (`game.py:71`) → laggy.
- Names with leading spaces / only spaces saved (screenshot: `"1 -       got"`).
- Config `highscore_filename` ignored; `highscores.json` committed with test names; no BACK button.
- Debug prints `input_widget.py:39-43`.

**Files:** `src/logic/save_scores.py`, `src/graphic/screens/highscores.py`, `src/graphic/widgets/input_widget.py`,
`src/logic/game.py:71`, `.gitignore`

**Suggested fix:** one `load_scores(filename)` that never crashes and never writes (try/except, keep only
`{"name": str, "score": int >= 0}`, sort, top 10), used by screen + save; refresh only on "Highscores"
click; `name.strip()` + ignore empty; title "TOP 10" + BACK; `.gitignore` += `highscores.json`.

---

### Step 6 — Instructions text polish **(screenshot)**

| Line | Now | Suggested |
|------|-----|-----------|
| `instructions.py:20` | `Arrow Keys < ^ >` | `Arrow keys` (down arrow missing) |
| `instructions.py:21` | `ESC / SPACE : Restart / pause` | `pause` — ESC does not restart |
| `instructions.py:28` | ghost respawn line commented | put back: `Eaten ghost back after {ghost_pwn_time} s` |
| all lines | `_____` fillers | font isn't monospace + lines centered → never align, looks noisy. Use `"Pacgum : +10 pts"` |
| title | `H0w T0 P1aY:` | `HOW TO PLAY` |
| cheat block | must match code (step 1 key change) | — |

---

### Step 7 — Config loader safe (subject V.3) **(tested)**

**Problem:** unknown key → crash; one wrong value → whole file refused; `{"width": 5, "height": 5}` → crash
(42 pacgums > 25 cells); no max size; comment lines removed → wrong error line number.

**Files:** `src/parser/load_config.py:13, 29-35, 44-58`

**Suggested fix:** `extra='ignore'` + message; check key by key → bad value = message + default; delete
`check_consistency`; clamp width/height 5..40; replace comment lines by empty lines.

---

### Step 8 — Small bugs + cleanup

| Problem | Files | Suggested fix |
|---------|-------|---------------|
| `minimum_size = (1500, 1000)` → window can't be smaller | `game.py:29` | `(800, 600)` |
| Resize re-lays out HUD / menu / pause only | `game.py` WINDOWRESIZED | `calculate_layout()` on every screen |
| Debug prints (`Game____O___OVER`, name prints) | `game.py:285+`, `input_widget.py:39-43` | delete |
| `elements[4]`, `elements[2]` magic indexes | `game.py:191,193,290,319` | refs `self.input`, `self.score_text` |
| `self.suface` typo (2nd copy of `self.surface`) | `hud_menu.py:13` | use `self.surface` |
| `random.seed()` resets global random | `game.py` `level_win` | `random.Random()` object |
| Dead lives fallback | `game_state.py:22-23` | delete |
| Dead files | `rules.py`, `screens/test.py`, `main.py`, `ghosts_move.py` | delete |
| Commented code, empty `start_evil` / `end_game` | several | delete |
| Root `setting.json` (`pacgum: 1`) + `src/setting.json` | root, src | one real `config.json` at root |

---

### Step 9 — Delivery (subject VII, VIII, IX)

- `pac-man.py` at root → `python3 pac-man.py config.json`.
- `config.json` at root with real values (`levels: 10`, `pacgum: 42`, `afraid_time`, `ghost_pwn_time`).
- `Makefile`: `debug` → `python3 -m pdb pac-man.py config.json`; `run` → `python3 pac-man.py config.json`;
  `lint` exact flags from III.2 (already OK).
- Packaging: build script/spec **at repo root** (e.g. PyInstaller `pacman.spec`), upload to itch.io
  (unlisted), in-package instructions (controls, config).
- `make lint` clean: flake8 + mypy, type hints + docstrings everywhere (III.1).
- README with all required sections (IX), `docs/project_management/` (VIII).

---

## 4. PARKED — Pause (later)

Works: SPACE / ESC pause, Resume / Main menu buttons, nothing moves while paused.

| Problem | Suggested fix (to discuss) |
|---------|----------------------------|
| No key leaves pause (ESC / SPACE ignored), mouse only | `if state == "PAUSE" and key in (ESC, SPACE): state = "PLAYING"` |
| ESC on WINNER / LOSER does nothing, no hint | ESC → MENU (skip save) + grey text "Type your name + Enter" |
| Ghost timers (afraid, respawn, scatter) + step 2 countdown keep running in pause | `ghost.shift_timers(paused_ms)` on resume |
| Pause screen opaque → game hidden | `SRCALPHA` surface + `(0, 0, 20, 160)` + redraw game under it |

---

## 5. SUBJECT CHECK — are we on track?

| Subject | Requirement | Us | Step |
|---------|-------------|----|------|
| III.1 | flake8 + mypy pass, type hints, docstrings | ❌ many errors / missing | 9 |
| III.1 | No crash (unhandled exception = non-functional) | ❌ `__main__` commented, highscores, config | 0, 5, 7 |
| III.2 | Makefile install / run / debug (pdb) / clean / lint | 🟡 `debug` uses `pudb` | 9 |
| III.3 | `.gitignore` | ✅ (+ add `highscores.json`) | 5 |
| IV | Graphic lib "MLX or similar" | 🟡 pygame-ce: only basic draw / fonts / events — keep it simple, be ready to justify | — |
| IV | Deploy on itch.io | ❌ | 9 |
| V.1 | `python3 pac-man.py config.json`, exactly 1 arg, no traceback | ❌ no `pac-man.py`, check commented | 0, 9 |
| V.2 | JSON + `#` comment lines | ✅ (`#` and `//` removed) | — |
| V.3 | Bad / missing values → default + message; unknown keys ignored | ❌ | 7 |
| V.4 | Use assigned generator as-is, PERFECT=False, failure handled | ✅ (`perfect=False`, try in `new_maze`) | — |
| V.5 | Highscores: robust file, name ≤ 10 alnum + spaces, top 10, shown in menu | 🟡 not robust, leading spaces | 5 |
| VI.1 | Level 1 fixed seed, next random | ✅ | — |
| VI.1 | Pacgums "in most corridors" | ⚠ **`pacgum` = count (42 of ~380 cells)** → maze mostly empty. Subject also suggests `pacgum: 42`. **To discuss:** read it as "fill most cells" (e.g. all free cells, `pacgum` = cells left empty) or keep count + say it in README | discuss |
| VI.1 | Super pacgums in 4 corners, 4 ghosts in corners, Pac-Man in the middle | ✅ | — |
| VI.2 | Arrows **or** WASD | ✅ arrows. ⚠ if WASD added later: **A and W clash with cheat keys** | — |
| VI.2 | 3 lives, respawn middle, game over at 0 | ✅ | — |
| VI.3 | Ghosts chase, flee when edible, respawn in corner after a while | ✅ (BFS = better chase, step 3) | 3 |
| VI.5 | Cheat mode that really helps review | ✅ P A C M N W L (HUD fix step 1) | 1 |
| VI.6 | Score never decreases | ✅ | — |
| VI.7 | ≥ 10 levels, time limit, keep score + lives, pause / resume | ✅ default `levels: 10` (setting files use test values) | 9 |
| VI.8 | Main menu (Start / Highscores / Instructions / Exit) | ✅ | — |
| VI.8 | HUD: score, lives, level, time always visible | ❌ cheat ON hides score | 1 |
| VI.8 | Pause menu Resume / Main menu | ✅ | — |
| VI.8 | Game over + victory screens: score + name prompt | ✅ (colors step 4) | 4 |
| VII | Packaging script/spec at repo root, itch.io build | ❌ | 9 |
| VIII | `docs/project_management/` (timeline, tracking, risks, team, tests) | ❌ (this file can start it) | 9 |
| IX | README: italic first line, Description, Instructions, Resources + AI use, Configuration, Highscore, Maze Generation, Implementation, Architecture, Project Management | ❌ empty | 9 |
| X | Defense: small live code change | — be able to explain every part (BFS, states, config) | — |

**Verdict:** gameplay is on track. Biggest risks for the grade: crash paths (steps 0, 5, 7),
HUD hidden data (step 1), and the non-code deliverables (step 9: README, docs, itch.io, lint).

---

## 6. EDGE CASES

| Case | Now | Step |
|------|-----|------|
| `uv run -m src` without arg / missing config | traceback | 0 |
| Cheat ON | Score + last cheat line off screen | 1 |
| P → A → P | invincible while cheat shows OFF | 1 |
| Pinky / Inky target outside maze or in "42" | ghost moves dumb (with BFS) | 3 |
| Button hover | black text on dark grey | 4 |
| Highscores file garbage / `{}` / read-only folder | crash | 5 |
| Name `"   got"` | saved with leading spaces | 5 |
| Config 5x5 / unknown key / wrong type | crash | 7 |
| Window < 1500x1000 | impossible | 8 |
| Pause + afraid ghosts | afraid ends during pause | parked |
| ESC on WINNER / LOSER | ignored | parked |

Checked OK (tested): every cheat key (M +1 life, N next level, W → WINNER, L → LOSER, A invincible);
win + ghost same frame → WINNER; spawn guard; timer 0 → 1 life; Start loads once; all maze cells
reachable; ghosts never cross walls; Instructions fit at 1500x1000; BACK returns to menu.

---

## DONE log

| Date | Commit | What |
|------|--------|------|
| 2026-11-23/24 | `7924927`, `3792ffb` | Cheat block in HUD (overflow bug, step 1); Instructions text tweaks |
| 2026-11-22 | `c2b3b84` | Cheat keys P A C M N W L (`cheat_ways`), `GameState.invincible`, collision uses `invincible`; BACK branch fixed; win + death same frame fixed (`else`) |
| 2026-11-22 | `e2c7f6f` | Instructions page (1 widget per line, comma, font `min()`), config passed; spawn-kill guard. ⚠ same commit commented `__main__` (step 0) |
| 2026-11-22 | `c4169db` | `__main__` argc + try (commented again later, step 0); title "PAC-MAN"; pause bg dark |
| 2026-11-22 | `3f06ae4` | Timer typo; pacgum check in PLAYING + `return` in `level_win`; Start loads once |
| 2026-11-22 | `1e724e6` | Lives reset on new game; keyboard in game; ESC pauses; hover fix |
| 2026-11-20 | `e051b69` | Pause menu |
| 2026-11-18 | `87241d6` / `84f203c` | Highscores screen reload; ghost-in-wall fix |
| ≤ `9f69bc7` | | Seeds, mazegenerator wheel, Winner/Loser + top-10 save, menu buttons |
