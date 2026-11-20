# Pac-Man project — TASKS

Reviewed: 2026-11-20, code on **`main` @ `425d435`** ("Merge pull request #3 … pause_menu").
All `file:line` below = that commit. Every bug marked **(tested)** was reproduced headless
(SDL dummy driver, fake clock), not only read. Fixed items are removed from this file (see DONE log at the end).

---

## PROGRESS

| # | Task | Status | Open |
|---|------|--------|------|
| 2 | Pacgums: placement, eating, score, HUD | DONE | |
| 3 | Ghost collision, lives, respawn | DONE | spawn kill (E3) |
| 4 | Afraid mode, eat ghost, ghost respawn | DONE | timers run during pause (P6) |
| 5 | Levels, timer, win/lose | 3 bugs | P2 lives, P3 timer typo, P4 `return` removed again |
| 6 | State machine (Menu → Game → End → Name → Menu) | PARTIAL | P1 keys dead in MENU/PAUSE, P5 double load + pacgum check, Enum |
| 7 | Pause menu (Resume / Main menu) | PARTIAL | P6: ESC does not pause, timers keep running, game hidden, hover spam |
| — | Ghost chase quality | TODO | Section G (BFS + fixes) |
| 10 | Cheat mode | TODO | Section C |
| 8 | Highscores JSON | PARTIAL | P7: 10 crash cases, file read every frame, config filename ignored |
| 9 | Winner / Loser / Instructions screens | PARTIAL | P9 instructions text |
| 1 | Config loader | PARTIAL | P10 |
| — | No traceback / argc check | TODO | P8 |
| 11 | Entry point, Makefile, packaging, itch.io | PARTIAL | no `pac-man.py`, `debug` uses `pudb`, `clean` incomplete, test values in config |
| 12 | Lint: flake8 + mypy strict | TODO (at end) | today: flake8 **9**, mypy **55 errors / 18 files** |
| 14 | README + `docs/project_management/` | TODO | README empty |

---

## ROADMAP (do in this order)

**Phase 0 — game-breaking, small (≈1 h)**
1. P3 timer typo (1 line).
2. P2 lives reset on new game.
3. P1 keyboard dead in MENU/PAUSE (indentation of the hover `if`).
4. P4 + P5: pacgum check inside `"PLAYING"`, put `return` back in `level_win`, delete extra `load_level()`.

**Phase 1 — finish pause (task 7)**
5. P6: ESC toggles pause, freeze ghost timers, draw game under the menu, hover only on mouse move.

**Phase 2 — ghosts (section G)**
6. BFS distance + target snapping (G2).
7. Spawn-kill guard (E3) + global, pause-safe mode timer (G3).
8. Optional polish: Inky classic target, staggered release, afraid = random (G4).

**Phase 3 — robustness (subject: never crash)**
9. P7 highscores loader + refresh only on enter + config filename.
10. P10 config loader per-key defaults.
11. P8 `__main__` argc + try/except, `run()` out of `__init__`.

**Phase 4 — content**
12. C cheat mode → 13. P9 instructions + screens polish → 14. P11-c resize all screens.

**Phase 5 — delivery**
15. Task 11 (`pac-man.py`, root `config.json`, Makefile, real config values) → 16. P12 cleanup
→ 17. Lint → 18. README + docs + itch.io.

---

## PROBLEMS — what, where, fix

### P1. Keys ignored in MENU and PAUSE **(tested)** — NEW, from pause PR

**Where:** `src/logic/game.py:149-154`. The new hover block was added at the wrong level:
```python
            if MazeEnv.state == "MENU":          # <- new `if` chain inside the event loop
                self.main_menu.check_hovers()
            elif MazeEnv.state == "PAUSE":
                self.pause_menu.check_hovers()

            elif event.type == pygame.KEYDOWN:   # <- now belongs to the `if state` chain!
```
→ KEYDOWN only handled when state is **not** MENU and **not** PAUSE.
Test: SPACE → PAUSE, then ESC → still PAUSE, SPACE → still PAUSE. Only the mouse can leave pause.
Also `check_hovers()` runs for **every** event (keys, resize…), re-renders every button and prints
`"We have a hover heeeere"` / `"…noooo hover…"` (`button_widget.py:47,51`) → terminal spam.

