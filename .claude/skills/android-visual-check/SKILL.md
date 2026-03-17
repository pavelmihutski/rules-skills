---
name: android-visual-check
description: Capture a screenshot from a running Android TV emulator via ADB, compare it visually against a reference design screenshot (Figma export or similar), report differences, and loop until the implementation matches the design.
metadata:
  author: Pavel Mihutski
  version: "1.3"
  tags: android-tv, emulator, screenshot, visual-testing, design-qa
---

# Android TV Visual Check Skill

## Purpose

Automate the visual feedback loop between a **reference design** (Figma export, mockup, etc.) and the **live Android TV emulator** output:

1. Take a screenshot from the running Android TV emulator via ADB
2. Read both images and compare them visually
3. Report what matches and what differs
4. If differences exist — describe what needs to change, then repeat after the fix

## When to Use This Skill

Invoke when the user:
- Asks to check if the UI matches a design
- Provides a reference image path and wants it compared with the emulator
- Wants a visual QA loop: implement → screenshot → compare → fix → repeat
- Asks to navigate to a screen and take a screenshot (reference is optional)

---

## Android TV — D-pad Only

This skill is **exclusively for Android TV**. Android TV uses D-pad remote control navigation. **Touch/tap events do not work** — never use `input tap` by coordinates. All navigation must use D-pad keyevents.

Key D-pad keycodes:
| Action | Keyevent |
|--------|----------|
| DPAD_UP | `19` |
| DPAD_DOWN | `20` |
| DPAD_LEFT | `21` |
| DPAD_RIGHT | `22` |
| DPAD_CENTER / Enter | `23` |
| Back | `4` |
| Home | `3` |

### focused vs selected (Android TV) — UiAutomator is unreliable

UiAutomator `focused` and `selected` attributes reflect Android **accessibility** state, NOT the internal `react-tv-space-navigation` focus. **Do not rely on them to determine which tab or item has D-pad focus.** The only reliable way to verify navigation state is to **take a screenshot and look at it visually** — the focused element shows a purple/colored highlight ring.

### How the nav bar focus system works in this app

This app uses `react-tv-space-navigation` with a custom `MenuProvider`. The nav bar is a separate `NavigationRoot` that only becomes active when `isFocused` (menu focus state) is `true`. Key rules:

- **`input tap` by coordinates does NOT work** on this TV app — never use it for navigation
- **Pressing DPAD_UP once does NOT activate the nav bar** — it just moves focus within page content
- **The nav bar activates only when upward movement is exhausted in the page** — keep pressing DPAD_UP until focus can't go higher, then one more UP triggers `focusMenu(true)` and activates the nav bar
- **When the nav bar activates**, focus automatically lands on the currently active tab (not the logo)
- **Pressing DPAD_RIGHT then moves between tabs** (one press per tab), and **DPAD_CENTER selects** the focused tab and navigates to it
- **Pressing DPAD_RIGHT from the logo does NOT work** — the logo is outside the SpatialNavigationView; pressing right from it has no effect

### Correct nav bar navigation procedure

1. Press `DPAD_UP` **5 times** (with 300ms delays) from within page content to exhaust all upward movement and activate the menu
2. Take a screenshot to confirm the nav bar is highlighted and which tab has focus
3. Press `DPAD_RIGHT` or `DPAD_LEFT` to move to the target tab — **one press per tab position**
4. Take a screenshot to verify the correct tab is focused (purple highlight)
5. Press `DPAD_CENTER` to navigate to the focused tab
6. Wait 2–3 seconds and take a screenshot to confirm the page loaded

---

## Inputs

The user must provide:

| Input | Description | Example |
|---|---|---|
| `reference` | (Optional) Path to the design/mockup screenshot | `~/designs/home-screen.png` |
| `adb_id` | (Optional) ADB device ID if multiple devices connected | `emulator-5554` |
| `steps` | (Optional) Navigation steps to reach the target screen before capture | see below |

If `adb_id` is not provided, use the first device from `adb devices`.

If `reference` is not provided — just navigate and capture, skip the comparison step.

### Navigation Steps Format

The user provides steps as a plain list of actions. Each step is one of:

```
dpad up                  # press DPAD_UP (keyevent 19)
dpad down                # press DPAD_DOWN (keyevent 20)
dpad left                # press DPAD_LEFT (keyevent 21)
dpad right               # press DPAD_RIGHT (keyevent 22)
dpad center              # press DPAD_CENTER / Enter (keyevent 23)
nav_tab "Tab Name"       # navigate to a named tab in the nav bar using D-pad (see below)
swipe up                 # swipe in a direction (up/down/left/right)
swipe 540 1500 540 500   # swipe from (x1 y1) to (x2 y2)
wait 1000                # wait N milliseconds
back                     # press the back button (keyevent 4)
home                     # press the home button (keyevent 3)
input "search text"      # type text into the focused field
```

#### `nav_tab "Tab Name"` — smart tab navigation (Android TV)

This step navigates to a named tab in the nav bar reliably:

1. Press `DPAD_UP` **5 times** (300ms apart) from page content to exhaust upward movement and activate the menu
2. Take a screenshot and read it — identify which tab currently has the purple focus highlight
3. Use a UI dump to collect tab names and their order (left to right) in the nav bar
4. Calculate how many RIGHT (or LEFT) presses are needed from the currently focused tab to reach the target
5. Press `DPAD_RIGHT` or `DPAD_LEFT` that many times (300ms apart)
6. Take a screenshot to visually confirm the target tab is now focused (purple highlight)
7. If not on the correct tab — adjust with more LEFT/RIGHT presses and re-verify
8. Press `DPAD_CENTER` to navigate to the tab
9. Wait 2500ms for content to load, then take a final screenshot to confirm

```python
# Parse nav items from UI dump to get tab order
import xml.etree.ElementTree as ET
tree = ET.parse('/tmp/ui.xml')
nav_items = []
for elem in tree.getroot().iter('node'):
    b = elem.get('bounds', '')
    coords = b.replace('[','').replace(']',',').split(',')
    try:
        x1,y1,x2,y2 = int(coords[0]),int(coords[1]),int(coords[2]),int(coords[3])
        if 30 <= y1 and y2 <= 80 and elem.get('clickable') == 'true':
            nav_items.append({
                'desc': elem.get('content-desc',''),
                'bounds': b
            })
    except: pass
# NOTE: do NOT use focused/selected from this dump — they are unreliable.
# Use the screenshot to determine current focus position visually.
```

**Important**: After pressing DPAD_CENTER on a tab that leads to a new screen, the menu deactivates. To navigate the nav bar again, repeat the 5x DPAD_UP activation process.

Example user input:
```
steps:
  - nav_tab "Our Livestreams"
  - dpad down
  - dpad center
```

**If no steps are provided** — screenshot whatever is currently on screen.

**If steps are provided but a step fails** (element not found, ADB error) — stop, report which step failed and what was on screen at that point, wait for user to fix the steps or type `RECHECK`.

---

## Workflow

### Step 1 — Verify ADB Connection

```bash
adb devices
```

- Check at least one device/emulator is listed and in `device` state
- If none found: stop and tell the user to start the emulator or connect a device
- If multiple devices and no `adb_id` provided: list them and ask the user to specify

### Step 2 — Navigate to Target Screen (if `steps` provided)

Execute each step in order using ADB shell commands:

**`dpad up/down/left/right/center`**:
```bash
adb -s <device-id> shell input keyevent 19   # up
adb -s <device-id> shell input keyevent 20   # down
adb -s <device-id> shell input keyevent 21   # left
adb -s <device-id> shell input keyevent 22   # right
adb -s <device-id> shell input keyevent 23   # center/enter
```

**`nav_tab "Tab Name"`** — see smart tab navigation above.

**`swipe up` / `swipe down` / `swipe left` / `swipe right`** — use center of screen with a fixed offset:
```bash
# swipe up
adb -s <device-id> shell input swipe 540 1200 540 400 300
# swipe down
adb -s <device-id> shell input swipe 540 400 540 1200 300
# swipe left
adb -s <device-id> shell input swipe 900 800 200 800 300
# swipe right
adb -s <device-id> shell input swipe 200 800 900 800 300
```

**`swipe <x1> <y1> <x2> <y2>`**:
```bash
adb -s <device-id> shell input swipe <x1> <y1> <x2> <y2> 300
```

**`wait <ms>`**:
```bash
sleep <ms/1000>
```

**`back`**:
```bash
adb -s <device-id> shell input keyevent 4
```

**`home`**:
```bash
adb -s <device-id> shell input keyevent 3
```

**`input "text"`**:
```bash
adb -s <device-id> shell input text "text"
```

