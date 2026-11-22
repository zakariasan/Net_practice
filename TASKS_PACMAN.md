# Pac-Man project — TASKS

Reviewed: 2026-11-22 on **`main` @ `c4169db` + uncommitted changes** (`__main__.py`, `menu_widget.py`,
`instructions.py`, `game.py`). All `file:line` = current working tree.
**(tested)** = reproduced headless. Each task = **Problem → Files → Suggested fix** (to discuss, not final).

> ⚠ This file is **not in git** and was already lost once. `git add TASKS_PACMAN.md` (or move it to `docs/`).

---

## 1. WHERE WE ARE

| Area | State |
|------|-------|
| Game starts (working tree) | ✅ yes (tested) — `origin/main` @ `c4169db` still crashes, push your fix |
| Menu → Start → play → levels → WINNER / LOSER → name → Menu | ✅ works (tested) |
| Lives / score / timer / level win | ✅ works (tested) |
| Instructions screen | 🟡 shows, fits the window — 2 bugs left (step 0) |
| `__main__.py` (no traceback, argc) | 🔴 protection commented out in working tree (step 0) |
| Cheat mode | 🔴 keys listed on Instructions, **none coded** (step 2) |
| Ghost AI | 🟡 works, but ignores walls → gets stuck (step 3) |
| Highscores | 🟡 works with a good file, crashes on a bad file (step 5) |
| Config loader | 🟡 works with a good file, crashes on reviewer edits (step 6) |
| Pause | ⏸ **parked** — works with the mouse, details later (section 4) |
| Lint / README / docs / `pac-man.py` | 🔴 not started (step 9) |

---

## 2. ROADMAP

| Step | Task | Size | Files |
|------|------|------|-------|
| 0 | Finish Instructions + `__main__` → test → commit + push | 15 min | `instructions.py`, `game.py`, `__main__.py` |
| 1 | Win + death in the same frame | 5 min | `game.py` |
| 2 | Cheat mode (your keys) | 40 min | `game.py`, `game_state.py`, `hud_menu.py` |
| 3 | Ghost BFS | 45 min | `ghosts_move.py`, `ghost.py` |
| 4 | Spawn kill | 10 min | `ghost.py` |
| 5 | Highscores safe | 40 min | `save_scores.py`, `highscores.py`, `input_widget.py`, `game.py` |
| 6 | Config loader safe | 45 min | `load_config.py` |
| 7 | Window size / resize | 15 min | `game.py` |
| 8 | Small bugs + cleanup | 45 min | several |
| 9 | Delivery: `pac-man.py`, `config.json`, Makefile, lint, README, docs | 2 h | root |
| — | Pause polish | parked | section 4 |

**Before every push:** `make run` → Start → play → ESC → Main menu → Instructions → BACK → Highscores →
ESC → Exit. Any crash → don't push.

---

## 3. TASKS

### Step 0 — Finish Instructions + `__main__`

#### 0-a. Missing comma → 2 rules glued in one line **(tested)**

**Problem:** on screen: `"Ghosts turn blue..:.7 secBlue ghost........:.+200 pts"`.
Python joins 2 strings written next to each other without a comma.

**Files:** `src/graphic/screens/instructions.py:27`

**Suggested fix:** add `,` at the end of line 27.

#### 0-b. BACK button does nothing **(tested: click BACK → still INSTRUCTIONS)**

**Problem:** `game.py:143` `elif MazeEnv.state == "INSTRUCTIONS":` is inside the **MENU** branch
(the `if clicked_button == ...` chain). In that branch the state is always `"MENU"` → never true.

**Files:** `src/logic/game.py:130-153`

**Suggested fix:** move it out, at the same level as `if MazeEnv.state == "MENU":` / `elif MazeEnv.state == "PAUSE":`
```python
if MazeEnv.state == "MENU":
    ...
elif MazeEnv.state == "INSTRUCTIONS":
    if self.instructions_screen.check_clicks(mouse_pos) == "BACK":
        MazeEnv.state = "MENU"
elif MazeEnv.state == "PAUSE":
    ...
```

#### 0-c. `__main__.py` protection commented **(tested)**