**Fix:**
```python
            elif event.type == pygame.MOUSEMOTION:
                if MazeEnv.state == "MENU":
                    self.main_menu.check_hovers()
                elif MazeEnv.state == "PAUSE":
                    self.pause_menu.check_hovers()
            elif event.type == pygame.KEYDOWN:
                self.handle_key(event)          # move the key code into its own method
```
`button_widget.py:44-53`: delete the 2 prints, re-render only if `is_hovered` changed:
```python
def check_hover(self):
    hovered = bool(self.rect and self.rect.collidepoint(pygame.mouse.get_pos()))
    if hovered != self.is_hovered:
        self.is_hovered = hovered
        self.render_button()
```

### P2. Lives not reset on new game **(tested)**

**Where:** `src/logic/game_state.py:22-23` `setup()`: `if cls.lives == 0: cls.lives = config.lives`;
`game_state.py:32` `reset_game()`: lives line commented out.

**Bug:** lives refill only when exactly 0. Test: lives=1 → Main menu → Start → **1 life**.
Win with 1 life left → next game starts with **1 life**.

**Fix:**
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

### P3. Timer at 0 → all lives lost in 3 frames **(tested)**

**Where:** `src/logic/game.py:287` `self.time_left = …` → typo, must be `GameState.time_left`.
Test: `time_left = 0.01` → 3 frames later lives 3 → 0, LOSER.

**Fix:** `GameState.time_left = float(self.config.level_max_time)`. Also `game_state.py:16` `time_left: int` → `float`.

### P4. `return` in `level_win` removed again (regression from `84f203c`)

**Where:** `game.py:302` `# return`.
Why it was removed: the pacgum check (`game.py:102-103`) runs in **every** state, so with `return` the empty
pacgum dict calls `level_win()` every frame on the WINNER screen (level 3, 4, 5…). Removing `return`
"fixes" it by building one more useless level (new maze + ghosts + pacgums) behind the WINNER screen.
Side effect: if that extra maze fails, state is set to `"MENU"` → WINNER screen skipped, score not saved.

**Fix:** with P5 (check only in PLAYING) put `return` back.

### P5. Start loads level twice / pacgum check in every state **(tested: 2 `load_level` calls per Start)**

**Where:** `game.py:131-134` `start_new_game()` (already calls `load_level()`) then `self.load_level()` again.
`game.py:102-103` `if not GameState.pacgums: self.level_win()` outside the PLAYING branch.

**Fix:** Start → `self.start_new_game(); MazeEnv.state = "PLAYING"`. Move the pacgum check right after
`self.pacman.eat_pacgum()` in the `"PLAYING"` branch.
Later (task 6 clean): `class State(Enum)` MENU, PLAYING, PAUSE, WINNER, LOSER, HIGHSCORES, INSTRUCTIONS.
Replace all string states (game.py, input_widget.py:42, maze_env.py:13).

### P6. Pause menu: what's left (task 7, subject VI.7/VI.8) **(tested)**

Done in `e051b69`: `PauseMenu` (Resume / Main menu), SPACE pauses, ghosts/timer don't move while paused.

| # | Problem | Where | Fix |
|---|---------|-------|-----|
| a | ESC in PLAYING quits to MENU (game lost), doesn't pause | `game.py:155-156` | ESC: PLAYING → PAUSE, PAUSE → PLAYING, WINNER/LOSER → MENU (skip save), others → MENU. Keep SPACE as 2nd key |
| b | Ghost timers keep running: afraid 7 s, paused 9.6 s, resume → afraid gone | `ghost.py:14,164,176,180,189` use `get_ticks()` | see below |
| c | Pause menu covers whole window with near-white `(250,250,250)` → game hidden | `pause_menu.py:8`, `game.py:191` | see below |
| d | Hover spam | P1 | P1 |
| e | Resume by click only (P1) | P1 | P1 |

