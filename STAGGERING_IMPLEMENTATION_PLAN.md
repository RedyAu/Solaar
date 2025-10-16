# Mouse Gesture Staggering Feature - Implementation Plan

## Executive Summary

This document provides a comprehensive analysis of Solaar's mouse gesture architecture and a detailed implementation plan for adding "staggering" support. Staggering will allow gesture-based rules to trigger repeatedly at fixed distance intervals while the gesture button is held and the mouse is moving.

**Problem:** Currently, mouse gesture rules only trigger once after the gesture button is released, preventing use cases like continuous volume adjustment while dragging.

**Solution:** Add staggering mode where gestures can trigger every N pixels of movement in a direction while the button remains pressed.

## Current Architecture Analysis

### Input Processing Flow

```
Hardware (Mouse)
    ↓ [Raw XY HID++ notifications]
RawXYProcessing.handler()
    ↓ [Detects button press/release, routes XY movements]
MouseGesturesXY
    ↓ [Accumulates dx/dy, manages FSM]
    ├─ press_action()     → Initialize, set state to PRESSED
    ├─ move_action()      → Accumulate dx/dy (NO NOTIFICATION SENT HERE!)
    ├─ key_action()       → Handle other key presses during gesture
    └─ release_action()   → Send ONE complete notification
                            ↓
diversion.process_notification()
    ↓ [Evaluates notification against rules]
MouseGesture.evaluate()
    ↓ [Pattern matching on complete gesture sequence]
Rule Actions (KeyPress, MouseScroll, etc.)
```

### Key Files and Their Roles

#### 1. `/lib/logitech_receiver/settings.py` - Base Processing
- **Class:** `RawXYProcessing` (lines 774-852)
- **Purpose:** Base class for processing raw XY movement notifications
- **Key Methods:**
  - `handler()`: Receives HID++ notifications, detects button press/release
  - `start(key)`: Enables raw XY reporting for a button
  - `stop(key)`: Disables raw XY reporting

#### 2. `/lib/logitech_receiver/settings_templates.py` - Gesture Generation
- **Class:** `MouseGesturesXY` (lines 819-884)
- **Purpose:** Converts raw XY movements into gesture data structures
- **FSM States:** IDLE → PRESSED → (accumulate movements) → IDLE
- **Key Methods:**
  - `press_action(key)`: Button pressed, initialize gesture tracking
  - `move_action(dx, dy)`: **CRITICAL** - Accumulates movement, NO notification sent
  - `release_action()`: **BOTTLENECK** - Only time notification is sent
  - `push_mouse_event()`: Formats accumulated dx/dy into gesture data

**Key Insight:** `move_action()` is called continuously during movement but only accumulates data. The notification is sent once on release. This is the bottleneck preventing staggering.

#### 3. `/lib/logitech_receiver/diversion.py` - Rule Evaluation
- **Class:** `MouseGesture` (lines 1031-1084)
- **Purpose:** Condition class that matches gesture patterns
- **Current Behavior:** Matches complete gesture sequences
- **Data Format:** 
  ```python
  [key_code, 0, x1, y1, 0, x2, y2, ...]  # 0 = movement, 1 = key press
  ```
- **Key Methods:**
  - `__init__(movements)`: Configure expected gesture pattern
  - `evaluate()`: Match notification data against pattern
  - `data()`: Serialize for storage

#### 4. `/lib/solaar/ui/rule_conditions.py` - User Interface
- **Class:** `MouseGestureUI` (lines 517-616)
- **Purpose:** GUI for configuring mouse gesture rules
- **Current UI:** 
  - Label with description
  - Dynamic list of movement fields
  - Add/Delete buttons for movements

### Current Data Flow Example

**User Action:** Press gesture button → drag up 100px → release

```python
# In MouseGesturesXY:
press_action(key_0xC4)     # data = [0xC4]
move_action(2, -5)         # dx += 2, dy += -5
move_action(3, -7)         # dx += 5, dy += -12
move_action(1, -3)         # dx += 6, dy += -15
release_action()           # Calls push_mouse_event()
                          # data = [0xC4, 0, 6, -15]
                          # Sends notification

# In MouseGesture.evaluate():
data = [0xC4, 0, 6, -15]
# Matches: initiating key 0xC4, then "Mouse Up" (negative y)
# Returns: True (rule triggers once)
```

