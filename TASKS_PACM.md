# Pac-Man project — TASKS

Reviewed: 2026-11-22 on **`main` @ `1e724e6` + uncommitted changes** in `game.py` / `game_state.py`.
All `file:line` = current working tree. **(tested)** = reproduced headless (SDL dummy driver, fake clock).

How to read each problem: **Problem** (what goes wrong, how to see it) → **Files** → **Fix**.
When a problem is fixed: move it to the DONE log at the end.

> **Commit your current changes first** — the timer typo, the `return` in `level_win` and the double
> `load_level()` are fixed in your working tree but not committed yet (all 3 re-tested OK).

---

## ROADMAP (order to work)

| Step | Problem | Size | Files |
|------|---------|------|-------|
| 0 | Commit current fixes | 1 min | `game.py`, `game_state.py` |
| 1 | P1 win + death in same frame | 5 min | `game.py` |
| 2 | P2 keys in PAUSE / WINNER / LOSER | 20 min | `game.py`, `winner.py`, `loser.py` |
| 3 | P3 ghost timers run during pause | 20 min | `ghost.py`, `game.py` |
| 4 | P4 pause hides the game | 15 min | `pause_menu.py`, `game.py` |
| 5 | P5 `TextWidget.font` missing | 1 min | `text_widget.py` |
| 6 | P6 ghost AI with BFS | 45 min | `ghosts_move.py`, `ghost.py` |
| 7 | P7 spawn kill | 10 min | `ghost.py` |
| 8 | P8 highscores crashes | 40 min | `save_scores.py`, `highscores.py`, `input_widget.py`, `game.py` |
| 9 | P9 config loader | 45 min | `load_config.py` |
| 10 | P10 `__main__` no traceback | 15 min | `__main__.py`, `game.py` |
| 11 | P11 cheat mode | 40 min | `game.py`, `game_state.py`, `hud_menu.py` |
| 12 | P12 instructions screen | 40 min | `instructions.py`, `menu_widget.py`, `game.py` |
| 13 | P13 window size / resize | 15 min | `game.py` |
| 14 | P14 small bugs | 30 min | several |
| 15 | P15 cleanup dead code | 20 min | several |
| 16 | P16 delivery: entry point, config, Makefile, lint, README | 2 h | root files |

---

## PROGRESS

| # | Task | Status |
|---|------|--------|
| 2 | Pacgums: placement, eating, score, HUD | DONE |
| 3 | Ghost collision, lives, respawn | DONE (P7 spawn kill open) |
| 4 | Afraid mode, eat ghost, ghost respawn | DONE (P3 pause timers open) |
| 5 | Levels, timer, win/lose | DONE after commit (P1 edge open) |
| 6 | State machine | PARTIAL (P2) |
| 7 | Pause menu | PARTIAL (P2, P3, P4) |
| — | Ghost chase quality | TODO (P6) |
| 10 | Cheat mode | TODO (P11) |
| 8 | Highscores JSON | PARTIAL (P8) |
| 9 | Winner / Loser / Instructions screens | PARTIAL (P12) |
| 1 | Config loader | PARTIAL (P9) |
| — | No traceback / argc check | TODO (P10) |
| 11 | Entry point, Makefile, packaging, itch.io | PARTIAL (P16) |
| 12 | Lint: flake8 + mypy strict | TODO (flake8 12, mypy 56) |
| 14 | README + `docs/project_management/` | TODO |

---

## PROBLEMS

### P1. Win the last level and touch a ghost in the same frame → LOSER **(tested, new)**

**Problem:**
The pacgum check is now inside PLAYING (good), but after `level_win()` the same frame still runs
`ghost.move()`, `check_ghost_collision()` and `update_timer()`.
If Pac-Man eats the last pacgum while a ghost touches him with 1 life left:
`level_win()` sets `"WINNER"`, then the collision calls `lose_life()` → state becomes `"LOSER"`.
Test: level 2/2, 1 life, last gum + ghost on same cell → **LOSER, lives 0** (the player actually won).
Same if the timer reaches 0 in that frame.

**Files:** `src/logic/game.py:80-86`

