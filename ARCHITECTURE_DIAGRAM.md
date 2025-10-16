# Mouse Gesture Architecture - Visual Guide

## Current System (Batch Mode)

```
┌────────────────────────────────────────────────────────────────────┐
│                         HARDWARE LAYER                              │
│                                                                     │
│   Mouse/Trackball with Gesture Button                              │
│   ├─ Button Press Event                                            │
│   ├─ Raw XY Movement Events (continuous while button held)         │
│   └─ Button Release Event                                          │
└────────────────────────────────────────────────────────────────────┘
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                      HID++ NOTIFICATION LAYER                       │
│                    (logitech_receiver/base.py)                     │
│                                                                     │
│   HIDPPNotification objects with raw data                          │
│   ├─ Feature: REPROG_CONTROLS_V4                                   │
│   ├─ Report 0x00: Button press/release (CID list)                  │
│   └─ Report 0x10: XY movement deltas                               │
└────────────────────────────────────────────────────────────────────┘
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                    RAW XY PROCESSING LAYER                          │
│                   (settings.py:774-852)                            │
│                                                                     │
│   RawXYProcessing.handler()                                         │
│   ├─ Detects button press/release from CID changes                 │
│   ├─ Routes XY movements to active processor                       │
│   └─ Manages lifecycle (start/stop reporting)                      │
└────────────────────────────────────────────────────────────────────┘
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                    GESTURE ACCUMULATION LAYER                       │
│                (settings_templates.py:819-884)                     │
│                                                                     │
│   MouseGesturesXY (Finite State Machine)                           │
│                                                                     │
│   State: IDLE                                                       │
│      ▼ (button press)                                              │
│   State: PRESSED                                                    │
│      │                                                              │
│      │  ┌─────────────────────────────────────┐                   │
│      │  │ move_action(dx, dy) called ~100 Hz   │                   │
│      │  │                                      │                   │
│      │  │  self.dx += dx  ◄─── ACCUMULATION   │                   │
│      │  │  self.dy += dy       (NO NOTIFICATION!)                 │
│      │  │                                      │                   │
│      │  │  After 200ms timeout:                │                   │
│      │  │  └─ push_mouse_event()               │                   │
│      │  │     └─ Append [0, x, y] to data      │                   │
│      │  └─────────────────────────────────────┘                   │
│      │                                                              │
│      ▼ (button release)                                            │
│   release_action()                                                  │
│   ├─ push_mouse_event() one final time                             │
│   ├─ Pack ALL accumulated data                                     │
│   │  Example: [0xC4, 0, 5, -20, 0, 10, -35]                        │
│   │           [key , movement1  , movement2  ]                     │
│   └─ Send SINGLE notification                                      │
│      ▼                                                              │
│   State: IDLE                                                       │
└────────────────────────────────────────────────────────────────────┘
                              ▼
            ⚠️  BOTTLENECK: Only ONE notification sent ⚠️
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                        RULE EVALUATION LAYER                        │
│                     (diversion.py:1500-1554)                       │
│                                                                     │
│   process_notification()                                            │
│   └─ Calls evaluate_rules()                                        │
│      └─ For each rule...                                           │
└────────────────────────────────────────────────────────────────────┘
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                     GESTURE MATCHING LAYER                          │
│                     (diversion.py:1055-1080)                       │
│                                                                     │
│   MouseGesture.evaluate()                                           │
│   ├─ Unpack notification data                                      │
│   ├─ Match against expected pattern                                │
│   │  Example: ["Mouse Up"]  (button configured separately)         │
│   │  Data:    [0xC4, 0, 15, -55] ✓ Matches!                        │
│   └─ Return: True (match) or False (no match)                      │
│                                                                     │
│   ⚠️  Only evaluates ONCE per button press/release cycle ⚠️        │
└────────────────────────────────────────────────────────────────────┘
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                         ACTION LAYER                                │
│                     (diversion.py:1187+)                           │
│                                                                     │
│   If gesture matches:                                               │
│   └─ Execute action (KeyPress, MouseScroll, etc.)                  │
│      └─ Triggered ONCE per gesture                                 │
└────────────────────────────────────────────────────────────────────┘
```

## Proposed System (With Staggering)