**Problem:** `python -m src` → `IndexError` traceback; missing file → `ConfigError` traceback.
Dead code after `sys.exit(0)` uses undefined `e`. Subject V.1: never a traceback.
Also `Game.__init__` still calls `self.run()` (`game.py:52`) → `run()` runs twice.

**Files:** `src/__main__.py:12-22`, `src/logic/game.py:52`

**Suggested fix:** `git checkout src/__main__.py` (committed version was right), delete `game.py:52`.
To debug without the try: use `make debug`, don't comment the try.

#### 0-d. Text content (small)

| Line | Now | Suggested |
|------|-----|-----------|
| `instructions.py:20` | `Arrow Keys < ^ >` | `Arrow keys` (down arrow missing) |
| `instructions.py:21` | `ESC / SPACE : restart / pause` | `pause` only — ESC doesn't restart |
| `instructions.py:29` | `ghost back stronge` | `Eaten ghost back` |
| all lists | dots `.....` to align | the font is **not** monospace + every line is centered → dots never align. Option: drop dots, use `"Pacgum : +10 pts"`, or render key and value in 2 columns later |
| `instructions.py:31-36` | `{` … `}` across lines inside f-strings | works on Python 3.13, but hard to read → `t = self.con.level_max_time` then `f"... {t} s"` |

---

### Step 1 — Win + death in the same frame **(tested)**

**Problem:** after `level_win()` (`game.py:83-84`) the same frame still runs ghosts, collision and timer.
Last pacgum eaten + ghost touching Pac-Man with 1 life → `WINNER` then `LOSER` → **player who won sees LOSER**.

**Files:** `src/logic/game.py:81-88`

**Suggested fix:**
```python
self.pacman.eat_pacgum()
if not GameState.pacgums:
    self.level_win()
else:
    for ghost in self.ghosts:
        ghost.move()
    self.check_ghost_collision()
    self.update_timer()
```

---

### Step 2 — Cheat mode (keys from your Instructions screen)

**Problem:** the Instructions screen lists 7 keys, the code has none. Today `GameState.cheat_mode` exists
but nothing turns it on, and it is used as "invincible" in the collision (`game.py:273`).

**Your keys:**

| Key | Effect (your text) | Suggested code | Where |
|-----|--------------------|----------------|-------|
| `P` | cheat mode ON/OFF | `GameState.cheat_mode = not GameState.cheat_mode` | `game.py` PLAYING keys |
| `A` | invincible | `GameState.invincible = not GameState.invincible` | idem |
| `C` | skip level | `GameState.pacgums.clear()` → next frame `level_win()` | idem |
| `M` | +1 life | `GameState.lives += 1` | idem |
| `N` | next level | `GameState.pacgums.clear()` | idem |
| `W` | win the game | `GameState.level = self.config.levels` then `GameState.pacgums.clear()` → `level_win()` → WINNER | idem |
| `L` | lose the game | `GameState.lives = 1` then `self.lose_life()` → LOSER | idem |

Rule: keys `A C M N W L` work **only when** `cheat_mode` is ON (`P` first). Only in `PLAYING`
(so typing a name on WINNER/LOSER is not affected).

**Suggested code** (`game.py`, inside `if MazeEnv.state == "PLAYING":` key block):
```python
elif event.key == pygame.K_p:
    GameState.cheat_mode = not GameState.cheat_mode
elif GameState.cheat_mode:
    self.cheat_key(event.key)
```
```python
def cheat_key(self, key: int) -> None:
    """Cheat keys, active only when cheat mode is ON."""
    if key == pygame.K_a:
        GameState.invincible = not GameState.invincible
    elif key in (pygame.K_c, pygame.K_n):
        GameState.pacgums.clear()
    elif key == pygame.K_m:
        GameState.lives += 1
    elif key == pygame.K_w:
        GameState.level = self.config.levels
        GameState.pacgums.clear()
    elif key == pygame.K_l:
        GameState.lives = 1
        self.lose_life()
```
Other changes:
- `game_state.py`: add `invincible: bool = False`; `reset_game()` sets `cheat_mode = False`, `invincible = False`.
- `game.py:273` collision: `if not GameState.invincible:` (instead of `cheat_mode`).
- `hud_menu.py`: line `"CHEAT ON"` (+ `"INVINCIBLE"`) when active, so the reviewer sees it.