**b — freeze timers (simple):**
```python
# ghost.py
def shift_timers(self, ms: int) -> None:
    self.afraid_until += ms
    self.pwn_time += ms
    self.last_mode_switch += ms

# game.py — use for EVERY way in / out of pause (button, ESC, SPACE)
def pause(self) -> None:
    self.pause_start = pygame.time.get_ticks()
    MazeEnv.state = "PAUSE"

def resume(self) -> None:
    paused = pygame.time.get_ticks() - self.pause_start
    for ghost in self.ghosts:
        ghost.shift_timers(paused)
    MazeEnv.state = "PLAYING"
```
Cleaner (with G3): count all ghost timers in seconds with `dt`, like the level timer → pause is free.

**c — see the game under the menu:**
```python
# calculate_dimensions
self.pause_menu_surface = pygame.Surface((w, h), pygame.SRCALPHA)
# pause_menu.py
self.bg_color = (0, 0, 0, 160)
# run(), "PAUSE" branch: redraw the frozen game first (no move), then the overlay
self.draw_game()                  # = the draw half of the PLAYING branch, extracted
self.pause_menu.draw()
self.screen.blit(self.pause_menu_surface, (0, 0))
```
Without `draw_game()` each frame blends black on the old frame → goes fully black in < 1 s.

### P7. Highscores **(tested: 10 crash cases)**

`87241d6` made the screen reload the file and survive a missing file / bad JSON. Still:

| Content of `highscores.json` | Screen | `save_score` |
|---|---|---|
| `{"a": 1}` | CRASH TypeError | CRASH AttributeError |
| `[{"name": "x"}]` | CRASH KeyError | CRASH KeyError |
| `[1, 2]` | CRASH TypeError | CRASH TypeError |
| `[{"name":"x","score":"10"}]` | CRASH TypeError | CRASH TypeError |
| `null` | CRASH TypeError | CRASH TypeError |
| `{broken` | file **overwritten with `[]`** (scores lost) | CRASH JSONDecodeError |
| 15 entries | shows 15 rows | ok |

Other:
- `game.py:69` `get_file_content()` called **every frame** → open + parse + 11 new `Font` objects.
  Measured **13 ms/frame** (785 ms / 60 frames) of a 16 ms budget → laggy screen.
- `highscores.py:30-32` the except writes the file → `OSError` crash if dir read-only; also creates
  `highscores.json` in the CWD just by starting the game.
- List not sorted on display, `save_score` assumes last entry = lowest.
- Filename hard-coded, config `highscore_filename` ignored. `highscores.json` committed with test names.
- Name `"   "` (spaces only) accepted.

**Fix:**
1. One loader in `save_scores.py` used by screen + save:
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
   Never write in the loader. `save_score` writes inside `try/except OSError` (print, no crash).
2. Screen: `refresh()` when the menu button "Highscores" is clicked, **not** in `run()`. "No scores yet" if empty.
3. Use `config.highscore_filename` everywhere (add `.json` if no extension).
4. `.gitignore` += `highscores.json`; `git rm --cached highscores.json`.
5. `input_widget.py`: `name = typed_text.strip()`; ignore if empty.

### P8. Tracebacks still possible (subject V.1)

**Where:** `src/__main__.py:12-26` try/except commented. Accepts 0 args (subject: exactly 1).
`game.py:50` `run()` inside `__init__`.

**Fix:**
```python
if len(sys.argv) != 2:
    print("Usage: python3 pac-man.py config.json", file=sys.stderr)
    sys.exit(1)
try:
    game = Game(Config.from_json_file(sys.argv[1]))
    game.run()
except KeyboardInterrupt:
    sys.exit(0)
except Exception as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
```

### P9. Instructions screen (placeholder text + broken layout)

**Where:** `src/graphic/screens/instructions.py` (unchanged since last review).
1. Text = "Hellooooo"… Subject VI.8: must show **controls and rules**.
2. Layout `"HUD"` → font = 10 % of window width (~150 px) → lines off screen.
3. l.10-13 copies score/lives/level → unused.
4. No Back button, player not told about ESC.

