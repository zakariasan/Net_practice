# Pac-Man project — TASKS

Reviewed: 2026-11-24 on **`main` @ `42359b2`** ("fix : colors scale and graphical menu remove no working parts").
All `file:line` = that commit. **(tested)** = reproduced headless (SDL dummy driver, fake clock, screenshots).
Each task = **Problem → Files → Suggested fix** (to discuss, not final).

> ⚠ This file is **not in git** (lost once already) → `git add TASKS_PACMAN.md` or move it to
> `docs/project_management/` (the subject asks for that folder anyway).

---

## 1. WHERE WE ARE

| Area | State |
|------|-------|
| Menu → play → levels → WINNER / LOSER → name → Menu | ✅ (tested) |
| Lives, score, timer, level win, win + death same frame, spawn guard | ✅ (tested) |
| Cheat keys P A C M N W L, invincible resets with P OFF | ✅ (tested) |
| HUD with cheat ON | ✅ fits 800x600 → 1920x1080 (tested) · ⚠ overflows on ultra-wide / with the power line (step 2) |
| Ghost blinking in the last 2 s | ✅ done (`ghost.py:58-62`) |
| Super-gum countdown in HUD | 🟡 computed in `game.py:90-92`, HUD lines commented (step 2) |
| `colors.py` | 🟡 file created, **imported nowhere** (step 3) |
| Ghost BFS | 🟡 `MOVES` added in `ghost.py:8-13`, not used yet (step 1) |
| Dead files `main.py`, `test.py`, `rules.py`, `ghosts_move.py` | ✅ deleted |
| Window min size 800x600 | ✅ |
| `__main__.py` (no traceback, 1 arg) | 🔴 still commented (step 0) |
| Highscores robustness | 🔴 crashes on bad file (step 5) |
| Config loader robustness | 🔴 crashes on reviewer edits (step 6) |
| Pause polish | ⏸ parked (section 4) |
| Lint: flake8 **23**, mypy **61 errors** | 🔴 (step 7) |
| `pac-man.py`, packaging, README, docs | 🔴 (step 8) |

---

## 2. ROADMAP

| Step | Task | Size | Files |
|------|------|------|-------|
| 0 | Restore `__main__.py` protection | 5 min | `__main__.py`, `game.py` |
| 1 | Ghost BFS (simple, inside `Ghost`) | 20 min | `ghost.py` |
| 2 | Power countdown in HUD + HUD height limit | 20 min | `hud_menu.py`, `menu_widget.py`, `game.py` |
| 3 | Load `colors.py` everywhere | 30 min | 11 files (list below) |
| 4 | Instructions text polish | 10 min | `instructions.py` |
| 5 | Highscores safe | 40 min | `save_scores.py`, `highscores.py`, `input_widget.py`, `game.py` |
| 6 | Config loader safe | 45 min | `load_config.py` |
| 7 | Dead code + small bugs + lint | 1 h | several |
| 8 | Delivery: `pac-man.py`, `config.json`, Makefile, packaging, README, docs | 3 h | root |
| — | Pause polish | parked | section 4 |

**Before every push:** `make run` → Start → eat a super pacgum (countdown?) → P (cheat) → ESC →
Main menu → Instructions → BACK → Highscores → ESC → Exit. Also `uv run -m src` (no arg) → usage, no traceback.

---

## 3. TASKS

### Step 0 — `__main__.py` protection still commented **(tested)**

**Problem:** `uv run -m src` → `IndexError` traceback; missing file → `ConfigError` traceback.
`print(f"ERR0R: {e}")` after `sys.exit(0)` = dead code, `e` undefined (flake8 F821). Subject V.1 + III.1:
no traceback, a crash = non-functional. `Game.__init__` still calls `self.run()` (`game.py:52`) → runs twice.

**Files:** `src/__main__.py:12-22`, `src/logic/game.py:52`

**Suggested fix:** `git checkout c4169db -- src/__main__.py`, errors to `sys.stderr`, delete `game.py:52`.
Debug with `make debug`, don't comment the try.

---

### Step 1 — Ghost BFS: simple version that fits the current code **(tested)**

