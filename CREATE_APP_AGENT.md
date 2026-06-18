# AnberNic RG40XXV App Development Guide

## Structure

Each app consists of a `.sh` launcher script in the root and a subdirectory of the same name containing the Python app. An optional `Imgs/<name>.png` icon provides a preview in the launcher menu.

```
APPS - AnberNic - RG40XXV/
├── <app>.sh                     # Bash launcher (always 13 lines for Python apps)
├── <app>/                       # Python app directory (same name as .sh)
│   ├── main.py                  # Hardware detection + entry point (identical across apps)
│   ├── app.py                   # Application logic + state machine (app-specific)
│   ├── graphic.py               # SDL2 rendering wrapper (similar across apps)
│   ├── input.py                 # Gamepad input from /dev/input/event1 (identical across apps)
│   ├── language.py              # Translator(i18n) class (identical across apps)
│   ├── anbernic.py              # SD card storage management (similar across apps)
│   ├── sdl2.zip                 # Pre-packaged PySDL2 module (identical across apps)
│   ├── components/              # Additional code modules (optional)
│   ├── font/
│   │   └── font.ttf             # App-specific font (optional)
│   ├── lang/
│   │   ├── en_US.json           # English strings
│   │   ├── de_DE.json           # German strings
│   │   └── ...                  # Additional locales
│   ├── sound/
│   │   └── sound.wav            # Sound effects (optional)
│   └── log.txt                  # Runtime log (generated)
├── Imgs/
│   └── <app>.png               # Launcher icon (optional)
└── res/                         # Shared resources (e.g. videos)
```

## The .sh Launcher Script (Python Apps)

Every Python-based `.sh` follows this exact 13-line pattern:

```bash
#!/bin/bash

progdir="$(cd $(dirname "$0") || exit; pwd)"/<app_dir>

export PYSDL2_DLL_PATH="/usr/lib"

program="python3 ${progdir}/main.py"
log_file="${progdir}/log.txt"
[ -f /mnt/mod/ctrl/volumeCtrl.dge ] && /mnt/mod/ctrl/volumeCtrl.dge &

$program > "$log_file" 2>&1

kill -9 $(pidof volumeCtrl.dge)
```

- `progdir` = `(script's directory)/<subdir>`
- `PYSDL2_DLL_PATH` always `/usr/lib`
- `log_file` always `progdir/log.txt`
- `volumeCtrl.dge` backgrounded before app, killed after
- stdout+stderr redirected to `log.txt`
- For menu/settings apps with a config arg: `python3 ${progdir}/main.py ${progdir}/config.json`
- For Go-based apps (e.g. cloud/): use a different pattern (no SDL2, no volumeCtrl)

## `main.py` — Entry Point (identical boilerplate across all apps)

```python
#!/usr/bin/env python3

from pathlib import Path
import zipfile
import os

board_mapping = {
    'RGcubexx': 1,
    'RG34xx': 2,    'RG34xxSP': 2,
    'RG28xx': 3,
    'RG35xx+_P': 4,
    'RG35xxH': 5,
    'RG35xxSP': 6,
    'RG40xxH': 7,
    'RG40xxV': 8,
    'RG35xxPRO': 9
}
system_list = ['zh_CN', 'zh_TW', 'en_US', 'ja_JP', 'ko_KR', 'es_LA', 'ru_RU', 'de_DE', 'fr_FR', 'pt_BR']

try:
    board_info = Path("/mnt/vendor/oem/board.ini").read_text().splitlines()[0]
except (FileNotFoundError, IndexError):
    board_info = 'RG35xxH'

try:
    lang_info = Path("/mnt/vendor/oem/language.ini").read_text().splitlines()[0]
except (FileNotFoundError, IndexError):
    lang_info = 2

try:
    hdmi_info = Path("/sys/class/extcon/hdmi/state").read_text().splitlines()[0]
except (FileNotFoundError, IndexError):
    hdmi_info = 'HDMI=0'

hw_info = board_mapping.get(board_info, 5)
system_lang = system_list[int(lang_info)]

def ensure_sdl2():
    try:
        import sdl2
        return True
    except ImportError:
        try:
            program = os.path.dirname(os.path.abspath(__file__))
            module_file = os.path.join(program, "sdl2.zip")
            with zipfile.ZipFile(module_file, 'r') as zip_ref:
                zip_ref.extractall("/")
            print("Successfully installed sdl2")
            return True
        except Exception as e:
            print(f"Failed to install sdl2: {e}")
            return False

def main():
    if ensure_sdl2():
        import app
    app.start()
    while True:
        app.update()

if __name__ == "__main__":
    main()
```