**Fix:** only move ghosts / check collision / timer when the level is not finished:
```python
self.pacman.move()
self.pacman.eat_pacgum()
if not GameState.pacgums:
    self.level_win()            # WINNER, or next level loaded
else:
    for ghost in self.ghosts:
        ghost.move()
    self.check_ghost_collision()
    self.update_timer()
```
The draw part below stays as is.

---

### P2. Keys in PAUSE, WINNER, LOSER **(tested)**

**Problem:**

| State | ESC | SPACE | What's wrong |
|---|---|---|---|
| PLAYING | → PAUSE ✓ | → PAUSE ✓ | — |
| PAUSE | nothing | nothing | no key can resume, mouse click only |
| MENU / INSTRUCTIONS / HIGHSCORES | → MENU ✓ | — | — |
| WINNER / LOSER | nothing | nothing | ESC is sent to the name box and ignored. Only way out = type a name + Enter. Empty Enter does nothing, no hint on screen |

**Files:**
- `src/logic/game.py:155-182` (KEYDOWN block)
- `src/graphic/screens/winner.py`, `src/graphic/screens/loser.py` (hint text)

**Fix:**
```python
elif event.type == pygame.KEYDOWN:
    if MazeEnv.state == "PAUSE":
        if event.key in (pygame.K_ESCAPE, pygame.K_SPACE):
            self.resume()                         # see P3
    elif MazeEnv.state == "PLAYING":
        if event.key in (pygame.K_ESCAPE, pygame.K_SPACE):
            self.pause()                          # see P3
        elif event.key == pygame.K_UP:
            self.pacman.new_direction = Direction.UP
        ...                                       # other arrows
    elif MazeEnv.state in ("WINNER", "LOSER"):
        screen = (self.winner_screen if MazeEnv.state == "WINNER"
                  else self.loser_screen)
        if event.key == pygame.K_ESCAPE:
            MazeEnv.state = "MENU"                # leave without saving
        else:
            screen.elements[4].handle_event(event)
    elif event.key == pygame.K_ESCAPE:
        MazeEnv.state = "MENU"
```
Winner / Loser screens: add `TextWidget("Type your name + Enter  (ESC = skip)", (150, 150, 150))`
as the last element (keep the input at index 4, or better keep a ref `self.input`, see P14-b).

---

### P3. Ghost timers keep running during pause **(tested)**

**Problem:**
Ghost timers use `pygame.time.get_ticks()` (real clock). Pause does not stop the real clock.
Test: ghosts afraid for 7 s → pause 9.6 s → resume → **afraid already over**.
Same for eaten-ghost respawn (`pwn_time`) and scatter/chase switch (`last_mode_switch`).

**Files:**
- `src/graphic/characters/ghost.py:14, 155, 164, 174, 189` (timers)
- `src/logic/game.py:144-147` (Resume click) + key block (P2)

**Fix:**
```python
# ghost.py
def shift_timers(self, ms: int) -> None:
    """Push every timer forward by the paused time."""
    self.afraid_until += ms
    self.pwn_time += ms
    self.last_mode_switch += ms

# game.py — use these 2 methods for EVERY way in / out (keys + Resume button)
def pause(self) -> None:
    self.pause_start = pygame.time.get_ticks()
    MazeEnv.state = "PAUSE"

def resume(self) -> None:
    paused = pygame.time.get_ticks() - self.pause_start
    for ghost in self.ghosts:
        ghost.shift_timers(paused)
    MazeEnv.state = "PLAYING"
```
"Main menu" from pause doesn't need `resume()` (Start creates new ghosts).

---

### P4. Pause screen hides the game

**Problem:**
`pause_menu_surface` is the full window, filled near-white `(250, 250, 250)` → the maze disappears and it
looks like another menu. Expected: game visible behind, darkened, buttons on top.

**Files:**
- `src/graphic/menus/pause_menu.py:8`
- `src/logic/game.py:72-77` (PAUSE draw), `game.py:196` (surface creation)