**Idea in 4 lines:**
1. Make a grid the size of the maze, **every cell = -1** ("not reached").
2. Target cell = **0**. Put it in a list.
3. Walk the list: for each cell, every open neighbor still at -1 gets **my value + 1** and goes at the end of the list.
4. At the end, each cell holds **how many steps** it is from the target through the corridors.
   The ghost looks at its open neighbors and takes the one with the **smallest number**.

#### Neighbors (uses the `MOVES` you already added, `ghost.py:8-13`)

| Direction | Wall bit | Blocked if | `dy, dx` |
|-----------|----------|-----------|----------|
| UP | `1` | `cell & 1` | `-1, 0` |
| RIGHT | `2` | `cell & 2` | `0, +1` |
| DOWN | `4` | `cell & 4` | `+1, 0` |
| LEFT | `8` | `cell & 8` | `0, -1` |

`9 = 8 + 1` → walls LEFT + UP → open RIGHT, DOWN. `15` → closed everywhere ("42" logo).

#### Example (3x3, checked with code)

```
      x=0 x=1 x=2
     +---+---+---+
y=0  | A   B | C |       A=9   B=7   C=11
     +   +---+   +
y=1  | D   E   F |       D=12  E=1   F=6
     +---+   +---+
y=2  | G   H |###|       G=13  H=6   I=15
     +---+---+---+

start (all -1, target C = 0)      end
 -1 -1  0                          4  5  0
 -1 -1 -1                          3  2  1
 -1 -1 -1                          4  3 -1   ← I is closed, stays -1
```
Ghost on **A** wants **C**: RIGHT → B = 5, DOWN → D = 3 → **DOWN** ✅ (straight-line picked B = dead end ❌).

#### Code — 1 new method in `class Ghost`

```python
    def bfs_steps(self, target: tuple[int, int]) -> list[list[int]]:
        """Steps from target to every cell through corridors.

        -1 means the cell was never reached (wall-closed).
        """
        assert MazeEnv.maze is not None
        maze = MazeEnv.maze
        rows, cols = len(maze), len(maze[0])
        steps = [[-1] * cols for _ in range(rows)]   # 1. whole maze = -1
        steps[target[0]][target[1]] = 0              # 2. target = 0
        queue = [target]
        for y, x in queue:                           # 3. list grows while we walk it
            for bit, dy, dx in MOVES.values():
                if maze[y][x] & bit:                 # wall on this side
                    continue
                ny, nx = y + dy, x + dx
                if (0 <= ny < rows and 0 <= nx < cols
                        and steps[ny][nx] == -1):
                    steps[ny][nx] = steps[y][x] + 1  # neighbor = me + 1
                    queue.append((ny, nx))
        return steps
```
(`for y, x in queue` while appending is on purpose: Python keeps reading the new items → no `deque` needed.)

#### Changes in `next_direction()` (`ghost.py:92-139`) — keep everything, change 2 places

**a) After choosing `target_y, target_x` (after l.105), add:**
```python
        # target must be inside the maze and on an open cell
        rows, cols = len(MazeEnv.maze), len(MazeEnv.maze[0])
        target_y = min(max(target_y, 0), rows - 1)    # Pinky/Inky can aim outside
        target_x = min(max(target_x, 0), cols - 1)
        if MazeEnv.maze[target_y][target_x] == 15:    # aimed inside "42"
            target_y, target_x = self.pacman.current_cell
        steps = self.bfs_steps((target_y, target_x))
```
**b) Replace l.131** `dist = (target_x - test_x)**2 + (target_y - test_y)**2` **by:**
```python
            dist = steps[test_y][test_x]
            if dist == -1:
                dist = 9999          # unreachable = very far
```
Everything else stays: `possible_directions`, no U-turn, `if self.is_afraid: dist = -dist` (afraid = pick the
**biggest** number = run away), `min_dist` loop. Also delete the commented block l.94-98.

**Tested** (this exact code, real `Blinky/Pinky/Inky/Clyde`, 30 mazes 20x20, 2 min, never on a `15` cell):