## Problem Analysis

### Why Staggering Doesn't Work Currently

1. **Batch Processing:** Movements are accumulated and sent as ONE notification on release
2. **Single Evaluation:** Rules evaluate the complete gesture sequence once
3. **No State Tracking:** No mechanism to track "distance traveled since last trigger"

### What Staggering Needs

1. **Incremental Notifications:** Send updates DURING movement, not just on release
2. **Distance Tracking:** Accumulate distance traveled in target direction
3. **Repeated Triggering:** Match and trigger every N pixels moved
4. **State Persistence:** Remember accumulated distance across multiple move events

## Implementation Plan

### Phase 1: Data Model Changes

#### 1.1 Extend MouseGesture Class (`diversion.py`)

**Location:** Lines 1031-1084

```python
class MouseGesture(Condition):
    def __init__(self, movements, warn=True):
        # Support both formats:
        # Legacy: ["Mouse Up"]
        # New:    {"movements": ["Mouse Up"], "staggering": True, "distance": 50}
        
        if isinstance(movements, dict):
            self.movements = movements.get("movements", [])
            self.staggering = movements.get("staggering", False)
            self.stagger_distance = movements.get("distance", 50)
        else:
            if isinstance(movements, str):
                movements = [movements]
            self.movements = movements
            self.staggering = False
            self.stagger_distance = 0
```

**Backward Compatibility:** Existing rules with list format continue working.

#### 1.2 Add Global State Tracking

```python
# In diversion.py, add module-level:
_stagger_accumulators = {}  # Key: (device_id, gesture_hash), Value: accumulated_distance

def _get_accumulator_key(device, gesture):
    """Create unique key for tracking gesture state"""
    gesture_id = hash(tuple(gesture.movements))
    return (id(device), gesture_id)
```

### Phase 2: Notification Generation Changes

#### 2.1 Modify MouseGesturesXY.move_action() (`settings_templates.py`)

**Current (lines 849-862):**
```python
def move_action(self, dx, dy):
    if self.fsmState == State.PRESSED:
        # ... accumulate dx, dy ...
        self.dx += dx
        self.dy += dy
        # NO notification sent here!
```

**Proposed:**
```python
def move_action(self, dx, dy):
    if self.fsmState == State.PRESSED:
        # ... existing accumulation code ...
        self.dx += dx
        self.dy += dy
        self.lastEv = now
        
        # NEW: Send incremental notification for staggering support
        if dx != 0 or dy != 0:  # Only if there's actual movement
            # Create minimal incremental notification
            # Format: [key_code, -1, dx, dy]  # -1 flags as "incremental"
            incremental_data = [self.data[0], -1, int(dx), int(dy)]
            payload = struct.pack("!" + (len(incremental_data) * "h"), *incremental_data)
            notification = base.HIDPPNotification(0, 0, 0, 0, payload)
            diversion.process_notification(self.device, notification, _F.MOUSE_GESTURE)
```

**Key Decision:** Use `-1` as marker for "incremental" vs `0` for "complete" movement.

#### 2.2 Update release_action() for Clarity

```python
def release_action(self):
    if self.fsmState == State.PRESSED:
        self.push_mouse_event()
        # ... existing code ...
        # This remains the "complete" notification for batch gestures
```

### Phase 3: Evaluation Logic Changes

#### 3.1 Update MouseGesture.evaluate() (`diversion.py`)

**Location:** Lines 1055-1080