```
┌────────────────────────────────────────────────────────────────────┐
│                         HARDWARE LAYER                              │
│   (unchanged)                                                       │
└────────────────────────────────────────────────────────────────────┘
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                      HID++ NOTIFICATION LAYER                       │
│   (unchanged)                                                       │
└────────────────────────────────────────────────────────────────────┘
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                    RAW XY PROCESSING LAYER                          │
│   (unchanged)                                                       │
└────────────────────────────────────────────────────────────────────┘
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                    GESTURE ACCUMULATION LAYER                       │
│                (settings_templates.py:819-884)                     │
│                        ✨ MODIFIED ✨                               │
│                                                                     │
│   MouseGesturesXY (Finite State Machine)                           │
│                                                                     │
│   State: IDLE                                                       │
│      ▼ (button press)                                              │
│   State: PRESSED                                                    │
│      │                                                              │
│      │  ┌─────────────────────────────────────┐                   │
│      │  │ move_action(dx, dy) called ~100 Hz   │                   │
│      │  │                                      │                   │
│      │  │  self.dx += dx  ◄─── ACCUMULATION   │                   │
│      │  │  self.dy += dy       (existing)      │                   │
│      │  │                                      │                   │
│      │  │  ✨ NEW: Send incremental notification                  │
│      │  │  incremental_data = [key, -1, dx, dy]│                   │
│      │  │  process_notification(...)           │ ◄─┐               │
│      │  │                                      │   │               │
│      │  │  After 200ms timeout:                │   │               │
│      │  │  └─ push_mouse_event()               │   │               │
│      │  │     └─ Append [0, x, y] to data      │   │               │
│      │  └─────────────────────────────────────┘   │               │
│      │                                             │               │
│      │  ┌─────────────────────────────────────┐   │               │
│      │  │     PARALLEL NOTIFICATION PATHS      │   │               │
│      │  │                                      │   │               │
│      │  │  Batch Path (existing):              │   │               │
│      │  │  └─ Complete gesture on release      │   │               │
│      │  │                                      │   │               │
│      │  │  Staggering Path (new): ─────────────┼───┘               │
│      │  │  └─ Incremental updates during move  │                   │
│      │  └─────────────────────────────────────┘                   │
│      │                                                              │
│      ▼ (button release)                                            │
│   release_action()                                                  │
│   ├─ push_mouse_event() (existing)                                 │
│   ├─ Pack ALL accumulated data                                     │
│   ├─ Send complete notification (marker: 0)                        │
│   └─ ✨ NEW: Clear stagger accumulators                            │
│      ▼                                                              │
│   State: IDLE                                                       │
└────────────────────────────────────────────────────────────────────┘
                    ▼                              ▼
         (incremental, continuous)    (complete, once on release)
                    │                              │
                    ▼                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        RULE EVALUATION LAYER                         │
│                     (diversion.py:1500-1554)                        │
│                         (unchanged)                                  │
│                                                                      │
│   process_notification() - processes both notification types         │
└─────────────────────────────────────────────────────────────────────┘
                    │                              │
                    ▼                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     GESTURE MATCHING LAYER                           │
│                     (diversion.py:1055-1080)                        │
│                        ✨ MODIFIED ✨                                │
│                                                                      │
│   MouseGesture.evaluate() - SPLIT INTO TWO MODES:                   │
│                                                                      │
│   ┌─────────────────────────┐     ┌─────────────────────────┐      │
│   │  STAGGERING MODE        │     │  BATCH MODE (existing)   │      │
│   │                         │     │                          │      │
│   │  Processes: Incremental │     │  Processes: Complete     │      │
│   │  (marker: -1)           │     │  (marker: 0)             │      │
│   │                         │     │                          │      │
│   │  Logic:                 │     │  Logic:                  │      │
│   │  1. Extract dx, dy      │     │  1. Match full pattern   │      │
│   │  2. Calculate distance  │     │  2. Return once          │      │
│   │     in target direction │     │                          │      │
│   │  3. Accumulate distance │     │  Use case:               │      │
│   │  4. Trigger when >=     │     │  - Traditional gestures  │      │
│   │     stagger_distance    │     │  - Complex sequences     │      │
│   │  5. Subtract threshold, │     │                          │      │
│   │     keep remainder      │     │                          │      │
│   │                         │     │                          │      │
│   │  Use case:              │     │                          │      │
│   │  - Volume control       │     │                          │      │
│   │  - Continuous scrolling │     │                          │      │
│   │  - Zooming              │     │                          │      │
│   └─────────────────────────┘     └─────────────────────────┘      │
│            │                                  │                      │
│            └──────────┬───────────────────────┘                      │
│                       ▼                                              │
│   Return: True (trigger) or False (don't trigger)                   │
│                                                                      │
│   ✨ NEW: Distance accumulator tracking                             │
│   _stagger_accumulators = {                                         │
│     (device_id, gesture_hash): accumulated_distance                 │
│   }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         ACTION LAYER                                 │
│                     (diversion.py:1187+)                            │
│                      ✨ ENHANCED ✨                                  │
│                                                                      │
│   If gesture matches:                                                │
│   └─ Execute action (KeyPress, MouseScroll, etc.)                   │
│      │                                                               │
│      ├─ Batch gestures: Triggered ONCE per button cycle             │
│      └─ Staggering gestures: Triggered REPEATEDLY every N pixels    │
│                              ▲                                       │
│                              └───────┐                               │
│                                      │                               │
│   Example with staggering:           │                               │
│   User drags up 150 pixels (stagger_distance=50)                    │
│   └─ Action triggers 3 times (50px, 100px, 150px)                   │
└─────────────────────────────────────────────────────────────────────┘
```