- Always includes `hw_info`, `system_lang`, `hdmi_info` globals
- `board_mapping` has exactly 9 entries + fallback to 5 (RG35xxH)
- `system_list` has exactly 10 entries
- `ensure_sdl2` is always present
- Tiny_scraper variant passes `sys.argv[1]` to `app.start(path)`

## File Classification

| File | Type | What to change |
|------|------|---------------|
| `main.py` | **Identical** (1:1 copy) | Nothing — same boilerplate in every app |
| `input.py` | **Identical** (1:1 copy) | Nothing — same in every app |
| `language.py` | **Identical** (1:1 copy) | Nothing — same in every app |
| `sdl2.zip` | **Identical** (1:1 copy) | Nothing — same pre-packaged PySDL2 in every app |
| `graphic.py` | **Similar** (minor tweaks) | Adjust `screen_resolutions` dict if different `max_elem` needed (e.g. 7 instead of 11). Add app-specific methods like `display_image()` for image viewers or clock font param for clock |
| `anbernic.py` | **Similar** (minor tweaks) | Change `__sd1_rom_storage_path` / `__sd2_rom_storage_path` to point to the app's data directory. Add/customize methods for app-specific storage needs (e.g. `get_bezels_cfg_path()` in bezels_m) |
| `app.py` | **App-specific** | Contains all app logic, state machine, and input handling. Written from scratch per app |
| `lang/*.json` | **App-specific** | Translation strings specific to the app |
| `font/font.ttf` | **Optional** | Only if app needs a custom font (e.g. clock app) |
| `components/` | **Optional** | Additional Python modules (e.g. `bezels.py`, `config.py`, `weather.py`, `set.py`, `systems.py`) |
| `sound/` | **Optional** | `.wav` files for sound effects (e.g. timer alarm) |

## `app.py` — Application Logic Pattern

```python
from main import hw_info, system_lang
from graphic import screen_resolutions, UserInterface
from language import Translator
import input
import sys
import time
import threading
import os
from anbernic import Anbernic

ver = "v1.2"
translator = Translator(system_lang)
gr = UserInterface()            # singleton
an = Anbernic()
skip_input_check = True
current_window = "console"      # state machine variable
selected_index = 0

x_size, y_size, max_elem = screen_resolutions.get(hw_info, (640, 480, 11))
button_x = x_size - 120
button_y = y_size - 30
ratio = y_size / x_size

script_dir = os.path.dirname(os.path.abspath(__file__))

def start():
    print("[INFO]Starting App...")
    gr.draw_log(f"{translator.translate('welcome')}", fill=gr.colorBlue, outline=gr.colorBlueD1)
    gr.draw_paint()
    time.sleep(2)
    handle_initial_window_input()

def update():
    global current_window, skip_input_check
    if skip_input_check:
        input.reset_input()
        skip_input_check = False
    else:
        input.check()
    if input.key("SELECT"):     # universal exit
        gr.draw_log(f"{translator.translate('Exiting...')}", fill=gr.colorBlue, outline=gr.colorBlueD1)
        gr.draw_paint()
        time.sleep(1)
        gr.draw_end()
        print("[INFO]Exiting...")
        sys.exit()
    # dispatch to current window handler
    if current_window == "window1":
        handle_window1_input()
    ...

def handle_window_input():
    # 1. Process input (DY, DX, L1, R1, L2, R2, A, B, X, Y, SELECT, START)
    # 2. gr.draw_clear()
    # 3. gr.draw_rectangle_r(...)       # inner panel
    # 4. gr.draw_text(...)               # title
    # 5. Loop + gr.row_list(...)         # list items
    # 6. gr.draw_help(...)               # help text
    # 7. gr.button_circle(...) / gr.button_rectangle(...)  # button hints
    # 8. gr.draw_paint()
```

