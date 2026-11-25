# Project Brief: FTC Shoot-on-the-Move (SOTM) TOF Video Tagging Tool

**Deadline:** 1 week

**Language / Recommended Stack:** Python 3 (`opencv-python`, `Pillow`, `tkinter` — built into Python)

## 1. Context & Goal

To make our robot **Shoot on the Move (SOTM)** accurately, we need to know the exact **Time of Flight (TOF)** of a game piece from the moment it leaves the shooter to the moment it hits the goal, across different distances and robot/shooter states.

We will record calibration runs on a **Sony a6400 at 120 fps** alongside robot telemetry logs. Your job is to build a **desktop GUI tool** that lets us step through the video frame-by-frame, mark the launch and landing frames for each shot from the telemetry log, review saved shots in a live list panel on the right, and export the final dataset as JSON.

## 2. Inputs

### A. Calibration Video (`video.mp4`)

* Constant **120 fps** video (default `fps = 120.0`, configurable via an **FPS input box** in the UI).

* Shows both the robot's shooter exit and the goal in the same frame.

### B. Robot Telemetry Log (`telemetry.json`)

A JSON list of event dictionaries logged during the run. **Only entries where `"action" == "shoot"` are shots**—ignore everything else (`"drive"`, `"intake"`, etc.).

Each `"shoot"` entry includes:

* `distance` (float): Distance to the goal at shot release.

* `velocity` (float): Robot speed/velocity scalar at shot release.

* `shooter_velocity` (float): Shooter wheel/launch velocity at shot release.

* `x_velocity` (float): Robot velocity along the X axis at shot release.

* `y_velocity` (float): Robot velocity along the Y axis at shot release.

* `millis` (int/float): Milliseconds since OpMode start when the shot occurred.

* `battery_voltage` (float): Battery voltage at shot release.

**Example `telemetry.json`:**

```json
[
  {
    "action": "drive",
    "distance": 1.45,
    "velocity": 0.82,
    "shooter_velocity": 0.0,
    "x_velocity": 0.65,
    "y_velocity": 0.50,
    "millis": 1020,
    "battery_voltage": 13.1
  },
  {
    "action": "shoot",
    "distance": 1.50,
    "velocity": 0.01,
    "shooter_velocity": 1420.0,
    "x_velocity": 0.01,
    "y_velocity": 0.00,
    "millis": 2340,
    "battery_voltage": 12.9
  },
  {
    "action": "shoot",
    "distance": 2.15,
    "velocity": 0.45,
    "shooter_velocity": 1610.0,
    "x_velocity": 0.32,
    "y_velocity": -0.31,
    "millis": 4890,
    "battery_voltage": 12.8
  },
  {
    "action": "shoot",
    "distance": 2.80,
    "velocity": 0.02,
    "shooter_velocity": 1790.0,
    "x_velocity": 0.02,
    "y_velocity": 0.00,
    "millis": 7610,
    "battery_voltage": 12.7
  }
]
```

## 3. Required Output Format

When exporting, the tool must save a `.json` file containing **only** a top-level list of objects with `"distance"` and `"tof"` (in seconds):

```json
[
  {"distance": 1.5, "tof": 0.325},
  {"distance": 2.15, "tof": 0.416667},
  {"distance": 2.8, "tof": 0.508333}
]
```

Where for each saved shot:

$$
\text{tof} = \frac{\text{land\_frame} - \text{launch\_frame}}{\text{fps}}
$$

* Skipped, deleted, or incomplete shots must **not** be included in the exported JSON.

* Both `distance` and `tof` must be JSON numbers (`float`), not strings.

## 4. Layout & Core Features Checklist

### Window Layout

* **Left / Main Panel:** Video frame canvas, timeline scrubber, playback/frame-stepping buttons, FPS input field (default `120`), and active shot telemetry + frame-marking controls.

* **Right Panel (Saved Shots List):** A scrollable list (`tkinter.Listbox` or `ttk.Treeview`) showing all saved/tagged shots in real time (e.g., `#1 | d=1.50m | v_s=1420 | vx=0.01, vy=0.00 | tof=0.325s`), plus a count header (`Saved: X / N shots`).

### Video Playback & Frame Stepping

* Open a `.mp4` video and `.json` telemetry file (via file picker dialogs or command-line arguments).

* **Configurable FPS Input:** Include an editable FPS field in the UI **defaulting to `120`** so timestamps and `tof` calculations update dynamically if a different frame rate is entered.

* Display the current frame clearly, along with **Current Frame Index** and **Timestamp (`frame / fps`)**.

