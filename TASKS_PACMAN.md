# Pac-Man project — TASKS

Reviewed: 2026-11-24 on **`main` @ `dd482ef`** ("update : Readme and ui thinks").
All `file:line` = that commit. **(tested)** = reproduced / verified headless.
Each step = **Problem → Files → How to fix (numbered) → How to check**.

> ⚠ `TASKS_PACMAN.md` is **not in git** (lost once) → `git add` it or move it to `docs/project_management/`.
> `en.subject.pdf` is no longer in the project folder — keep a copy somewhere for the defense.

---

## 1. WHERE WE ARE

| Area | State |
|------|-------|
| Menu → play → levels → WINNER / LOSER → name → Menu | ✅ |
| Ghost BFS chase, afraid (run away) + blink, exact `afraid_time` | ✅ (tested) |
| Instructions scaling + resize of every screen, HUD relayout only on change | ✅ (tested) |
| HUD with cheat + power | 🔴 lines hidden on maximized windows (step 1) |
| Timed scatter mode | 🔴 not asked by the subject + never happens → **delete it** (step 2) |
| Ghost speed per level | 🔴 same all game (step 3) |
| Inky target | 🟡 simplified (step 4) |
| Colors | 🟡 white bg → invisible text / BACK (step 5) |
| `__main__.py` protection | 🔴 commented (step 6) |
| Highscores / config robustness | 🔴 (steps 7, 8) — **fix code tested** |
| Lint, `pac-man.py`, packaging, `docs/` | 🔴 (steps 9, 10) · README ✅ |
| Pause polish | ⏸ parked |

---

## 2. ROADMAP

| Step | Task | Size | Files |
|------|------|------|-------|
| 1 | HUD font limited by height | 5 min | `menu_widget.py` |
| 2 | Delete timed scatter (subject = chase + run away only) | 5 min | `ghost.py` |
| 3 | Ghost speed per level | 5 min | `game.py` |
| 4 | Inky classic target | 10 min | `ghost.py`, `game.py` |
| 5 | Colors | 10 min | `menu_widget.py`, `text_widget.py`, `button_widget.py`, `instructions.py`, `colors.py` |
| 6 | `__main__.py` protection | 10 min | `__main__.py`, `game.py` |
| 7 | Highscores safe | 30 min | `save_scores.py`, `highscores.py`, `input_widget.py`, `game.py`, `.gitignore` |
| 8 | Config loader safe | 20 min | `load_config.py`, config files |
| 9 | Dead code + lint | 1 h | several |
| 10 | Delivery | 3 h | root |

**Before every push:** `make run` → Start → eat a super pacgum → P (cheat) → ESC → Main menu →
Instructions → resize → BACK → Highscores → ESC → Exit. Plus `uv run -m src` (no arg) → usage, no traceback.

---

## 3. STEPS

### Step 1 — HUD: power + cheat lines hidden **(tested)**

**Problem:** HUD font = 10 % of HUD **width** only (`menu_widget.py:17-20`), gap = 0.8 × font. With cheat (6 lines)
+ power (1 line) the HUD is taller than the window. Maximized windows are shorter than the screen
(title bar + taskbar): 1920x1000, 1600x860, 1366x700 → **"Power Left" + last cheat line hidden**; 2560x1080 →
Score hidden too. Power line is inserted first (`hud_menu.py:63`) → first one cut.

**Files:** `src/graphic/menus/menu_widget.py:17-20`, `src/graphic/menus/hud_menu.py:61-63`

**How to fix:**
1. `menu_widget.py` HUD branch — font from the number of lines (same idea as Instructions):
   ```python
   if self.menu_name == "HUD":
       menu_width, menu_height = self.surface.get_size()
       lines = len(self.elements)
       dynamic_font_size = min(int(menu_width * 0.10),
                               int(menu_height / (lines * 1.3)))
       self.gap = int(dynamic_font_size * 0.6)
   ```
