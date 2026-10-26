# Three badge hacks to try on your GitHub Universe badge

An endless runner built from text. A pocket gallery of animated art. A hidden
screen unlocked by a secret button combination. These projects began as Arm
Create badge hacks, and you can build each one on the GitHub Universe badge
using the code in this post.

Every app below includes its complete source. Choose a hack, create the named
file on your badge, and paste in the code. The visible apps also include their
launcher icon definitions. You can complete the projects in any order; none
requires another hack to be installed.

This guide targets the Universe image with the four-column app grid, Mona,
photos, and achievements. Its launcher uses a shared icon dictionary and an
`update()` callback. The examples below fit that setup. If your menu files do
not contain the lines shown in the launcher steps, compare your version
before applying those edits.

## Before you start

### 1. Connect the badge and back up the launcher

You need your badge, a USB data cable, a computer, and a plain-text code editor
such as VS Code. All three hacks work offline using the firmware already on
the badge.

Connect the badge and double-tap **RESET** to enter Disk Mode. You can also
open the **Mass Storage** app from the badge's launcher. Open the mounted USB
drive in your file manager; it should contain `apps` and `assets`.

Copy the entire `apps/menu` folder into a dated backup folder on your computer.
Back up any existing `pipeline_runner`, `pocket_art_studio`, or `secret` folder
before reusing that name. Keep backups outside the badge's `apps` directory,
because folders containing app code can become launcher tiles themselves.

### 2. Recognize the paths and button names

All file creation and editing in this guide happens on the mounted badge
drive, while the badge is in Disk Mode. For example, `apps/menu/app.py` means
open the existing `apps` folder on that drive, then `menu`, then `app.py`.

The code uses `/system` when referring to the same files at runtime:

| In the mounted USB drive | In Python running on the badge |
| --- | --- |
| `apps/pipeline_runner` | `/system/apps/pipeline_runner` |
| `apps/pocket_art_studio` | `/system/apps/pocket_art_studio` |
| `apps/secret` | `/system/apps/secret` |
| `apps/menu` | `/system/apps/menu` |
| `assets/fonts/MonaSans-Medium.af` | `/system/assets/fonts/MonaSans-Medium.af` |

Use the drive's existing `apps` folder; the `/system` prefix belongs in the
code. Confirm that `assets/fonts/MonaSans-Medium.af` is present. The examples
use this bundled scalable font so their text sizes work with the Universe
image. They also use Universe's logical **SELECT**, **BACK**, **LEFT**,
**RIGHT**, **UP**, and **DOWN** controls. Follow the orientation used by your
badge's normal menu.

### 3. Use this save-and-test cycle

Save Python files as plain text, using the exact name `__init__.py` with two
underscores on each side of `init`. Check that your editor has not added
`.txt`. Preserve the indentation when copying code; spaces inside a function
or loop matter in Python.

After completing a hack's file and launcher steps:

1. Save every edited file and close the editor's files on the badge drive.
2. Safely eject the drive through your operating system and wait for ejection
   to finish.
3. Press **RESET once** to reboot into the launcher.
4. Open the app and test its controls.
5. Return to Disk Mode before making the next edit.

Reset after installing an app or changing the launcher: this Universe image
preloads menu modules and caches the app list. Each section below includes
its own test so you can finish one project before starting another.

## Hack 1: Pipeline Runner

Build a small arcade game whose player, obstacles, and collectibles are all
text. The complete app includes a title screen, jumping, ducking, increasing
speed, collisions, scoring, and a retry screen. Start with the working game,
then give it your own theme.

### 1. Create the app and paste in the complete code

In Disk Mode, create a folder named **`pipeline_runner`** inside the badge's
existing `apps` folder. Inside it, create **`__init__.py`**. Paste this entire
block into that file and save it:

```python
"""Jump and duck Arm technology blocks, and collect ASCII compute tokens."""

import math
import random


# ---------------------------------------------------------------------------
# CUSTOMIZATION STARTS HERE
# ---------------------------------------------------------------------------

# Change the game's name and the short message on its title screen.
GAME_TITLE = "PIPELINE RUNNER"
TAGLINE = "KEEP MODERN Arm MOVING"

# The player is ordinary text. Keep each row roughly the same width.
PLAYER_ART = (
  ".------.",
  "| CORE |",
  "'--[]--'",
)

# This shorter sprite appears while DOWN is held.
DUCK_ART = (
  ".------.",
  "|_CORE_|",
)

# Try another skin by uncommenting both assignments below.
# PLAYER_ART = (" /\\_/\\ ", "( o.o )", " > ^ < ")
# DUCK_ART = (" /\\_/\\ ", "( -.- )")

# Arm technology names decorate the game's obstacles and collectible tokens.
# These are game labels, not features provided by the badge's processor.
GROUND_HAZARDS = ("SME2", "SVE", "SVE2", "NGP", "NFRU", "CSS", "SoC")
OVERHEAD_HAZARDS = ("NSS", "NSSD", "RTOS", "MCP")
TOKEN_LABELS = ("NPU", "GPU", "DSP", "SME2", "SVE2", "AGI", "NEON", "TOSA", "LLVM")

# Movement values are pixels per second. Change one value at a time.
# A 50% faster scroll also puts 50% more distance between hazards because the
# spawn interval below remains unchanged.
BASE_SPEED = 157.5
MAX_SPEED = 210
SPEED_STEP = 9
GRAVITY = 450
JUMP_SPEED = 330

# One jump is the classic runner rule. Uncomment the second line for a
# forgiving double-jump.
MAX_JUMPS = 1
# MAX_JUMPS = 2

# Change the palette with red, green and blue values from 0 to 255.
BACKGROUND = color.rgb(5, 13, 24)
FOREGROUND = color.rgb(245, 247, 250)
ACCENT = color.rgb(0, 193, 222)
BONUS = color.rgb(255, 174, 45)
DANGER = color.rgb(255, 105, 75)
MUTED = color.rgb(115, 135, 150)

# Optional teaching and visual features. Change False to True.
SHOW_DEBUG_HITBOXES = False
ENABLE_SCANLINES = False
FLASH_CASE_LIGHTS = False

# Ignore this many pixels at each text row's edges for forgiving near misses.
# Set to 0 for tighter collisions; debug outlines show the exact active boxes.
HITBOX_INSET = 2

# ---------------------------------------------------------------------------
# CUSTOMIZATION ENDS HERE
# The game engine below is intentionally commented as a learning reference.
# ---------------------------------------------------------------------------

MENU = "/system/apps/menu"

FONT_SIZE = 15
SMALL_SIZE = 12
LINE_HEIGHT = 17
GROUND_Y = 190
PLAYER_X = 34
MAX_FRAME_TIME = 0.05

TITLE_ART = (
  "+--------------------+",
  "|    SVE2 + SME2     |",
  "|  CPU > GPU > NPU   |",
  "+------[ Arm ]-------+",
)

TRACK_TEXT = "==> SVE2 ==> SME2 ==> NEON ==> NPU ==> GPU ==> DSP ==> TOSA ==> LLVM "

badge.mode(HIRES | VSYNC)
screen.antialias = image.X2
game_font = font.load("/system/assets/fonts/MonaSans-Medium.af")
screen.font = game_font


# Return the rendered width of one line of ASCII art.
def text_width(text, size=FONT_SIZE):
  return screen.measure_text(text, size)[0]


# Layout height includes line spacing; collision uses the text rows below.
def art_size(lines):
  return max(text_width(line) for line in lines), len(lines) * LINE_HEIGHT


PLAYER_WIDTH, PLAYER_HEIGHT = art_size(PLAYER_ART)
DUCK_WIDTH, DUCK_HEIGHT = art_size(DUCK_ART)


# Measure once, ignoring blank rows and leading/trailing whitespace. Keeping a
# separate box for each row avoids collisions in an ASCII sprite's empty space.
def art_hitboxes(lines):
  boxes = []
  for row, line in enumerate(lines):
    text = line.strip()
    if not text:
      continue
    leading = len(line) - len(line.lstrip())
    left = text_width(line[:leading])
    width, height = screen.measure_text(text, FONT_SIZE)
    inset = max(0, min(HITBOX_INSET, (min(width, height) - 1) / 2))
    boxes.append((left + inset, row * LINE_HEIGHT + inset,
                  width - 2 * inset, height - 2 * inset))
  return boxes


PLAYER_HITBOXES = art_hitboxes(PLAYER_ART)
DUCK_HITBOXES = art_hitboxes(DUCK_ART)


# Use the same integer origin as draw_art(), including during a jump.
def placed_hitboxes(boxes, x, y):
  return [(int(x) + left, int(y) + top, width, height)
          for left, top, width, height in boxes]


# Build a hazard around any attendee-supplied label.
def boxed_art(label):
  middle = "| " + label + " |"
  edge = "+" + "-" * (len(middle) - 2) + "+"
  return (edge, middle, edge)


# Draw multi-line ASCII with its bottom edge fixed to the supplied position.
def draw_art(lines, x, bottom, pen, size=FONT_SIZE):
  screen.font = game_font
  screen.pen = pen
  y = int(bottom - len(lines) * LINE_HEIGHT)
  for line in lines:
    screen.text(line, int(x), int(y), size)
    y += LINE_HEIGHT


# Axis-aligned rectangles are sufficient for this intentionally simple game.
def overlaps(a, b):
  return (a[0] < b[0] + b[2] and a[0] + a[2] > b[0]
          and a[1] < b[1] + b[3] and a[1] + a[3] > b[1])


def hitboxes_overlap(first, second):
  return any(overlaps(a, b) for a in first for b in second)


# Debug outlines make collision detection visible to curious attendees.
def draw_outline(box, pen):
  x, y, w, h = box
  screen.pen = pen
  screen.rectangle(int(x), int(y), int(w), 1)
  screen.rectangle(int(x), int(y + h - 1), int(w), 1)
  screen.rectangle(int(x), int(y), 1, int(h))
  screen.rectangle(int(x + w - 1), int(y), 1, int(h))


# Mutable game state is kept together here so reset_game() is easy to follow.
state = "title"
player_bottom = GROUND_Y
player_velocity = 0.0
jumps_used = 0
ducking = False
score = 0.0
high_score = 0
hazards = []
tokens = []
spawn_in = 0.8
case_light_until = 0


def reset_game():
  """Restore the starting state for a new run."""
  global state, player_bottom, player_velocity, jumps_used, ducking
  global score, hazards, tokens, spawn_in

  state = "playing"
  player_bottom = GROUND_Y
  player_velocity = 0.0
  jumps_used = 0
  ducking = False
  score = 0.0
  hazards = []
  tokens = []
  spawn_in = 1.0


def current_speed():
  """Increase speed every 100 points, without exceeding MAX_SPEED."""
  level = int(score) // 100
  return min(MAX_SPEED, BASE_SPEED + level * SPEED_STEP)


def player_boxes():
  """Follow the active text rows, leaving line spacing out of collisions."""
  boxes = DUCK_HITBOXES if ducking else PLAYER_HITBOXES
  height = DUCK_HEIGHT if ducking else PLAYER_HEIGHT
  return placed_hitboxes(boxes, PLAYER_X, player_bottom - height)


def hazard_boxes(hazard):
  """Use the same measured rows and position as the visible obstacle."""
  return placed_hitboxes(hazard["hitboxes"], hazard["x"],
                         hazard["bottom"] - hazard["h"])


def token_boxes(token):
  return placed_hitboxes(token["hitboxes"], token["x"], token["y"])


def spawn_hazard():
  """Add an obstacle: jump ground boxes and duck under overhead alerts."""
  global spawn_in

  overhead = random.randint(0, 99) < 25
  if overhead:
    art = ("< " + random.choice(OVERHEAD_HAZARDS) + " >",)
    bottom = GROUND_Y - DUCK_HEIGHT
  else:
    art = boxed_art(random.choice(GROUND_HAZARDS))
    bottom = GROUND_Y

  width, height = art_size(art)
  x = screen.width + 8
  hazards.append({
    "x": x,
    "bottom": bottom,
    "overhead": overhead,
    "art": art,
    "w": width,
    "h": height,
    "hitboxes": art_hitboxes(art),
  })

  # A token above some hazards rewards a well-timed jump.
  if not overhead and random.randint(0, 99) < 55:
    token = "[" + random.choice(TOKEN_LABELS) + "]"
    tokens.append({
      "x": x + width / 2 - text_width(token) / 2,
      "y": GROUND_Y - height - 31,
      "text": token,
      "w": text_width(token),
      "hitboxes": art_hitboxes((token,)),
    })

  # Faster levels get slightly shorter gaps, but never an unfair wall.
  speed_part = (current_speed() - BASE_SPEED) / max(1, MAX_SPEED - BASE_SPEED)
  spawn_in = 1.35 + random.randint(0, 55) / 100 - speed_part * 0.35


def update_player(dt):
  """Read controls, then apply velocity and gravity."""
  global player_bottom, player_velocity, jumps_used, ducking

  on_ground = player_bottom >= GROUND_Y
  if on_ground:
    player_bottom = GROUND_Y
    player_velocity = 0.0
    jumps_used = 0

  ducking = on_ground and badge.held(BUTTON_DOWN)

  wants_jump = badge.pressed(BUTTON_SELECT) or badge.pressed(BUTTON_UP)
  if wants_jump and jumps_used < MAX_JUMPS:
    player_velocity = -JUMP_SPEED
    jumps_used += 1
    ducking = False

  player_velocity += GRAVITY * dt
  player_bottom += player_velocity * dt

  if player_bottom > GROUND_Y:
    player_bottom = GROUND_Y
    player_velocity = 0.0


def finish_run():
  """Move to the game-over state and retain a session high score."""
  global state, high_score
  state = "game_over"
  high_score = max(high_score, int(score))


def update_world(dt):
  """Scroll objects, collect tokens, detect crashes, and update the score."""
  global score, spawn_in, hazards, tokens, case_light_until

  speed = current_speed()
  score += dt * 10
  spawn_in -= dt
  if spawn_in <= 0:
    spawn_hazard()

  for hazard in hazards:
    hazard["x"] -= speed * dt
  for token in tokens:
    token["x"] -= speed * dt

  hazards = [hazard for hazard in hazards if hazard["x"] + hazard["w"] > -5]
  tokens = [token for token in tokens if token["x"] + token["w"] > -5]

  player = player_boxes()
  for hazard in hazards:
    if hitboxes_overlap(player, hazard_boxes(hazard)):
      finish_run()
      return

  remaining = []
  for token in tokens:
    if hitboxes_overlap(player, token_boxes(token)):
      score += 25
      case_light_until = badge.ticks + 140
    else:
      remaining.append(token)
  tokens = remaining


def draw_track():
  """Repeat a scrolling stream of Arm technology beneath the player."""
  screen.font = game_font
  screen.pen = MUTED
  width = text_width(TRACK_TEXT, SMALL_SIZE)
  offset = -int((badge.ticks * current_speed() / 1000) % width)
  x = offset
  while x < screen.width:
    screen.text(TRACK_TEXT, x, GROUND_Y + 9, SMALL_SIZE)
    x += width

  screen.pen = ACCENT
  screen.rectangle(0, GROUND_Y + 2, screen.width, 2)


def draw_world():
  """Draw one complete game frame from back to front."""
  screen.pen = BACKGROUND
  screen.clear()

  screen.font = game_font
  screen.pen = ACCENT
  screen.text(GAME_TITLE, 12, 10, FONT_SIZE)
  screen.pen = FOREGROUND
  score_text = "%04d" % int(score)
  screen.text(score_text, screen.width - text_width(score_text) - 12, 10,
              FONT_SIZE)

  draw_track()
  draw_art(DUCK_ART if ducking else PLAYER_ART, PLAYER_X, player_bottom,
           FOREGROUND)

  for hazard in hazards:
    draw_art(hazard["art"], hazard["x"], hazard["bottom"], DANGER)
  for token in tokens:
    screen.pen = BONUS
    screen.text(token["text"], int(token["x"]), int(token["y"]), FONT_SIZE)

  if SHOW_DEBUG_HITBOXES:
    for box in player_boxes():
      draw_outline(box, BONUS)
    for hazard in hazards:
      for box in hazard_boxes(hazard):
        draw_outline(box, BONUS)
    for token in tokens:
      for box in token_boxes(token):
        draw_outline(box, BONUS)


def draw_title():
  """Draw an inviting start screen using only text and ASCII characters."""
  screen.pen = BACKGROUND
  screen.clear()

  screen.font = game_font
  screen.pen = ACCENT
  title_width = text_width(GAME_TITLE, 25)
  screen.text(GAME_TITLE, int((screen.width - title_width) / 2), 22, 25)

  draw_art(TITLE_ART, (screen.width - art_size(TITLE_ART)[0]) / 2,
           124, BONUS)

  screen.pen = FOREGROUND
  prompt = "SELECT / UP  START + JUMP"
  screen.text(prompt, int((screen.width - text_width(prompt)) / 2), 145,
              FONT_SIZE)
  screen.pen = MUTED
  help_text = "DOWN DUCK   BACK MENU"
  screen.text(help_text, int((screen.width - text_width(help_text)) / 2), 169,
              SMALL_SIZE)
  screen.text(TAGLINE, int((screen.width - text_width(TAGLINE, SMALL_SIZE)) / 2),
              202, SMALL_SIZE)


def draw_game_over():
  """Overlay the result while leaving the crashed run visible underneath."""
  screen.pen = color.rgb(0, 0, 0, 220)
  screen.rectangle(35, 55, screen.width - 70, 105)

  screen.font = game_font
  screen.pen = DANGER
  message = "PIPELINE STALLED"
  screen.text(message, int((screen.width - text_width(message, 23)) / 2),
              70, 23)
  screen.pen = FOREGROUND
  result = "SCORE %d   BEST %d" % (int(score), high_score)
  screen.text(result, int((screen.width - text_width(result)) / 2), 108,
              FONT_SIZE)
  screen.pen = BONUS
  prompt = "SELECT RETRY   BACK MENU"
  screen.text(prompt, int((screen.width - text_width(prompt, SMALL_SIZE)) / 2),
              137, SMALL_SIZE)


def update_case_lights():
  """Optional hardware feedback: briefly flash after collecting a token."""
  if FLASH_CASE_LIGHTS and badge.ticks < case_light_until:
    pulse = 0.5 + math.sin(badge.ticks / 25) * 0.25
    badge.caselights(pulse, 0.1, pulse, 0.1)
  else:
    badge.caselights(0)


on_exit = None

while on_exit is None:
  # BACK always provides a predictable route back to the app launcher.
  if badge.pressed(BUTTON_BACK):
    badge.caselights(0)
    on_exit = MENU
    break

  if state == "title":
    if (badge.pressed(BUTTON_SELECT) or badge.pressed(BUTTON_UP)
        or badge.pressed(BUTTON_RIGHT)):
      reset_game()
    draw_title()

  elif state == "playing":
    dt = min(MAX_FRAME_TIME, badge.ticks_delta / 1000)
    update_player(dt)
    update_world(dt)
    draw_world()
    if state == "game_over":
      draw_game_over()

  else:
    draw_world()
    draw_game_over()
    if (badge.pressed(BUTTON_SELECT) or badge.pressed(BUTTON_UP)
        or badge.pressed(BUTTON_RIGHT)):
      reset_game()

  update_case_lights()

  # A scanline filter is a one-line example of Badgeware post-processing.
  if ENABLE_SCANLINES:
    screen.grid(4, 24, 0.45)

  badge.update()
```