| Scenario | Now (straight-line) | BFS |
|---|---|---|
| Blinky alone, Pac-Man still | caught **18/30**, avg 13.9 s | **30/30**, avg 8.0 s, max 10.3 s |
| 4 ghosts, Pac-Man still | 29/30, **max 83.5 s** | 30/30, max 7.9 s |
| 4 ghosts, Pac-Man random walk | 30/30, avg 10.4 s | 30/30, avg 8.0 s |

Cost ≈ 0.3 ms per call on 20x20, only when a ghost reaches a cell center.
Also: `MOVES` block (`ghost.py:8-13`) → fix indentation (4 spaces) + **2 blank lines** before `class Ghost` (flake8 E302).
Defense sentence: *"Each ghost picks a target cell (its personality), BFS counts the steps from that target to
every cell, and the ghost moves to the neighbor with the fewest steps."*

---

### Step 2 — Super-gum countdown in the HUD **(tested)**

**Now:** `game.py:90-92` computes `power_left`, `hud_menu.py:63-68` is commented. Ghost blink ✅ already done.

**Problem 1 — rounding:** `power_left // 1000` rounds **down** → 6.9 s shows "6", the last second shows "0"
(and the line would disappear 1 s before ghosts are back).

| ms left | now `// 1000` | fix `(ms + 999) // 1000` |
|---|---|---|
| 6999 | 6 | 7 |
| 999 | 0 | 1 |
| 300 | 0 | 1 |

**Problem 2 — `elements.pop()`** in the commented code would remove the **last** element = a cheat line when
cheat is ON, not the power line. Use `remove(self.power_mode)`.

**Problem 3 — height:** HUD font = 10 % of HUD width only (`menu_widget.py:17-20`). Tested with cheat ON:
fits up to 1920x1080 (bottom 1034 / 1080), **overflows on 2560x1080**; one more line (power) → overflows
1920x1080 too.

**Files:** `src/logic/game.py:90-92` (also E501 103 chars), `src/graphic/menus/hud_menu.py:42, 63-68`,
`src/graphic/menus/menu_widget.py:17-20`

**Suggested fix:**
```python
# game.py
now = pygame.time.get_ticks()
power_ms = max((g.afraid_until - now for g in self.ghosts
                if g.is_afraid), default=0)
power_left = (power_ms + 999) // 1000
```
```python
# hud_menu.py update_values
if power_left > 0:
    self.power_mode.update_text(f"Ghosts blue: {power_left} s")
    if self.power_mode not in self.elements:
        self.elements.insert(8, self.power_mode)   # after Time + its separator
        self.calculate_layout()
elif self.power_mode in self.elements:
    self.elements.remove(self.power_mode)
    self.calculate_layout()
```
Color: `AFRAID` from `colors.py` (same blue as the ghosts → player links them). Optional: white when ≤ 2 s
(same moment the ghosts blink).
```python
# menu_widget.py HUD branch — limit by height too
menu_width, menu_height = self.surface.get_size()
dynamic_font_size = min(int(menu_width * 0.10), menu_height // 14)
self.gap = int(dynamic_font_size * 0.8)
```
Note: pause (parked) doesn't stop this countdown — same clock as the ghosts.

---

### Step 3 — Load `colors.py` everywhere

**Problem:** `src/graphic/colors.py` exists but **no file imports it**. Colors are still hard-coded in 11 files;
Winner / Loser / Highscores / HUD are still **white**; button hover = black text on dark grey (unreadable).

**How to load:** `from ..colors import BG, TEXT, TITLE` (from `menus/`, `screens/`, `widgets/`,
`characters/`, `pacgums/`) or `from .colors import WALL` (from `graphic/maze_graphic.py`).

**Where to change:**

