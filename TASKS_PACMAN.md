# Pac-Man project — TASKS

Reviewed: 2026-11-18, code on **`origin/main` @ `84f203c`** ("fix: lives and main checker also ghosts glitch").
All `file:line` below = that commit. Fixed items are removed from this file (see DONE log at the end).

> **Before you start your try:** local `zak_features` (@ `f1bf2b3`) is **4 commits behind `origin/main`**.
> `origin/main` moved `setting.json` → `src/setting.json` and replaced the `mazegenerator/` folder by the wheel.
> 1. `git fetch && git merge origin/main` (on `zak_features`).
> 2. `uv sync` (installs the wheel; without it `import mazegenerator` fails).
> 3. `make run` (uses `src/setting.json`).

---

## PROGRESS

| # | Task | Status | Open |
|---|------|--------|------|
| 2 | Pacgums: placement, eating, score, HUD | DONE | |
| 3 | Ghost collision, lives, respawn | DONE | |
| 4 | Afraid mode, eat ghost, ghost respawn | DONE | |
| 5 | Levels, timer, win/lose | 2 bugs | P2 lives on new game, P3 timer typo |
| — | Ghost chase quality | TODO | Section B (BFS) |
| 6 | State machine (Menu → Game → End → Name → Menu) | PARTIAL | P5 (double load, pacgum check), Enum, PAUSED |
| 7 | Pause menu (Resume / Main menu) | TODO | P6 |
| 10 | Cheat mode | TODO | Section C |
| 8 | Highscores JSON | PARTIAL | P7 (robust load, refresh, config filename) |
| 9 | Winner / Loser / Instructions screens | PARTIAL | P9 (instructions text) |
| 1 | Config loader | PARTIAL | P10 |
| — | No traceback / argc check | TODO | P8 |
| 11 | Entry point, Makefile, packaging, itch.io | PARTIAL | No `pac-man.py`, `debug` uses `pudb`, `clean` incomplete |
| 12 | Lint: flake8 + mypy strict | TODO (at end) | + `ghost.py:168` 82 chars (E501) |
| 14 | README + `docs/project_management/` | TODO | README empty |

Order of remaining work:
**P3 + P2 → P5 → BFS chase (section B) → P6 pause → C cheat → P7 → P9 → P10 → P8 → 11 → P12 cleanup → 12 lint → 14**

---

## PROBLEMS — what, where, fix

### P2. Lives not reset on new game

**Where:** `src/logic/game_state.py:22-23` `setup()`: `if cls.lives == 0: cls.lives = config.lives`;
`game_state.py:32` `reset_game()`: lives line commented out (`MazeEnv.config` is `None` on the very first call).

**Bug:** lives only refill when exactly 0. ESC mid-game with 1 life → Start → new game with **1 life**.
Win the game with 2 lives → next game starts with **2 lives**.

**Fix:** give the value to `reset_game`, keep `setup` away from lives:
```python
# game_state.py
@classmethod
def reset_game(cls, lives: int) -> None:
    cls.score = 0
    cls.lives = lives
    cls.level = 1
    cls.cheat_mode = False

# setup(): delete the 2 lives lines

# game.py start_new_game()
GameState.reset_game(self.config.lives)
```

### P3. Timer at 0 → all lives lost in 3 frames

**Where:** `src/logic/game.py:256` `self.time_left = float(self.config.level_max_time)`.
Typo: `self.time_left` instead of `GameState.time_left` → `GameState.time_left` stays ≤ 0 → `lose_life()` every frame.

**Fix:**
```python
GameState.time_left = float(self.config.level_max_time)
```
Also `game_state.py:16` `time_left: int` → `float`.

### P5. Start loads level twice / level check runs in every state

**Where:**
- `game.py:120-123` click "Start": `start_new_game()` (already calls `load_level()`) then `self.load_level()` again
  → pacgums placed twice, wasted work.
- `game.py:93-94` `if not GameState.pacgums: self.level_win()` runs in **every** state (menu, screens...).

**Fix:** click Start → `self.start_new_game(); MazeEnv.state = "PLAYING"` (delete extra `load_level()`).
Move the pacgum check inside the `"PLAYING"` branch, right after `self.pacman.eat_pacgum()`.

Later (task 6 clean): `class State(Enum)` with MENU, PLAYING, PAUSED, WINNER, LOSER, HIGHSCORES, INSTRUCTIONS.
Replace all string states (game.py, input_widget.py:38, maze_env.py:13).