### 2. Add the processor icon to the Universe launcher

Open the badge's existing **`apps/menu/app.py`** and find this import near the
top:

```python
from icons import ICONS
```

Immediately below that import, paste the following block. If you already
added Pocket Art Studio's icon there, keep it and add this block beside it.
Keep one definition for each app name.

```python
ICONS["pipeline_runner"] = (
  (-0.275862, -0.258621, 0.275862, -0.258621, 0.275862, 0.258621, -0.275862, 0.258621),
  (-0.413793, -0.189655, -0.310345, -0.189655, -0.310345, -0.137931, -0.413793, -0.137931),
  (-0.413793, -0.034483, -0.310345, -0.034483, -0.310345, 0.017241, -0.413793, 0.017241),
  (-0.413793, 0.12069, -0.310345, 0.12069, -0.310345, 0.172414, -0.413793, 0.172414),
  (0.310345, -0.189655, 0.413793, -0.189655, 0.413793, -0.137931, 0.310345, -0.137931),
  (0.310345, -0.034483, 0.413793, -0.034483, 0.413793, 0.017241, 0.310345, 0.017241),
  (0.310345, 0.12069, 0.413793, 0.12069, 0.413793, 0.172414, 0.310345, 0.172414),
  (-0.172414, -0.362069, -0.12069, -0.362069, -0.12069, -0.293103, -0.172414, -0.293103),
  (-0.034483, -0.362069, 0.017241, -0.362069, 0.017241, -0.293103, -0.034483, -0.293103),
  (0.103448, -0.362069, 0.155172, -0.362069, 0.155172, -0.293103, 0.103448, -0.293103),
  (-0.172414, 0.293103, -0.12069, 0.293103, -0.12069, 0.362069, -0.172414, 0.362069),
  (-0.034483, 0.293103, 0.017241, 0.293103, 0.017241, 0.362069, -0.034483, 0.362069),
  (0.103448, 0.293103, 0.155172, 0.293103, 0.155172, 0.362069, 0.103448, 0.362069),
)
```