## Data Format Comparison

### Current (Batch Mode)
```python
# Complete gesture notification sent on release
# Format: [key_code, type, x, y, type, x, y, ...]
# type: 0 = movement, 1 = key press

# Example: Button 0xC4, moved right 50px, then up 30px
notification.data = struct.pack("!7h", 
    0xC4,  # Initiating key
    0,     # Movement marker
    50,    # X delta
    0,     # Y delta  
    0,     # Movement marker
    0,     # X delta
    -30    # Y delta (negative = up)
)
# Unpacked: [0xC4, 0, 50, 0, 0, 0, -30]
```

### Proposed (Staggering Mode)
```python
# Incremental notification sent during movement
# Format: [key_code, -1, dx, dy]  # -1 marks as "incremental"

# Example: Button 0xC4, single movement step of 5px up
notification.data = struct.pack("!4h",
    0xC4,  # Initiating key
    -1,    # Incremental marker (NEW!)
    0,     # X delta (this step only)
    -5     # Y delta (this step only)
)
# Unpacked: [0xC4, -1, 0, -5]

# MouseGesture tracks accumulation internally
# Triggers when accumulated distance >= stagger_distance
```

## State Tracking

### Stagger Accumulator Structure
```python
_stagger_accumulators = {
    # Key: (device_id, gesture_hash)
    # Value: accumulated_distance (float)
    
    (140234567890123, 12345): 25.3,  # Device A, Gesture 1: 25.3px accumulated
    (140234567890123, 67890): 103.7, # Device A, Gesture 2: 103.7px accumulated
    (140234567890456, 12345): 0.0,   # Device B, Gesture 1: just triggered, reset
}

# Cleaned up on:
# 1. Button release (gesture complete)
# 2. Gesture match + trigger (reset to remainder)
# 3. Device disconnect (periodic cleanup)
```

## Example: Volume Control Use Case

```yaml
# User configuration
---
- MouseGesture:
    movements: [Mouse Up]
    staggering: true
    distance: 50
- KeyPress: [XF86_AudioRaiseVolume, click]
---
```

**Note:** The gesture button (e.g., side button) is configured separately in device settings under "Key/Button Diversion" → "Mouse Gestures".

### Execution Flow
```
1. User presses gesture button (e.g., side button)
   └─ MouseGesturesXY: State = PRESSED
   
2. User drags up 20 pixels
   └─ move_action(0, -20) called
      ├─ Accumulate: dx=0, dy=-20
      └─ Send incremental: [0xC4, -1, 0, -20]
         └─ MouseGesture.evaluate()
            ├─ Direction: "Mouse Up" ✓
            ├─ Distance: 20px (in target direction)
            ├─ Accumulator: 0 + 20 = 20px
            └─ Threshold: 50px ✗ (don't trigger)

3. User drags up another 35 pixels (total: 55px)
   └─ move_action(0, -35) called
      ├─ Accumulate: dx=0, dy=-55
      └─ Send incremental: [0xC4, -1, 0, -35]
         └─ MouseGesture.evaluate()
            ├─ Direction: "Mouse Up" ✓
            ├─ Distance: 35px
            ├─ Accumulator: 20 + 35 = 55px
            ├─ Threshold: 50px ✓ TRIGGER!
            ├─ Action: KeyPress(XF86_AudioRaiseVolume) ← Volume +1
            └─ Accumulator: 55 - 50 = 5px (remainder)

4. User drags up another 50 pixels (total: 105px from start)
   └─ move_action(0, -50) called
      └─ Send incremental: [0xC4, -1, 0, -50]
         └─ MouseGesture.evaluate()
            ├─ Accumulator: 5 + 50 = 55px
            ├─ Threshold: 50px ✓ TRIGGER!
            ├─ Action: KeyPress(XF86_AudioRaiseVolume) ← Volume +1
            └─ Accumulator: 55 - 50 = 5px

5. User releases button
   └─ release_action() called
      ├─ Send complete notification (for batch gestures)
      └─ Clear accumulator
```