2. `hud_menu.py:61` text with unit: `f"Ghosts blue: {power_left} s"`.
3. Optional `hud_menu.py:63`: `insert(8, …)` → power line under Time, Score stays on top.

**How to check:** tested at 1280x720, 1366x700, 1600x860, 1920x1000, 1920x1080, 2560x1080 with cheat + power +
1 extra line → all visible. In game: resize the window small with P on + super pacgum eaten.

---

### Step 2 — Delete the timed scatter mode

**Why:** subject VI.3 asks only:
- move through corridors · **chase** when not edible · **run away when edible** · respawn in the corner after
  being eaten.

"Run away when edible" = our **afraid** mode (after a super pacgum) ✅ already done.
The **timed** scatter (chase 20 s ↔ scatter 5 s, each ghost goes to its corner) is from the arcade game, not the
subject — and today it never even happens (every death restarts its 20 s clock). Delete it = simpler code,
nothing to explain at defense.

| Word | Meaning | Keep? |
|------|---------|-------|
| chase | go to the ghost's target (Blinky / Pinky / Inky / Clyde) | ✅ |
| afraid (frightened) | after super pacgum: blue, slow, run away, blink, can be eaten | ✅ |
| scatter (timed) | every 20 s go to own corner for 5 s | ❌ delete |
| Clyde "go to corner" | Clyde's chase rule when close to Pac-Man | ✅ (it's his chase target, not scatter) |

**Files:** `src/graphic/characters/ghost.py`

**How to fix — only deletions:**
1. `__init__` l.19-22: delete `is_scatter_mode`, `last_mode_switch`, `scatter_duration`, `chase_duration`.
2. Delete the whole `update_mode()` method (l.90-99).
3. `next_direction()`: delete `self.update_mode()` (l.111), the commented block (l.113-116), and the
   `elif self.is_scatter_mode:` branch (l.120-121). Result:
   ```python
   if self.is_afraid:
       target_y, target_x = self.pacman.current_cell   # used with -dist = run away
   else:
       target_y, target_x = self.define_target()
   ```
4. `reset()`: delete l.191 (`is_scatter_mode = False`) and l.193 (`last_mode_switch = …`).
5. `README.md:214`: replace the "chase 20 s ↔ scatter 5 s" line with
   `Modes: **chase** (always) and **afraid** after a super pacgum (run away, slower, blue, blinks before the end).`

**How to check:** `grep -rn scatter src` → nothing. Play: ghosts always chase; after a super pacgum they turn,
go blue, run away, blink the last 2 s, then chase again.

---

### Step 3 — Ghost speed per level **(tested)**

**Problem:** ghosts `0.05` every level (`ghost.py:23`) → level 10 as easy as level 1.

**Files:** `src/logic/game.py` `load_level()` (l.248-259)

**How to fix:**
1. After `self.ghosts = [...]`, add:
   ```python
   ghost_speed = min(0.05 + 0.01 * ((GameState.level - 1) // 2), 0.09)
   for ghost in self.ghosts:
       ghost.speed = ghost.normal_speed = ghost_speed
   ```
2. Keep speeds with 2 decimals (movement uses `round(…, 2)`). Cap `0.09` < Pac-Man `0.1`.

| Level | 1-2 | 3-4 | 5-6 | 7-8 | 9-10 |
|-------|-----|-----|-----|-----|------|
| Speed | 0.05 | 0.06 | 0.07 | 0.08 | 0.09 |
| Avg catch time (tested, 15 mazes) | 7.7 s | 5.9 s | 5.7 s | 4.6 s | 4.4 s |

`afraid_of()` sets `0.03`; `move()` and `reset()` restore `normal_speed` → nothing else to change.

**How to check:** cheat `N` to jump levels → ghosts visibly faster at levels 3, 5, 7, 9. Tested: no wall crossing,
no missed collision.

---

### Step 4 — Inky classic target **(tested)**

**Problem:** Inky = 2 cells ahead of Pac-Man (`ghost.py:260-276`) → like a short Pinky.
Classic: point 2 cells ahead of Pac-Man, line **from Blinky** to that point, **doubled** → pincer with Blinky.
```
target = 2 × (2 cells ahead) − Blinky
```