**Fix:**
```python
# game.py calculate_dimensions
self.pause_menu_surface = pygame.Surface((window_width, window_height),
                                         pygame.SRCALPHA)
# pause_menu.py
self.bg_color = (0, 0, 0, 160)          # transparent black
# game.py run(): move the draw half of PLAYING into draw_game()
elif MazeEnv.state == "PAUSE":
    self.draw_game()                    # frozen frame, nothing moves
    self.pause_menu.draw()
    self.screen.blit(self.pause_menu_surface, (0, 0))
```
The game must be redrawn every frame: without `draw_game()` the transparent black stacks up → full black in < 1 s.

---

### P5. `TextWidget` lost `self.font` **(tested)**

**Problem:**
`1e724e6` replaced `self.font = None` with `self.font_size = 0`. `render_text()` does `if self.font:`
→ `update_text()` before `update_size()` = **AttributeError**. Not hit today, but the next screen that
sets a text before layout will crash the game.
Also the main menu title adds a fixed `+100` px to the font → 175 px on a big screen, too big on a small one.

**Files:** `src/graphic/widgets/text_widget.py:8,14`, `src/graphic/menus/main_menu.py:13`

**Fix:**
```python
# text_widget.py __init__
self.font = None
self.font_size = 0
# delete the commented SysFont line (flake8 E265/E501)
```
Title: use a multiplier (e.g. `font_scale = 2.0` → `int(new_size * self.font_scale)`) instead of `+100`.

---

### P6. Ghosts chase with straight-line distance (walls ignored) **(tested)**

**Problem:**
At each cell, a ghost takes the open direction closest to its target **as the crow flies**.
Walls are ignored → the ghost circles a block forever and never reaches Pac-Man.

Measured (20x20, seeds 1-30, 2 min limit):

| Scenario | Now (straight-line) | BFS (fix) |
|---|---|---|
| Blinky alone, Pac-Man still | caught **18/30**, avg 13.9 s | **30/30**, avg 8.0 s, max 10.3 s |
| 4 ghosts, Pac-Man still | 29/30, **max 83.5 s** | 30/30, max 7.9 s |
| 4 ghosts, Pac-Man random walk | 30/30, avg 10.4 s | 30/30, avg 8.0 s |

Second problem: Pinky / Inky targets (4 / 2 cells ahead) often land **outside** the maze or **inside the
"42" logo** (closed cells = 15). BFS from a closed cell reaches nothing → the ghost picks a random-looking way.

**Files:**
- `src/logic/ghosts_move.py` (empty today → put BFS here)
- `src/graphic/characters/ghost.py:82-129` (`next_direction`)

**Fix:**
```python
# ghosts_move.py
from collections import deque

WALLS = [(1, -1, 0), (2, 0, 1), (4, 1, 0), (8, 0, -1)]   # bit, dy, dx (N, E, S, W)


def snap_target(maze: list[list[int]],
                target: tuple[int, int]) -> tuple[int, int]:
    """Clamp into the maze; if closed (15) take the nearest open cell."""
    rows, cols = len(maze), len(maze[0])
    y = min(max(target[0], 0), rows - 1)
    x = min(max(target[1], 0), cols - 1)
    if maze[y][x] != 15:
        return (y, x)
    return min(((r, c) for r in range(rows) for c in range(cols)
                if maze[r][c] != 15),
               key=lambda cell: abs(cell[0] - y) + abs(cell[1] - x))


def bfs_distances(maze: list[list[int]],
                  start: tuple[int, int]) -> dict[tuple[int, int], int]:
    """Real path length (through corridors) from start to every cell."""
    rows, cols = len(maze), len(maze[0])
    start = snap_target(maze, start)
    dist = {start: 0}
    queue = deque([start])
    while queue:
        y, x = queue.popleft()
        for bit, dy, dx in WALLS:
            ny, nx = y + dy, x + dx
            if (not maze[y][x] & bit and 0 <= ny < rows and 0 <= nx < cols
                    and (ny, nx) not in dist):
                dist[(ny, nx)] = dist[(y, x)] + 1
                queue.append((ny, nx))
    return dist
```
```python
# ghost.py next_direction(): compute once before the loop, replace the distance line
dist_map = bfs_distances(MazeEnv.maze, (target_y, target_x))
...
dist = dist_map.get((test_y, test_x), 10_000)
if self.is_afraid:
    dist = -dist
```
Cost: 400 cells (20x20), only when a ghost reaches a cell center → nothing.
Defense line: *"Each ghost picks a target cell (its personality), then follows the shortest real path."*