**Target:**
```
                 HOW TO PLAY
                  CONTROLS
          Arrows / WASD   move Pac-Man
          ESC / SPACE     pause
                   RULES
          Pacgum           +{points_per_pacgum}
          Super pacgum     +{points_per_super_pacgum}, ghosts blue {afraid_time} s
          Blue ghost       +{points_per_ghost}, back after {ghost_pwn_time} s
          Touch a ghost    lose 1 life ({lives} lives)
          Eat all pacgums  next level ({levels} levels)
          Timer {level_max_time} s     time out = lose 1 life
                  GHOSTS
     Blinky chases you      Pinky cuts you off
     Inky ambushes          Clyde is shy when near
            CHEAT MODE (press C)    <- only keys that really exist (section C)
                  [ Back ]
```
Numbers from **config** (reviewer changes them at defense).

**Layout** (`menu_widget.py` `calculate_layout`):
```python
elif self.menu_name == 'INSTRUCTIONS':
    menu_width, menu_height = self.surface.get_size()
    dynamic_font_size = max(14, min(int(menu_width * 0.022),
                                    int(menu_height * 0.032)))
    self.gap = int(dynamic_font_size * 0.15)
```
Screen = list of `TextWidget(text, color)` (`TextWidget("")` = spacer, works) + `ButtonWidget("Back")`.
Wire: `InstructionScreen(surface, self.config)`; click "Back" in INSTRUCTIONS → MENU.
Note: `menu_widget.py:33` gives every element `fixed_width = 30 %` → Back button OK, text ignores it.

**Cheap UI wins:** title "PAC-MAN" on main menu; highscores title "TOP 10" + Back; winner/loser:
"Type your name + Enter" hint; shared `src/graphic/colors.py`.

### P10. Config loader crashes on reviewer edits (task 1, subject V.3)

**Where:** `src/parser/load_config.py`
- l.13 `extra='forbid'` → unknown key crashes → `extra='ignore'` + message.
- l.13 `strict=True` + whole-model validate → one bad value rejects all.
- l.29-35 `check_consistency` → **`{"width": 5, "height": 5}` alone crashes (tested)**: default pacgum 42 > 25.
  Delete it (placement already clamps with `min()`).
- l.50-56 missing file / bad JSON → `ConfigError` → traceback while P8 not done.
- No max for `width/height` → clamp 5..40 (40x40 tested OK: generator fine, BFS cheap).
- Comment lines removed → JSON error line numbers wrong → replace by `"\n"`.
- `afraid_time`, `ghost_pwn_time` not in subject key list → keep as optional extras, document in README.

**Fix:** per-key loop: validate each field alone; on error print
`[config] 'lives'=-4 invalid, using default 3`; unknown key → `[config] unknown key 'x' ignored`.

### P11. Other bugs / risks

| # | Where | Problem | Fix |
|---|-------|---------|-----|
| a | `button_widget.py:47,51`, `input_widget.py:39,41,43`, `game.py:270` | Debug prints | Delete |
| b | `game.py:29` | `minimum_size = (1500, 1000)` > many screens | `(800, 600)` |
| c | `game.py:113-125` | Resize re-layouts HUD, main menu, pause only. Winner/loser/highscores/instructions keep old font size (surface itself is fine: `Window.get_surface()` returns the same object, tested) | `calculate_layout()` on every screen |
| d | `game.py:174,176,271,300` | Magic indexes `elements[4]`, `elements[2]` | Keep refs `self.input`, `self.score_text` in screens |
| e | `input_widget.py` | ESC leaves typed name in the box → shows on next WINNER/LOSER | Clear `text` when entering the screen |
| f | `game.py:293` | `random.seed()` reseeds global random; seed range only ~1300 values → same maze twice possible | `self.rng = random.Random()`; skip seed == previous |
| g | `game_state.py:60` | Pacgum placement uses global random | `random.Random(level_seed)` |
| h | `pacgum.py:15-17` | `MazeEnv.config` may be `None` → mypy | `assert` |
| i | `character.py:31-32` | `self.y: int` becomes float | `float` |
| j | `pacman.py:13-22` | Start fallback tries 3 cells only. Tested 20x20 seeds 1-30 + sizes 5..40: always OK today, but depends on generator | BFS from center to nearest cell `!= 15` |
| k | `pacman.py:6` | `config` unused | remove |
| l | `src/setting.json` | Test values: `levels: 1`, `pacgum: 1` | `levels: 10`, `pacgum: 42`; root `config.json` |
| m | `game.py` | Movement per frame, not `dt` → slower game if FPS drops (timer uses `dt` → inconsistent) | Optional: `speed * dt * 60` |
| n | `character.py:55-76` | Pacman turns back only at cell center (laggy) | Opposite key while moving → swap `current_cell`/`target_cell` |
| o | `character.py:83-93` | `is_valid_direction` trusts wall bits only, no bounds check | Add bounds check (cheap safety) |