**Files:** `src/graphic/characters/ghost.py:260-276`, `src/logic/game.py:255-258`

**How to fix:**
1. `Inky.__init__(self, start_corner, pacman, blinky)` → store `self.blinky = blinky`.
2. `define_target()`:
   ```python
   py, px = self.pacman.current_cell
   dy, dx = {Direction.UP: (-1, 0), Direction.DOWN: (1, 0),
             Direction.LEFT: (0, -1), Direction.RIGHT: (0, 1)
             }.get(self.pacman.direction, (0, 0))
   ay, ax = py + 2 * dy, px + 2 * dx        # 2 cells ahead
   by, bx = self.blinky.current_cell
   return (2 * ay - by, 2 * ax - bx)
   ```
3. `game.py` `load_level()`:
   ```python
   blinky = Blinky(Corner.TOP_RIGHT, self.pacman)
   self.ghosts = [blinky,
                  Pinky(Corner.TOP_LEFT, self.pacman),
                  Inky(Corner.BOTTOM_LEFT, self.pacman, blinky),
                  Clyde(Corner.BOTTOM_RIGHT, self.pacman)]
   ```

**How to check:** target outside the maze 41 % of the time (tested) → already clamped in `next_direction()`.
Inky now comes from the side opposite Blinky.

---

### Step 5 — Colors **(screenshot)**

**Problem:** `menu_widget.py:12` default bg `(255, 255, 255)` → on Instructions `"[> Rules <]"` (WHITE) invisible,
BACK button = light text (`TEXT`) on light grey. HUD / Winner / Loser / Highscores still white.

**How to fix (all together, or some text disappears):**
1. `menu_widget.py:12` → `self.bg_color = BG` (import `from ..colors import BG`).
2. `text_widget.py:5` default `color=TEXT` (light text on dark bg).
3. `button_widget.py:5` default `text_color=B_TEXT` (black on grey, black on yellow hover).
4. `instructions.py`: lines in `B_TEXT` → `TEXT` (or no color = default).
5. `input_widget.py:32` typed name `'black'` → `TEXT`.
6. `colors.py`: one name per color — delete `GREY` duplicate, `YELLOW` (= `TITLE`), `WHITE` (= `TEXT`),
   `BG_2` (= `B_TEXT`), update the imports.

**How to check:** every screen dark, all text readable, button text readable normal + hover.

---

### Step 6 — `__main__.py` protection **(tested)**

**Problem:** argc check + try/except commented → `uv run -m src` = `IndexError` traceback, missing file =
`ConfigError` traceback; dead `print(f"ERR0R: {e}")` (F821). `Game.__init__` calls `self.run()` (`game.py:52`).

**How to fix:**
1. `src/__main__.py`:
   ```python
   import sys
   from .parser.load_config import Config, ConfigError
   from .logic.game import Game


   def main() -> int:
       """Check args, load config, run the game. Return exit code."""
       if len(sys.argv) != 2:
           print("Usage: python3 pac-man.py config.json", file=sys.stderr)
           return 1
       try:
           game = Game(Config.from_json_file(sys.argv[1]))
           game.run()
       except ConfigError as e:
           print(f"Config error: {e}", file=sys.stderr)
           return 1
       except KeyboardInterrupt:
           return 0
       except Exception as e:
           print(f"Error: {e}", file=sys.stderr)
           return 1
       return 0


   if __name__ == "__main__":
       sys.exit(main())
   ```
2. `game.py:52` delete `self.run()`.

**How to check:** `uv run -m src` → usage; `uv run -m src nofile.json` → one clear line; Ctrl+C in terminal → no
traceback.

---

### Step 7 — Highscores safe **(code tested)**

**Problem:** bad file → crash: `{}`, `null`, `[1, 2]`, entry without `score`, `"score": "10"` (screen + save);
`{broken` → screen overwrites with `[]`; screen re-reads the file **every frame** (`game.py:70-71`); spaces-only
names saved; config `highscore_filename` ignored (hard-coded in `save_scores.py:6`, `highscores.py:17`);
`highscores.json` committed.

