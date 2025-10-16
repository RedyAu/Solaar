# Mouse Gesture Staggering - Testing Guide

## Installation Instructions for Testing

### Prerequisites
- Kubuntu with Solaar installed via apt
- Git installed
- Python 3 with GTK3 bindings
- A Logitech device with a programmable button

### Step 1: Clone and Setup Test Environment

```bash
# Navigate to a temporary directory
cd ~/Downloads

# Clone the repository
git clone https://github.com/RedyAu/Solaar.git
cd Solaar

# Check out the PR branch
git checkout copilot/refactor-mouse-gestures

# Verify you're on the correct branch
git branch
# Should show: * copilot/refactor-mouse-gestures
```

### Step 2: Stop the System Solaar

```bash
# Stop the system Solaar service/daemon
solaar unregister
killall solaar

# Or use systemd if running as service
systemctl --user stop solaar.service
```

### Step 3: Run Test Version from Source

```bash
# Run Solaar directly from the cloned repository
cd ~/Downloads/Solaar
./bin/solaar
```

This will launch the test version of Solaar with the staggering feature enabled.

**Alternative (if above doesn't work):**
```bash
# Set Python path and run
cd ~/Downloads/Solaar
PYTHONPATH=./lib python3 bin/solaar
```

## What to Test

### Test 1: Basic Staggering - Volume Control

**Setup:**
1. Launch test Solaar: `cd ~/Downloads/Solaar && ./bin/solaar`
2. Select your Logitech device in the left panel
3. Find a programmable button (e.g., side button, back button)
4. In the button's settings, set it to "Mouse Gestures"
5. Go to "Rules" tab at the bottom
6. Click "+ Add Rule"

**Create the Rule:**
1. Condition: "Mouse Gesture"
   - Add movement: "Mouse Up"
   - ☑ Enable Staggering (trigger repeatedly every N pixels)
   - Stagger Distance: 50 pixels
2. Action: "Key Press"
   - Key: XF86_AudioRaiseVolume (or XF86AudioRaiseVolume)
   - Action: click
3. Save the rule

**Test Procedure:**
1. Press and hold the gesture button (the one you configured)
2. While holding, drag your mouse upward slowly
3. **Expected:** Volume should increase in steps as you drag
   - Every 50 pixels of upward movement = 1 volume increase
   - You should hear/see volume going up repeatedly
4. Stop moving → volume should stop increasing
5. Continue moving up → volume should increase again
6. Release button → all accumulators reset

**Success Criteria:**
- ✅ Volume increases continuously while dragging up
- ✅ Volume stops increasing when you stop moving
- ✅ Volume continues increasing when you resume upward movement
- ✅ No volume change when dragging sideways or down
- ✅ Releasing button doesn't trigger extra volume change

### Test 2: Bi-directional Control

**Create Two Rules:**
1. **Rule 1:** Mouse Up (staggering: 50px) → Volume Up
2. **Rule 2:** Mouse Down (staggering: 50px) → Volume Down

**Test Procedure:**
1. Press gesture button
2. Drag up → volume increases
3. Drag down → volume decreases
4. Drag up and down in waves → volume goes up and down accordingly
5. Release button

**Success Criteria:**
- ✅ Upward motion increases volume
- ✅ Downward motion decreases volume
- ✅ Direction changes work smoothly
- ✅ No interference between the two rules
- ✅ Each direction maintains its own accumulator

### Test 3: Different Distances

**Test with various stagger distances:**
- 10 pixels: Very sensitive, triggers frequently
- 50 pixels: Default, good balance
- 100 pixels: Less sensitive, requires more movement
- 200 pixels: Coarse control

**Success Criteria:**
- ✅ Smaller distances trigger more frequently
- ✅ Larger distances require more movement
- ✅ All distances work correctly

### Test 4: Horizontal Gestures

**Create Rule:**
- Mouse Right (staggering: 50px) → Next Track (or Right Arrow key)

**Test:**
- Drag right while holding gesture button
- Should trigger repeatedly

### Test 5: UI Functionality

**Test the rule editor UI:**
1. Create a staggering gesture rule
2. Save and close the rule editor
3. Reopen the rule to edit it
4. **Verify:**
   - ✅ Checkbox state is preserved
   - ✅ Distance value is preserved
   - ✅ Can change distance while editing
   - ✅ Can disable staggering (uncheck)
   - ✅ Distance field grays out when unchecked

### Test 6: Legacy Compatibility

**Test existing non-staggering gestures:**
1. Create a regular (non-staggering) gesture
   - Mouse Up → Volume Up
   - Staggering: unchecked
2. Use the gesture
3. **Expected:** Triggers ONCE on button release (old behavior)

**Success Criteria:**
- ✅ Old gestures still work as before
- ✅ No staggering behavior on legacy rules
- ✅ Triggers once on release, not during movement

### Test 7: Multiple Devices

If you have multiple Logitech devices:
1. Configure staggering gestures on Device 1
2. Configure staggering gestures on Device 2
3. Use both alternately

**Success Criteria:**
- ✅ Each device maintains separate accumulators
- ✅ No interference between devices

## Debugging

### Enable Debug Logging

```bash
# Run with debug logging
cd ~/Downloads/Solaar
./bin/solaar -ddd
```

Look for log messages like:
```
DEBUG: incremental notification sent: dx=0, dy=-15
DEBUG: staggering gesture triggered: Mouse Up (accumulated: 55, threshold: 50)
```

### Common Issues

**Issue:** Volume doesn't change
- **Fix:** Check audio keys in your system
- Try different keys: XF86_AudioRaiseVolume vs XF86AudioRaiseVolume
- Test with a simple key like "Up" or "Right" first

**Issue:** Triggers too often/rarely
- **Fix:** Adjust the stagger distance
- Smaller = more sensitive (10-30px)
- Larger = less sensitive (100-200px)

**Issue:** UI doesn't show staggering controls
- **Fix:** Make sure you're running the test version
- Check git branch: `git branch` should show `copilot/refactor-mouse-gestures`

## Performance Testing

### Monitor CPU Usage

```bash
# In another terminal while testing
top -p $(pgrep -f "solaar")
```

**Expected:**
- Idle: <1% CPU
- During staggering: 2-5% CPU (should be minimal)
- No memory leaks over time

### Test Rate Limiting

1. Move mouse very quickly while holding gesture button
2. Check debug logs for notification frequency
3. **Expected:** Max 50 notifications per second (20ms interval)

## Reporting Issues

If you find bugs, please report:
1. **What you did:** Step-by-step reproduction
2. **What happened:** Actual behavior
3. **What you expected:** Expected behavior
4. **Logs:** Debug output from `./bin/solaar -ddd`
5. **Device:** Your Logitech device model
6. **System:** Ubuntu/Kubuntu version

## Switching Back to System Solaar

After testing:

```bash
# Stop test version
# Press Ctrl+C in the terminal running ./bin/solaar

# Restart system version
solaar register
solaar &

# Or with systemd
systemctl --user start solaar.service
```

## Configuration Files

Test configuration is saved to:
- Rules: `~/.config/solaar/rules.yaml`
- Settings: `~/.config/solaar/config.yaml`

**Backup before testing:**
```bash
cp ~/.config/solaar/rules.yaml ~/.config/solaar/rules.yaml.backup
cp ~/.config/solaar/config.yaml ~/.config/solaar/config.yaml.backup
```

**Restore after testing:**
```bash
cp ~/.config/solaar/rules.yaml.backup ~/.config/solaar/rules.yaml
cp ~/.config/solaar/config.yaml.backup ~/.config/solaar/config.yaml
```

## Expected Results Summary

✅ **Working correctly:**
- Volume/brightness control with continuous adjustment
- Smooth scrolling through items/pages
- Zoom in/out with mouse gestures
- Bi-directional control (up/down, left/right)
- No interference between different staggering rules
- Legacy non-staggering gestures still work

❌ **Not working (report as bug):**
- Actions not triggering during movement
- Triggers only on button release (old behavior)
- Direction changes cause issues
- Memory leaks or high CPU usage
- UI not showing/saving staggering parameters
- Multiple directions causing conflicts

## Advanced Testing

### Test Configuration via YAML

Directly edit `~/.config/solaar/rules.yaml`:

```yaml
%YAML 1.3
---
- Rule:
  - MouseGesture:
      movements: [Mouse Up]
      staggering: true
      distance: 50
  - KeyPress: [XF86_AudioRaiseVolume, click]
```

Restart Solaar and verify the rule loads correctly.

### Test Edge Cases

1. **Zero movement:** Hold button without moving → no triggers
2. **Opposite direction:** Drag up then down → only relevant rule triggers
3. **Quick button tap:** Press/release without moving → no triggers
4. **Device disconnect:** Unplug device → accumulators cleared
5. **Very large distance:** 500px setting → works but requires big movement
6. **Very small distance:** 10px setting → triggers very frequently

## Video Recording

For bug reports, record your test session:

```bash
# Using SimpleScreenRecorder or similar
simplescreenrecorder

# Or ffmpeg
ffmpeg -video_size 1920x1080 -framerate 30 -f x11grab -i :0.0 test_staggering.mp4
```

Show:
1. Opening Solaar UI
2. Creating the rule
3. Using the gesture
4. Expected vs actual behavior
