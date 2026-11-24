# Pac-Man project — TASKS

Reviewed: 2026-11-24 on **`main` @ `898f5ff`** ("update : colors ghosts mouvemets bfs").
All `file:line` = that commit. **(tested)** = reproduced headless (SDL dummy driver, fake clock, screenshots).
Each task = **Problem → Files → Suggested fix** (to discuss, not final).

> ⚠ `TASKS_PACMAN.md` is **not in git** (lost once already) → `git add` it, or move it to
> `docs/project_management/` (README links there, folder doesn't exist yet).
> `README.md` written (was empty) → review it, then commit.

---

## 1. WHERE WE ARE

| Area | State |
|------|-------|
| Menu → play → levels → WINNER / LOSER → name → Menu | ✅ (tested) |
| Ghost BFS (`bfs_steps` in `Ghost`, target clamp, closed-cell fallback) | ✅ **done, works** (tested: same results as the measured BFS) |
| Ghost blink last 2 s | ✅ |
| Power countdown in HUD | 🟡 shows — **but afraid lasts 10 s too long** (debug `+ 10000`) + HUD relayout every frame (step 1) |
| `colors.py` imported | 🟡 in 10 files — **3 color bugs**: white-on-white Instructions, unreadable buttons, HUD still white (step 2) |
| Instructions scaling | 🔴 **broken**: font 100 px at 1500x1000, text 4x taller than the screen (step 3) |
| Debug prints in name input | ✅ removed |
| Cheat mode, spawn guard, win + death same frame | ✅ |
| `__main__.py` protection | 🔴 still commented (step 4) |
| Highscores / config robustness | 🔴 (steps 6, 7) |
| Lint: flake8 **23**, mypy **63** | 🔴 (step 8) |
| `pac-man.py`, packaging, `docs/` | 🔴 (step 9) · README ✅ draft |
| Pause polish | ⏸ parked (section 5) |

---

## 2. ROADMAP

| Step | Task | Size | Files |
|------|------|------|-------|
| 1 | Remove `+ 10000` + HUD relayout only on change | 10 min | `ghost.py`, `hud_menu.py`, `game.py` |
| 2 | 3 color bugs | 10 min | `menu_widget.py`, `text_widget.py`, `button_widget.py` |
| 3 | Instructions scaling + resize | 10 min | `menu_widget.py`, `game.py` |
| 4 | Restore `__main__.py` protection | 5 min | `__main__.py`, `game.py` |
| 5 | Ghost behavior polish (optional, section 4) | 30 min | `ghost.py`, `game.py` |
| 6 | Highscores safe | 40 min | `save_scores.py`, `highscores.py`, `input_widget.py`, `game.py` |
| 7 | Config loader safe | 45 min | `load_config.py` |
| 8 | Dead code + lint | 1 h | several |
| 9 | Delivery: `pac-man.py`, `config.json`, Makefile, packaging, `docs/` | 3 h | root |

**Before every push:** `make run` → Start → eat a super pacgum (countdown = `afraid_time`?) → P (cheat) →
ESC → Main menu → Instructions (all visible?) → resize window → BACK → Highscores → ESC → Exit.

---

## 3. TASKS

### Step 1 — Power mode: 10 s too long + HUD slow **(tested)**

**Problem 1 — debug leftover:** `ghost.py:202`
```python
self.afraid_until = pygame.time.get_ticks() + duration + 10000
```
Config `afraid_time = 7` → ghosts stay blue **17 s** (tested; HUD shows "Power Left: 16"). Instructions say 7 s.

**Problem 2 — HUD relayout every frame:** `hud_menu.py:81` calls `self.calculate_layout()` at the end of
`update_values()`, every frame → all HUD fonts rebuilt 60 times/s.
Measured: **9.0 ms per frame** vs **1.0 ms** without it (budget 16.7 ms). On a slower PC the game drops
under 60 FPS → and since movement is per frame, **everything moves slower**.

**Problem 3 — rounding:** `game.py:92` `power_left // 1000` → 6.9 s shows "6", last second shows nothing.

**Files:** `ghost.py:202`, `hud_menu.py:59-81`, `game.py:91-92`

**Suggested fix:**
```python
# ghost.py:202
self.afraid_until = pygame.time.get_ticks() + duration

# game.py:91-92
power_ms = max((g.afraid_until - now for g in self.ghosts
                if g.is_afraid), default=0)
power_left = (power_ms + 999) // 1000
```
```python
# hud_menu.py update_values — relayout only when the list changes
changed = False
if power_left > 0:
    self.power_mode.update_text(f"Ghosts blue: {power_left} s")
    if self.power_mode not in self.elements:
        self.elements.insert(0, self.power_mode)
        changed = True
elif self.power_mode in self.elements:
    self.elements.remove(self.power_mode)
    changed = True
# ... cheat block: set changed = True when it adds / removes lines
if changed:
    self.calculate_layout()
```
(`update_text()` re-renders only that line — enough when only the number changes.)
Delete the commented lines `hud_menu.py:65-67` and `ghost.py:156`.

---

### Step 2 — 3 color bugs after loading `colors.py` **(screenshots)**

| # | Problem | Cause | Fix |
|---|---------|-------|-----|
| a | **Instructions: white text on white bg** — nothing readable except the yellow lines | `instructions.py` bg line removed → default `menu_widget.py:12` `(255, 255, 255)` | `menu_widget.py:12` → `self.bg_color = BG` (also makes HUD, Winner, Loser, Highscores dark) |
| b | **Buttons: light text on light grey** (Start, Exit, BACK almost invisible) | `button_widget.py:5` `text_color=TEXT` (230) on `BUTTON` (200) | `text_color=BG_2` (black) — readable on grey and on yellow hover |
| c | After (a), default black text disappears on dark bg (HUD, Winner, Loser, Highscores) | `text_widget.py:5` `color=BG_2` (black) | `color=TEXT` |

Do **a + c together** (one without the other = invisible text somewhere).
Also: `colors.py` defines `GREY` twice, `YELLOW` = `TITLE`, `WHITE` = `TEXT` → keep one name each.
HUD separators: `"________________" * 30` in `TITLE` yellow → fine on dark; long surface (~10 000 px) → `"_" * 40` is enough.

---

### Step 3 — Instructions scaling + resize **(tested)**

**Problem:** `menu_widget.py:37-43`
```python
dynamic_font_size = min(int(menu_width * 0.1), int(menu_hight * 0.1))
self.gap = int(dynamic_font_size * 0.8)
```
26 lines × (font + gap). Tested: 800x600 → font 60, text from y = -896 to 1496;
1500x1000 → font 100, **y = -1494 to 2493** (screen is 1000). Only ~5 lines visible.
Also `game.py` WINDOWRESIZED (l.~132) re-lays out HUD / main / pause only → Instructions keep the old size.

**Why it must depend on the number of lines:** the screen has 26 lines; each line takes ≈ 0.95 × font size
(text + gap). So the biggest font that fits the height is `height / (lines × 0.95)`. Width limit
`width × 0.026` keeps the longest line inside the window.

**Suggested fix (tested at 800x600, 1024x768, 1366x768, 1500x1000, 1920x1080, 2560x1440 → all FIT):**
```python
elif self.menu_name == 'INSTRUCTIONS':
    menu_width, menu_height = self.surface.get_size()
    lines = len(self.elements)
    dynamic_font_size = max(12, min(int(menu_width * 0.026),
                                    int(menu_height / (lines * 0.95))))
    self.gap = int(dynamic_font_size * 0.15)
```
```python
# game.py WINDOWRESIZED: add with the others
self.instructions_screen.calculate_layout()
self.winner_screen.calculate_layout()
self.loser_screen.calculate_layout()
self.highscores_screen.calculate_layout()
```
Adding / removing a line later → still fits (formula uses `len(self.elements)`).
Text polish still open: `"Restart / pause"` → `pause`; `"Arrow Keys < ^ >"`; `_____` fillers; `H0w T0 P1aY`;
add `"Eaten ghost back after {ghost_pwn_time} s"` (commented l.~24).

---

### Step 4 — `__main__.py` protection still commented **(tested)**

**Problem:** no arg → `IndexError` traceback; missing file → `ConfigError` traceback; `print(f"ERR0R: {e}")`
after `sys.exit(0)` uses undefined `e` (flake8 F821). `Game.__init__` calls `self.run()` (`game.py:52`).

**Suggested fix:** `git checkout c4169db -- src/__main__.py`, errors to `sys.stderr`, delete `game.py:52`.

---

### Step 6 — Highscores safe **(tested)**

Bad file → crash (`{}`, `null`, `[1, 2]`, missing / string `score`); `{broken` → overwritten with `[]`;
file read every frame on that screen (`game.py:71`); names with spaces only / leading spaces saved;
config `highscore_filename` ignored (default `"highscores.json"` in `save_scores.py:6`, `highscores.py:17`);
`highscores.json` committed; no BACK button.

**Suggested fix:** one `load_scores(filename)` (try/except, keep valid `{"name": str, "score": int >= 0}`,
sort, top 10, never writes), used by screen + save; refresh on "Highscores" click only; `name.strip()`;
`.gitignore` += `highscores.json`.

---

### Step 7 — Config loader safe (subject V.3) **(tested)**

`load_config.py:13` `extra='forbid', strict=True` → unknown key / one bad value = crash;
`check_consistency` (l.29-35) → `{"width": 5, "height": 5}` crashes; no max size; comment lines removed
(wrong error line). Config files: root `setting.json` (`pacgum: 1`), `src/setting.json` (`levels: 3, pacgum: 1`).

**Suggested fix:** `extra='ignore'` + message; validate key by key (bad value → message + default); delete
`check_consistency`; clamp 5..40; comment line → empty line; one real `config.json` at root.

---

### Step 8 — Dead code + lint

| What | Where |
|------|-------|
| `start_evil()`, `end_game()` empty | `game.py` |
| `self.suface` typo + unused `menu_width` (F841) | `hud_menu.py:13, 70` |
| commented `#self.invi`, power block | `hud_menu.py:16, 65-67` |
| commented old distance | `ghost.py:156` |
| commented code | `pacgum.py` (27 lines), `character.py` (17), `game.py` (13), `ghost.py` (10), `maze_env.py` (8) |
| `config` param unused | `pacman.py:6` |
| `random.seed()` resets global random | `game.py` `level_win` |
| `elements[4]`, `elements[2]` magic indexes | `game.py` winner / loser |
| Dead lives fallback | `game_state.py:22-23` |
| `C` and `N` same cheat | `cheat_ways` → free one for `F` freeze ghosts (subject VI.5) |

flake8 23: E265 ×7, E302 ×5 (1 blank line before `class` in `main_menu.py`, `pause_menu.py`, `button_widget.py`,
`text_widget.py`, `ghost.py`), E501 ×6, E203 (`YELLOW , WHITE`), E115, F821, F841, W391. mypy 63 (type hints).

---

### Step 9 — Delivery

- `pac-man.py` at root (`python3 pac-man.py config.json`) — README already documents it.
- `Makefile`: `run` → `python3 pac-man.py config.json`; `debug` uses `pudb` (not installed) → `python3 -m pdb`.
- Packaging script/spec at root (PyInstaller), itch.io unlisted build.
- `docs/project_management/` (README links it): timeline, progress (this file), risks, who did what, tests.

---

## 4. GHOSTS vs REAL PAC-MAN — what we have, what's different

| Behavior | Classic Pac-Man | Ours | Verdict |
|----------|-----------------|------|---------|
| Path choice | at each crossing, neighbor closest to target, no U-turn | BFS steps (better than classic straight-line), no U-turn | ✅ (smarter than classic, fine) |
| Blinky target | Pac-Man cell | same | ✅ |
| Pinky target | 4 cells ahead | same | ✅ |
| Inky target | vector from Blinky through 2 cells ahead × 2 | 2 cells ahead (simplified) | 🟡 OK, subject: "up to you" |
| Clyde target | Pac-Man if > 8 cells, else own corner | same (straight-line distance, `distance > 64` l.287) | ✅ |
| Scatter / chase | global timer: 7/20/7/20/5/20/5/∞ s, all ghosts together | per ghost 5 s / 20 s, each ghost has its own clock → out of sync after one is eaten / reset | 🟡 optional: one clock in `Game` |
| Mode switch | ghosts reverse direction | no reverse | 🟡 optional (visual hint) |
| Frightened | reverse, slow, random turns, blink before end | reverse ✅, slow 0.03 ✅, flee by BFS (not random), blink ✅ | ✅ (BFS flee = harder to catch; random = classic) |
| Frightened duration | shorter each level | fixed `afraid_time` (+10 s bug, step 1) | 🔴 step 1 |
| Eaten | eyes run home, revive | invisible, respawn in corner after `ghost_pwn_time`, only if Pac-Man ≥ 3 cells away | ✅ subject VI.3: "respawn to their corner after a while" |
| Ghost points | 200 / 400 / 800 / 1600 in a row | flat `points_per_ghost` | ✅ subject uses one value Z |
| Start | leave the house one by one | all 4 start at t = 0 from the corners | 🟡 optional stagger 0 / 2 / 4 / 6 s |
| Speed | ghosts ≈ 75 % of Pac-Man, faster each level | 50 % (0.05 vs 0.1), same every level | 🟡 optional: `0.05 + 0.005 × (level - 1)`, max 0.08 |
| Pac-Man death | ghosts + Pac-Man back to start | same | ✅ |

**Tested:** 4 ghosts catch a still Pac-Man in 30/30 mazes (max 7.9 s), random-walking Pac-Man 30/30; ghosts
never enter a closed cell; afraid ghosts reverse once and flee.
**Verdict:** behavior matches the subject (chase, flee when edible, respawn in corner). Only real bug = `+ 10000`.

---

## 5. PARKED — Pause

| Problem | Suggested fix |
|---------|---------------|
| No key leaves pause, mouse only | `if state == "PAUSE" and key in (ESC, SPACE): state = "PLAYING"` |
| ESC on WINNER / LOSER does nothing | ESC → MENU (skip save) + hint text |
| Ghost timers + power countdown run during pause | `ghost.shift_timers(paused_ms)` on resume |
| Pause screen opaque | `SRCALPHA` + `(0, 0, 20, 160)` + redraw game under it |

---

## 6. SUBJECT CHECK

| Subject | Requirement | Us | Step |
|---------|-------------|----|------|
| III.1 | flake8 + mypy pass, type hints, docstrings | ❌ 23 / 63 | 8 |
| III.1 | No crash | ❌ `__main__`, highscores, config | 4, 6, 7 |
| III.2 | Makefile rules | 🟡 debug = pudb | 9 |
| V.1 | `python3 pac-man.py config.json`, 1 arg, no traceback | ❌ | 4, 9 |
| V.2 | JSON + `#` comments | ✅ | — |
| V.3 | Bad values → default + message, unknown keys ignored | ❌ | 7 |
| V.4 | Generator as-is, PERFECT=False, failure handled | ✅ | — |
| V.5 | Highscores robust, names ≤ 10 alnum + spaces, top 10, in menu | 🟡 | 6 |
| VI.1 | Pacgums "in most corridors" | ⚠ 42 dots in ~380 cells → **discuss** | — |
| VI.1 | Corners: super pacgums + ghosts, Pac-Man middle, level 1 fixed seed | ✅ | — |
| VI.2 | Arrows or WASD | ✅ arrows (WASD would clash with cheat A, W) | — |
| VI.3 | Chase / flee / respawn in corner | ✅ BFS | — |
| VI.5 | Cheat mode | ✅ | — |
| VI.7 | ≥ 10 levels, time limit, pause / resume | ✅ default 10 (config files: 3 / 10) | 7 |
| VI.8 | HUD always shows score, lives, level, time | ✅ | — |
| VI.8 | Instructions shows controls + rules | ❌ unreadable now (white on white, too big) | 2, 3 |
| VII | Packaging + itch.io | ❌ | 9 |
| VIII | `docs/project_management/` | ❌ | 9 |
| IX | README | ✅ draft written — check facts, add itch.io link, keep in sync | — |

---

## 7. EDGE CASES

| Case | Now | Step |
|------|-----|------|
| Super pacgum with `afraid_time: 7` | ghosts blue 17 s | 1 |
| Slow PC | HUD relayout 8 ms/frame → < 60 FPS → game slower | 1 |
| Instructions at any window size | text off screen, white on white | 2, 3 |
| Resize while on Instructions / Highscores / Winner / Loser | old font size | 3 |
| Any button | text barely visible | 2 |
| No arg / missing config | traceback | 4 |
| Highscores garbage / `{}` / read-only folder | crash | 6 |
| Config 5x5 / unknown key / wrong type | crash | 7 |
| Pause + power mode | countdown runs during pause | parked |

Checked OK (tested): BFS chase 30/30 mazes; ghosts never on closed cells; target clamp; cheat keys; spawn guard;
win + ghost same frame; timer 0 → 1 life.

---

## DONE log

| Date | Commit | What |
|------|--------|------|
| 2026-11-24 | `898f5ff` | BFS `bfs_steps` in `Ghost` + target clamp ✅; `colors.py` imported in 10 files; power countdown in HUD; input debug prints removed; title `font_size` hack removed. ⚠ `+ 10000` in `afraid_of`, Instructions font, 3 color bugs |
| 2026-11-24 | `42359b2` | `colors.py`, `TextWidget.scale`, HUD cheat fits, P OFF resets invincible, ghost blink, min window 800x600, dead files deleted, `MOVES` |
| 2026-11-23/24 | `7924927`, `3792ffb` | Cheat block in HUD; Instructions text |
| 2026-11-22 | `c2b3b84`, `e2c7f6f` | Cheat keys, `invincible`, BACK, win + death fix, Instructions page, spawn guard (⚠ `__main__` commented) |
| ≤ 2026-11-22 | `c4169db` … `9f69bc7` | Title, timer, pacgum check, lives reset, keyboard, pause menu, highscores screen, ghost-in-wall fix, seeds, Winner/Loser + save, menu |
| 2026-11-24 | *(not committed)* | `README.md` draft (all subject IX sections) |