**Files:** `src/logic/save_scores.py`, `src/graphic/screens/highscores.py`, `src/graphic/widgets/input_widget.py:39`,
`src/logic/game.py:70-71, 148-149`, `.gitignore`

**How to fix:**
1. `save_scores.py` — replace all:
   ```python
   import json


   def load_scores(filename: str) -> list[dict[str, object]]:
       """Top 10 valid scores from the file. Never crashes, never writes."""
       try:
           with open(filename, encoding="utf-8") as f:
               data = json.load(f)
       except (OSError, json.JSONDecodeError):
           return []
       if not isinstance(data, list):
           return []
       good = [e for e in data
               if isinstance(e, dict)
               and isinstance(e.get("name"), str) and e["name"].strip()
               and type(e.get("score")) is int and e["score"] >= 0]
       good.sort(key=lambda e: e["score"], reverse=True)
       return good[:10]


   def save_score(name: str, score: int, filename: str) -> bool:
       """Add a score and keep the top 10. False if not saved."""
       name = name.strip()
       if not name or score < 0:
           return False
       scores = load_scores(filename)
       scores.append({"name": name, "score": score})
       scores.sort(key=lambda e: e["score"], reverse=True)
       try:
           with open(filename, "w", encoding="utf-8") as f:
               json.dump(scores[:10], f, indent=4)
       except OSError as e:
           print(f"[highscores] cannot save to '{filename}': {e.strerror}")
           return False
       return True
   ```
2. `highscores.py`: `get_file_content(self, filename)` uses `load_scores(filename)` — no `open`, no write, no
   try needed. Empty → `TextWidget("No scores yet", GREY)`. Add a `ButtonWidget("BACK")` at the end.
3. `game.py:70-71`: delete `get_file_content()` from the frame loop. Call it **once** on click
   (`game.py:148-149`): `self.highscores_screen.get_file_content(self.config.highscore_filename)`.
4. `input_widget.py:39`: `save_score(typed_text, GameState.score, MazeEnv.config.highscore_filename)`;
   leave the screen only if `typed_text.strip()` is not empty.
5. `load_config.py`: default `highscore_filename = 'highscores.json'` (today `'scores'`, no extension).
6. `.gitignore` += `highscores.json`, then `git rm --cached highscores.json`.
7. Click BACK on Highscores → MENU (same as Instructions branch in `game.py`).

**How to check (tested with the code above):** `{}`, `null`, `[1, 2]`, missing / string / bool score, broken JSON →
0 valid entries, no crash, save writes a clean file; valid + invalid mix → only valid kept, sorted; missing file →
empty, save creates it; read-only folder → message, `False`, no crash; `"   "` → refused; `"  bob "` → `"bob"`.

---

### Step 8 — Config loader safe (subject V.3) **(code tested)**

**Problem:** `extra='forbid', strict=True` + one `model_validate(data)` → unknown key or one bad value = whole file
refused (crash); `check_consistency` → `{"width": 5, "height": 5}` crashes (42 > 25); no max size; comment lines
removed → wrong JSON error line.

**Files:** `src/parser/load_config.py`, `setting.json`, `src/setting.json`, `Makefile:1`

**How to fix:**
1. Delete `check_consistency` (l.29-35). Pacgum placement already uses `min(config.pacgum, free cells)`.
2. Add upper limits: `levels le=100`, `width/height le=40`, `lives le=99`, `level_max_time ge=10`;
   `pacgum ge=0`; `highscore_filename` default `'highscores.json'`.