**Optional after:**
- One global scatter/chase clock in `Game` counted with `dt` (today each ghost has its own → out of sync
  after one is eaten; also solves P3 for mode switch).
- Inky classic: target = `2 * (2 cells ahead of Pac-Man) - Blinky position` (needs `snap_target`).
- Afraid ghosts: random open direction at crossings (easier to catch).
- Ghost speed up per level: `0.05 + 0.005 * (level - 1)`, max `0.08`.
- Staggered start: Blinky 0 s, Pinky 2 s, Inky 4 s, Clyde 6 s.

---

### P7. Spawn kill **(tested)**

**Problem:**
An eaten ghost comes back in its corner after `ghost_pwn_time`, even if Pac-Man stands there → instant life
lost (test: lives 3 → 2). Super pacgums are in the corners → this happens often.

**Files:** `src/graphic/characters/ghost.py:173-178` (`move`, eaten branch)

**Fix:**
```python
if self.is_eaten:
    if now >= self.pwn_time:
        sy, sx = self.get_start_pos()
        py, px = self.pacman.current_cell
        if abs(sy - py) + abs(sx - px) >= 3:      # Pac-Man far enough
            self.reset()
    return
```

---

### P8. Highscores crash on bad file **(tested)**

**Problem:**

| Content of `highscores.json` | Highscores screen | Saving a score |
|---|---|---|
| `{"a": 1}` | CRASH TypeError | CRASH AttributeError |
| `[{"name": "x"}]` | CRASH KeyError | CRASH KeyError |
| `[1, 2]` | CRASH TypeError | CRASH TypeError |
| `[{"name":"x","score":"10"}]` | CRASH TypeError | CRASH TypeError |
| `null` | CRASH TypeError | CRASH TypeError |
| `{broken` | file **overwritten with `[]`** (scores lost) | CRASH JSONDecodeError |

Also:
- File read + 11 fonts created **every frame** (`game.py:69`) → 13 ms of a 16 ms frame → laggy.
- The screen writes the file when it's missing → crash if the folder is read-only; creates `highscores.json` on start.
- Not sorted on display; more than 10 shown if the file is edited.
- Config `highscore_filename` ignored (hard-coded `"highscores.json"`). File committed with test names.
- A name of spaces only is accepted. Debug prints when saving (`input_widget.py:39,41,43`).

**Files:**
- `src/logic/save_scores.py` (loader + save)
- `src/graphic/screens/highscores.py:17-34`
- `src/graphic/widgets/input_widget.py:35-43`
- `src/logic/game.py:69` and the Highscores click (`game.py:136`)
- `.gitignore`

**Fix:**
1. One safe loader in `save_scores.py`, used by the screen AND `save_score`:
   ```python
   def load_scores(filename: str) -> list[dict[str, object]]:
       try:
           with open(filename, encoding="utf-8") as f:
               data = json.load(f)
       except (OSError, json.JSONDecodeError):
           return []
       if not isinstance(data, list):
           return []
       good = [e for e in data
               if isinstance(e, dict) and isinstance(e.get("name"), str)
               and type(e.get("score")) is int and e["score"] >= 0]
       good.sort(key=lambda e: e["score"], reverse=True)
       return good[:10]
   ```
   The loader never writes. `save_score` writes inside `try/except OSError` (print message, no crash).
2. Read the file only when the "Highscores" button is clicked (`refresh()`), not in `run()`.
   Show "No scores yet" when empty.
3. Pass `config.highscore_filename` (add `.json` if no extension).
4. `input_widget.py`: `name = typed_text.strip()`, ignore if empty; delete the 3 prints.
5. `.gitignore` += `highscores.json`; `git rm --cached highscores.json`.

---

### P9. Config loader crashes on reviewer edits (subject V.3) **(tested)**

**Problem:**
- Unknown key → crash (`extra='forbid'`).
- One wrong value → whole file rejected (`strict=True`).
- `{"width": 5, "height": 5}` alone → **crash** (default pacgum 42 > 25 cells, `check_consistency`).
- Missing file / bad JSON → `ConfigError` → traceback (P10).
- No max for `width` / `height`.
- Comment lines removed → JSON error line numbers wrong.