| File:line | Now | Use |
|-----------|-----|-----|
| `menus/menu_widget.py:12` | `bg_color = (255, 255, 255)` | `BG` → fixes HUD, Winner, Loser, Highscores at once |
| `menus/main_menu.py:9` | `(0, 0, 10)` | `BG` (or delete: default) |
| `menus/main_menu.py:11` | `(255, 255, 0)` | `TITLE` |
| `menus/main_menu.py:13` | `font_size = 100` | delete (dead since `scale`) |
| `menus/pause_menu.py:8` | `(0, 0, 20)` | delete (default) |
| `menus/hud_menu.py:7` | `GREY = (150,150,150)` | `DIM` |
| `menus/hud_menu.py:25` | separator `(0, 0, 15)` | `DIM` — invisible on dark bg otherwise |
| `menus/hud_menu.py:27` | `(255, 0, 255)` | `CHEAT` |
| `menus/hud_menu.py:42` | `(64, 224, 208)` | `AFRAID` |
| `screens/instructions.py:5-7,16` | own `YELLOW/WHITE/GREY`, `(0,0,20)` | `TITLE/TEXT/DIM`, delete bg line |
| `widgets/text_widget.py:5` | default `color=(0, 0, 0)` black | `TEXT` (black text on dark bg = invisible) |
| `widgets/button_widget.py:5` | `bg_color=(200,200,200)` | `BUTTON` |
| `widgets/button_widget.py:32` | hover `(50, 50, 50)` | `BUTTON_HOVER` (yellow, black text readable) |
| `widgets/input_widget.py:32` | text `'black'` | `TEXT` |
| `maze_graphic.py:8` | `(0, 0, 0)` | `BG` (same as HUD) |
| `maze_graphic.py:9` | wall `(255, 255, 255)` | `WALL` (blue) |
| `pacgums/pacgum.py:12` | `(255, 255, 255)` | `PACGUM` |
| `characters/ghost.py:58,62` | `(33,33,255)`, `(255,255,255)` | `AFRAID`, `BLINK` (and delete l.58: l.59-62 already set it) |
| `characters/ghost.py:208,217,235,253` | ghost colors | optional: add `RED, PINK, CYAN, ORANGE` to `colors.py` |
| `characters/pacman.py:7` | `(255, 255, 0)` | optional: `PACMAN = (255, 255, 0)` in `colors.py` |

⚠ `AFRAID` `(60,60,255)` vs `WALL` `(33,33,222)`: both blue. OK because walls are thin lines and ghosts are
filled shapes — check on screen; if confusing, `AFRAID = (40, 40, 255)` + walls a bit darker `(25, 25, 170)`.

---

### Step 4 — Instructions text polish

| Line | Now | Suggested |
|------|-----|-----------|
| `instructions.py:20` | `Arrow Keys < ^ >` | `Arrow keys` (down arrow missing) |
| `instructions.py:21` | `ESC / SPACE : Restart / pause` | `pause` — ESC does not restart |
| `instructions.py:28` | ghost respawn line commented | put back: `Eaten ghost back after {ghost_pwn_time} s` |
| new line | — | `Super pacgum : countdown in the HUD, ghosts blink 2 s before the end` |
| all lines | `_____` fillers | font is not monospace + lines centered → never align. `"Pacgum : +10 pts"` |
| title | `H0w T0 P1aY:` | `HOW TO PLAY` |
| HUD `cheat_doc` (`hud_menu.py:35-40`) | 2 columns with spaces | same problem (proportional font) → 1 key per line or keep short |

---

### Step 5 — Highscores safe **(tested)**

**Problem:** bad file → crash (screen AND save): `{}`, `null`, `[1, 2]`, entry without `score`, `"score": "10"`;
`{broken` → screen overwrites file with `[]`; file read + fonts rebuilt **every frame** (`game.py:71`);
leading / only spaces accepted (`"1 -       got"` seen); config `highscore_filename` ignored;
`highscores.json` committed; no BACK button; debug prints `input_widget.py:39-43`.

**Files:** `src/logic/save_scores.py`, `src/graphic/screens/highscores.py`, `src/graphic/widgets/input_widget.py`,
`src/logic/game.py:71`, `.gitignore`

**Suggested fix:** one `load_scores(filename)` that never crashes and never writes (try/except, keep only
`{"name": str, "score": int >= 0}`, sort, top 10), used by screen + save; refresh only on "Highscores"
click; `name.strip()` + ignore empty; title "TOP 10" + BACK; `.gitignore` += `highscores.json`.

---

### Step 6 — Config loader safe (subject V.3) **(tested)**

**Problem:** unknown key → crash; one wrong value → whole file refused; `{"width": 5, "height": 5}` → crash
(42 pacgums > 25 cells); no max size; comment lines removed → wrong error line number.
Both config files have test values: root `setting.json` `pacgum: 1`, `src/setting.json` `levels: 3, pacgum: 1`.