3. Replace `from_json_file`:
   ```python
   @classmethod
   def from_json_file(cls, filepath: str) -> 'Config':
       """Read the file; bad or unknown keys -> message + default."""
       try:
           with open(filepath, encoding='utf-8') as f:
               lines = ["\n" if ln.lstrip().startswith(("#", "//")) else ln
                        for ln in f]               # keep line numbers right
           data = json.loads(''.join(lines))
       except OSError as e:
           raise ConfigError(f"cannot read '{filepath}': {e.strerror}") from e
       except json.JSONDecodeError as e:
           raise ConfigError(f"'{filepath}' is not valid JSON (line {e.lineno}, "
                             f"column {e.colno}): {e.msg}") from e
       if not isinstance(data, dict):
           raise ConfigError(f"'{filepath}' must contain a JSON object {{...}}")
       defaults = cls()
       good = {}
       for key, value in data.items():
           if key not in cls.model_fields:
               print(f"[config] unknown key '{key}' ignored")
               continue
           try:
               cls.model_validate({key: value})   # check this key alone
               good[key] = value
           except ValidationError:
               print(f"[config] '{key}': {value!r} is invalid, "
                     f"using default {getattr(defaults, key)!r}")
       return cls.model_validate(good)
   ```
   (import `ValidationError` from pydantic; `extra='forbid', strict=True` can stay: unknown keys never reach it,
   strict refuses `"3"`, `3.5`, `true` for ints.)
4. One `config.json` at root with real values (`levels: 10`, `pacgum: 42`), delete `setting.json` +
   `src/setting.json`, `Makefile` `ARG ?= config.json`.

**How to check (tested):** comments + valid keys → loaded; `lives: -4`, `width: "abc"`, `levels: 3.5`,
`seed: true`, `height: 99` → 5 messages, defaults used; unknown key → message, ignored; 5x5 → OK; `{}` → all
defaults; broken JSON → `ConfigError` with the **right line**; `[1, 2]` / missing file → `ConfigError` (caught in
step 6 → clean message).

---

### Step 9 — Dead code + lint (subject III.1: flake8 + mypy must pass)

**How to fix, in this order:**
1. Delete dead code:
   - `game.py`: `start_evil()`, `end_game()`, commented lines, `self.run()` (step 6).
   - `hud_menu.py`: `self.suface` (use `self.surface`), unused `menu_width` (F841), commented `#self.invi`,
     commented power block.
   - commented blocks: `pacgum.py` (old `Pacgums` class, 27 lines), `character.py` (17), `maze_env.py` (8),
     `ghost.py` (old distance line), `pacman.py` (unused `config` param + comments), `highscores.py`, `winner.py`.
   - `game_state.py:22-23` lives fallback (`reset_game(lives)` does it).
2. Fix small bugs:
   - `hud_menu.py:69-82` cheat block: rebuild the list from flags instead of append / pop
     (Start with cheat ON adds lines while OFF, popped 1 per frame):
     ```python
     def build(self) -> None:
         """Lines to show, from the current flags."""
         self.elements = [self.score_text, self.gap, self.lives_text, self.gap,
                          self.level_text, self.gap, self.time_text, self.gap]
         if self.power_left > 0:
             self.elements.append(self.power_mode)
         if self.cheat_mode:
             self.elements.append(self.cheat_mode_txt)
             self.elements += self.cheat_lines       # built once in __init__
         self.calculate_layout()
     ```
     call `build()` only when `cheat_mode` or "power > 0" changes.
   - `elements[4]` / `elements[2]` (`game.py:201, 203, 300, 329`) → `self.input` / `self.score_text` refs in
     `WinnerScreen` / `LoserScreen`.
   - `random.seed()` in `level_win` → `self.rng = random.Random()` in `__init__`, `self.rng.randint(...)`.
   - `C` and `N` same cheat → `F` = freeze ghosts (subject VI.5): `GameState.ghosts_frozen`, skip `ghost.move()`.
3. flake8 (23 today): `uvx flake8 src` → fix E302 (2 blank lines before `class`), E265 (`# ` comments), E501
   (lines > 79), E203 (`YELLOW , WHITE`), W391.
4. mypy (63 today): add type hints to every `def` (params + `-> None` / return type), `assert x is not None` before
   using `MazeEnv.maze`, `MazeEnv.config`, `MazeEnv.surface`.