### P6. No pause — ESC kills the game (subject VI.7 + VI.8: pause menu mandatory)

**Where:** `game.py:131-132` ESC → `"MENU"` from any state.

**Fix (task 7):**
1. ESC in PLAYING → `"PAUSED"`. ESC in PAUSED → `"PLAYING"`. ESC in other screens → `"MENU"`.
2. New `src/graphic/screens/pause.py`: `MenuWidget` with buttons "Resume", "Main menu". Draw it over the
   last game frame (don't call `move()` when PAUSED).
3. Timers: ghost afraid / eaten / scatter use `pygame.time.get_ticks()` → keep running while paused.
   Simplest: store pause start, on resume add pause duration to each ghost
   (`afraid_until`, `pwn_time`, `last_mode_switch`). Cleaner: count with `dt` like the level timer.

### P7. Highscores not robust / not refreshed

**Where:**
- `src/graphic/screens/highscores.py:17-23`: `open` + `json.load` no try → **crash at startup** if file missing or broken
  (screen is built in `Game.__init__`).
- Read only once in `__init__` → new score not shown until restart.
- `src/logic/save_scores.py:8-10`: same, no try; no check each entry has `name`/`score`.
- Filename hard-coded `"highscores.json"`, config `highscore_filename` ignored (default `"scores"`).
- `highscores.json` committed with test names.

**Fix:**
1. In `save_scores.py` add one loader used by both:
   ```python
   def load_scores(filename: str) -> list[dict[str, object]]:
       try:
           with open(filename, encoding="utf-8") as f:
               data = json.load(f)
       except (OSError, json.JSONDecodeError):
           return []
       if not isinstance(data, list):
           return []
       return [e for e in data
               if isinstance(e, dict) and isinstance(e.get("name"), str)
               and isinstance(e.get("score"), int) and e["score"] >= 0][:10]
   ```
   `save_score` uses it + wraps the write in `try/except OSError` (print message, no crash).
2. `HighScoresScreen.refresh()` rebuilds `elements` + `calculate_layout()`. Call it when the menu button
   "Highscores" is clicked. Show "No scores yet" when empty.
3. Pass `config.highscore_filename` everywhere (add `.json` if no extension).
4. `.gitignore` → add `highscores.json`, remove it from repo (`git rm --cached highscores.json`).

### P8. Tracebacks still possible (subject V.1: never a traceback)

**Where:** `src/__main__.py:12-26` — try/except commented out. `Config` errors, maze errors, game errors → traceback.
Also accepts 0 args (subject: exactly 1).

**Fix:**
```python
if len(sys.argv) != 2:
    print("Usage: python3 pac-man.py config.json", file=sys.stderr)
    sys.exit(1)
try:
    config = Config.from_json_file(sys.argv[1])
    game = Game(config)
except KeyboardInterrupt:
    sys.exit(0)
except Exception as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
```
And move `self.run()` out of `Game.__init__` (`game.py:48`) → `game.run()` inside the try.

### P9. Instructions screen (placeholder text + broken layout)

**Where:** `src/graphic/screens/instructions.py` + `src/graphic/menus/menu_widget.py` + `src/logic/game.py`.

**Problems now:**
1. Text = "Hellooooo", "Its meeeee", "what do you want?" (l.15-17). Subject VI.8: must show **controls and rules**.
2. Uses layout `"HUD"` (l.8) → font = `width * 0.10` of the **whole window** (~150 px) → long lines go off screen.
3. l.10-13 copies score/lives/level → unused, delete.
4. No way back except ESC, and the player isn't told.
5. Not re-laid out on window resize (`game.py:104-114` only HUD + main menu).

**Target look** (dark bg, yellow titles, white text, ghost names in their colors, button at bottom):
```
                 HOW TO PLAY

                  CONTROLS
          Arrows / WASD   move Pac-Man
          ESC             pause / back

                   RULES
          Pacgum           +10 pts
          Super pacgum     +50 pts, ghosts turn blue (7 s)
          Blue ghost       +200 pts, comes back after 5 s
          Touch a ghost    lose 1 life (3 lives)
          Eat all pacgums  next level  (10 levels)
          Timer 90 s       time out = lose 1 life

                  GHOSTS
     Blinky chases you      Pinky cuts you off
     Inky follows close     Clyde is shy when near

            CHEAT MODE (press C)
     I invincible  N skip level  F freeze ghosts
     L +1 life     V speed x2    T +30 s

                  [ Back ]
```
Numbers come from **config** (reviewer will change them at defense → screen must follow).

**Step 1 — layout for this screen** (`menu_widget.py`, in `calculate_layout`, add a branch):
```python
elif self.menu_name == 'INSTRUCTIONS':
    menu_width, menu_height = self.surface.get_size()
    # ~26 lines must fit: limit by height too, not only width
    dynamic_font_size = max(14, min(int(menu_width * 0.022),
                                    int(menu_height * 0.032)))
    self.gap = int(dynamic_font_size * 0.15)
```

**Step 2 — the screen** (`src/graphic/screens/instructions.py`, replace all):
```python
from ..menus.menu_widget import MenuWidget
from ..widgets.text_widget import TextWidget
from ..widgets.button_widget import ButtonWidget

YELLOW = (255, 220, 0)
WHITE = (230, 230, 230)
GREY = (150, 150, 150)
RED = (255, 0, 0)
PINK = (255, 184, 255)
CYAN = (70, 191, 238)
ORANGE = (240, 131, 0)


class InstructionScreen(MenuWidget):
    """Controls, rules, ghosts and cheat keys. Values read from config."""

    def __init__(self, surface, config) -> None:
        super().__init__(surface, "INSTRUCTIONS")
        self.bg_color = (0, 0, 0)
        self.back_button = ButtonWidget("Back")
        c = config
        self.elements = [
            TextWidget("HOW TO PLAY", YELLOW),
            TextWidget(""),
            TextWidget("CONTROLS", YELLOW),
            TextWidget("Arrows / WASD : move Pac-Man", WHITE),
            TextWidget("ESC : pause / back", WHITE),
            TextWidget(""),
            TextWidget("RULES", YELLOW),
            TextWidget(f"Pacgum : +{c.points_per_pacgum} pts", WHITE),
            TextWidget(f"Super pacgum : +{c.points_per_super_pacgum} pts,"
                       f" ghosts turn blue for {c.afraid_time} s", WHITE),
            TextWidget(f"Blue ghost : +{c.points_per_ghost} pts,"
                       f" back after {c.ghost_pwn_time} s", WHITE),
            TextWidget(f"Touch a ghost : lose 1 life"
                       f" (you have {c.lives})", WHITE),
            TextWidget(f"Eat all pacgums : next level"
                       f" ({c.levels} levels)", WHITE),
            TextWidget(f"Timer {c.level_max_time} s :"
                       f" time out = lose 1 life", WHITE),
            TextWidget(""),
            TextWidget("GHOSTS", YELLOW),
            TextWidget("Blinky chases you", RED),
            TextWidget("Pinky cuts you off", PINK),
            TextWidget("Inky follows close behind", CYAN),
            TextWidget("Clyde runs home when near", ORANGE),
            TextWidget(""),
            TextWidget("CHEAT MODE (press C)", YELLOW),
            TextWidget("I invincible   N skip level   F freeze ghosts",
                       GREY),
            TextWidget("L +1 life   V speed x2   T +30 s", GREY),
            TextWidget(""),
            self.back_button,
        ]
        self.calculate_layout()
```
Note: `TextWidget("")` = empty line spacer (checked: pygame-ce renders `""` as 0 x font-height surface, layout works).
Keep the cheat lines in sync with section C (only list keys that really exist).

**Step 3 — wire it** (`game.py`):
1. Build: `InstructionScreen(self.instructions_surface, self.config)`.
2. Click in INSTRUCTIONS state (in `check_events`, MOUSEBUTTONDOWN):
   ```python
   elif MazeEnv.state == "INSTRUCTIONS":
       if self.instructions_screen.check_clicks(mouse_pos) == "Back":
           MazeEnv.state = "MENU"
   ```
3. Resize: add `self.instructions_screen.surface = self.window.get_surface()` +
   `self.instructions_screen.calculate_layout()` (same for winner/loser/highscores, P11-c).

**Same idea for the other screens (cheap UI wins):**
- Main menu: add title "PAC-MAN" in yellow above the buttons (`TextWidget("PAC-MAN", YELLOW)` first element)
  + small grey line at bottom "ESC = back / pause".
- Highscores: title "TOP 10" yellow, rank 1 in yellow, others white, "No scores yet" if empty, "Back" button.
- Winner / Loser: black bg like the menu, score line yellow, under the input: "Type your name + Enter"
  (grey) so the player knows what to do (P11-e).
- One shared colors file `src/graphic/colors.py` (YELLOW, WHITE, GREY, ghost colors) → reuse in ghost.py too.

**Done when:** Instructions readable at 800x600 and full screen, numbers change when config changes,
Back button + ESC return to menu, resize keeps it fitting.

### P10. Config loader crashes on reviewer edits (task 1, subject V.3)

**Where:** `src/parser/load_config.py`
- l.13 `extra='forbid'` → unknown key crashes. Must be ignored → `extra='ignore'`.
- l.13 `strict=True` + whole-model validate (l.58) → one bad value rejects everything. Must clamp/default + message.
- l.29-35 `check_consistency` raises → remove (clamp pacgum at placement, already done with `min()`).
- l.50-56 missing file / bad JSON → `ConfigError` (traceback while P8 not fixed). Choose: defaults + message.
- `width/height` no max → huge maze = generator crash/slow. Clamp e.g. 5..40.
- Comment lines removed → JSON error line numbers wrong. Replace comment line by `"\n"` instead of dropping.

**Fix:** per-key validation loop: for each field, try validate alone; on error print
`[config] 'lives'=-4 invalid, using default 3`, keep default. Print `[config] unknown key 'x' ignored`.

### P11. Other bugs / risks

| # | Where | Problem | Fix |
|---|-------|---------|-----|
| a | several | Debug prints left | Delete: `input_widget.py:35-39`, `highscores.py:14`, `game.py:239`, commented `game.py:52` |
| b | `game.py:28` | `minimum_size = (1500, 1000)` > many screens / reviewer VM | `(800, 600)` |
| c | `game.py:104-114` | Resize only re-layouts HUD + main menu. Winner/loser/highscores/instructions keep old font size; their surfaces are the old `window.get_surface()` | On resize: `self.screen = ...` and give every screen the new surface + `calculate_layout()` |
| d | `game.py:147-150, 240-241, 269-270` | Magic indexes `elements[4]`, `elements[2]` | Keep refs: `self.input = InputWidget()`, `self.score_text = ...` in screens |
| e | `input_widget.py:31-38` | Empty name → nothing happens, no hint; ESC leaves without saving silently | Show "Type a name + Enter". OK to keep ESC = skip |
| f | `game.py:262` | `random.seed()` reseeds global random | Use own `self.rng = random.Random()` for level seeds |
| g | `game_state.py:59` | Level-1 pacgums placed with unseeded random → differ each run | Optional: `random.Random(config.seed)` for level 1 |
| h | `pacgum.py:15-17` | `MazeEnv.config` may be `None` → mypy error | `assert MazeEnv.config is not None` |
| i | `character.py:31-32` | `self.y: int` but becomes float | type `float` |
| j | `pacman.py:13-22` | Start fallback only tries 3 cells around center (closed `15` "42" logo cells) | BFS from center to nearest cell `!= 15` |
| k | `pacman.py:6` | `config` param unused | remove or use for speed |
| l | `src/setting.json` | Test values committed: `pacgum: 1`, `levels: 3` | `pacgum: 42`, `levels: 10`; add `afraid_time`, `ghost_pwn_time`; rename `config.json` at root (subject usage) |
| m | `game.py:20-48` | `Game.__init__` calls `start_new_game()` and `run()` | `run()` out (P8) |
| n | `character.py:55-76` | Pacman can only turn back at a cell center (feels laggy) | Optional: if `new_direction` is opposite of `direction` while moving → swap `current_cell`/`target_cell` + flip direction (same trick as ghost afraid fix) |

### P12. Dead files / cleanup

- `src/logic/rules.py` — only `from .game import Game` → circular import risk. Delete.
- `src/graphic/screens/test.py` — test script, runs code at import. Delete.
- `src/logic/ghosts_move.py` — empty. Delete (or put BFS here, section B).
- `main.py` — `uv init` stub. Replace by `pac-man.py` (task 11).
- `pacgum.py:38-69` — old commented `Pacgums` class. Delete.
- `character.py:8-27`, `maze_env.py:28-34`, `pacman.py:8-11`, `game.py:8,133-137` — commented code. Delete.
- `game.py:248-250` `start_evil()` and `278-280` `end_game()` — empty. Delete.

---

## A. How ghosts chase Pac-Man today (`ghost.py`)

Every time a ghost reaches a cell center (`get_target_cell` → `next_direction`, l.82-129):

1. **Pick a target cell** by mode:

   | Mode | Target |
   |------|--------|
   | Chase (20 s) | Blinky (red) = Pacman cell. Pinky (pink) = 4 cells ahead of Pacman. Inky (cyan) = 2 cells ahead. Clyde (orange) = Pacman if far (> 8 cells), else his corner |
   | Scatter (5 s) | own corner |
   | Afraid | Pacman cell, distance negated → picks the move **farthest** from Pacman |
   | Eaten | invisible, no move; after `ghost_pwn_time` s → reset to corner |

2. **List open directions** (no wall bit) except U-turn.
3. **Greedy choice**: for each, next cell → straight-line distance² to target → take smallest.

**Weak points:**
- Straight-line distance ignores walls. Ghost goes toward Pacman "as the crow flies" → in dead ends / U-shapes
  it loops around a block and never reaches Pacman. Non-perfect maze helps but not always.
- 4 ghosts often follow the same path (all greedy on nearly same cell).
- **Measured (2026-11-18, headless, 30 mazes 20x20 seeds 1-30, Pacman standing still at center, chase only):**
  greedy Blinky reached Pacman in **10/30** mazes within 2 min (avg 15.0 s). Other 20: loops forever.
  Same test with BFS (section B): **30/30**, avg 8.8 s, max 13.8 s.
- Modes timed with `get_ticks()` → keep running in pause (P6).

## B. Better chase: BFS distance (recommended, ~25 lines)

Keep the same structure, only change **how distance is measured**: real path length through corridors.

```python
from collections import deque

WALLS = [(1, -1, 0), (2, 0, 1), (4, 1, 0), (8, 0, -1)]   # bit, dy, dx (N, E, S, W)

def bfs_distances(maze: list[list[int]],
                  start: tuple[int, int]) -> dict[tuple[int, int], int]:
    """Path length from start to every reachable cell."""
    rows, cols = len(maze), len(maze[0])
    y0 = min(max(start[0], 0), rows - 1)       # Pinky/Inky targets can be outside maze
    x0 = min(max(start[1], 0), cols - 1)
    dist = {(y0, x0): 0}
    queue = deque([(y0, x0)])
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
In `next_direction()`:
```python
dist_map = bfs_distances(MazeEnv.maze, (target_y, target_x))
...
d_val = dist_map.get((test_y, test_x), 10_000)     # unreachable (e.g. "42" cell) = far
if self.is_afraid:
    d_val = -d_val                                  # run away = maximize path length
```
Cost: 20x20 = 400 cells, run only at cell centers → nothing for pygame.
Defense explanation: "each ghost targets a cell (personality), then takes the corridor with the shortest real path."

Optional variety: afraid ghosts pick a **random** open direction (classic behavior, easier to catch).

## C. Cheat mode (task 10, subject VI.5 — must really help the reviewer)

**Where:** `game.py` `check_events()` (PLAYING branch), `game_state.py`, `hud_menu.py`, `ghost.py`.

| Key | Effect | How |
|-----|--------|-----|
| `C` | Toggle cheat mode ON/OFF | `GameState.cheat_mode = not ...`; HUD shows "CHEAT ON". Other cheat keys work only when ON |
| `I` | Invincible | Already used: `check_ghost_collision` skips `lose_life()` if `cheat_mode` (l.231). Make it its own flag `GameState.invincible` |
| `N` | Skip level | `GameState.pacgums.clear()` → normal `level_win()` path runs |
| `F` | Freeze ghosts | `GameState.ghosts_frozen = not ...`; in `run()` skip `ghost.move()` when frozen |
| `L` | +1 life | `GameState.lives += 1` |
| `V` | Pacman speed x2 toggle | `self.pacman.speed = 0.2 / 0.1` (keep values that divide 1.0 cleanly: 0.1, 0.2, 0.25, 0.5) |
| `T` | +30 s timer | `GameState.time_left += 30` |

Also: timer `T` + skip `N` let reviewer test WINNER screen in seconds; `L` + invincible test LOSER with `lives` config.
`reset_game()` already turns cheat off (l.33) — also reset the new flags there.
List all keys on the Instructions screen + README.

---

## D. How the game works now (origin/main)

```
make run → uv run -m src src/setting.json
   │
   ├─ src/__main__.py       Config.from_json_file(path) → Game(config)
   │
   └─ Game.__init__  (src/logic/game.py)
        ├─ new_maze(config.seed)     MazeGenerator(size, seed, perfect=False).maze  (wheel package)
        ├─ pygame window: HUD left 30%, maze right 70%, cellsize fits
        ├─ build screens: MainMenu, Instructions, Winner, Loser, HighScores
        ├─ start_new_game()  → load_level() + GameState.reset_game()
        └─ run()             loop 60 fps, dispatch on MazeEnv.state (starts "MENU")

Shared state (class attributes, no instances):
   MazeEnv     maze, surface, offsets, cellsize, config, state   (src/logic/maze_env.py)
   GameState   pacman, ghosts, pacgums, score, lives, level, time_left, cheat_mode  (src/logic/game_state.py)

load_level(): MazeEnv.setup → Pacman (center) + 4 ghosts (corners) → GameState.setup
              → set_pacgums (4 super in corners + N random) → time_left = level_max_time

Frame in "PLAYING":
   check_events → pacman.move → pacman.eat_pacgum (+score, super → ghosts afraid)
   → ghosts move (AI section A) → check_ghost_collision → update_timer → HUD → draw all

Rules:
   gum +10 | super +50 + afraid (afraid_time s) | afraid ghost +200, respawn after ghost_pwn_time s
   normal ghost → -1 life, everyone back to start | timer 0 → -1 life, timer reset
   no gums left → level_win: level+1, new maze (random seed), same score
   level > levels → "WINNER" screen | lives 0 → "LOSER" screen
   Winner/Loser: type name (≤10, alnum + space) + Enter → save_score (top 10) → "MENU"

Maze cell = int, bit = wall: 1 N, 2 E, 4 S, 8 W, 15 = closed ("42" logo)
```

Module map:
```
src/
├─ __main__.py
├─ setting.json                  test values inside (P11-l)
├─ parser/load_config.py         Config (pydantic)
├─ logic/
│   ├─ game.py                   loop, events, levels, collisions, timer
│   ├─ game_state.py             score/lives/level/pacgums (shared)
│   ├─ maze_env.py               maze/surface/config/state (shared)
│   ├─ save_scores.py            top-10 JSON save
│   ├─ rules.py, ghosts_move.py  dead (P12)
└─ graphic/
    ├─ maze_graphic.py
    ├─ characters/  init.py (enums), character.py, pacman.py, ghost.py
    ├─ pacgums/pacgum.py
    ├─ menus/       menu_widget.py, main_menu.py, hud_menu.py
    ├─ screens/     winner.py, loser.py, highscores.py, instructions.py, test.py (dead)
    └─ widgets/     text_widget.py, button_widget.py, input_widget.py

To add: screens/pause.py, pac-man.py, config.json (root), docs/project_management/, README content
```

---

## Final checklist before defense

- [ ] `python3 pac-man.py config.json` works; 0 args / 2 args → usage message
- [ ] Config with comment, unknown key, `"lives": -4`, `"width": "abc"`, missing file → starts, message, no traceback
- [ ] Generator error → clean message
- [ ] Menu → play → win/lose → name → menu, many times without restart
- [ ] Lives + score kept between levels; ≥ 10 levels default
- [ ] Pause (Resume / Main menu); afraid timer does not run out during pause
- [ ] Ghost never crosses walls (eat super gum mid-chase many times)
- [ ] Highscores: delete file / write garbage in it → no crash, list refreshes after a game
- [ ] Cheat keys documented in Instructions + README
- [ ] `make lint` clean (flake8 + mypy strict)
- [ ] README + docs/project_management + itch.io link

---

## DONE log (removed from problems list)

| Date | Commit | What |
|------|--------|------|
| 2026-11-18 | `84f203c` | Ghost went into walls after super gum → `afraid_of` swaps `current_cell`/`target_cell` |
| 2026-11-18 | `84f203c` | WINNER: `return` restored in `level_win`, no extra level built |
| 2026-11-18 | `84f203c` | `start_new_game`: `reset_game()` now before `load_level()` |
| 2026-11-18 | `84f203c` | Lives no longer reset every level (but see P2) |
| 2026-11-18 | `84f203c` | `print("State: ...")` every frame removed |
| ≤ `9f69bc7` | | Level 1 = `config.seed`, `level = 1`, maze `None` → RuntimeError, `dt` reset per level, seed range never crashes |
| ≤ `9f69bc7` | | Mazegenerator installed as wheel (folder removed), Winner/Loser screens + name input + top-10 save, Start/Highscores/Instructions/Exit buttons |
