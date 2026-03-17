---
name: ios-visual-check
description: Capture a screenshot from a running Apple TV (tvOS) simulator via xcrun simctl, compare it visually against a reference design screenshot (Figma export or similar), report differences, and loop until the implementation matches the design.
metadata:
  author: Pavel Mihutski
  version: "1.1"
  tags: apple-tv, tvos, simulator, screenshot, visual-testing, design-qa
---

# iOS TV Visual Check Skill

## Purpose

Automate the visual feedback loop between a **reference design** (Figma export, mockup, etc.) and the **live Apple TV (tvOS) simulator** output:

1. Take a screenshot from the running tvOS simulator via `xcrun simctl`
2. Read both images and compare them visually
3. Report what matches and what differs
4. If differences exist — describe what needs to change, then repeat after the fix

## When to Use This Skill

Invoke when the user:
- Asks to check if the UI matches a design on iOS/tvOS
- Provides a reference image path and wants it compared with the simulator
- Wants a visual QA loop: implement → screenshot → compare → fix → repeat
- Asks to navigate to a screen and take a screenshot (reference is optional)

---

## tvOS Simulator — D-pad via Keyboard

This skill is **exclusively for Apple TV (tvOS) simulator**. The tvOS simulator is controlled via keyboard shortcuts that mirror the Siri Remote D-pad. **Touch/tap events do not work** — all navigation must use keyboard key codes sent via AppleScript/osascript.

Key mappings:
| Action | Key Code | Key |
|--------|----------|-----|
| DPAD_UP | `126` | ↑ Up Arrow |
| DPAD_DOWN | `125` | ↓ Down Arrow |
| DPAD_LEFT | `123` | ← Left Arrow |
| DPAD_RIGHT | `124` | → Right Arrow |
| Select / Enter | `36` | Return |
| Menu / Back | `53` | Escape |

### How to send key events to the tvOS Simulator

Use `osascript` to send key presses to the Simulator app:

```bash
# Bring Simulator to foreground first
osascript -e 'tell application "Simulator" to activate'

# Send a single key press (e.g. DPAD_UP)
osascript -e 'tell application "System Events" to tell process "Simulator" to key code 126'

# Wait between presses (e.g. 300ms)
sleep 0.3
```

### osascript permission error

If osascript fails with "not allowed to send keystrokes (1002)", fix it with:

```bash
tccutil reset Accessibility com.apple.Terminal
```

Then retry — no system restart needed.

### focused vs selected (tvOS Simulator)

React Native / `react-tv-space-navigation` manages its own focus state internally — it does **not** map reliably to accessibility attributes. **Never rely on accessibility inspection to determine focus.** The only reliable way to verify navigation state is to **take a screenshot and look at it visually** — the focused element shows a highlight ring.

### How the nav bar focus system works in this app

This app uses `react-tv-space-navigation` with a custom `MenuProvider`. The nav bar is a separate `NavigationRoot` that only becomes active when `isFocused` (menu focus state) is `true`. Key rules:

- **Pressing DPAD_UP once does NOT activate the nav bar** — it just moves focus within page content
- **The nav bar activates only when upward movement is exhausted in the page** — keep pressing DPAD_UP until focus can't go higher, then one more UP triggers `focusMenu(true)` and activates the nav bar
- **When the nav bar activates**, focus automatically lands on the currently active tab (not the logo)
- **Pressing DPAD_RIGHT then moves between tabs** (one press per tab), and **Select (Return)** navigates to the focused tab
- **Pressing DPAD_DOWN from the nav bar** deactivates the menu and returns focus to page content

### Nav bar active tab indicator vs focused tab

The white pill background on a tab in the nav bar is the **active tab indicator** (always visible for the current screen). It is **NOT** a focus indicator. When the nav bar has focus, the focused tab shows an additional subtle highlight or border. If pressing DPAD_RIGHT after activating the nav bar causes the hero carousel content to change instead of moving between tabs, the nav bar was **not actually activated** — repeat the upward exhaustion process.

### Tab order in this app

The nav bar has these tabs in order (left to right):
1. Home
2. Upcoming Livestreams
3. Recent Livestreams
4. On-Demand Concerts
5. Our Livestreams (My Livestreams)
6. Settings (gear icon)

From "Home" (index 0), pressing RIGHT N times reaches the Nth tab.

### Correct nav bar navigation procedure