## `input.py` — Gamepad Input (identical across all apps)

```python
import struct

code = 0
codeName = ""
value = 0

mapping = {
    304: "A",   305: "B",   306: "Y",   307: "X",
    308: "L1",  309: "R1",  314: "L2",  315: "R2",
    17: "DY",   16: "DX",
    310: "SELECT",  311: "START",  312: "MENUF",
    114: "V+",  115: "V-",
}

def check():
    global type, code, codeName, codeDown, value, valueDown
    with open("/dev/input/event1", "rb") as f:
        while True:
            event = f.read(24)
            if event:
                (tv_sec, tv_usec, type, kcode, kvalue) = struct.unpack('llHHI', event)
                if kvalue != 0:
                    if kvalue != 1:
                        kvalue = -1
                    code = kcode
                    codeName = mapping.get(code, str(code))
                    value = kvalue
                    return

def key(keyCodeName, keyValue=99):
    global code, codeName, value
    if codeName == keyCodeName:
        if keyValue != 99:
            return value == keyValue
        return True

def slide_key():
    global codeName
    if codeName:
        return True

def reset_input():
    global codeName, value
    codeName = ""
    value = 0
```

### Input conventions
| Button | Action |
|--------|--------|
| `DY` | Scroll up/down by 1 |
| `DX` | Scroll by 5 |
| `L1` / `R1` | Scroll by one page (max_elem items) |
| `L2` / `R2` | Scroll by 100 items |
| `A` | Select / Confirm / Open |
| `B` | Back / Go up (or Exit in clock app) |
| `X` | Special action (slideshow, reset bezels) |
| `Y` | Secondary action (switch SD, help) |
| `SELECT` | **Exit app (universal)** |
| `START` | Start action (timer, stopwatch) |

Usage: `input.check()` blocks until a button event; `input.key("A")` checks if pressed; `input.key("DY")` + `input.value` gives direction (-1 or 1); `input.slide_key()` is non-blocking check for threaded loops; `input.reset_input()` clears the current event before entering new input mode.

## `graphic.py` — SDL2 Rendering Wrapper

Singleton `UserInterface` class wrapping SDL2 + Pillow. This file is copied into each app directory and is mostly identical across apps, with app-specific additions (e.g., `display_image` in img_browser, clock font in clock).