```python
def evaluate(self, feature, notification: HIDPPNotification, device, last_result):
    if feature == SupportedFeature.MOUSE_GESTURE:
        d = notification.data
        data = struct.unpack("!" + (int(len(d) / 2) * "h"), d)
        
        # Detect notification type
        is_incremental = len(data) == 4 and data[1] == -1
        
        if self.staggering and is_incremental:
            # Staggering mode: process incremental movement
            return self._evaluate_staggering(data, device)
        elif not self.staggering and not is_incremental:
            # Batch mode: process complete gesture (existing logic)
            return self._evaluate_batch(data)
        else:
            # Mismatched: staggering rule got batch or vice versa
            return False

def _evaluate_staggering(self, data, device):
    """Evaluate incremental movement for staggering"""
    key_code, marker, dx, dy = data
    
    # Verify initiating key if specified
    if self.movements and self.movements[0] not in self.MOVEMENTS:
        if self.movements[0] != str(CONTROL[key_code]):
            return False
        target_direction = self.movements[1] if len(self.movements) > 1 else None
    else:
        target_direction = self.movements[0] if self.movements else None
    
    if not target_direction:
        return False
    
    # Calculate distance in target direction
    directional_distance = self._calculate_directional_distance(dx, dy, target_direction)
    
    # Track accumulation
    acc_key = _get_accumulator_key(device, self)
    _stagger_accumulators[acc_key] = _stagger_accumulators.get(acc_key, 0.0) + directional_distance
    
    # Trigger if threshold exceeded
    if _stagger_accumulators[acc_key] >= self.stagger_distance:
        _stagger_accumulators[acc_key] -= self.stagger_distance  # Keep remainder
        return True
    
    return False

def _calculate_directional_distance(self, dx, dy, direction):
    """Calculate distance moved in specific direction"""
    # Map direction to axis and sign
    direction_map = {
        "Mouse Up": (0, -1),      # Negative Y
        "Mouse Down": (0, 1),     # Positive Y
        "Mouse Left": (-1, 0),    # Negative X
        "Mouse Right": (1, 0),    # Positive X
        # Diagonals use both components
        "Mouse Up-left": (-0.707, -0.707),
        "Mouse Up-right": (0.707, -0.707),
        "Mouse Down-left": (-0.707, 0.707),
        "Mouse Down-right": (0.707, 0.707),
    }
    
    x_factor, y_factor = direction_map.get(direction, (0, 0))
    # Project movement onto direction vector
    distance = (dx * x_factor) + (dy * y_factor)
    return max(0, distance)  # Only count positive movement in target direction

def _evaluate_batch(self, data):
    """Evaluate complete gesture (existing logic)"""
    # ... existing evaluation code unchanged ...
```

#### 3.2 Add State Cleanup on Button Release

```python
# In process_notification() or MouseGesturesXY.release_action():
# Clear stagger accumulator when button released
if is_button_release_event:
    acc_key = _get_accumulator_key(device, active_gestures)
    _stagger_accumulators.pop(acc_key, None)
```

### Phase 4: UI Changes

#### 4.1 Add Staggering Controls (`rule_conditions.py`)

**Location:** MouseGestureUI.create_widgets()

```python
def create_widgets(self):
    # ... existing widgets ...
    
    # NEW: Staggering checkbox
    self.staggering_checkbox = Gtk.CheckButton(
        label=_("Enable Staggering"),
        halign=Gtk.Align.START,
        tooltip_text=_("Trigger repeatedly every N pixels while dragging")
    )
    self.staggering_checkbox.connect(GtkSignal.TOGGLED.value, self._on_staggering_toggled)
    
    # NEW: Distance spinner
    self.stagger_distance_label = Gtk.Label(
        label=_("Stagger Distance (pixels):"),
        halign=Gtk.Align.END
    )
    
    self.stagger_distance_field = Gtk.SpinButton.new_with_range(10, 500, 5)
    self.stagger_distance_field.set_value(50)
    self.stagger_distance_field.set_tooltip_text(
        _("Distance in pixels to travel before triggering again")
    )
    self.stagger_distance_field.connect(GtkSignal.VALUE_CHANGED.value, self._on_update)
    
    # Initially hide distance controls
    self.stagger_distance_label.set_no_show_all(True)
    self.stagger_distance_field.set_no_show_all(True)

def _on_staggering_toggled(self, checkbox):
    """Show/hide distance field based on checkbox"""
    enabled = checkbox.get_active()
    self.stagger_distance_label.set_visible(enabled)
    self.stagger_distance_field.set_visible(enabled)
    self._on_update()
```

#### 4.2 Update show() and collect_value()