1. Press `DPAD_UP` **5–8 times** (with 300ms delays) from within page content to exhaust all upward movement and activate the menu
2. Take a screenshot — confirm the nav bar tab shows a focus highlight (distinct from just the active pill)
3. If D-pad RIGHT still scrolls hero content instead of moving tab focus — press DPAD_UP more times and retry
4. Press `DPAD_RIGHT` or `DPAD_LEFT` to move to the target tab — **one press per tab position**
5. Take a screenshot to verify the correct tab is focused
6. Press `Select` (Return, key code 36) to navigate to the focused tab
7. Wait 2–3 seconds and take a screenshot to confirm the page loaded

### Faster alternative: change `initialRouteName` (for specific tab screens)

When nav bar navigation is unreliable, navigate directly by temporarily changing `initialRouteName` in `src/app/navigation/TabNavigator.tsx` and restarting the app:

```ts
// Change from:
initialRouteName={ScreenNames.HomePageScreen}
// To:
initialRouteName={ScreenNames.SettingsScreen}  // or whatever target
```

Then kill and relaunch:
```bash
xcrun simctl terminate <udid> net.nugs.multiband
sleep 1
xcrun simctl launch <udid> net.nugs.multiband
sleep 10
# Take screenshot
```

**Always revert this change after the visual check is complete.**

---

## App-Specific Details

- **Bundle ID**: `net.nugs.multiband`
- **Find bundle ID** from simulator if unknown:
  ```bash
  ls ~/Library/Developer/CoreSimulator/Devices/<udid>/data/Containers/Bundle/Application/
  # Then check the .app inside that folder:
  /usr/libexec/PlistBuddy -c "Print CFBundleIdentifier" <path>/nugs.app/Info.plist
  ```
- **Launch app directly** (without rebuilding):
  ```bash
  xcrun simctl launch <udid> net.nugs.multiband
  ```

---

## Inputs

The user must provide:

| Input | Description | Example |
|---|---|---|
| `reference` | (Optional) Path to the design/mockup screenshot | `~/designs/home-screen.png` |
| `simulator_id` | (Optional) Simulator UDID if multiple tvOS simulators are booted | `XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX` |
| `steps` | (Optional) Navigation steps to reach the target screen before capture | see below |

If `simulator_id` is not provided, use the first booted tvOS simulator from `xcrun simctl list devices --json`.

If `reference` is not provided — just navigate and capture, skip the comparison step.

### Navigation Steps Format

The user provides steps as a plain list of actions. Each step is one of:

```
dpad up                  # press DPAD_UP (key code 126)
dpad down                # press DPAD_DOWN (key code 125)
dpad left                # press DPAD_LEFT (key code 123)
dpad right               # press DPAD_RIGHT (key code 124)
dpad center              # press Select / Enter (key code 36)
nav_tab "Tab Name"       # navigate to a named tab in the nav bar using D-pad (see below)
wait 1000                # wait N milliseconds
back                     # press Menu / Back (key code 53)
```

#### `nav_tab "Tab Name"` — smart tab navigation (tvOS)

This step navigates to a named tab in the nav bar reliably:

1. Press `DPAD_UP` **5–8 times** (300ms apart) from page content to exhaust upward movement and activate the menu
2. Take a screenshot — identify which tab currently has the focus highlight (not just the active pill)
3. If the nav bar isn't focused (D-pad right scrolls hero instead), press UP more times and re-verify
4. Calculate how many RIGHT (or LEFT) presses are needed from the currently focused tab to reach the target (use the tab order above)
5. Press `DPAD_RIGHT` or `DPAD_LEFT` that many times (300ms apart)
6. Take a screenshot to visually confirm the target tab is now focused
7. If not on the correct tab — adjust with more LEFT/RIGHT presses and re-verify
8. Press `Select` (key code 36) to navigate to the tab
9. Wait 2500ms for content to load, then take a final screenshot to confirm

**Important**: After pressing Select on a tab that leads to a new screen, the menu deactivates. To navigate the nav bar again, repeat the 5–8x DPAD_UP activation process.

Example user input:
```
steps:
  - nav_tab "Our Livestreams"
  - dpad down
  - dpad center
```

**If no steps are provided** — screenshot whatever is currently on screen.

**If steps are provided but a step fails** — stop, report which step failed and what was on screen at that point, wait for user to fix the steps or type `RECHECK`.

---

## Workflow

### Step 1 — Verify Simulator Connection