**To discuss:**
- `C` and `N` do the same thing. Keep one, give the other a new effect? Ideas: `F` freeze ghosts,
  `T` +30 s timer, `G` all ghosts blue.
- Save a highscore after cheating? Suggest: no (or name gets a `*`).

---

### Step 3 — Ghost BFS (walls-aware chase)

**Problem:** today `ghost.py:121` measures distance **as the crow flies**:
`dist = (target_x - test_x)**2 + (target_y - test_y)**2` → walls ignored → ghost enters dead ends, circles
blocks. Measured (20x20, 30 mazes, Pac-Man still): Blinky alone catches **18/30** in 2 min; with BFS **30/30**
in max 10 s.

**Idea:** keep everything (targets, no U-turn, afraid), change only the distance:
**number of steps through corridors** from each cell to the target, computed by BFS.

#### 3-1. How to find the neighbors of a cell

Each cell is a number. Each bit = one wall:

| Move | Wall bit | Cell is closed on that side if | Next cell |
|------|----------|--------------------------------|-----------|
| UP | `1` (North) | `cell & 1` | `(y - 1, x)` |
| RIGHT | `2` (East) | `cell & 2` | `(y, x + 1)` |
| DOWN | `4` (South) | `cell & 4` | `(y + 1, x)` |
| LEFT | `8` (West) | `cell & 8` | `(y, x - 1)` |

`cell & bit == 0` → no wall → you can go there. `15 = 1+2+4+8` → closed everywhere ("42" logo).

Example: `cell = 9 = 8 + 1` → walls West + North → open RIGHT and DOWN.

#### 3-2. Example maze (3x3, checked with code)

```
      x=0 x=1 x=2
     +---+---+---+
y=0  | A   B | C |       A=9   B=7   C=11
     +   +---+   +
y=1  | D   E   F |       D=12  E=1   F=6
     +---+   +---+
y=2  | G   H |###|       G=13  H=6   I=15 (closed)
     +---+---+---+
```
Neighbors: A → B, D · E → F, H, D · I → none.

BFS from target **C** (0,2) — each step adds 1:
```
step 0: C
step 1: F            (C's only open side: DOWN)
step 2: E
step 3: D, H
step 4: A, G
step 5: B
```
Distance grid (`-1` = never reached):
```
 4  5  0
 3  2  1
 4  3 -1
```
**Ghost on A wants C.** Open moves: RIGHT → B, DOWN → D.
- Crow-flies: B is 1 cell from C, D is ~2.2 → picks **B = dead end** (B only opens back to A).
- BFS: B = 5 steps, D = 3 steps → picks **D** → D, E, F, C. ✅

#### 3-3. Suggested code (`src/logic/ghosts_move.py`, plain lists)

```python
# (wall bit, dy, dx) for UP, RIGHT, DOWN, LEFT
MOVES = [(1, -1, 0), (2, 0, 1), (4, 1, 0), (8, 0, -1)]


def open_neighbors(maze: list[list[int]], y: int,
                   x: int) -> list[tuple[int, int]]:
    """Cells reachable in one step from (y, x)."""
    result = []
    for bit, dy, dx in MOVES:
        if maze[y][x] & bit:            # wall on this side
            continue
        ny, nx = y + dy, x + dx
        if 0 <= ny < len(maze) and 0 <= nx < len(maze[0]):
            result.append((ny, nx))
    return result


def bfs_grid(maze: list[list[int]],
             start: tuple[int, int]) -> list[list[int]]:
    """Steps from start to every cell. -1 = not reachable."""
    rows, cols = len(maze), len(maze[0])
    dist = [[-1] * cols for _ in range(rows)]
    dist[start[0]][start[1]] = 0
    queue = [start]
    i = 0                                # read position in the list
    while i < len(queue):
        y, x = queue[i]
        i += 1
        for ny, nx in open_neighbors(maze, y, x):
            if dist[ny][nx] == -1:       # first time we reach it = shortest
                dist[ny][nx] = dist[y][x] + 1
                queue.append((ny, nx))
    return dist
```
(`i` instead of `queue.pop(0)`: same result, no shifting of the list.)