**Files:** `src/parser/load_config.py:13, 29-35, 44-58`

**Suggested fix:** `extra='ignore'` + message; check key by key → bad value = message + default; delete
`check_consistency`; clamp width/height 5..40; replace comment lines by empty lines. One `config.json` at root.

---

### Step 7 — Dead code, unused stuff, small bugs, lint

**Unused / dead (found by grep + flake8):**

| What | Where | Action |
|------|-------|--------|
| `colors.py` never imported | `src/graphic/colors.py` | step 3 |
| `MOVES` never used | `ghost.py:8-13` | step 1 |
| `self.game_title.font_size = 100` (no effect since `scale`) | `main_menu.py:13` | delete |
| `self.suface` (typo copy of `self.surface`) | `hud_menu.py:13,70` | use `self.surface` |
| `self.invi` never read | `hud_menu.py:19` | delete (or show "INVINCIBLE" in HUD) |
| `menu_width` assigned, unused (F841) | `hud_menu.py:70` | delete |
| commented `ele.update_size(14, …)` + power block | `hud_menu.py:63-68,74` | delete / step 2 |
| `start_evil()`, `end_game()` empty | `game.py:304, 334` | delete |
| commented code | `pacgum.py` (27 lines), `character.py` (17), `game.py` (13), `ghost.py` (10), `maze_env.py` (8), `pacman.py`, `input_widget.py`, `highscores.py`, `text_widget.py` | delete |
| `config` param unused | `pacman.py:6` | delete |
| 2 config files with test values | `setting.json`, `src/setting.json` | one `config.json` (step 6/8) |

**Small bugs:**

| Problem | Where | Fix |
|---------|-------|-----|
| Cheat OFF → 8 lines appended again, then popped 1 per frame (tested: 18 elements right after, back to 8 within 20 frames) | `hud_menu.py:69-79` | rebuild list from flags: `elements = base (+ power) (+ cheat block)` then `calculate_layout()` |
| Resize re-lays out HUD / menu / pause only | `game.py` WINDOWRESIZED | `calculate_layout()` on every screen |
| `elements[4]`, `elements[2]` magic indexes | `game.py` winner / loser | keep refs `self.input`, `self.score_text` |
| `random.seed()` resets global random | `game.py` `level_win` | `random.Random()` object |
| Debug prints `Game____O___OVER`, name prints | `game.py` `lose_life`, `input_widget.py:39-43` | delete |
| Dead lives fallback | `game_state.py:22-23` | delete |
| `C` and `N` = same cheat | `game.py` `cheat_ways` | free one → `F` freeze ghosts (subject VI.5 suggests it) |

**Lint now:** flake8 **23** (E265 ×10 in `__main__.py` + `hud_menu.py`, F821, F841, E501 ×5, E115),
mypy **61 errors** (missing type hints). Subject III.1: both must pass.

---

### Step 8 — Delivery (subject VII, VIII, IX)

- `pac-man.py` at root → `python3 pac-man.py config.json`.
- `Makefile`: `run` → `python3 pac-man.py config.json`; `debug` uses `pudb` → `python3 -m pdb pac-man.py config.json`.
- Packaging script/spec at repo root (e.g. PyInstaller), itch.io unlisted build, in-package instructions.
- README with every required section (IX), `docs/project_management/` (VIII).

---

## 4. PARKED — Pause (later)

Works: SPACE / ESC pause, Resume / Main menu buttons, nothing moves while paused.

| Problem | Suggested fix (to discuss) |
|---------|----------------------------|
| No key leaves pause (ESC / SPACE ignored), mouse only | `if state == "PAUSE" and key in (ESC, SPACE): state = "PLAYING"` |
| ESC on WINNER / LOSER does nothing, no hint | ESC → MENU (skip save) + DIM text "Type your name + Enter" |
| Ghost timers + power countdown keep running in pause | `ghost.shift_timers(paused_ms)` on resume |
| Pause screen opaque → game hidden | `SRCALPHA` surface + `(0, 0, 20, 160)` + redraw game under it |

---

## 5. SUBJECT CHECK

