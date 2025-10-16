# Quick Start: Mouse Gesture Staggering

## What is Staggering?

Staggering allows mouse gesture rules to trigger **repeatedly** at fixed distance intervals while the gesture button is held and the mouse is moving.

**Example:** Increase volume continuously while dragging the mouse upward.

## Quick Test (5 minutes)

### 1. Install Test Version

```bash
cd ~/Downloads
git clone https://github.com/RedyAu/Solaar.git
cd Solaar
git checkout copilot/refactor-mouse-gestures

# Stop system Solaar
killall solaar

# Run test version
./bin/solaar
```

### 2. Configure a Button

1. Select your device in Solaar
2. Find a button (e.g., "Back Button" or "Side Button")
3. Set it to: **Mouse Gestures**

### 3. Create a Staggering Rule

1. Go to "Rules" tab
2. Click "+ Add Rule"
3. Configure:
   ```
   Condition: Mouse Gesture
     - Movement: Mouse Up
     - ☑ Enable Staggering (trigger repeatedly every N pixels)
     - Stagger Distance: 50 pixels
   
   Action: Key Press
     - Key: XF86_AudioRaiseVolume
     - Action: click
   ```
4. Save

### 4. Test It!

1. Press and **hold** the gesture button
2. Drag mouse **upward** slowly
3. **Hear:** Volume increases continuously!
4. Stop moving → Volume stops increasing
5. Move more → Volume increases again
6. Release button

## Key Features

✅ **Triggers repeatedly** while moving (not just on release)  
✅ **Distance-based** (default: 50 pixels)  
✅ **Direction-aware** (only triggers for target direction)  
✅ **Multi-rule support** (up=volume up, down=volume down)  
✅ **Backward compatible** (old gestures still work)

## Use Cases

### Volume Control
- **Up:** Volume Up (50px)
- **Down:** Volume Down (50px)

### Brightness Control
- **Up:** Brightness Up (50px)
- **Down:** Brightness Down (50px)

### Scrolling
- **Right:** Next Tab (100px)
- **Left:** Previous Tab (100px)

### Zoom
- **Up:** Zoom In (75px)
- **Down:** Zoom Out (75px)

## Tips

- **Sensitive control:** Use 10-30 pixels
- **Normal control:** Use 40-60 pixels (default: 50)
- **Coarse control:** Use 100-200 pixels
- **Very coarse:** Use 300-500 pixels

## Troubleshooting

**Problem:** Nothing happens  
→ Check the button is set to "Mouse Gestures"  
→ Try debug mode: `./bin/solaar -ddd`

**Problem:** Only triggers once (on release)  
→ Make sure "Enable Staggering" checkbox is checked  
→ Verify distance is reasonable (50px is good default)

**Problem:** Too sensitive / Not sensitive enough  
→ Adjust the "Stagger Distance" value  
→ Smaller = more triggers, Larger = fewer triggers

**Problem:** Volume keys don't work  
→ Try: XF86AudioRaiseVolume (without underscore)  
→ Or test with simple keys first: "Up" or "Right"

## Full Documentation

See `TESTING_GUIDE.md` for comprehensive testing instructions and advanced use cases.

## Switching Back

```bash
# Stop test Solaar (Ctrl+C)
# Start system Solaar
solaar &
```