```python
import ctypes
import os
from main import hw_info, hdmi_info
from typing import Optional

import sdl2
from PIL import Image, ImageDraw, ImageFont

script_dir = os.path.dirname(os.path.abspath(__file__))
local_font_list = os.path.join(script_dir, "font/font.ttf")
sys_font_file = os.path.join("/mnt/vendor/bin/default.ttf")
font_file = local_font_list if os.path.exists(local_font_list) else sys_font_file

color_text = "#ffffff"

screen_resolutions = {
    1: (720, 720, 18),    # RGcubexx, RG35xxSP
    2: (720, 480, 11),    # landscape devices (fallback)
    # fallback: (640, 480, 11)
}

class UserInterface:
    _instance: Optional["UserInterface"] = None
    _initialized: bool = False

    screen_width, screen_height, max_elem = screen_resolutions.get(hw_info, (640, 480, 11))
    colorBlue = "#0072bb"
    colorBlueD1 = "#004f7f"
    colorGray = "#292929"
    colorGrayL1 = "#383838"
    colorGrayD2 = "#141414"
    colorGreen = "#00ff00"
    colorRed = "#cb0202"
    colorYellow = "#ffd700"
    colorGrayL2 = "#00d7ff"

    active_image: Image.Image
    active_draw: ImageDraw.ImageDraw

    def __init__(self):
        if self._initialized:
            return
        self.window = self._create_window()
        self.renderer = self._create_renderer()
        self.draw_start()
        self.opt_stretch = True
        self._initialized = True

    def __new__(cls):
        if not cls._instance:
            cls._instance = super(UserInterface, cls).__new__(cls)
        return cls._instance

    ### WINDOW MANAGEMENT

    def create_image(self):
        return Image.new("RGBA", (self.screen_width, self.screen_height), color="black")

    def draw_start(self):
        sdl2.SDL_SetRenderDrawColor(self.renderer, 0, 0, 0, 255)
        sdl2.SDL_RenderClear(self.renderer)
        self.active_image = self.create_image()
        self.active_draw = ImageDraw.Draw(self.active_image)

    def _create_window(self):
        window = sdl2.SDL_CreateWindow(
            "App".encode("utf-8"),
            sdl2.SDL_WINDOWPOS_UNDEFINED,
            sdl2.SDL_WINDOWPOS_UNDEFINED,
            0, 0,
            sdl2.SDL_WINDOW_FULLSCREEN_DESKTOP | sdl2.SDL_WINDOW_SHOWN,
        )
        if not window:
            print(f"Failed to create window: {sdl2.SDL_GetError()}")
            raise RuntimeError("Failed to create window")
        return window

    def _create_renderer(self):
        renderer = sdl2.SDL_CreateRenderer(
            self.window, -1, sdl2.SDL_RENDERER_ACCELERATED
        )
        if not renderer:
            renderer = sdl2.render.SDL_CreateRenderer(
                self.window, -1, sdl2.render.SDL_RENDERER_SOFTWARE
            )
            if not renderer:
                print(f"Failed to create renderer: {sdl2.SDL_GetError()}")
                raise RuntimeError("Failed to create renderer")
        sdl2.SDL_SetHint(sdl2.SDL_HINT_RENDER_SCALE_QUALITY, b"0")
        return renderer

    def draw_paint(self):
        # Rotate 90 for RG28xx (hw_info==3) when HDMI is not connected
        if hw_info == 3 and hdmi_info != "HDMI=1":
            rotated_image = self.active_image.rotate(90, expand=True)
            rgba_data = rotated_image.tobytes()
            temp_width, temp_height = rotated_image.size
        else:
            rgba_data = self.active_image.tobytes()
            temp_width, temp_height = self.screen_width, self.screen_height

        surface = sdl2.SDL_CreateRGBSurfaceWithFormatFrom(
            rgba_data, temp_width, temp_height, 32, temp_width * 4,
            sdl2.SDL_PIXELFORMAT_RGBA32,
        )
        texture = sdl2.SDL_CreateTextureFromSurface(self.renderer, surface)
        sdl2.SDL_FreeSurface(surface)

        window_width = ctypes.c_int()
        window_height = ctypes.c_int()
        sdl2.SDL_GetWindowSize(
            self.window, ctypes.byref(window_width), ctypes.byref(window_height)
        )
        window_width, window_height = window_width.value, window_height.value

        if not self.opt_stretch:
            scale = min(window_width / temp_width, window_height / temp_height)
            dst_width = int(temp_width * scale)
            dst_height = int(temp_height * scale)
            dst_x = (window_width - dst_width) // 2
            dst_y = (window_height - dst_height) // 2
            dst_rect = sdl2.SDL_Rect(dst_x, dst_y, dst_width, dst_height)
        else:
            dst_rect = sdl2.SDL_Rect(0, 0, window_width, window_height)

        sdl2.SDL_RenderCopy(self.renderer, texture, None, dst_rect)
        sdl2.SDL_RenderPresent(self.renderer)
        sdl2.SDL_DestroyTexture(texture)

    def draw_end(self):
        sdl2.SDL_DestroyRenderer(self.renderer)
        sdl2.SDL_DestroyWindow(self.window)
        sdl2.SDL_Quit()

    ### DRAWING FUNCTIONS

    def draw_clear(self):
        self.active_draw.rectangle(
            [0, 0, self.screen_width, self.screen_height], fill="black"
        )

    def draw_text(self, position, text, font=21, color=color_text, **kwargs):
        self.active_draw.text(
            position, text, font=ImageFont.truetype(font_file, font), fill=color, **kwargs
        )

    def draw_rectangle(self, position, fill=None, outline=None, width=1):
        self.active_draw.rectangle(position, fill=fill, outline=outline, width=width)

    def draw_rectangle_r(self, position, radius, fill=None, outline=None):
        self.active_draw.rounded_rectangle(position, radius, fill=fill, outline=outline)

    def row_list(self, text, pos, width, selected):
        self.draw_rectangle_r(
            [pos[0], pos[1], pos[0] + width, pos[1] + 32],
            5,
            fill=(self.colorBlue if selected else self.colorGrayL1),
        )
        self.draw_text((pos[0] + 5, pos[1] + 5), text)

    def draw_circle(self, position, radius, fill=None, outline=color_text):
        self.active_draw.ellipse(
            [position[0], position[1], position[0] + radius, position[1] + radius],
            fill=fill, outline=outline,
        )

    def draw_log(self, text, fill="Black", outline="black", width=500, font=21):
        x = (self.screen_width - width) / 2
        y = (self.screen_height - 80) / 2
        self.draw_rectangle_r([x, y, x + width, y + 80], 5, fill=fill, outline=outline)
        font_obj = ImageFont.truetype(font_file, font)
        padding = 10
        max_width = width - 2 * padding
        lines = []
        current_line = ""
        for word in text.split():
            test_line = f"{current_line} {word}".strip() if current_line else word
            bbox = font_obj.getbbox(test_line)
            if (bbox[2] - bbox[0]) <= max_width:
                current_line = test_line
            else:
                if current_line:
                    lines.append(current_line)
                    current_line = ""
                bbox = font_obj.getbbox(word)
                if (bbox[2] - bbox[0]) <= max_width:
                    current_line = word
                else:
                    remaining = word
                    while remaining:
                        substring = ""
                        for char in remaining:
                            temp_sub = substring + char
                            if (font_obj.getbbox(temp_sub)[2] - font_obj.getbbox(temp_sub)[0]) <= max_width:
                                substring = temp_sub
                            else:
                                break
                        if substring:
                            lines.append(substring)
                            remaining = remaining[len(substring):]
                        else:
                            break
        if current_line:
            lines.append(current_line)
        ascent, descent = font_obj.getmetrics()
        line_height = int((ascent + descent) * 1.2)
        total_height = len(lines) * line_height
        start_y = y + (80 - total_height) // 2
        for i, line in enumerate(lines):
            self.draw_text((x + width / 2, start_y + i * line_height + ascent - 5), line, font, anchor="mm")

    def draw_help(self, text, fill="Black", outline="black", font=21):
        x = 20
        y = self.screen_height - 180
        rect_width = self.screen_width - 40
        rect_height = 130
        self.draw_rectangle_r([x, y, self.screen_width - 20, y + rect_height], 5, fill=fill, outline=outline)
        font_obj = ImageFont.truetype(font_file, font)
        padding = 10
        max_width = rect_width - 2 * padding
        lines = []
        current_line = ""
        for word in text.split():
            test_line = f"{current_line} {word}".strip() if current_line else word
            bbox = font_obj.getbbox(test_line)
            if (bbox[2] - bbox[0]) <= max_width:
                current_line = test_line
            else:
                if current_line:
                    lines.append(current_line)
                    current_line = ""
                bbox = font_obj.getbbox(word)
                if (bbox[2] - bbox[0]) <= max_width:
                    current_line = word
                else:
                    remaining = word
                    while remaining:
                        substring = ""
                        for char in remaining:
                            temp_sub = substring + char
                            if (font_obj.getbbox(temp_sub)[2] - font_obj.getbbox(temp_sub)[0]) <= max_width:
                                substring = temp_sub
                            else:
                                break
                        if substring:
                            lines.append(substring)
                            remaining = remaining[len(substring):]
                        else:
                            break
        if current_line:
            lines.append(current_line)
        ascent, descent = font_obj.getmetrics()
        line_height = int((ascent + descent) * 1)
        total_height = len(lines) * line_height
        start_y = y + (rect_height - total_height) // 2
        for i, line in enumerate(lines):
            self.draw_text((self.screen_width // 2, start_y + i * line_height + 10), line, font, anchor="mm")

    def get_text_width(self, text, font):
        image = Image.new('RGB', (1, 1))
        draw = ImageDraw.Draw(image)
        font_obj = ImageFont.truetype(font_file, font)
        bbox = draw.textbbox((0, 0), text, font=font_obj)
        return bbox[2] - bbox[0]

    def button_circle(self, pos, button, text, color=colorBlueD1):
        self.draw_circle(pos, 25, fill=color)
        self.draw_text((pos[0] + 12, pos[1] + 12), button, anchor="mm")
        self.draw_text((pos[0] + 30, pos[1] + 12), text, font=19, anchor="lm")

    def button_rectangle(self, pos, button, text):
        self.draw_rectangle_r(
            (pos[0], pos[1], pos[0] + 60, pos[1] + 25), 5, fill=self.colorGrayL1
        )
        self.draw_text((pos[0] + 30, pos[1] + 12), button, anchor="mm")
        self.draw_text((pos[0] + 65, pos[1] + 12), text, font=19, anchor="lm")

    def display_image(self, image_path,
                    target_width=None, target_height=None,
                    zoom=None, rotation=0):
        if hw_info == 3 and '/anbernic/bootlogo/' in image_path and hdmi_info != "HDMI=1":
            target_width = 480
            target_height = 640
        img = Image.open(image_path)
        if rotation != 0:
            img = img.rotate(rotation, expand=True)
        if zoom is None:
            x_zoom = self.screen_width / img.width
            y_zoom = self.screen_height / img.height
            zoom = min(x_zoom, y_zoom)
        if target_width is None:
            target_width = int(img.width * zoom)
        if target_height is None:
            target_height = int(img.height * zoom)
        img = img.resize((target_width, target_height), Image.LANCZOS)
        paste_x = (self.screen_width - target_width) // 2
        paste_y = (self.screen_height - target_height) // 2
        if hw_info == 3 and '/anbernic/bootlogo/' in image_path and hdmi_info != "HDMI=1":
            img = img.rotate(-90, expand=True)
        self.active_image.paste(img, (paste_x, paste_y))
        self.draw_paint()

    def preview_image(self, image_path, target_x=0, target_y=0, target_width=None, target_height=None):
        if target_width is None:
            target_width = self.screen_width - target_x
        if target_height is None:
            target_height = self.screen_height - target_y
        img = Image.open(image_path)
        img.thumbnail((target_width, target_height))
        paste_x = target_x + (target_width - img.width) // 2
        paste_y = target_y + (target_height - img.height) // 2
        if hw_info == 3 and '/anbernic/bootlogo/' in image_path and hdmi_info != "HDMI=1":
            img = img.rotate(-90, expand=True)
        self.active_image.paste(img, (paste_x, paste_y))
        self.draw_paint()
```