**Files:** `src/parser/load_config.py:13, 29-35, 44-58`

**Fix:**
- `extra='ignore'` + print `[config] unknown key 'x' ignored`.
- Validate key by key: bad value → print `[config] 'lives'=-4 invalid, using default 3`, keep the default.
- Delete `check_consistency` (pacgum placement already uses `min()`).
- Clamp `width` / `height` to 5..40 (40x40 tested fine).
- Replace comment lines by `"\n"` instead of removing them.
- Missing file / bad JSON → message + defaults (or clean exit, no traceback).

---

### P10. Traceback possible / wrong args (subject V.1)

**Problem:** try/except is commented out. 0 args runs with defaults, extra args are ignored (subject: exactly 1).
`Game.__init__` calls `self.run()` → can't be wrapped cleanly.

**Files:** `src/__main__.py:12-26`, `src/logic/game.py:50`

**Fix:**
```python
if len(sys.argv) != 2:
    print("Usage: python3 pac-man.py config.json", file=sys.stderr)
    sys.exit(1)
try:
    game = Game(Config.from_json_file(sys.argv[1]))
    game.run()                       # delete self.run() from Game.__init__
except KeyboardInterrupt:
    sys.exit(0)
except Exception as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
```

---

### P11. Cheat mode (subject VI.5) — not started

**Problem:** no cheat keys. The reviewer must be able to reach every screen fast.

**Files:** `src/logic/game.py` (PLAYING keys, run loop), `src/logic/game_state.py` (flags + reset),
`src/graphic/menus/hud_menu.py` ("CHEAT ON" line)

**Fix:**

| Key | Effect | How |
|-----|--------|-----|
| `C` | Cheat mode ON/OFF | `GameState.cheat_mode = not ...`; HUD shows "CHEAT ON". Other keys work only when ON |
| `I` | Invincible | `GameState.invincible`; collision skips `lose_life()` (`game.py:266` already checks `cheat_mode`) |
| `N` | Skip level | `GameState.pacgums.clear()` → normal `level_win()` |
| `F` | Freeze ghosts | `GameState.ghosts_frozen`; skip `ghost.move()` |
| `L` | +1 life | `GameState.lives += 1` |
| `V` | Speed x2 | Pac-Man speed `0.1 ↔ 0.2` |
| `T` | +30 s | `GameState.time_left += 30` |

`reset_game()` resets every flag.

---

### P12. Instructions screen — placeholder text

**Problem:**
- Text is "Hellooooo" / "Its meeeee" / "what do you want?" → subject VI.8 wants **controls + rules**.
- Uses layout `"HUD"` → font = 10 % of window width (~150 px) → long lines go off screen.
- l.10-13 copies score/lives/level → unused. No Back button.

**Files:**
- `src/graphic/screens/instructions.py` (content)
- `src/graphic/menus/menu_widget.py:9-33` (new `'INSTRUCTIONS'` layout)
- `src/logic/game.py:42` (pass config), click handler (Back button), resize handler

**Target screen — 3 blocks, one rule per line:**
```
                      HOW TO PLAY

 ───────────────────── [ CONTROLS ] ─────────────────────
   Arrow keys ........ move Pac-Man
   ESC / SPACE ....... pause / resume
   ESC (menus) ....... back to main menu
   Enter ............. save your name (end screen)

 ────────────────────── [ RULES ] ───────────────────────
   Pacgum ............ +{points_per_pacgum} pts
   Super pacgum ...... +{points_per_super_pacgum} pts, ghosts turn blue {afraid_time} s
   Blue ghost ........ +{points_per_ghost} pts, comes back after {ghost_pwn_time} s
   Touch a ghost ..... lose 1 life  (you have {lives})
   Timer ............. {level_max_time} s per level, time out = lose 1 life
   All pacgums eaten . next level  ({levels} levels to win)

 ──────────────────── [ CHEAT MODE ] ────────────────────
   C ................. cheat mode ON / OFF
   I ................. invincible
   N ................. skip level
   F ................. freeze ghosts
   L ................. +1 life
   V ................. speed x2
   T ................. +30 seconds

                        [ Back ]
```
Numbers come from **config** (the reviewer changes them at defense → the screen must follow).
Cheat block: list only keys that really exist (update when P11 is done).