```bash
xcrun simctl list devices --json | python3 -c "
import json, sys
data = json.load(sys.stdin)
booted = []
for runtime, devices in data.get('devices', {}).items():
    if 'tvOS' in runtime:
        for d in devices:
            if d.get('state') == 'Booted':
                booted.append({'name': d['name'], 'udid': d['udid'], 'runtime': runtime})
print(json.dumps(booted, indent=2))
"
```

- If at least one tvOS simulator is booted — proceed
- If **none booted**: find all available tvOS simulators and boot the first one automatically:

```bash
# Find all tvOS simulators (booted or not)
xcrun simctl list devices --json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for runtime, devices in data.get('devices', {}).items():
    if 'tvOS' in runtime:
        for d in devices:
            print(d['state'], d['udid'], d['name'])
"

# Boot the first one
xcrun simctl boot <udid>
open -a Simulator
sleep 10  # wait for boot
```

- Verify it's now booted, then proceed

### Step 1.5 — Verify App is Running

After confirming the simulator is booted, check if the app is already running by taking a quick screenshot. If the simulator shows the tvOS home screen (not the app), launch the app:

```bash
xcrun simctl launch <udid> net.nugs.multiband
sleep 8
```

If the app shows a **native module error** (`NativeModule.XXX is null`), the native dependencies are not properly installed. Fix:

```bash
# Kill any Metro processes
kill -9 $(lsof -ti:8081) 2>/dev/null

# Run pod install
cd ios && pod install

# Rebuild and reinstall
yarn ios --simulator "Apple TV 4K (3rd generation)"
# Wait ~2 minutes for build to complete
```

If the app shows a **React Native LogBox overlay** (red/pink console error panel), it cannot be dismissed via D-pad on tvOS. Fix:

1. Temporarily add `LogBox.ignoreAllLogs()` to `index.js`:
```js
import { AppRegistry, LogBox } from 'react-native'
LogBox.ignoreAllLogs()
// ... rest of file
```
2. Reload the bundle: `osascript -e 'tell application "System Events" to tell process "Simulator" to keystroke "r"'`
3. Wait 5 seconds for reload
4. **Revert `index.js` after the visual check is complete**

### Step 1.6 — Handle Login Screen (Auth Bypass)

If the app lands on the **login screen** and you need to reach a screen behind auth, temporarily bypass it:

In `src/app/navigation/RootStack.tsx`, change:
```tsx
{hasAuthorizedLaunch ? (
```
to:
```tsx
{true || hasAuthorizedLaunch ? (
```

Reload the bundle with keystroke "r", wait 5–8 seconds. The main tab navigator will render directly.

**Always revert this change after the visual check is complete.**

### Step 2 — Navigate to Target Screen (if `steps` provided)

First, bring Simulator to the foreground:
```bash
osascript -e 'tell application "Simulator" to activate'
sleep 0.5
```

Execute each step in order:

**`dpad up/down/left/right`**:
```bash
osascript -e 'tell application "System Events" to tell process "Simulator" to key code 126'  # up
osascript -e 'tell application "System Events" to tell process "Simulator" to key code 125'  # down
osascript -e 'tell application "System Events" to tell process "Simulator" to key code 123'  # left
osascript -e 'tell application "System Events" to tell process "Simulator" to key code 124'  # right
```

**`dpad center` (Select)**:
```bash
osascript -e 'tell application "System Events" to tell process "Simulator" to key code 36'
```

**`back` (Menu)**:
```bash
osascript -e 'tell application "System Events" to tell process "Simulator" to key code 53'
```

**`nav_tab "Tab Name"`** — see smart tab navigation above.

**`wait <ms>`**:
```bash
sleep <ms/1000>
```

After **each step**, wait 500ms by default to allow the UI to settle. After the last step, wait **2000ms** before capturing (tvOS screens can have skeleton loading states that take 3–5 seconds to resolve — if the screenshot shows a black/skeleton screen, wait longer and retry).

If a step fails — capture an immediate screenshot, read it, report which step failed and what was visible, then stop and wait for `RECHECK`.

### Step 3 — Capture Screenshot from Simulator

```bash
xcrun simctl io <simulator-udid> screenshot /tmp/simulator-current.png
```

Save to `/tmp/simulator-current.png`.

Verify the file exists and is non-empty. If capture fails, report the error and stop.

### Step 4 — Read Both Images

Use the Read tool to load both images:
- **Reference**: path provided by the user
- **Current**: `/tmp/simulator-current.png`

### Step 5 — Visual Comparison

Analyze both images and produce a structured diff report:

```
## Visual Comparison Report

### Overall Match
[PASS ✅ | FAIL ❌] — [brief one-line verdict]

### Differences Found
| Area | Reference | Current | Severity |
|------|-----------|---------|----------|
| [region] | [what design shows] | [what simulator shows] | High/Med/Low |

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

**Ignore as expected dev-mode differences:**
- Focused state on the default-focused element (e.g. first button highlighted on load)
- Placeholder values for data loaded from API (email, user name, etc.)

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

## Cleanup — Revert Temporary Code Changes

After every visual check session, always revert any temporary changes made to:

| File | Change to revert |
|------|-----------------|
| `index.js` | Remove `LogBox.ignoreAllLogs()` and `LogBox` from import |
| `src/app/navigation/RootStack.tsx` | Revert `{true \|\| hasAuthorizedLaunch ?` → `{hasAuthorizedLaunch ?` |
| `src/app/navigation/TabNavigator.tsx` | Revert `initialRouteName` back to `ScreenNames.HomePageScreen` |

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

1. **Minimize code changes** — only modify code temporarily for navigation/debug purposes (LogBox, auth bypass, initialRouteName); always revert after
2. **Always show both images in the report context** — read both before comparing
3. **Be specific in diff output** — "button label is 'Submit' but design shows 'Continue'" not "button text is wrong"
4. **Report coordinates/regions** — describe where on screen the difference is (top-left, center, bottom nav, etc.)
5. **Capture fresh screenshot each RECHECK** — never reuse the previous capture
6. **Save all screenshots to `screenshots/` folder** — copy `/tmp/simulator-current.png` to `screenshots/<screen-name>-ios.png` after every successful capture
7. **Screenshots are the ground truth for navigation state** — never rely on accessibility inspection to determine focus; always take a screenshot and visually verify
8. **`input tap` / coordinate tapping is forbidden for TV navigation** — never use coordinate-based tapping; use D-pad key codes only
9. **Kill duplicate Metro processes before rebuilding** — `kill -9 $(lsof -ti:8081)` to avoid bundle conflicts

---

## Error Handling

| Error | Action |
|---|---|
| No booted tvOS simulator found | Automatically find tvOS simulators, boot the first one with `xcrun simctl boot <udid>` + `open -a Simulator`, wait 10s |
| App not installed / shows home screen | Launch app directly: `xcrun simctl launch <udid> net.nugs.multiband` |
| `NativeModule.XXX is null` crash | Run `cd ios && pod install`, then rebuild with `yarn ios` |
| React Native LogBox overlay | Add `LogBox.ignoreAllLogs()` to `index.js`, reload with keystroke "r", revert after check |
| Login screen blocking access | Temporarily set `{true \|\| hasAuthorizedLaunch ?` in `RootStack.tsx`, reload, revert after check |
| `xcrun simctl io screenshot` fails | Report error output, suggest checking simulator state |
| osascript "not allowed to send keystrokes" | Run `tccutil reset Accessibility com.apple.Terminal` and retry |
| Reference image not found | Stop, ask user to provide correct path |
| Nav bar D-pad right scrolls hero instead of tabs | Nav bar wasn't activated — press DPAD_UP more times (try 8–10) and retry |

---

## Example Invocations

**Just take a screenshot of current screen:**
> `/ios-visual-check` — just save a screenshot to the screenshots folder

**Navigate to a tab and screenshot (no reference):**
> `/ios-visual-check` — go to My Livestreams and screenshot

Steps used internally:
```
- nav_tab "Our Livestreams"
```

**Navigate into a detail screen:**
> `/ios-visual-check` — go to My Livestreams, open the first item

Steps used internally:
```
- nav_tab "Our Livestreams"
- dpad down
- dpad center
```

**Full visual QA with reference design:**
> `/ios-visual-check` — reference is `screenshots/settings-screen.png`, check Settings screen

You:
1. Check for booted tvOS simulator; boot one automatically if needed
2. Verify app is running; launch if needed; fix native module errors or LogBox overlays
3. Handle auth if login screen appears
4. Activate Simulator, execute navigation steps (if provided), verifying UI state after each step
5. Capture screenshot, save to `screenshots/`
6. If reference provided: read both images and output comparison report
7. If no reference: just show the screenshot to the user
8. If differences found: stop and wait for `RECHECK`
9. On `RECHECK`: re-run navigation steps + fresh screenshot + compare again
10. **Revert all temporary code changes** when done