```python
def show(self, component, editable=True):
    # ... existing code ...
    
    # Load staggering state
    if hasattr(component, 'staggering'):
        with self.ignore_changes():
            self.staggering_checkbox.set_active(component.staggering)
            self.stagger_distance_field.set_value(component.stagger_distance)
            self.stagger_distance_label.set_visible(component.staggering)
            self.stagger_distance_field.set_visible(component.staggering)

def collect_value(self):
    movements = [f.get_active_text().strip() for f in self.fields if f.get_visible()]
    
    if self.staggering_checkbox.get_active():
        return {
            "movements": movements,
            "staggering": True,
            "distance": int(self.stagger_distance_field.get_value())
        }
    else:
        return movements  # Legacy format for backward compatibility
```

### Phase 5: Serialization Changes

#### 5.1 Update MouseGesture.data() (`diversion.py`)

```python
def data(self):
    if self.staggering:
        return {
            "MouseGesture": {
                "movements": [str(m) for m in self.movements],
                "staggering": True,
                "distance": self.stagger_distance
            }
        }
    else:
        return {"MouseGesture": [str(m) for m in self.movements]}
```

#### 5.2 Ensure YAML Handling

The existing YAML save/load in `diversion.py` should handle the dict format automatically, but verify:

```python
# In _save_config_rule_file() - should already work
# In _load_rule_config() - should already work
```

## Testing Strategy

### Unit Tests

Create `/tests/logitech_receiver/test_mouse_gesture_staggering.py`:

```python
import pytest
from logitech_receiver import diversion
from logitech_receiver.base import HIDPPNotification

def test_staggering_initialization():
    """Test staggering parameters in dict format"""
    config = {
        "movements": ["Mouse Up"],
        "staggering": True,
        "distance": 75
    }
    gesture = diversion.MouseGesture(config)
    assert gesture.staggering == True
    assert gesture.stagger_distance == 75
    assert gesture.movements == ["Mouse Up"]

def test_legacy_format_still_works():
    """Ensure backward compatibility"""
    gesture = diversion.MouseGesture(["Mouse Up"])
    assert gesture.staggering == False
    assert gesture.movements == ["Mouse Up"]

def test_incremental_notification_accumulation():
    """Test distance accumulation and triggering"""
    gesture = diversion.MouseGesture({
        "movements": ["Mouse Up"],
        "staggering": True,
        "distance": 50
    })
    device = MockDevice()
    
    # First movement: 20 pixels up
    data = struct.pack("!hhhh", 0xC4, -1, 0, -20)
    notif = HIDPPNotification(0, 0, 0, 0, data)
    result = gesture.evaluate(SupportedFeature.MOUSE_GESTURE, notif, device, None)
    assert result == False  # Below threshold
    
    # Second movement: 35 more pixels up (total: 55)
    data = struct.pack("!hhhh", 0xC4, -1, 0, -35)
    notif = HIDPPNotification(0, 0, 0, 0, data)
    result = gesture.evaluate(SupportedFeature.MOUSE_GESTURE, notif, device, None)
    assert result == True  # Exceeded threshold (50)
    
    # Accumulator should have remainder (5)
    # Third movement: 10 pixels (total: 15, below threshold again)
    data = struct.pack("!hhhh", 0xC4, -1, 0, -10)
    notif = HIDPPNotification(0, 0, 0, 0, data)
    result = gesture.evaluate(SupportedFeature.MOUSE_GESTURE, notif, device, None)
    assert result == False

def test_directional_filtering():
    """Test that only movement in target direction counts"""
    gesture = diversion.MouseGesture({
        "movements": ["Mouse Up"],
        "staggering": True,
        "distance": 50
    })
    device = MockDevice()
    
    # Movement to the right (shouldn't count for "up")
    data = struct.pack("!hhhh", 0xC4, -1, 50, 0)
    notif = HIDPPNotification(0, 0, 0, 0, data)
    result = gesture.evaluate(SupportedFeature.MOUSE_GESTURE, notif, device, None)
    assert result == False
    
    # Movement down (opposite direction, shouldn't count)
    data = struct.pack("!hhhh", 0xC4, -1, 0, 50)
    notif = HIDPPNotification(0, 0, 0, 0, data)
    result = gesture.evaluate(SupportedFeature.MOUSE_GESTURE, notif, device, None)
    assert result == False
```

### Integration Tests

Test the complete flow:
1. Configure rule with staggering
2. Simulate button press + movement
3. Verify multiple action triggers
4. Verify accumulator reset on release

### Manual Testing