| Subject | Requirement | Us | Step |
|---------|-------------|----|------|
| III.1 | flake8 + mypy pass, type hints, docstrings | ❌ 23 / 61 | 7 |
| III.1 | No crash | ❌ `__main__`, highscores, config | 0, 5, 6 |
| III.2 | Makefile install / run / debug (pdb) / clean / lint | 🟡 debug = pudb | 8 |
| V.1 | `python3 pac-man.py config.json`, 1 arg, no traceback | ❌ | 0, 8 |
| V.2 | JSON + `#` comments | ✅ | — |
| V.3 | Bad values → default + message, unknown keys ignored | ❌ | 6 |
| V.4 | Assigned generator as-is, PERFECT=False, failure handled | ✅ | — |
| V.5 | Highscores robust, name ≤ 10 alnum + spaces, top 10, in menu | 🟡 | 5 |
| VI.1 | Pacgums "in most corridors" | ⚠ `pacgum` = count (42 of ~380 cells) → **discuss** | — |
| VI.1 | Super pacgums + ghosts in corners, Pac-Man in middle, level 1 fixed seed | ✅ | — |
| VI.2 | Arrows **or** WASD | ✅ arrows (⚠ WASD would clash with cheat A, W) | — |
| VI.3 | Ghosts chase / flee / respawn in corner | ✅ (BFS better, step 1) | 1 |
| VI.5 | Cheat mode helps review | ✅ | — |
| VI.7 | ≥ 10 levels, time limit, pause / resume | ✅ code default 10 · ⚠ config files say 3 / 10 | 6 |
| VI.8 | HUD always shows score, lives, level, time | ✅ up to 1920x1080 (step 2 for wider) | 2 |
| VI.8 | Menu, pause menu, game over + victory with score + name | ✅ | — |
| VII | Packaging spec at root, itch.io | ❌ | 8 |
| VIII | `docs/project_management/` | ❌ | 8 |
| IX | README with all sections | ❌ empty | 8 |

---

## 6. EDGE CASES

| Case | Now | Step |
|------|-----|------|
| No arg / missing config | traceback | 0 |
| Pinky / Inky target outside maze or in "42" | handled by BFS target clamp | 1 |
| Last second of power mode | countdown shows 0 | 2 |
| Power line + cheat ON on 1920x1080, or cheat ON on 2560x1080 | HUD overflow | 2 |
| Dark bg + default black text (any new screen) | invisible text | 3 |
| Button hover | black on dark grey | 3 |
| Highscores garbage / `{}` / read-only folder | crash | 5 |
| Name `"   got"` | saved with spaces | 5 |
| Config 5x5 / unknown key / wrong type | crash | 6 |
| Pause + power mode | countdown runs during pause | parked |

Checked OK (tested): all cheat keys; P OFF resets invincible; HUD fits 800x600 → 1920x1080 with cheat ON;
win + ghost same frame → WINNER; spawn guard; timer 0 → 1 life; all maze cells reachable; ghosts never
cross walls; BFS example + 30-maze test.

---

## DONE log

| Date | Commit | What |
|------|--------|------|
| 2026-11-24 | `42359b2` | `colors.py` created; `TextWidget.scale` (title 2.5, cheat 0.5); HUD cheat fits; P OFF resets invincible; ghost blink last 2 s; `power_left` computed; min window 800x600; deleted `main.py`, `test.py`, `rules.py`, `ghosts_move.py`; `MOVES` table added |
| 2026-11-23/24 | `7924927`, `3792ffb` | Cheat block in HUD; Instructions text tweaks |
| 2026-11-22 | `c2b3b84` | Cheat keys P A C M N W L, `GameState.invincible`; BACK branch; win + death same frame (`else`) |
| 2026-11-22 | `e2c7f6f` | Instructions page, spawn guard. ⚠ commented `__main__` (step 0) |
| 2026-11-22 | `c4169db` / `3f06ae4` | Title, pause bg; timer typo, pacgum check in PLAYING, Start loads once |
| ≤ 2026-11-22 | `1e724e6` … `9f69bc7` | Lives reset, keyboard, pause menu, highscores screen, ghost-in-wall fix, seeds, Winner/Loser + save, menu |