### P12. Dead files / cleanup

- `src/logic/rules.py` (circular import risk), `src/graphic/screens/test.py` (runs at import, flake8 W291),
  `src/logic/ghosts_move.py` (empty → or put BFS here), `main.py` (uv stub) → delete.
- `pacgum.py:38-69`, `character.py:8-27`, `maze_env.py:28-34`, `pacman.py:8-11`, `game.py:9,48,54,147-148,157-161,185-186`,
  `input_widget.py:44-47`, `highscores.py:12-15`, `winner.py:10` → commented code, delete.
- `game.py:279-281` `start_evil()`, `309-311` `end_game()` → empty, delete.
- Root `setting.json` (untracked) duplicates `src/setting.json` → keep one.

---

## G. Ghost AI — what it does, what's wrong, how it should be

### G1. Today (`ghost.py:82-129`)
At each cell center: pick target (Blinky = Pac cell, Pinky = 4 ahead, Inky = 2 ahead, Clyde = Pac if
> 8 cells else corner; scatter = own corner; afraid = flee Pac) → list open directions except U-turn →
take the one with smallest **straight-line** distance² to target.

**Measured today** (headless, real `Ghost` classes, 20x20 non-perfect, seeds 1-30, 2 min limit, 60 fps):

| Scenario | Greedy (now) | BFS (G2) |
|---|---|---|
| Blinky alone, Pac still | caught **18/30**, avg 13.9 s, max 30.6 s | **30/30**, avg 8.0 s, max 10.3 s |
| 4 ghosts, Pac still | 29/30, avg 13.0 s, **max 83.5 s** | 30/30, avg 6.6 s, max 7.9 s |
| 4 ghosts, Pac random walk | 30/30, avg 10.4 s, max 21.5 s | 30/30, avg 8.0 s, max 13.6 s |

Greedy loops around wall blocks (crow-flies ignores walls). 4 ghosts hide it; one ghost shows it.