Use case: Volume control while dragging
```yaml
# Example rule configuration
---
- MouseGesture:
    movements: [Mouse Up]
    staggering: true
    distance: 50
- KeyPress: [XF86_AudioRaiseVolume, click]
---
```

**Note:** The gesture button (e.g., side button) is configured separately in device settings under "Key/Button Diversion" → "Mouse Gestures". Only movement direction(s) are specified in the MouseGesture rule.

## Implementation Phases

### Phase 1 (Week 1): Core Logic
- [ ] Add staggering parameters to MouseGesture class
- [ ] Implement _evaluate_staggering() method
- [ ] Add global accumulator tracking
- [ ] Write unit tests for evaluation logic

### Phase 2 (Week 2): Notification Generation
- [ ] Modify MouseGesturesXY.move_action() to send incremental notifications
- [ ] Add incremental notification marker (-1)
- [ ] Add accumulator cleanup on button release
- [ ] Test notification flow

### Phase 3 (Week 3): UI Implementation
- [ ] Add staggering checkbox to MouseGestureUI
- [ ] Add distance spinner
- [ ] Implement show/collect_value updates
- [ ] Test UI interaction

### Phase 4 (Week 4): Integration & Testing
- [ ] End-to-end testing
- [ ] Performance testing (notification frequency)
- [ ] Edge case testing (rapid movements, direction changes)
- [ ] Documentation updates

## Edge Cases & Considerations

### 1. Performance
- **Issue:** Sending notifications on every move_action() call might be frequent
- **Solution:** 
  - Add minimum time threshold (e.g., 10ms between incremental notifications)
  - Or batch small movements before sending

### 2. Direction Changes
- **Issue:** User changes direction mid-gesture
- **Solution:** Reset accumulator when direction changes significantly (>90°)

### 3. Multiple Staggering Rules
- **Issue:** Multiple rules with different stagger distances
- **Solution:** Each rule maintains its own accumulator (already handled by _get_accumulator_key)

### 4. Memory Leaks
- **Issue:** Accumulators not cleaned up
- **Solution:** 
  - Clean on button release (already planned)
  - Add periodic cleanup of stale entries (device disconnected, etc.)

### 5. Backward Compatibility
- **Issue:** Existing rules must continue working
- **Solution:** 
  - Support both list and dict formats in __init__
  - Non-staggering rules ignore incremental notifications
  - Default staggering=False for all existing rules

## Open Questions

1. **Notification Frequency:** Should there be a rate limit on incremental notifications?
   - **Recommendation:** Yes, max 50 Hz (20ms minimum interval)

2. **Diagonal Movements:** How to calculate distance for diagonal gestures?
   - **Recommendation:** Use vector projection (already in plan)

3. **UI Placement:** Where to place staggering controls in the UI?
   - **Recommendation:** Below movement list, above action buttons

4. **Default Stagger Distance:** What's a good default?
   - **Recommendation:** 50 pixels (based on typical mouse DPI and hand movement)

## Success Criteria

1. ✅ User can enable staggering on a mouse gesture rule
2. ✅ Rule triggers repeatedly every N pixels of movement
3. ✅ Existing rules without staggering continue working
4. ✅ UI is intuitive and clearly labeled
5. ✅ Performance impact is minimal (<5% CPU increase)
6. ✅ Configuration persists across Solaar restarts
7. ✅ Documentation is clear and includes examples

## References

### Key Code Locations

- **MouseGesture Class:** `/lib/logitech_receiver/diversion.py:1031-1084`
- **MouseGesturesXY Class:** `/lib/logitech_receiver/settings_templates.py:819-884`
- **RawXYProcessing Base:** `/lib/logitech_receiver/settings.py:774-852`
- **MouseGestureUI Class:** `/lib/solaar/ui/rule_conditions.py:517-616`
- **Rule Evaluation:** `/lib/logitech_receiver/diversion.py:1500-1554` (process_notification)

### Related Issues
- GitHub Issue #2211: Continuous gesture processing request

### Dependencies
- No new external dependencies required
- Uses existing GTK3 widgets
- Uses existing YAML serialization

---

**Document Version:** 1.0  
**Last Updated:** 2025-10-16  
**Status:** Planning Phase - Ready for Review