Each tuple describes one filled contour, using x/y coordinates centred on
zero. The existing launcher builds a vector shape from these coordinates and
draws it in white over its own tile colour. The key `pipeline_runner` must
match the app folder name.

Keep the existing `icons.mpy` and other menu files. This assignment extends
its imported dictionary, so there is no compilation step. Universe does not
load an app's `icon.svg` automatically; the dictionary entry supplies the
artwork for this launcher.

The new app is discovered automatically and sorts after the names in
`APP_ORDER`. To place it earlier, find the existing `"mass_storage",` entry
inside the `APP_ORDER` tuple and insert `"pipeline_runner",` immediately
above it. Keep the other entries and do this only once.

### 3. Personalize the game

Open `apps/pipeline_runner/__init__.py` and find the comments marking
**CUSTOMIZATION STARTS HERE**. Edit the existing assignments in that block.
For example, replace the title, tagline, and label settings with:

```python
GAME_TITLE = "UNIVERSE RUNNER"
TAGLINE = "SHIP SMALL. KEEP MOVING."
GROUND_HAZARDS = ("BUG", "BUILD", "MERGE")
OVERHEAD_HAZARDS = ("REVIEW", "CI")
TOKEN_LABELS = ("PR", "STAR", "SHIP")
```

The original Arm technology names are game decorations; they do not describe
the badge's hardware features. Use short labels so obstacles leave room to
play. `TITLE_ART` and `TRACK_TEXT`, farther down the file, control the large
ASCII title graphic and the scrolling text beneath the track.

| Setting | What to change |
| --- | --- |
| `PLAYER_ART` | Replace the standing character's rows with your own ASCII sprite. |
| `DUCK_ART` | Use fewer rows to make a shorter ducking sprite. |
| `BACKGROUND`, `ACCENT`, `BONUS`, `DANGER` | Change RGB values from 0 to 255. |
| `JUMP_SPEED` | Increase it for a faster upward launch. |
| `GRAVITY` | Increase it for a stronger downward pull. |
| `BASE_SPEED` | Change the initial scrolling speed. |
| `MAX_JUMPS` | Set to `2` to allow a second jump. |
| `SHOW_DEBUG_HITBOXES` | Set to `True` to see the collision boxes. |
| `FLASH_CASE_LIGHTS` | Set to `True` to flash the rear lights when collecting a token. |
| `ENABLE_SCANLINES` | Set to `True` for a display filter. |

Change one physics value at a time. The engine measures each nonblank text
row and uses the same position for drawing and collision checks. Empty space
around a sprite does not create a solid rectangular hitbox. Mona Sans has
proportional glyph widths, so inspect your ASCII artwork on the badge after
changing it. `HITBOX_INSET` adds a small forgiveness margin at each row's
edges; the debug outlines show the active boxes.

### 4. Eject, launch, and play

Save `apps/pipeline_runner/__init__.py` and `apps/menu/app.py`, safely eject,
and press RESET once. Find **Pipeline Runner** in the grid and check that it
has the processor icon.

| Control | Action |
| --- | --- |
| SELECT or UP | Start; jump while playing. |
| Hold DOWN | Duck while on the ground. |
| SELECT, UP, or RIGHT | Retry after a collision. |
| BACK | Return to the launcher. |

Jump a ground obstacle, duck under an overhead sign, collect a token, and
check that the score increases. Then crash, retry, and press BACK to return
to the menu. The best score lasts for the current app session.

For a next challenge, add a rare token worth 100 points or a temporary shield.
The small functions in the engine separate controls, movement, collisions,
and drawing so you can explore one part at a time.

## Hack 2: Pocket Art Studio

Turn the badge into a gallery of spirals, kaleidoscopes, and ribbons. Each
mode draws a complete animated scene. A seed sets the starting design, and
the buttons let you remix it while it runs.

### 1. Create the app and paste in the complete code

Return to Disk Mode. In the badge's existing `apps` folder, create
**`pocket_art_studio`**, then create **`__init__.py`** inside it. Paste the
entire block below and save it. You can start with this hack even if you
skipped Pipeline Runner.