**Result:** Volume increased 2 times during the gesture!

## UI Mockup

```
┌───────────────────────────────────────────────────────────────┐
│  Rule Editor - Mouse Gesture Condition                        │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  Mouse gesture with optional initiating button followed by    │
│  zero or more mouse movements.                                │
│                                                                │
│  ┌──────────────────────────────┐ ┌────────┐                 │
│  │ Mouse Up                 ▾   │ │ Delete │                 │
│  └──────────────────────────────┘ └────────┘                 │
│                                                                │
│  ┌─────────────────┐                                          │
│  │  Add movement   │                                          │
│  └─────────────────┘                                          │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ ☑ Enable Staggering                                     │ │
│  │                                                          │ │
│  │   Trigger repeatedly every N pixels while dragging       │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                                │
│  Stagger Distance (pixels):  ┌───┐                            │
│                              │ 50│ ◀ ▶                        │
│                              └───┘                            │
│  (Range: 10-500 pixels)                                       │
│                                                                │
│  Recommended:                                                  │
│  • Volume control: 30-50 pixels                               │
│  • Scrolling: 20-40 pixels                                    │
│  • Zooming: 40-60 pixels                                      │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

## Performance Considerations

### Notification Frequency
- XY movements arrive at ~100 Hz (10ms intervals) from hardware
- Each move_action() call currently just accumulates
- With staggering: Each call sends incremental notification

**Concern:** Too many notifications?

**Solution:** Add minimum interval throttle
```python
# In move_action():
MIN_NOTIFICATION_INTERVAL_MS = 20  # 50 Hz max

if (now - self.last_incremental_notification) >= MIN_NOTIFICATION_INTERVAL_MS:
    # Send incremental notification
    self.last_incremental_notification = now
```

### Memory Usage
- Accumulator dict grows with number of active staggering gestures
- Each entry: ~50 bytes (key tuple + float)
- Typical usage: 1-5 active gestures = 250 bytes
- Cleanup on button release prevents leaks

**Negligible impact** ✓

### CPU Impact
- Additional notification processing
- Additional distance calculations
- Estimated: <2% CPU increase (benchmarking needed)

**Acceptable for feature value** ✓

## Testing Checklist

### Unit Tests
- [ ] MouseGesture initialization with dict format
- [ ] Distance accumulation logic
- [ ] Directional filtering (only count movement in target direction)
- [ ] Threshold triggering with remainder handling
- [ ] Accumulator cleanup
- [ ] Backward compatibility with list format

### Integration Tests
- [ ] Complete flow: press → move → trigger → move → trigger → release
- [ ] Multiple simultaneous staggering gestures
- [ ] Batch + staggering rules active simultaneously
- [ ] Accumulator persistence across movements

### Manual Tests
- [ ] Volume control while dragging
- [ ] Scrolling with staggering
- [ ] Direction changes mid-gesture
- [ ] Rapid back-and-forth movements
- [ ] Very slow movements (edge case)
- [ ] Configuration save/load

### Edge Cases
- [ ] Zero movements (button press/release immediately)
- [ ] Movements in opposite direction (shouldn't count)
- [ ] Diagonal movements with non-diagonal target
- [ ] Device disconnect during gesture
- [ ] Multiple rules with different stagger distances

---

**Legend:**
- ✨ = New/Modified component
- ⚠️ = Bottleneck/Issue in current design
- ✓ = Verified/Acceptable
- ◄─ = Data flow direction