#### 3-4. Fix the target before BFS

Pinky (4 ahead) / Inky (2 ahead) can point **outside the maze** or **into a 15 cell** → BFS from there
reaches nothing → all `-1` → ghost moves randomly.
```python
def safe_target(maze: list[list[int]], target: tuple[int, int],
                fallback: tuple[int, int]) -> tuple[int, int]:
    """Clamp into the maze; closed cell → use fallback (Pac-Man cell)."""
    y = min(max(target[0], 0), len(maze) - 1)
    x = min(max(target[1], 0), len(maze[0]) - 1)
    return fallback if maze[y][x] == 15 else (y, x)
```

#### 3-5. Use it in `ghost.py` `next_direction()` (`ghost.py:82-129`)

```python
target = safe_target(MazeEnv.maze, (target_y, target_x),
                     self.pacman.current_cell)
dist = bfs_grid(MazeEnv.maze, target)        # once per decision
...
for d in possible_directions:
    # test_y, test_x = next cell (existing code)
    steps = dist[test_y][test_x]
    if steps == -1:
        steps = 9999                          # unreachable = very far
    if self.is_afraid:
        steps = -steps                        # afraid: go where it's FAR
    if steps < min_dist:
        min_dist = steps
        best_dir = d
```
Cost: 400 cells (20x20), only when a ghost reaches a cell center → nothing.
Defense sentence: *"Each ghost picks a target cell (its personality), then takes the move with the fewest
steps through the corridors (BFS)."*