**Fix — layout** (`menu_widget.py`, add a branch):
```python
elif self.menu_name == 'INSTRUCTIONS':
    menu_width, menu_height = self.surface.get_size()
    dynamic_font_size = max(14, min(int(menu_width * 0.026),
                                    int(menu_height * 0.036)))
    self.gap = int(dynamic_font_size * 0.15)
```
Checked headless with these 22 lines: fits at 800x600 (font 20), 1500x1000 (font 36), 1920x1080 (font 38).

**Fix — screen** (`instructions.py`, replace all):
```python
from ..menus.menu_widget import MenuWidget
from ..widgets.text_widget import TextWidget
from ..widgets.button_widget import ButtonWidget

YELLOW = (255, 220, 0)
WHITE = (230, 230, 230)
GREY = (150, 150, 150)


class InstructionScreen(MenuWidget):
    """Controls, rules and cheat keys. Numbers read from config."""

    def __init__(self, surface, config) -> None:
        super().__init__(surface, "INSTRUCTIONS")
        self.bg_color = (0, 0, 10)
        c = config
        controls = [
            "Arrow keys : move Pac-Man",
            "ESC / SPACE : pause / resume",
            "ESC (menus) : back to main menu",
            "Enter : save your name (end screen)",
        ]
        rules = [
            f"Pacgum : +{c.points_per_pacgum} pts",
            f"Super pacgum : +{c.points_per_super_pacgum} pts,"
            f" ghosts turn blue {c.afraid_time} s",
            f"Blue ghost : +{c.points_per_ghost} pts,"
            f" comes back after {c.ghost_pwn_time} s",
            f"Touch a ghost : lose 1 life (you have {c.lives})",
            f"Timer : {c.level_max_time} s per level,"
            f" time out = lose 1 life",
            f"All pacgums eaten : next level ({c.levels} levels to win)",
        ]
        cheats = [
            "C : cheat mode ON / OFF",
            "I : invincible     N : skip level     F : freeze ghosts",
            "L : +1 life     V : speed x2     T : +30 seconds",
        ]
        self.back_button = ButtonWidget("Back")
        self.elements = [TextWidget("HOW TO PLAY", YELLOW), TextWidget("")]
        for title, lines, color in (("[ CONTROLS ]", controls, WHITE),
                                    ("[ RULES ]", rules, WHITE),
                                    ("[ CHEAT MODE ]", cheats, GREY)):
            self.elements.append(TextWidget(title, YELLOW))
            self.elements += [TextWidget(line, color) for line in lines]
            self.elements.append(TextWidget(""))       # empty line between blocks
        self.elements.append(self.back_button)
        self.calculate_layout()
```

**Fix — wiring** (`game.py`):
```python
# __init__
self.instructions_screen = InstructionScreen(self.instructions_surface,
                                             self.config)
# check_events, MOUSEBUTTONDOWN
elif MazeEnv.state == "INSTRUCTIONS":
    if self.instructions_screen.check_clicks(mouse_pos) == "Back":
        MazeEnv.state = "MENU"
# WINDOWRESIZED: add
self.instructions_screen.calculate_layout()
```
Optional: MOUSEMOTION in INSTRUCTIONS → `self.instructions_screen.check_hovers()` (Back hover color).

**Done when:** readable at 800x600 and full screen, numbers change with config, Back + ESC → menu.

---

### P13. Window size and resize

**Problem:**
- `minimum_size = (1500, 1000)` → the window can't be smaller (test: resize to 800 refused). Many laptops /
  the reviewer's VM are smaller than that.
- Resize re-lays out HUD, main menu and pause only → Instructions, Highscores, Winner, Loser keep the old
  font size (the surface itself is fine, tested).

**Files:** `src/logic/game.py:29`, `game.py:114-126` (WINDOWRESIZED)

**Fix:** `minimum_size = (800, 600)`; in WINDOWRESIZED call `calculate_layout()` on every screen.

---

### P14. Small bugs