```python
"""A pocket gallery of animated patterns, ready to remix in Python.

SELECT: next mode. RIGHT: remix. UP/DOWN: speed. BACK: launcher.
Keep exploring at home: connect by USB, double-tap RESET for disk mode, and
back up apps/pocket_art_studio to your computer. Edit this file as plain text,
save it back as __init__.py, safely eject the drive, and press RESET once.
The badge supplies the graphics and buttons; run this app on the badge.
"""

import math
import random


# ---------------------------------------------------------------------------
# CUSTOMIZATION STARTS HERE
# ---------------------------------------------------------------------------

# Give your studio a name. Short titles fit best.
TITLE = "POCKET ART STUDIO"

# Start with "Spiral", "Kaleidoscope", or "Ribbons". SELECT cycles through them.
START_MODE = "Spiral"

# Change the colours using red, green and blue values from 0 to 255.
# Keep at least one colour in PALETTE. Add more lines to add more colours.
BACKGROUND = color.rgb(8, 5, 24)
PALETTE = (
  color.rgb(155, 95, 255),
  color.rgb(0, 220, 235),
  color.rgb(255, 110, 155),
  color.rgb(255, 195, 90),
)

# Try 3 for a simpler pattern, or 9 for more repeated arms and ribbons.
SYMMETRY = 6                 # 2 to 12
DETAIL = 28                  # 12 to 48 points per arm; fewer draws faster
DOT_SIZE = 3                 # 1 to 6 pixels; used by Spiral
SPEED = 1.0                  # 0.25 to 3.0; UP / DOWN also changes this live

# The same seed starts the same design. RIGHT remixes it using the next seed.
SEED = 42

# Labels disappear after a few seconds; any art control brings them back.
SHOW_LABELS = True
ENABLE_SCANLINES = False
PULSE_CASE_LIGHTS = False

# ---------------------------------------------------------------------------
# CUSTOMIZATION ENDS HERE
# Each draw_* function below is a small project to keep exploring at home.
# ---------------------------------------------------------------------------

MENU = "/system/apps/menu"
TAU = math.pi * 2
MAX_FRAME_TIME = 0.05
LABEL_SECONDS = 5.0

badge.mode(HIRES | VSYNC)
screen.antialias = image.X2
studio_font = font.load("/system/assets/fonts/MonaSans-Medium.af")
screen.font = studio_font

CX, CY = screen.width / 2, screen.height / 2
RADIUS = min(screen.width, screen.height) / 2 - 16

# Keep the amount of drawing bounded while experimenting with the settings.
symmetry = max(2, min(12, int(SYMMETRY)))
detail = max(12, min(48, int(DETAIL)))
dot_size = max(1, min(6, int(DOT_SIZE)))
speed = max(0.25, min(3.0, SPEED))
pens = PALETTE or (color.rgb(255, 255, 255),)
seed = SEED
animation_time = 0.0
labels_remaining = LABEL_SECONDS


def remix():
  """Pick the design once, rather than generating noisy randomness each frame."""
  global phase, twists, frequency, palette_offset
  random.seed(seed)
  phase = random.randint(0, 628) / 100
  twists = random.randint(2, 5)
  frequency = random.randint(2, 4)
  palette_offset = random.randint(0, len(pens) - 1)


def pick_pen(index):
  return pens[(index + palette_offset) % len(pens)]


def draw_spiral(t):
  """Grow coloured dots along rotating spiral arms."""
  for arm in range(symmetry):
    screen.pen = pick_pen(arm)
    for step in range(detail):
      fraction = step / (detail - 1)
      angle = arm * TAU / symmetry + fraction * TAU * twists + t * 0.5
      pulse = 0.85 + 0.12 * math.sin(t + fraction * TAU + phase)
      radius = (8 + fraction * (RADIUS - 8)) * pulse
      x = CX + math.cos(angle) * radius
      y = CY + math.sin(angle) * radius
      size = max(1, int(dot_size * (0.4 + fraction * 0.6)))
      screen.circle(int(x), int(y), size)


def draw_kaleidoscope(t):
  """Reflect a wavy line in each wedge, then rotate the whole pattern."""
  wedge = TAU / symmetry
  for arm in range(symmetry):
    for mirror in (-1, 1):
      previous = None
      for step in range(detail):
        fraction = step / (detail - 1)
        bend = math.sin(fraction * TAU * frequency + t + phase) * wedge * 0.42
        angle = arm * wedge + t * 0.12 + mirror * bend
        radius = (0.12 + fraction * 0.85) * RADIUS
        x = int(CX + math.cos(angle) * radius)
        y = int(CY + math.sin(angle) * radius)
        screen.pen = pick_pen(step // 4)
        if previous is not None:
          screen.line(previous[0], previous[1], x, y)
        previous = (x, y)


def draw_ribbons(t):
  """Layer two sine waves to make coloured ribbons weave across the screen."""
  for ribbon in range(symmetry):
    screen.pen = pick_pen(ribbon)
    offset = ribbon * TAU / symmetry + phase
    previous = None
    for step in range(detail * 2):
      fraction = step / (detail * 2 - 1)
      x = int(12 + fraction * (screen.width - 24))
      wave = math.sin(fraction * TAU * frequency + t + offset)
      ripple = math.cos(fraction * TAU * twists - t * 0.6 + offset)
      y = int(CY + wave * RADIUS * 0.65 + ripple * RADIUS * 0.25)
      if previous is not None:
        screen.line(previous[0], previous[1], x, y)
      previous = (x, y)


# To add a mode, write draw_your_mode(t) above, then add its name and function.
SCENES = (
  ("Spiral", draw_spiral),
  ("Kaleidoscope", draw_kaleidoscope),
  ("Ribbons", draw_ribbons),
)
scene_index = 0
for index, scene in enumerate(SCENES):
  if scene[0] == START_MODE:
    scene_index = index
remix()


def update_studio(dt):
  """Read each press once; advance time independently of the frame rate."""
  global scene_index, seed, speed, animation_time, labels_remaining
  dt = max(0, min(MAX_FRAME_TIME, dt))
  labels_remaining = max(0, labels_remaining - dt)

  if badge.pressed(BUTTON_SELECT):
    scene_index = (scene_index + 1) % len(SCENES)
    labels_remaining = LABEL_SECONDS
  if badge.pressed(BUTTON_RIGHT):
    seed += 1
    remix()
    animation_time = 0.0
    labels_remaining = LABEL_SECONDS
  if badge.pressed(BUTTON_UP):
    speed = min(3.0, speed + 0.25)
    labels_remaining = LABEL_SECONDS
  if badge.pressed(BUTTON_DOWN):
    speed = max(0.25, speed - 0.25)
    labels_remaining = LABEL_SECONDS

  animation_time += dt * speed


def draw_labels():
  """Keep the controls readable, then leave the art on its own."""
  if not SHOW_LABELS or labels_remaining <= 0:
    return
  screen.font = studio_font
  screen.pen = BACKGROUND
  screen.rectangle(0, 0, screen.width, 44)
  screen.rectangle(0, screen.height - 37, screen.width, 37)
  screen.pen = pick_pen(0)
  # Shrink long custom titles to fit the screen.
  title_size = 18
  while title_size > 8 and screen.measure_text(TITLE, title_size)[0] > screen.width - 20:
    title_size -= 1
  screen.text(TITLE, 10, 5, title_size)
  screen.pen = color.rgb(245, 240, 255)
  status = "%s  %.2fx  seed %d" % (SCENES[scene_index][0], speed, seed)
  screen.text(status, 10, 27, 12)
  screen.text("SELECT MODE  RIGHT REMIX", 10, screen.height - 32, 12)
  screen.text("UP/DOWN SPEED  BACK MENU", 10, screen.height - 17, 12)


def draw_studio():
  screen.pen = BACKGROUND
  screen.clear()
  SCENES[scene_index][1](animation_time)
  if ENABLE_SCANLINES:
    screen.grid(4, 24, 0.45)
  draw_labels()

  if PULSE_CASE_LIGHTS:
    pulse = (math.sin(animation_time) + 1) * 0.2
    badge.caselights(pulse, pulse * 0.4, pulse, pulse * 0.4)
  else:
    badge.caselights(0)


on_exit = None

while on_exit is None:
  if badge.pressed(BUTTON_BACK):
    badge.caselights(0)
    on_exit = MENU
    break

  update_studio(badge.ticks_delta / 1000)
  draw_studio()
  badge.update()
```

### 2. Add the kaleidoscope icon to the Universe launcher

Open **`apps/menu/app.py`** and find:

```python
from icons import ICONS
```

Paste the following block immediately below it. If Pipeline Runner's icon is
already there, keep that block too. The two assignments can sit next to each
other, after the import.