### Font Selection
1. `<app_dir>/font/font.ttf` (app-local)
2. `/mnt/vendor/bin/default.ttf` (system fallback)
The clock app uses a separate `clock_font_file` for time display.

### App-specific additions
- `img_browser/graphic.py`: adds `display_image(image_path, target_width, target_height, zoom, rotation)` for fullscreen image viewing and `preview_image(image_path, target_x, target_y, target_width, target_height)` for thumbnail preview
- `clock/graphic.py`: `draw_text()` accepts an extra `clock` parameter to select the clock-specific font file; `get_text_width()` uses `clock_font_file`
- `mod_set/graphic.py` and `mod_tools/graphic.py`: different `max_elem` values (7 or 14) in `screen_resolutions`
- `button_circle()` in some copies has an optional `color` parameter

## `anbernic.py` — SD Card Storage

```python
import os
from pathlib import Path

class Anbernic:

    def __init__(self):
        self.__sd1_rom_storage_path = "/mnt/mmc"
        self.__sd2_rom_storage_path = "/mnt/sdcard"
        self.__current_sd = 1

    def get_sd1_storage_path(self):
        return self.__sd1_rom_storage_path

    def get_sd2_storage_path(self):
        return self.__sd2_rom_storage_path

    def set_sd_storage(self, sd):
        if sd == 1 or sd == 2:
            self.__current_sd = sd

    def get_sd_storage(self):
        return self.__current_sd

    def switch_sd_storage(self):
        if self.__current_sd == 1:
            self.__current_sd = 2
        else:
            self.__current_sd = 1

    def get_sd_storage_path(self):
        if self.__current_sd == 1 or not any(Path("/mnt/sdcard").iterdir()):
            self.__current_sd = 1
            return self.get_sd1_storage_path()
        else:
            return self.get_sd2_storage_path()

    @staticmethod
    def get_current_path_files(path):
        try:
            entries = os.listdir(path)
            dirs = []
            images = []
            for entry in entries:
                full_path = os.path.join(path, entry)
                if os.path.isdir(full_path):
                    dirs.append(("[+] " + entry, full_path, "dir", entry.lower()))
                elif entry.lower().endswith(('.png', '.jpg', '.jpeg', '.bmp')):
                    images.append((entry, full_path, "image", entry.lower()))
            dirs.sort(key=lambda x: x[3])
            images.sort(key=lambda x: x[3])
            return dirs + images
        except Exception as e:
            print(f"Error reading path {path}: {str(e)}")
            return []
```