### G2. Fix: BFS distance + target snapping (~35 lines, put in `src/logic/ghosts_move.py`)
```python
from collections import deque

WALLS = [(1, -1, 0), (2, 0, 1), (4, 1, 0), (8, 0, -1)]   # bit, dy, dx (N, E, S, W)


def snap_target(maze: list[list[int]],
                target: tuple[int, int]) -> tuple[int, int]:
    """Clamp into the maze; if closed (15, "42" logo) take nearest open cell."""
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
In `next_direction()`: `dist_map = bfs_distances(MazeEnv.maze, (target_y, target_x))`, then
`d_val = dist_map.get((test_y, test_x), 10_000)`; afraid → `-d_val`.

**Why snapping matters (edge case, not in the old plan):** Pinky/Inky targets often land **outside** the maze
or **inside the "42" logo**. A closed cell has 4 walls → BFS from it reaches nothing → every neighbor = 10 000
→ tie → ghost always takes the first direction in the list (dumb, looks random). Snapping fixes it.

Cost: 400 cells (20x20), 1 600 (40x40), only at cell centers → negligible. Optional: cache per
`(target, frame)` so Blinky + Clyde share one BFS.
Defense line: *"Each ghost picks a target cell (its personality), then follows the shortest real path."*

### G3. Timers and fairness
- **Spawn kill (tested):** eaten ghost respawns in its corner after `ghost_pwn_time` even if Pac stands there
  → instant life lost (lives 3 → 2 in test). Super pacgums **are** in the corners → common case.
  Fix: in `Ghost.move()` eaten branch, respawn only if Pac is ≥ 3 cells away (BFS or Manhattan), else wait.
- **Per-ghost mode clock:** each ghost has its own `last_mode_switch` → after one ghost is eaten/reset,
  ghosts are out of sync (one scatters, others chase). Classic: one global clock in `Game`
  (scatter 7 → chase 20 → scatter 7 → chase 20 → scatter 5 → chase forever), counted with `dt`
  (pause-safe, replaces P6-b). On each switch every ghost reverses (visible signal to the player).
- **Start fairness:** all 4 ghosts leave at t=0 from the corners. Optional staggered release:
  Blinky 0 s, Pinky 2 s, Inky 4 s, Clyde 6 s.

### G4. Optional polish (if time)
- **Inky classic:** `p2 = 2 cells ahead of Pac`; `target = (2*p2y - blinky.y, 2*p2x - blinky.x)` (pass Blinky
  in constructor). Needs G2 snapping (target often outside maze).
- **Clyde:** use BFS distance (path) instead of crow-flies for the "> 8" test.
- **Afraid:** random open direction at each intersection (classic, easier to catch) instead of perfect flee;
  with BFS flee, ghosts are very hard to catch on big mazes.
- **Speed by level:** ghost `0.05` (half of Pac) is easy. E.g. `0.05 + 0.005 * (level-1)` capped `0.08`.
  Overshoot is safe (`align_with_target` snaps), `round(…, 2)` keeps it stable.
- Tie-break order UP, LEFT, DOWN, RIGHT (classic, deterministic).

---

## C. Cheat mode (task 10, subject VI.5 — must really help the reviewer)

**Where:** `game.py` key handler (PLAYING), `game_state.py`, `hud_menu.py`, `ghost.py`.

| Key | Effect | How |
|-----|--------|-----|
| `C` | Toggle cheat ON/OFF | `GameState.cheat_mode`; HUD "CHEAT ON". Other keys only when ON |
| `I` | Invincible | `GameState.invincible` (collision code already skips `lose_life` on `cheat_mode`, `game.py:262`) |
| `N` | Skip level | `GameState.pacgums.clear()` → normal `level_win()` (needs P4/P5 first) |
| `F` | Freeze ghosts | `GameState.ghosts_frozen`; skip `ghost.move()` |
| `L` | +1 life | `GameState.lives += 1` |
| `V` | Pac speed x2 | `0.1 ↔ 0.2` (values dividing 1.0) |
| `T` | +30 s | `GameState.time_left += 30` |

`reset_game()` resets every flag. List keys on Instructions + README.

---

## E. Edge cases found (2026-11-20)

| # | Case | Result now | Where fixed |
|---|------|-----------|-------------|
| E1 | Timer hits 0 | all lives lost in 3 frames | P3 |
| E2 | Main menu mid-game → Start | keeps old lives | P2 |
| E3 | Pac on a corner when eaten ghost respawns | instant death | G3 |
| E4 | Afraid ghost + pause > afraid_time | afraid gone on resume | P6-b |
| E5 | ESC / SPACE in PAUSE | ignored | P1 |
| E6 | Highscores file = `{}`, `null`, `[1]`, missing `score`, string score | crash | P7 |
| E7 | Highscores file = broken JSON | silently replaced by `[]` | P7 |
| E8 | Read-only directory | crash at startup (screen writes file) | P7 |
| E9 | Config `width=5,height=5` only | crash (pacgum 42 > 25) | P10 |
| E10 | Config unknown key / wrong type / comment line shift | crash / wrong line | P10 |
| E11 | 0 args / 2 args | runs with defaults / ignores extra | P8 |
| E12 | WINNER and extra level maze fails | WINNER screen skipped | P4 |
| E13 | Pinky/Inky target outside maze or in "42" | ghost moves dumb (with BFS) | G2 snap |
| E14 | Name of spaces only | saved as blank | P7-5 |
| E15 | ESC on WINNER/LOSER after typing | text stays for next time | P11-e |
| E16 | Window resize on Instructions/Highscores/Winner | old font size | P11-c |
| E17 | Window drag / slow PC | big `dt` jump → timer drops, movement doesn't | P11-m |
| E18 | Maze < 9x7 | generator prints "too small to add '42'" (fine, no crash, tested 5x5) | — (mention in README) |
| E19 | Highscores screen open | 13 ms/frame file I/O + fonts | P7-2 |

Checked OK (no action): all non-15 cells reachable from Pac start (20x20 seeds 1-30, sizes 5..40);
no ghost through walls after super pacgum; collision can't be skipped at current speeds (max 0.15 cell/frame
vs 0.5 hit box); resized window surface stays valid.

---

## D. How the game works now (main @ `425d435`)

```
make run → uv run -m src src/setting.json
   ├─ src/__main__.py       Config.from_json_file(path) → Game(config)
   └─ Game.__init__  (src/logic/game.py)
        ├─ new_maze(config.seed)   MazeGenerator(size, seed, perfect=False).maze (wheel)
        ├─ window: HUD left 30 %, maze right 70 %
        ├─ screens: MainMenu, Instructions, Winner, Loser, HighScores, PauseMenu
        ├─ start_new_game() → reset_game() + new_maze + load_level()
        └─ run()  60 fps, dispatch on MazeEnv.state (starts "MENU")