| # | Problem | Files | Fix |
|---|---------|-------|-----|
| a | Debug prints: `Game____O___OVER`, name prints | `game.py:275`, `input_widget.py:39,41,43` | delete |
| b | Magic indexes `elements[4]`, `elements[2]` → break if a line is added to the screen | `game.py:177,179,276,305` | keep refs in screens: `self.input`, `self.score_text` |
| c | Typed name stays in the box if the screen is left without Enter (after P2) | `input_widget.py` | clear `text` when entering WINNER/LOSER |
| d | `random.seed()` reseeds global random; only ~1300 seeds → same maze twice possible | `game.py:298` | `self.rng = random.Random()`, skip previous seed |
| e | Dead lives fallback | `game_state.py:22-23` | delete (`reset_game(lives)` does it) |
| f | Hovered button stays highlighted when coming back to the menu | `game.py` | call `check_hovers()` when entering MENU/PAUSE |
| g | `self.y: int` becomes float | `character.py:31-32` | type `float` |
| h | Pac-Man start: only 3 cells tried around center (OK today on all tested mazes) | `pacman.py:13-22` | BFS to nearest cell `!= 15` |
| i | Pac-Man turns back only at cell center (feels laggy) | `character.py:55-76` | opposite key while moving → swap `current_cell`/`target_cell` |
| j | Movement per frame, timer per `dt` → slow PC = slower game, same timer | `character.py`, `game.py` | optional `speed * dt * 60` |
| k | `is_valid_direction` trusts wall bits, no bounds check | `character.py:83-93` | add bounds check |
| l | `pacgum.py` uses `MazeEnv.config` without check → mypy | `pacgum.py:15-17` | `assert MazeEnv.config is not None` |

---

### P15. Dead code / files

**Problem:** unused files and commented code → flake8 errors, confusing at defense.

**Files + fix:**
- Delete files: `src/logic/rules.py` (F401, circular import risk), `src/graphic/screens/test.py`
  (runs at import), `main.py` (uv stub). `src/logic/ghosts_move.py` → use it for BFS (P6).
- Delete commented code: `pacgum.py:38-69`, `character.py:8-27`, `maze_env.py:28-34`, `pacman.py:8-11`,
  `game.py:9,48,54,135,149,157-161`, `input_widget.py:44-47`, `highscores.py:12-15`, `winner.py:10`,
  `text_widget.py:14`.
- Delete empty methods: `game.py:284-286` `start_evil()`, `game.py:314-316` `end_game()`.
- Root `setting.json` (untracked) = duplicate of `src/setting.json` → keep one.

---

### P16. Delivery

**Problem → Fix:**
- No `pac-man.py` at root (subject usage `python3 pac-man.py config.json`) → create it, it calls the `src` main.
- `src/setting.json` has test values (`levels: 3`, `pacgum: 1`) → root `config.json` with `levels: 10`,
  `pacgum: 42`, plus `afraid_time` and `ghost_pwn_time`.
- `Makefile`: `debug` uses `pudb` (not installed) → `python3 -m pdb`; `clean` also removes `*.pyc`;
  `run` → `python3 pac-man.py config.json`.
- Lint: flake8 12, mypy 56 errors → after P15, add type hints.
- README empty, `docs/project_management/` missing, no itch.io link.

**Files:** `pac-man.py` (new), `config.json` (new), `Makefile`, `README.md`, `docs/`

---

## Edge cases list (all re-tested 2026-11-22)

| # | Case | Now | Fix |
|---|------|-----|-----|
| E1 | Win last level + ghost touch same frame | LOSER | P1 |
| E2 | Win last level + timer 0 same frame | life lost after the win | P1 |
| E3 | ESC / SPACE in PAUSE | ignored | P2 |
| E4 | ESC on WINNER / LOSER | ignored, must type a name | P2 |
| E5 | Afraid ghost + pause longer than afraid time | afraid over on resume | P3 |
| E6 | Pac-Man on a corner when an eaten ghost respawns | life lost | P7 |
| E7 | Highscores file `{}`, `null`, `[1]`, missing / string score | crash | P8 |
| E8 | Highscores file broken JSON | replaced by `[]`, scores lost | P8 |
| E9 | Read-only folder | crash at start | P8 |
| E10 | Config `width=5, height=5` only | crash | P9 |
| E11 | Config unknown key / wrong type | crash | P9 |
| E12 | 0 args / 2 args | runs anyway | P10 |
| E13 | Pinky / Inky target outside maze or in "42" | ghost moves dumb (with BFS) | P6 snap |
| E14 | Name of spaces only | saved blank | P8 |
| E15 | `TextWidget.update_text()` before layout | AttributeError | P5 |
| E16 | Window smaller than 1500x1000 | impossible | P13 |
| E17 | Resize on Instructions / Highscores / end screens | old font size | P13 |