After **each step**, wait 500ms by default to allow the UI to settle. After the last step, wait **2000ms** before capturing (Android TV screens can have skeleton loading states that take 3–5 seconds to resolve — if the screenshot shows a black/skeleton screen, wait longer and retry).

If a step fails — capture an immediate screenshot to `/tmp/emulator-step-fail.png`, read it, report which step failed and what was visible, then stop and wait for `RECHECK`.

### Step 3 — Capture Screenshot from Emulator

```bash
adb -s <device-id> exec-out screencap -p > /tmp/emulator-current.png
```

Save to `/tmp/emulator-current.png`.

Verify the file exists and is non-empty. If capture fails, report the ADB error and stop.

### Step 4 — Read Both Images

Use the Read tool to load both images:
- **Reference**: path provided by the user
- **Current**: `/tmp/emulator-current.png`

### Step 5 — Visual Comparison

Analyze both images and produce a structured diff report:

```
## Visual Comparison Report

### Overall Match
[PASS ✅ | FAIL ❌] — [brief one-line verdict]

### Differences Found
| Area | Reference | Current | Severity |
|------|-----------|---------|----------|
| [region] | [what design shows] | [what emulator shows] | High/Med/Low |

### What Matches
- [list of elements that look correct]

### What Needs Fixing
1. [Specific actionable item — describe element, expected state, actual state]
2. ...
```

Severity guide:
- **High** — layout broken, element missing, wrong component
- **Medium** — wrong color, wrong spacing, wrong font size
- **Low** — minor pixel difference, slight alignment off

### Step 6 — Decision

**If PASS**: Report success and stop the loop.

**If FAIL**:
- Output the diff report
- List specific changes needed (actionable, file-level if possible)
- **MANDATORY STOP** — Output:
  ```
  ❌ VISUAL MISMATCH: Differences found (see report above).
  Fix the issues listed, then type RECHECK to run the comparison again.
  ```
- Wait for user to type `RECHECK` before re-running from Step 2

---

## Loop Behavior

Each `RECHECK` restarts from **Step 2** (navigation) and repeats through capture and comparison. The loop continues until:
- User receives a PASS report
- User explicitly stops

Valid commands:
- `RECHECK` — take new screenshot and compare again
- `STOP` — exit the loop

---

## Rules

1. **Never modify code** — this skill only observes and reports; code changes are the user's or another task's responsibility
2. **Always show both images in the report context** — read both before comparing
3. **Be specific in diff output** — "button label is 'Submit' but design shows 'Continue'" not "button text is wrong"
4. **Report coordinates/regions** — describe where on screen the difference is (top-left, center, bottom nav, etc.)
5. **Capture fresh screenshot each RECHECK** — never reuse the previous capture
6. **Save all screenshots to the project root** — copy `/tmp/emulator-current.png` to the project root with a descriptive name (e.g. `our-livestreams-screenshot.png`) after every successful capture
7. **Screenshots are the ground truth for navigation state** — never rely solely on UiAutomator dumps to determine D-pad focus position; always take a screenshot and visually verify
8. **`input tap` is forbidden for TV navigation** — never use coordinate-based tapping to navigate this app; use D-pad keyevents only

---

## Error Handling

| Error | Action |
|---|---|
| No ADB devices found | Stop, tell user to start emulator or enable USB debugging |
| `adb screencap` fails | Report error output, suggest checking ADB connection |
| Reference image not found | Stop, ask user to provide correct path |
| Image unreadable | Report and stop |

---

## Example Invocations

**Just take a screenshot of current screen:**
> `/android-visual-check` — just save a screenshot to the root folder

**Navigate to a tab and screenshot (no reference):**
> `/android-visual-check` — go to My Livestreams and screenshot

Steps used internally:
```
- nav_tab "My Livestreams"
```

**Navigate into a detail screen:**
> `/android-visual-check` — go to My Livestreams, open the first item

Steps used internally:
```
- nav_tab "My Livestreams"
- dpad down
- dpad center
```

**Full visual QA with reference design:**
> `/android-visual-check` — reference is `~/designs/my-livestreams.png`
> steps:
>   - nav_tab "My Livestreams"

You:
1. Run `adb devices`, confirm emulator is running
2. Execute navigation steps (if provided), verifying UI state after each step
3. Capture screenshot (wait for loading to finish — retry if skeleton screen)
4. If reference provided: read both images and output comparison report
5. If no reference: just show the screenshot to the user
6. If differences found: stop and wait for `RECHECK`
7. On `RECHECK`: re-run navigation steps + fresh screenshot + compare again