5. Docstrings on every class / function (PEP 257).

**How to check:** `make lint` → 0 errors.

---

### Step 10 — Delivery (subject V.1, III.2, VII, VIII)

1. `pac-man.py` at repo root:
   ```python
   """Entry point: python3 pac-man.py config.json"""
   import sys
   from src.__main__ import main

   if __name__ == "__main__":
       sys.exit(main())
   ```
2. `Makefile`: `run` → `uv run python3 pac-man.py $(ARG)`, `debug` → `uv run python3 -m pdb pac-man.py $(ARG)`
   (`pudb` isn't installed), `clean` += `*.pyc`, `ARG ?= config.json`.
3. Packaging: PyInstaller spec / script at repo root, include `config.json` + the generator; test the build on a
   clean machine; upload to itch.io (unlisted); in-package README (controls, config).
4. `docs/project_management/` (README links it): timeline, progress (this file), risks, who did what,
   acceptance tests (the checks of each step above), blocking points.
5. README: add itch.io link, keep in sync with the final cheat keys and config keys.

---

## 4. GHOSTS vs SUBJECT

| Subject VI.3 | Ours | After steps |
|--------------|------|-------------|
| Move autonomously through corridors | ✅ cell to cell, wall bits | ✅ |
| Chase when not edible (behavior up to you) | ✅ BFS, 4 personalities (Blinky direct, Pinky 4 ahead, Inky, Clyde shy) | ✅ Inky classic (step 4), faster per level (step 3) |
| Run away when edible | ✅ afraid: reverse, blue, slow, flee by BFS, blink last 2 s | ✅ |
| Respawn to corner after a while when eaten | ✅ after `ghost_pwn_time`, not on top of Pac-Man | ✅ |
| (arcade) timed scatter | ❌ not asked, never happens | deleted (step 2) |

---

## 5. PARKED — Pause

| Problem | Suggested fix |
|---------|---------------|
| No key leaves pause, mouse only | `if state == "PAUSE" and key in (ESC, SPACE): state = "PLAYING"` |
| ESC on WINNER / LOSER does nothing | ESC → MENU (skip save) + hint text |
| Afraid + power countdown run during pause (`get_ticks`) | `ghost.shift_timers(paused_ms)` on resume |
| Pause screen opaque | `SRCALPHA` + `(0, 0, 20, 160)` + redraw game under it |

---

## 6. EDGE CASES

| Case | Now | Step |
|------|-----|------|
| Maximized window + cheat + power | power + last cheat line hidden | 1 |
| Level 10 | ghosts as slow as level 1 | 3 |
| Instructions / BACK on white bg | invisible | 5 |
| No arg / missing config / broken JSON | traceback | 6, 8 |
| Highscores garbage / read-only folder / spaces name | crash / saved blank | 7 |
| Config bad values / unknown key / 5x5 | crash | 8 |
| Start new game with cheat ON | cheat lines appear while OFF, popped over frames | 9 |
| Pause + afraid | afraid ends during pause | parked |

---

## DONE log

| Date | Commit | What |
|------|--------|------|
| 2026-11-24 | `dd482ef` | README; `+ 10000` removed; HUD relayout only on change; Instructions font from line count; resize re-lays out every screen; highscores title |
| 2026-11-24 | `898f5ff` | BFS `bfs_steps` + target clamp; `colors.py` imported; power countdown; input prints removed |
| 2026-11-24 | `42359b2` | `TextWidget.scale`, HUD cheat block, P OFF resets invincible, ghost blink, min window 800x600, dead files deleted |
| 2026-11-22/23 | `c2b3b84` … `3792ffb` | Cheat keys, `invincible`, BACK, win + death fix, Instructions page, spawn guard |
| ≤ 2026-11-22 | `c4169db` … `9f69bc7` | Title, timer, pacgum check, lives reset, keyboard, pause menu, highscores screen, ghost-in-wall fix, seeds, Winner/Loser + save, menu |