Checked OK: timer 0 → 1 life lost + timer back to 90 ✓; Start builds the level once ✓; WINNER screen
stays (level doesn't climb) ✓; new game → full lives ✓; all maze cells reachable from start
(seeds 1-30, sizes 5..40) ✓; ghosts never cross walls ✓; collision can't be skipped at current speeds ✓.

---

## How the game works now

```
make run → uv run -m src src/setting.json
   ├─ src/__main__.py   Config.from_json_file(path) → Game(config)
   └─ Game.__init__ (src/logic/game.py)
        ├─ new_maze(config.seed)  MazeGenerator(size, seed, perfect=False)
        ├─ window: HUD left 30 %, maze right 70 %
        ├─ screens: MainMenu, Instructions, Winner, Loser, HighScores, PauseMenu
        ├─ start_new_game() → reset_game(config.lives) + new_maze + load_level()
        └─ run()  60 fps loop on MazeEnv.state

States:
   MENU --Start--> PLAYING --ESC/SPACE--> PAUSE --Resume (click)--> PLAYING
   PAUSE --Main menu--> MENU
   PLAYING --last level done--> WINNER --name + Enter--> MENU
   PLAYING --lives 0--> LOSER --name + Enter--> MENU
   MENU --Highscores / Instructions--> screen --ESC--> MENU
   MENU --Exit / window X--> quit (only way to close)

Frame in PLAYING: keys → pacman.move → eat_pacgum → (no gums? level_win)
                  → ghosts move → collision → timer → HUD → draw
Maze cell bits: 1 N, 2 E, 4 S, 8 W, 15 = closed ("42" logo)
```

---

## Final checklist before defense

- [ ] `python3 pac-man.py config.json` works; 0 / 2 args → usage message
- [ ] Bad config (comment, unknown key, `"lives": -4`, `"width": "abc"`, 5x5, missing file) → message, no traceback
- [ ] Menu → play → win/lose → name → menu, many times, full lives each new game
- [ ] Lives + score kept between levels; 10 levels by default
- [ ] Pause with ESC and SPACE, game visible behind, afraid timer frozen
- [ ] Ghosts reach a still Pac-Man in every maze (BFS); no spawn kill
- [ ] Highscores: delete file / garbage / `{}` → no crash, list refreshes after a game
- [ ] Instructions: 3 blocks (Controls / Rules / Cheat mode), numbers from config
- [ ] Cheat keys work and match Instructions + README
- [ ] `make lint` clean
- [ ] README + docs/project_management + itch.io link

---

## DONE log

| Date | Commit | What |
|------|--------|------|
| 2026-11-22 | *uncommitted* | Timer typo fixed (`GameState.time_left`, tested: 1 life lost, timer back to 90); pacgum check moved inside PLAYING + `return` back in `level_win` (WINNER stays, no extra level); Start builds the level once; `time_left: float` |
| 2026-11-22 | `1e724e6` | Lives reset on new game; keyboard works in game again (hover under `MOUSEMOTION`); ESC pauses instead of quitting; hover re-renders only on change, no prints; main menu title |
| 2026-11-20 | `e051b69` | Pause menu (Resume / Main menu), SPACE pauses, nothing moves while paused |
| 2026-11-18 | `87241d6` | Highscores screen reloads the file, survives missing file / broken JSON; button hover color |
| 2026-11-18 | `84f203c` | Ghost into walls after super gum fixed; lives kept between levels |
| ≤ `9f69bc7` | | Level 1 = `config.seed`; mazegenerator wheel; Winner/Loser + name input + top-10 save; menu buttons |