States: MENU · PLAYING · PAUSE · WINNER · LOSER · HIGHSCORES · INSTRUCTIONS  (strings)
   PLAYING --SPACE--> PAUSE --Resume--> PLAYING      PAUSE --Main menu--> MENU
   any --ESC--> MENU (except PAUSE: broken, P1)

Shared state (class attributes):
   MazeEnv   maze, surface, offsets, cellsize, config, state
   GameState pacman, ghosts, pacgums, score, lives, level, time_left, cheat_mode

Frame in PLAYING: events → pacman.move → eat_pacgum → ghosts move → collision → timer → HUD → draw
Rules: gum +10 | super +50 + afraid | blue ghost +200, back after ghost_pwn_time
       ghost → -1 life, all reset | timer 0 → -1 life | no gums → next level (random seed)
       level > levels → WINNER | lives 0 → LOSER | name + Enter → top-10 JSON → MENU
Maze cell bits: 1 N, 2 E, 4 S, 8 W, 15 = closed ("42")
```

---

## Final checklist before defense

- [ ] `python3 pac-man.py config.json` works; 0 / 2 args → usage message
- [ ] Config with comment, unknown key, `"lives": -4`, `"width": "abc"`, 5x5, missing file → starts, message, no traceback
- [ ] Menu → play → win/lose → name → menu, many times, lives always full on new game
- [ ] Lives + score kept between levels; ≥ 10 levels by default
- [ ] Pause with ESC and SPACE, game visible behind; afraid timer frozen during pause
- [ ] Ghosts reach a still Pac in every maze (BFS); no spawn kill
- [ ] Highscores: delete file / garbage / `{}` → no crash, list refreshes after a game
- [ ] Cheat keys work and are in Instructions + README
- [ ] `make lint` clean (flake8 + mypy strict)
- [ ] README + docs/project_management + itch.io link

---

## DONE log

| Date | Commit | What |
|------|--------|------|
| 2026-11-20 | `e051b69` | Pause menu screen (Resume / Main menu), SPACE pauses, nothing moves while paused, pause re-laid out on resize |
| 2026-11-18 | `87241d6` | Highscores screen reloads the file, survives missing file / broken JSON; button hover color |
| 2026-11-18 | `84f203c` | Ghost into walls after super gum fixed (swap cells); `reset_game()` before `load_level()`; lives kept between levels; per-frame state print removed |
| ≤ `9f69bc7` | | Level 1 = `config.seed`; maze `None` → RuntimeError; `dt` reset per level; mazegenerator wheel; Winner/Loser + name input + top-10 save; menu buttons |