**To discuss (later):** afraid = random direction instead of max distance (easier to catch); ghosts faster
each level; Inky classic target (uses Blinky's position).

---

### Step 4 — Spawn kill **(tested)**

**Problem:** eaten ghost comes back in its corner even if Pac-Man stands there → life lost (3 → 2).
Super pacgums are in the corners → happens often.

**Files:** `src/graphic/characters/ghost.py:175-178`

**Suggested fix:** respawn only when Pac-Man is ≥ 3 cells away:
```python
if self.is_eaten:
    if now >= self.pwn_time:
        sy, sx = self.get_start_pos()
        py, px = self.pacman.current_cell
        if abs(sy - py) + abs(sx - px) >= 3:
            self.reset()
    return
```

---

### Step 5 — Highscores safe **(tested)**

**Problem:**
- Bad file → crash: `{}`, `null`, `[1, 2]`, entry without `score`, `"score": "10"` (screen AND save).
- `{broken` → screen overwrites the file with `[]` (scores lost); save crashes.
- File read + fonts rebuilt **every frame** on the Highscores screen (`game.py:71`) → laggy.
- Config `highscore_filename` ignored; `highscores.json` committed with test names.
- Name of spaces only accepted; debug prints `input_widget.py:39,41,43`.

**Files:** `src/logic/save_scores.py`, `src/graphic/screens/highscores.py:17-34`,
`src/graphic/widgets/input_widget.py:35-43`, `src/logic/game.py:71`, `.gitignore`

**Suggested fix:** one `load_scores(filename)` that never crashes and never writes (try/except +
keep only `{"name": str, "score": int >= 0}` + sort + top 10), used by screen and save; read it only when
"Highscores" is clicked; `name.strip()`; `.gitignore` += `highscores.json`.

---

### Step 6 — Config loader safe (subject V.3) **(tested)**

**Problem:** unknown key → crash; one wrong value → whole file refused; `{"width": 5, "height": 5}` → crash
(42 pacgums > 25 cells); no max size; comment lines removed → wrong error line number.

**Files:** `src/parser/load_config.py:13, 29-35, 44-58`

**Suggested fix:** `extra='ignore'` + message; check key by key → bad value = message + default; delete
`check_consistency`; clamp width/height 5..40; replace comment lines by empty lines.

---

### Step 7 — Window size / resize

**Problem:** `minimum_size = (1500, 1000)` (`game.py:29`) → can't be smaller (tested), too big for many
laptops. Resize re-lays out HUD / main menu / pause only.

**Suggested fix:** `(800, 600)`; on resize call `calculate_layout()` for every screen (instructions,
highscores, winner, loser too).

---

### Step 8 — Small bugs + cleanup

| Problem | Files | Suggested fix |
|---------|-------|---------------|
| Debug prints (`Game____O___OVER`, name prints) | `game.py` `lose_life`, `input_widget.py:39-43` | delete |
| `elements[4]`, `elements[2]` magic indexes | `game.py:183,185,282,311` | keep refs `self.input`, `self.score_text` in screens |
| `TextWidget` has no `self.font = None` → `update_text()` before layout crashes | `text_widget.py:5-11` | add the line |
| Title `font_size = 100` added (huge on big screen) | `main_menu.py:13` | multiplier instead |
| `random.seed()` resets global random | `game.py` `level_win` | `random.Random()` object |
| Dead lives fallback | `game_state.py:22-23` | delete |
| Dead files | `rules.py`, `screens/test.py`, `main.py` | delete |
| Commented code / empty methods (`start_evil`, `end_game`) | several | delete |
| Root `setting.json` duplicate | root | keep one |

---

### Step 9 — Delivery

- `pac-man.py` at root (subject: `python3 pac-man.py config.json`).
- Root `config.json` with real values (`levels: 10`, `pacgum: 42`); `src/setting.json` still test values.
- `Makefile`: `debug` → `python3 -m pdb`, `run` → `python3 pac-man.py config.json`.
- `make lint` clean (flake8 + mypy), README, `docs/project_management/`, itch.io link.

---

## 4. PARKED — Pause (later)

Works today: SPACE / ESC pause, Resume / Main menu buttons, nothing moves while paused.
Left for later (tested, not urgent):

| Problem | Suggested fix (to discuss) |
|---------|----------------------------|
| No key leaves pause (ESC / SPACE ignored in PAUSE), mouse only | in KEYDOWN: `if state == "PAUSE" and key in (ESC, SPACE): state = "PLAYING"` |
| ESC on WINNER / LOSER does nothing, no hint | ESC → MENU (skip save) + grey text "Type your name + Enter" |
| Ghost timers (afraid, respawn, scatter) keep running during pause → afraid can end while paused | `ghost.shift_timers(paused_ms)` on resume |
| Pause screen opaque → game hidden | `SRCALPHA` surface + `(0, 0, 20, 160)` + redraw game under it |

---

## 5. EDGE CASES

| Case | Now | Step |
|------|-----|------|
| `origin/main` `make run` | crash at start | 0 (push) |
| 0 args / missing config file | traceback | 0-c |
| Click BACK on Instructions | nothing | 0-b |
| Win last level + ghost touch / timer 0 same frame | LOSER / life lost | 1 |
| Cheat keys on Instructions | do nothing | 2 |
| Pinky / Inky target outside maze or in "42" | ghost moves dumb (with BFS) | 3-4 |
| Pac-Man on corner when eaten ghost respawns | life lost | 4 |
| Highscores file garbage / `{}` / read-only folder | crash | 5 |
| Config 5x5 / unknown key / wrong type | crash | 6 |
| Window < 1500x1000 | impossible | 7 |
| Pause + afraid ghosts | afraid ends during pause | parked |

Checked OK: timer 0 → 1 life; Start builds the level once; WINNER stays; new game = full lives; all
maze cells reachable (30 seeds, sizes 5..40); ghosts never cross walls; Instructions fit at 1500x1000.

---

## DONE log

| Date | Commit | What |
|------|--------|------|
| 2026-11-22 | *uncommitted* | Instructions: 1 `TextWidget` per line (no more crash), 3 blocks, font `min()` fix |
| 2026-11-22 | `c4169db` | `__main__` argc + try (undone in working tree, 0-c); title "PAC-MAN"; pause bg dark |
| 2026-11-22 | `3f06ae4` | Timer typo; pacgum check in PLAYING + `return` in `level_win`; Start loads once |
| 2026-11-22 | `1e724e6` | Lives reset on new game; keyboard in game; ESC pauses; hover fix |
| 2026-11-20 | `e051b69` | Pause menu |
| 2026-11-18 | `87241d6` / `84f203c` | Highscores screen reload; ghost-in-wall fix |
| ≤ `9f69bc7` | | Seeds, mazegenerator wheel, Winner/Loser + top-10 save, menu buttons |