Customize `__sd1_rom_storage_path` / `__sd2_rom_storage_path` for the app's data location (e.g., `/mnt/mmc/anbernic/bezels`, `/mnt/mmc/anbernic/backup`).

## `language.py` — i18n (identical across all apps)

```python
import json
import os

class Translator:
    def __init__(self, lang_code='en_US'):
        self.lang_data = {}
        self.lang_code = lang_code
        self.load_language(lang_code)

    def load_language(self, lang_code):
        lang_file = f'{os.path.dirname(os.path.abspath(__file__))}/lang/{lang_code}.json'
        if not os.path.exists(lang_file):
            lang_file = f'{os.path.dirname(os.path.abspath(__file__))}/lang/en_US.json'
        try:
            with open(lang_file, 'r', encoding='utf-8') as f:
                self.lang_data = json.load(f)
        except FileNotFoundError:
            raise Exception(f"Language {lang_file} file {lang_code}.json not found!")

    def translate(self, key, **kwargs):
        message = self.lang_data.get(key, key)
        return message.format(**kwargs)

translator = Translator()
```

### Translation File Format

Create JSON files in `<app_dir>/lang/` for each supported locale. The key is a programmatic identifier (typically English), and the value is the translated string:

```json
{
    "Back": "Back",
    "Exit": "Exit",
    "Open": "Open",
    "Select": "Select",
    "Slideshow": "Slideshow",
    "Path": "Path",
    "No valid file found!": "No valid file found!",
    "Exiting...": "Exiting...",
    "Image Browser": "Image Browser",
    "Switch": "Switch",
    "welcome": "Welcome to Image Browser"
}
```