```python
ICONS["pocket_art_studio"] = (
  (0, -0.362069, 0.12069, -0.137931, 0, -0.034483, -0.12069, -0.137931),
  (0, 0.362069, -0.12069, 0.137931, 0, 0.034483, 0.12069, 0.137931),
  (-0.310345, -0.189655, -0.051724, -0.172414, -0.017241, -0.034483, -0.172414, 0.017241),
  (0.310345, 0.189655, 0.051724, 0.172414, 0.017241, 0.034483, 0.172414, -0.017241),
  (-0.310345, 0.189655, -0.172414, -0.017241, -0.017241, 0.034483, -0.051724, 0.172414),
  (0.310345, -0.189655, 0.172414, 0.017241, 0.017241, -0.034483, 0.051724, -0.172414),
  (0.068966, 0, 0.048759, 0.048759, 0, 0.068966, -0.048759, 0.048759, -0.068966, 0,
   -0.048759, -0.048759, 0, -0.068966, 0.048759, -0.048759),
)
```

The key `pocket_art_studio` matches the app's directory name. This adds a
static icon to the existing dictionary; keep `icons.mpy` and the other menu
assets. No animation file is needed. Editing `apps/menu/src/icons.py` alone
would not change the compiled icons used by the menu, so use the assignment
in `app.py` shown here.

Universe discovers the app automatically. To move its tile earlier, add
`"pocket_art_studio",` immediately before `"mass_storage",` inside the
existing `APP_ORDER` tuple. Preserve any other app entries you have added.

### 3. Choose your starting design and palette

In `apps/pocket_art_studio/__init__.py`, find **CUSTOMIZATION STARTS HERE**.
Change the existing settings, for example:

```python
TITLE = "MY POCKET UNIVERSE"
START_MODE = "Kaleidoscope"
SYMMETRY = 6
DETAIL = 28
SPEED = 0.75
SEED = 2026

PALETTE = (
  color.rgb(155, 95, 255),
  color.rgb(95, 237, 131),
  color.rgb(255, 125, 237),
)
```

Use `"Spiral"`, `"Kaleidoscope"`, or `"Ribbons"` for `START_MODE`, matching
the capitalization. Keep at least one colour in `PALETTE`.

| Setting | What it changes |
| --- | --- |
| `SYMMETRY` | Number of repeated arms, reflections, or ribbons; use 2–12. |
| `DETAIL` | Points per arm; use 12–48. Lower values draw less work per frame. |
| `DOT_SIZE` | Spiral dot size, from 1–6 pixels. |
| `SPEED` | Starting animation speed, from 0.25–3.0. |
| `SEED` | Starting design; the same seed recreates it on the same firmware. |
| `BACKGROUND`, `PALETTE` | Background and drawing colours. |
| `SHOW_LABELS` | Set to `False` to hide the text while retaining the controls. |
| `PULSE_CASE_LIGHTS` | Set to `True` for gently pulsing rear lights. |
| `ENABLE_SCANLINES` | Set to `True` for a display filter. |

### 4. Add an optional fourth drawing mode

Each mode has a short `draw_*` function. To add your own, paste this function
**above the existing `SCENES` tuple**, beside the other drawing functions:

```python
def draw_orbit(t):
  for dot in range(symmetry):
    angle = t + dot * TAU / symmetry
    x = CX + math.cos(angle) * RADIUS * 0.8
    y = CY + math.sin(angle * 2) * RADIUS * 0.6
    screen.pen = pick_pen(dot)
    screen.circle(int(x), int(y), dot_size)
```

Then replace the existing `SCENES` tuple with this version:

```python
SCENES = (
  ("Spiral", draw_spiral),
  ("Kaleidoscope", draw_kaleidoscope),
  ("Ribbons", draw_ribbons),
  ("Orbit", draw_orbit),
)
```

SELECT will now cycle through four modes. Set `START_MODE = "Orbit"` if you
want the new one to appear first. The function receives animation time `t`
in seconds, adjusted by your speed setting. The app clears the screen before
calling it, so draw the complete scene on each frame. `pick_pen(index)` cycles
through your palette.

### 5. Eject and test every control

Save `apps/pocket_art_studio/__init__.py` and `apps/menu/app.py`, safely
eject, and press RESET once. Open **Pocket Art Studio** and check its icon.

| Control | Action |
| --- | --- |
| SELECT | Cycle through drawing modes. |
| RIGHT | Remix using the next seed. |
| UP / DOWN | Increase / decrease animation speed. |
| BACK | Return to the launcher. |

Test every mode, remix, both speed buttons, and BACK. Labels disappear after
five seconds so the artwork fills the screen; any art control brings them
back. Reduce `DETAIL` or `SYMMETRY` if animation slows.

Live speed and seed changes last for the current run. To keep a favourite,
return to Disk Mode and update `SPEED` and `SEED` in the file. For another
experiment, change one `math.sin` to `math.cos` inside `draw_ribbons(t)` and
compare the movement.

## Hack 3: Secret Button Combo

Create a hidden screen and teach the Universe launcher to reveal it after
**UP, UP, DOWN, DOWN, LEFT, RIGHT, LEFT, RIGHT, SELECT**. The screen can hold
a message, an image, a GitHub QR code, or a clue for a scavenger hunt.

This hack has three required files. Keep the badge in Disk Mode until the
screen, hidden-app filter, and sequence handler are all saved:

| File on the mounted badge drive | What you will do |
| --- | --- |
| `apps/secret/__init__.py` | Create the complete hidden screen. |
| `apps/menu/app.py` | Exclude `secret` from tile discovery. |
| `apps/menu/__init__.py` | Add the tracker and call it inside the existing `update()`. |

### 1. Create the hidden app and paste in the complete code

In the badge's existing `apps` folder, create **`secret`**, then create
**`__init__.py`** inside it. Paste this entire block into that file and save:

```python
"""A hidden screen for attendees to discover and customize."""

import math


# ---------------------------------------------------------------------------
# CUSTOMIZATION STARTS HERE
# ---------------------------------------------------------------------------

# Change the words shown on the secret screen. Short titles fit best.
TITLE = "THE HIDDEN LAYER"
MESSAGE = "You found the badge's secret screen. Edit this app and make it yours."
HELP_TEXT = "Try adding a message, image, QR code or lights."
FOOTER = "BACK  MENU"

# Change the screen colours. Values are red, green and blue, from 0 to 255.
BACKGROUND = color.rgb(10, 16, 28)
ACCENT = color.rgb(255, 120, 30)
TEXT = color.rgb(255, 255, 255)
MUTED = color.rgb(170, 180, 195)

EXTRA_TEXT = None
# Optional extra text. Uncomment the example below to override None.
# EXTRA_TEXT = "I found the hidden screen at GitHub Universe!"

# Optional PNG. Copy the image into apps/secret, then uncomment its path.
# It will be scaled into the visual box, so a small square PNG works best.
IMAGE_PATH = None
# IMAGE_PATH = "/system/apps/secret/my-secret-image.png"

# Optional QR code. Use either QR_TEXT or IMAGE_PATH, not both. IMAGE_PATH wins
# when both are set.
QR_TEXT = None
# QR_TEXT = "https://github.com/your-name"

# Change False to True for softly pulsing rear case lights.
ANIMATE_CASE_LIGHTS = False

# ---------------------------------------------------------------------------
# CUSTOMIZATION ENDS HERE
# The drawing code below normally does not need to be changed.
# ---------------------------------------------------------------------------

MENU = "/system/apps/menu"

TITLE_SIZE = 27
MESSAGE_SIZE = 19
SMALL_SIZE = 14
MARGIN = 22
VISUAL_SIZE = 112

badge.mode(HIRES | VSYNC)
screen.antialias = image.X2

secret_font = font.load("/system/assets/fonts/MonaSans-Medium.af")
screen.font = secret_font


def build_qr(text, size):
  """Fill the visual box using the QR image's existing white border."""
  code = image.qr(text, image.QR_LOW, 1)
  if code is None or max(code.width, code.height) > size:
    return None

  plate = image(size, size)
  plate.pen = color.rgb(255, 255, 255)
  plate.clear()
  plate.blit(code, rect(0, 0, code.width, code.height),
             rect(0, 0, size, size))
  return plate


visual = None
visual_error = None

if IMAGE_PATH:
  try:
    visual = image.load(IMAGE_PATH)
  except (OSError, ValueError):
    visual_error = "IMAGE NOT FOUND"
elif QR_TEXT:
  try:
    visual = build_qr(QR_TEXT, VISUAL_SIZE)
    if visual is None:
      visual_error = "QR TEXT TOO LONG"
  except ValueError:
    visual_error = "QR TEXT TOO LONG"


def draw_centered(text, y, size, pen):
  screen.font = secret_font
  screen.pen = pen
  width, _ = screen.measure_text(text, size)
  screen.text(text, int((screen.width - width) / 2), y, size)


def draw_screen():
  screen.pen = BACKGROUND
  screen.clear()

  # A simple frame and accent rule make the default screen feel intentional.
  screen.pen = ACCENT
  screen.rectangle(0, 0, 7, screen.height)
  screen.rectangle(MARGIN, 62, screen.width - MARGIN * 2, 3)

  draw_centered(TITLE, 25, TITLE_SIZE, ACCENT)

  visual_x = screen.width - MARGIN - VISUAL_SIZE
  text_width = screen.width - MARGIN * 2
  if visual is not None or visual_error:
    text_width = visual_x - MARGIN - 14

  body_size = 15 if visual is not None or visual_error else MESSAGE_SIZE
  help_size = 12 if visual is not None or visual_error else SMALL_SIZE

  screen.font = secret_font
  screen.pen = TEXT
  screen.text(MESSAGE, rect(MARGIN, 82, text_width, 72), body_size,
              line_height=1.05)

  screen.pen = MUTED
  screen.text(HELP_TEXT, rect(MARGIN, 160, text_width, 32), help_size,
              line_height=1.05)

  if EXTRA_TEXT:
    screen.pen = ACCENT
    screen.text(EXTRA_TEXT, rect(MARGIN, 195, text_width, 20), SMALL_SIZE)

  if visual is not None:
    screen.blit(visual, rect(visual_x, 82, VISUAL_SIZE, VISUAL_SIZE))
  elif visual_error:
    screen.pen = ACCENT
    screen.text(visual_error, rect(visual_x, 120, VISUAL_SIZE, 30),
                SMALL_SIZE, align=(image.CENTER, image.CENTER))

  screen.pen = MUTED
  screen.text(FOOTER, MARGIN, screen.height - 24, SMALL_SIZE)

  # Optional display effects: remove the leading # from one line to try it.
  # screen.bloom(180, 130, 3.0)
  # screen.grid(4, 25, 0.5)
  # screen.glitch(20, 0.25)


def update_case_lights():
  if not ANIMATE_CASE_LIGHTS:
    badge.caselights(0)
    return

  pulse = (math.sin(badge.ticks / 350) + 1.0) / 2.0
  badge.caselights(pulse * 0.65, pulse * 0.2,
                   pulse * 0.65, pulse * 0.2)


on_exit = None

while on_exit is None:
  draw_screen()
  update_case_lights()

  if badge.pressed(BUTTON_BACK):
    badge.caselights(0)
    on_exit = MENU
    break

  badge.update()
```

### 2. Hide the app from the launcher grid

Open the badge's existing **`apps/menu/app.py`**. Above `APP_ORDER`, add:

```python
HIDDEN_APPS = ("menu", "secret")
```

If your file already has a hidden-app tuple, add `"secret"` to it and retain
its existing entries. Keep a single definition.

Inside **`Apps.__init__()`**, find the following existing lines:

```python
            if slug == "menu":
                continue
```

Replace those two lines with:

```python
            if slug in HIDDEN_APPS:
                continue
```

Preserve their indentation and the following check for `__init__.py` or
`__init__.mpy`. Keep the rest of the discovery code and any icon assignments
from the other hacks. Save the file.

The hidden screen stays launchable at `/system/apps/secret` but gets no tile.
It needs no icon. Universe discovers directories containing app code even
without artwork, so leaving out an SVG would not hide it.

### 3. Add the sequence settings and tracker

Open **`apps/menu/__init__.py`**. This Universe launcher already contains
`restore()` and `update()`, and ends with:

```python
on_exit = run(update).result
```

Keep those functions and that final line. Find **`def update():`** and paste
this whole block **immediately above it**, after the existing `restore()`
function has ended. The settings and new function definitions start at the
left edge, outside any other function:

```python
SECRET_APP = "/system/apps/secret"
SECRET_TIMEOUT_MS = 3000
SECRET_SEQUENCE = (
  BUTTON_UP, BUTTON_UP,
  BUTTON_DOWN, BUTTON_DOWN,
  BUTTON_LEFT, BUTTON_RIGHT,
  BUTTON_LEFT, BUTTON_RIGHT,
  BUTTON_SELECT,
)

secret_index = 0
secret_last_press = 0


def reset_secret_sequence():
  global secret_index, secret_last_press
  secret_index = 0
  secret_last_press = 0


def check_secret_sequence():
  global secret_index, secret_last_press
  pressed = badge.pressed()
  if not pressed:
    return False

  if len(pressed) != 1:
    reset_secret_sequence()
    return False

  now = badge.ticks
  if now - secret_last_press > SECRET_TIMEOUT_MS:
    secret_index = 0
  secret_last_press = now
  button = pressed[0]

  if button == SECRET_SEQUENCE[secret_index]:
    secret_index += 1
    if secret_index == len(SECRET_SEQUENCE):
      reset_secret_sequence()
      return True
  else:
    secret_index = 1 if button == SECRET_SEQUENCE[0] else 0

  return False
```

The tracker counts newly pressed buttons. Releasing UP and pressing it again
counts twice; holding UP counts once. A wrong press starts the attempt over,
with a new initial UP retained as the beginning of another try. More than
three seconds between inputs or multiple newly pressed buttons in one frame
resets the attempt.