* Frame-by-frame controls (both **buttons** and **keyboard shortcuts**):

  * Previous / Next **1 frame** (e.g., `Left` / `Right` arrows or `A` / `D`)

  * Previous / Next **10 frames** (e.g., `Shift+Left` / `Shift+Right` or `Q` / `E`)

  * Timeline slider/scrubber to jump quickly between shots

### Telemetry Pairing & Right-Panel Shot List

* Filter `telemetry.json` to `"action" == "shoot"` and sort by `millis`.

* Display the active shot being labeled (`Shot 1 of 12`), showing:

  * `distance`

  * `shooter_velocity`

  * `velocity`, `x_velocity`, and `y_velocity`

  * `millis` and `battery_voltage`

* Buttons + hotkeys to:

  * **Mark Launch Frame** (first frame where the game piece leaves the shooter)

  * **Mark Land Frame** (first frame where the game piece hits/enters the goal)

  * **Save / Commit Shot** (or auto-save once both frames are marked) -> immediately adds/updates the entry in the **right-hand Saved Shots list** and advances to the next shot.

  * **Next / Previous Shot** & **Skip Shot**

* **Interactive Right-Hand List:**

  * Clicking an item in the right panel selects that shot (and optionally jumps the video to its `launch_frame`) so you can inspect or re-mark it.

  * Include a **Delete / Remove Selected** button so bad/outlier shots can be removed directly from the saved list before exporting.

### Session Auto-Save & Export

* **Session Auto-Save:** Automatically persist in-progress marks and saved shots to a local draft file (e.g., `session_autosave.json`) whenever a shot is saved, updated, or deleted, and restore progress automatically if the app is reopened with the same files.

* **Export JSON:** An **Export JSON** button (below the saved shots list) that writes the clean `[{"distance": x, "tof": y}, ...]` list to disk.

## 5. Engineering Tips & Gotchas (Read Before Coding!)

1. **Watch out for backward seeking in OpenCV:**

   Calling `cap.set(cv2.CAP_PROP_POS_FRAMES, idx)` on compressed H.264/MP4 files (like Sony XAVC S) can be sluggish or off by 1–2 frames when stepping backward (`-1 frame`).

   * *Recommended fix:* Calibration clips are short (usually 15–60 seconds). Keep a **sliding cache of \~300 frames** around the current frame (or preload resized frames if the clip is short) so `-1` / `+1` frame stepping is instant and 100% frame-exact.

2. **Keep the GUI stack simple:**

   Use **`tkinter` + `Pillow` (`ImageTk`) + `opencv-python`**. A `ttk.Treeview` or `tk.Listbox` with a `tk.Scrollbar` packed on the right (`side="right", fill="y"`) gives you a clean saved-shots panel in \~20 lines of code.

3. **Resize the display frame, not the frame index:**

   A 1080p frame might take up too much width next to the right-hand list. Resize the rendered image (e.g., `960x540` or `1120x630`), while keeping all frame indices tied to the original video stream.

4. **Use `x_velocity`, `y_velocity`, and `shooter_velocity` as context:**

   Even though the exported JSON only contains `distance` and `tof`, displaying `shooter_velocity`, `x_velocity`, and `y_velocity` during tagging helps spot whether a shot happened while moving (`x_velocity` / `y_velocity` non-zero) or with an unspun/drooping shooter (`shooter_velocity` off target) so you can skip or flag bad calibration points.

5. **Sanity-check shot order:**

   The time gap between Shot 1 and Shot 2 in the video (`(launch_2 - launch_1) / fps`) should roughly match `(millis_2 - millis_1) / 1000.0` in `telemetry.json`. Reject or warn if `land_frame <= launch_frame`.

## 6. Suggested 1-Week Plan

* **Days 1–2:** Parse `telemetry.json` (`action == "shoot"` including `shooter_velocity`, `x_velocity`, `y_velocity`), load video with `cv2.VideoCapture`, display frames in Tkinter with `+1` / `-1` frame stepping and the FPS input (default `120`).

* **Days 3–4:** Add `Mark Launch`, `Mark Land`, shot navigation (`Prev`/`Next`/`Skip`), telemetry readout panel, and the **right-hand Saved Shots list** (click-to-jump + delete button).

* **Day 5:** Implement **session auto-save** and **JSON export** in the exact `[{"distance": x, "tof": y}]` schema, then test on a sample clip.

* **Days 6–7:** Fix edge cases (re-marking already-saved shots, invalid FPS input, end-of-video bounds, skipped/deleted shots) and hand over the working repo.