German example (`de_DE.json`):
```json
{
    "Back": "Zurück",
    "Exit": "Beenden",
    "Open": "Öffnen",
    "Select": "Auswählen",
    "Slideshow": "Diashow",
    "Path": "Pfad",
    "No valid file found!": "Keine gültige Datei gefunden!",
    "Exiting...": "Beende...",
    "Image Browser": "Bildbetrachter",
    "Switch": "Wechseln",
    "welcome": "Willkommen beim Bildbetrachter"
}
```

### How to Use Translations in Code

Instantiate the translator with the detected system language:
```python
translator = Translator(system_lang)
```

Wrap every user-facing string with `translator.translate()`:
```python
gr.draw_log(f"{translator.translate('welcome')}")
gr.draw_text((50, 20), f"{translator.translate('Path')}: {current_path}")
gr.button_circle((20, button_y), "A", f"{translator.translate('Open')}")
```

Use translated strings as dynamic state machine values:
```python
file_list = ["CLOCK", "TIMER", "STOPWATCH"]
# In UI rendering:
gr.row_list(translator.translate(entry), ...)
```

For variable substitution in translations, use `format()` placeholders:
```json
{"file_count": "Showing {count} files"}
```
```python
gr.draw_text((50, 20), translator.translate("file_count", count=42))
```

If a key is missing from the loaded JSON, `translate()` returns the key string itself as fallback — so the app still works even without translations.

### Creating a New Locale

1. Copy `en_US.json` to `<code>.json` (e.g. `fr_FR.json`)
2. Translate all values (keep keys unchanged)
3. The system will load it automatically when `language.ini` reports the matching index

## Threading Pattern

For real-time display with non-blocking input (clock, slideshow, timer, stopwatch):

```python
input.reset_input()
thread = threading.Thread(target=input.check)
thread.start()

while True:
    # update display...
    gr.draw_paint()

    if input.slide_key():    # non-blocking check
        # exit loop
        return
```

## Sound Playback