### 4. Call the tracker inside the existing `update()`

Adding the helper alone does not make the launcher watch the buttons. In the
same file, go **inside the existing `update()` function** and find:

```python
    picker = apps.picker
```

This line comes after the opening-animation and pending-achievement handling.
Insert the block below **directly above that line**. Keep `picker = apps.picker`
and all the existing code beneath it:

```python
    # Watch the ordinary grid, after opening/achievement state is updated.
    secret_ready = (
        opening is None
        and viewer is None
        and celebration is None
        and (apps.picker is None or not apps.picker.is_open)
    )
    if secret_ready:
        if check_secret_sequence():
            restore()
            return SECRET_APP
    else:
        reset_secret_sequence()
```

The outer indentation is four spaces to match Universe's existing `update()`
body. The surrounding code should remain in this order:

1. `def update():` and its existing `global` declaration.
2. Existing opening-animation handling.
3. Existing pending-achievement handling.
4. The new `secret_ready` block.
5. Existing `picker = apps.picker` and the normal input and drawing code.
6. The existing `on_exit = run(update).result`, outside the function.

Save the file. A completed combo calls the existing `restore()` before
returning the hidden app's path. That follows the normal launcher exit path,
including closing achievement listening and restoring display state. The
existing runner passes the returned path back to the badge's main program,
which opens the hidden screen.

The check must come before normal SELECT handling so the last press opens
the secret screen instead of the selected tile. Earlier direction presses
will also move the grid; that is expected. The tracker resets while the
opening animation, a photo viewer, an achievement picker, or a celebration
is active.

### 5. Make the secret screen yours

In `apps/secret/__init__.py`, find **CUSTOMIZATION STARTS HERE**. Replace the
existing text assignments, for example:

```python
TITLE = "THE OCTOVERSE"
MESSAGE = "A little curiosity goes a long way. Keep building."
HELP_TEXT = "This screen is yours to remix."
EXTRA_TEXT = "Unlocked at GitHub Universe"
ANIMATE_CASE_LIGHTS = True
```

Keep the title short enough to fit. If needed, reduce `TITLE_SIZE` farther
down the file. Change `BACKGROUND`, `ACCENT`, and `TEXT` using RGB values from
0 to 255.

To show a QR code, replace the existing `QR_TEXT = None` assignment with:

```python
QR_TEXT = "https://github.com/your-name"
```

Replace `your-name` with your GitHub username. Keep `IMAGE_PATH = None` for
this version. The app uses Universe's built-in `image.qr()` encoder, as the
badge's Contacts app does. It scales the QR image to fill the 112 × 112 pixel
box, keeping the encoder's white border without adding another layer of
padding. Use a short URL and test it with a phone. No extra Python module is
required.

For an image instead, put a small PNG of your own inside `apps/secret` and
set the existing assignments to:

```python
IMAGE_PATH = "/system/apps/secret/my-secret-image.png"
QR_TEXT = None
```

Use your PNG's exact filename. It is scaled into a 112 × 112 pixel box on the
right. Start with a small square image to keep memory use modest. You can also
leave both settings as `None` to use the full width for text.

To personalize the combo, edit the existing `SECRET_SEQUENCE` tuple in
`apps/menu/__init__.py`. Keep it nonempty, use direction buttons, and put
SELECT only at the end. An earlier SELECT would open the currently selected
menu tile. `SECRET_TIMEOUT_MS = 3000` allows three seconds between consecutive
inputs, rather than three seconds for the entire sequence. This is an Easter
egg: its source and message are readable over USB.

### 6. Eject and unlock the secret

Before ejecting, confirm all three files are saved:

- `apps/secret/__init__.py` contains the screen code.
- `apps/menu/app.py` contains both `HIDDEN_APPS` and the `slug` filter.
- `apps/menu/__init__.py` contains the tracker functions and the call inside
  the existing `update()`.

Safely eject, wait for completion, and press RESET once. Confirm that
**Secret has no tile**. Wait for the ordinary grid and close any photo viewer
or achievement picker. Then press:

**UP → UP → DOWN → DOWN → LEFT → RIGHT → LEFT → RIGHT → SELECT**

Release each button before pressing the next, keeping less than three seconds
between inputs. Your screen should open. Check its text, optional visual, and
case lights, then press **BACK** to return to the menu.

Try a wrong sequence and a pause longer than three seconds. Neither should
unlock the screen. Check that normal app selection, photos, achievements,
Settings, and Mass Storage still work.

## Fixing problems and continuing the experiment

| Symptom | What to check |
| --- | --- |
| App does not appear | Use the exact folder name and `__init__.py` filename; avoid an extra nested folder or `.txt` extension. Eject and reset after copying. |
| App shows the default icon | Put the icon assignment after `from icons import ICONS` in `apps/menu/app.py`. Its dictionary key must match the app folder. Reset after saving. |
| App returns immediately or reports a font error | Confirm `assets/fonts/MonaSans-Medium.af` exists. Copy the full app code, including its font path and final loop. |
| Secret reports `ImportError: no module named 'qrcode'` | An earlier version used an API from the Arm Create firmware. Remove `import qrcode`. If `QR_TEXT` is set, also replace `build_qr()` with the version above, which uses Universe's built-in `image.qr()`. |
| Python reports an indentation or syntax error | Copy only the Python between the code fences, preserving spaces. Check for missing lines or an extra `.txt` extension. |
| Secret has a visible tile | Check the filter inside `Apps.__init__()`. An app without artwork still appears unless it is filtered out. |
| The combo opens a normal app | Put the secret check before the existing SELECT handler and use SELECT only as the combo's final button. |
| The combo never triggers | Check that both the tracker and its call were saved. Start on the normal grid, release each press, and stay within the timeout. |
| Launcher fails after editing | Double-tap RESET for Disk Mode. Restore `apps/menu/app.py` and `apps/menu/__init__.py` from your computer's backup, then eject and reset. |

You can inspect exceptions with a MicroPython serial console such as Thonny;
close the console before switching to Disk Mode. These apps run on the badge,
where the firmware supplies `badge`, `screen`, `font`, and the button
constants. Pasting them into desktop Python does not provide that environment.

Keep your working backups on your computer and leave Wi-Fi credentials in
the badge's existing `secrets.py`, outside any code you publish. After an
experiment works, make a fresh backup before extending it.

For more ideas, the public Badgeware
[drawing API](https://github.com/pimoroni/badgeware-docs/blob/main/api/image.md),
[font API](https://github.com/pimoroni/badgeware-docs/blob/main/api/font.md), and
[button and timing API](https://github.com/pimoroni/badgeware-docs/blob/main/api/badge.md)
explain the building blocks used here. All code and icon definitions needed
for these three projects are included above.

*Compatibility note: the code and launcher edits were checked against the
supplied Universe image using firmware stand-ins, including app behaviour,
icon discovery, and sequence handling. Rehearse the display, physical buttons,
USB workflow, optional QR scanning, and launcher transitions on your badge.*