Use `aplay` via `subprocess` to play `.wav` files. Reference the sound file via a path relative to the app directory:

```python
import subprocess
import os

script_dir = os.path.dirname(os.path.abspath(__file__))
sound_file = os.path.join(script_dir, 'sound', 'sound.wav')

subprocess.run(["aplay", sound_file], check=True)
```

For vibration (clock timer uses the motor hardware):

```python
subprocess.run("echo 1 > /sys/class/power_supply/axp2202-battery/moto && sleep 0.3 && echo 0 > /sys/class/power_supply/axp2202-battery/moto && sleep 0.1", shell=True, check=True)
```

Both wrapped in `try/except subprocess.CalledProcessError` for robustness.

## Target Hardware (Anbernic RG40xxV)

The device running these apps is an **Anbernic RG40xxV** with the following specification:

| Component | Detail |
|-----------|--------|
| **SoC** | Allwinner H616 (sun50iw9) |
| **CPU** | 4× Cortex-A53 @ 1.5 GHz, ARMv8-A, crypto extensions (AES, SHA1/2, PMULL) |
| **GPU** | Mali-G31 MP2 (Midgard, arch 7.0.9 r0p0, 650 MHz) via `/dev/mali0`, `mali_kbase` kernel module |
| **RAM** | ~973 MB |
| **Storage** | 58.2 GB eMMC (partitioned: 7 GB root, 4 GB vendor, 2.5 GB data, 44 GB FAT `/mnt/mmc`) |
| **OS** | Ubuntu 22.04 LTS (custom Buildroot-style, kernel 4.9.170 PREEMPT) |
| **Display** | Direct framebuffer — no X11/Wayland. Resolution via `/sys/class/graphics/fb0`: 1280×1024 virtual, modes: 1280×1024p-59 / 640×480p-59 |
| **Input** | `/dev/input/event1` for gamepad (gpio_keys-joystick), `/dev/input/event0` for keys, `/dev/input/js0` legacy joystick |
| **Audio** | Internal audiocodec (card 0) + HDMI audio (card 2), ALSA `aplay` for playback |
| **Wireless** | Realtek 8821CS WiFi (`8821cs` module), interfaces `wlan0` + `wlan1` |
| **Power** | AXP2201 PMIC, battery motor at `/sys/class/power_supply/axp2202-battery/moto` |

### SDL2 Configuration

| Parameter | Value |
|-----------|-------|
| **Version** | SDL 2.0.12 (compiled static lib + shared) |
| **Video driver** | Default (fbcon based — no `SDL_VIDEO_DRIVER` set) |
| **Library path** | `/usr/lib/libSDL2.so`, `/usr/lib/libSDL2-2.0.so.0.12.0` |
| **Python bindings** | PySDL2 at `/usr/lib/python3/dist-packages/sdl2/` (full module with ext, image, ttf, mixer) |
| **Pillow** | 9.0.1 |
| **Python** | 3.10.12 |

### App Runtime Behavior

- SDL opens a **fullscreen desktop window** on the framebuffer
- Rendering goes through `Pillow` → RGBA bytes → SDL2 texture → hardware-scaled to window
- Resolution detection reads `board.ini` (`RG40xxV` = hw_info 8), falls back to 640×480 on unknown boards
- The display is **not rotated** except for RG28xx (hw_info 3) which rotates 90°
- No compositor or display manager runs — apps write directly to the framebuffer via SDL

## Creating a New App

1. Create `<app>.sh` using the 13-line template above
2. Create `<app>/` directory
3. Copy boilerplate files from an existing app:
   - `main.py` (hardware detection + entry point)
   - `graphic.py` (UI rendering — adjust `screen_resolutions` and `max_elem` if needed)
   - `input.py` (identical across all apps)
   - `language.py` (identical across all apps)
   - `sdl2.zip` (pre-packaged PySDL2)
4. Create `lang/` with at least `en_US.json`
5. Create `app.py` with your app logic (state machine pattern above)
6. Optionally add `components/` for additional Python modules
7. Optionally add `font/font.ttf` for custom fonts
8. Optionally add `sound/` with `.wav` files
9. Add `Imgs/<app>.png` icon (optional)
10. Write translation strings to `lang/<code>.json` files
