# Mouse Gesture Staggering - Quick Reference

## What Is This?

This PR adds planning for a "staggering" feature to Solaar's Mouse Gestures, allowing rules to trigger repeatedly at fixed distance intervals while the gesture button is held and moving.

**Example Use Case:** Hold gesture button + drag up = volume increases continuously (not just once)

## Problem

**Current Behavior:** Mouse gesture rules trigger ONCE after releasing the gesture button.

```
User: Press button → Drag up 200px → Release button
Solaar: [accumulate movements] → [ONE notification] → [trigger action ONCE]
```

**Why:** `MouseGesturesXY.move_action()` only accumulates dx/dy. Notification sent only in `release_action()`.

## Solution

**New Behavior:** With staggering enabled, trigger every N pixels during drag.

```
User: Press button → Drag up 200px → Release button
Solaar: [accumulate] → [notify at 50px] → [action] → [notify at 100px] → [action] → etc.
```

**How:** Send incremental notifications during `move_action()`, track distance in `evaluate()`.

## Files Changed

### 1. `lib/logitech_receiver/diversion.py` (Lines 1031-1084)
**Class:** `MouseGesture`

**Changes:**
- Add parameters: `self.staggering` (bool), `self.stagger_distance` (int)
- Support dict input: `{"movements": [...], "staggering": True, "distance": 50}`
- Split `evaluate()` into staggering and batch modes
- Track accumulated distance per gesture

**Key Methods:**
- `__init__()` - Parse staggering params
- `evaluate()` - Choose mode based on notification type and settings
- `_evaluate_staggering()` - New method for distance tracking
- `_evaluate_batch()` - Existing logic for complete gestures

### 2. `lib/logitech_receiver/settings_templates.py` (Lines 819-884)
**Class:** `MouseGesturesXY`

**Changes:**
- Modify `move_action()` to send incremental notifications
- Use marker `-1` for incremental, `0` for complete
- Continue accumulating for batch gestures (backward compatibility)

**Key Methods:**
- `move_action()` - Add incremental notification generation
- `release_action()` - Clear stagger accumulators

### 3. `lib/solaar/ui/rule_conditions.py` (Lines 517-616)
**Class:** `MouseGestureUI`

**Changes:**
- Add checkbox: "Enable Staggering"
- Add spinner: "Stagger Distance" (10-500 pixels, default 50)
- Update `show()` to load staggering state
- Update `collect_value()` to return dict format when enabled

**New Widgets:**
- `self.staggering_checkbox` - Gtk.CheckButton
- `self.stagger_distance_field` - Gtk.SpinButton
- `self.stagger_distance_label` - Gtk.Label

## Data Formats

### Legacy (List)
```python
movements = ["Mouse Up"]
# Serialized: {"MouseGesture": ["Mouse Up"]}
```

### New (Dict with Staggering)
```python
movements = {
    "movements": ["Mouse Up"],
    "staggering": True,
    "distance": 50
}
# Serialized: {"MouseGesture": {...}}
```

### Notification Formats

**Incremental (during movement):**
```python
data = [key_code, -1, dx, dy]  # -1 = incremental marker
```

**Complete (on release):**
```python
data = [key_code, 0, x1, y1, 0, x2, y2, ...]  # 0 = movement marker
```

## Implementation Checklist

### Phase 1: Core Logic (Week 1)
- [ ] Add `staggering` and `stagger_distance` to MouseGesture.__init__
- [ ] Implement dict format parsing with backward compatibility
- [ ] Create `_evaluate_staggering()` method
- [ ] Create `_evaluate_batch()` method (refactor existing)
- [ ] Add global `_stagger_accumulators` dict
- [ ] Implement `_get_accumulator_key()` helper
- [ ] Implement `_calculate_directional_distance()` helper
- [ ] Write unit tests for evaluation logic

### Phase 2: Notification Generation (Week 2)
- [ ] Modify MouseGesturesXY.move_action() to send incremental notifications
- [ ] Add `-1` marker for incremental vs `0` for complete
- [ ] Implement rate limiting (50 Hz max)
- [ ] Add accumulator cleanup in release_action()
- [ ] Test notification generation flow

### Phase 3: UI Implementation (Week 3)
- [ ] Create staggering checkbox in MouseGestureUI
- [ ] Create distance spinner with 10-500 range
- [ ] Implement `_on_staggering_toggled()` handler
- [ ] Update `show()` to load staggering params
- [ ] Update `collect_value()` to return dict format
- [ ] Add tooltips and help text
- [ ] Test UI interaction and validation

### Phase 4: Integration & Polish (Week 4)
- [ ] End-to-end testing with real hardware
- [ ] Performance benchmarking (CPU, memory)
- [ ] Edge case testing (see list below)
- [ ] Update user documentation
- [ ] Create example rule configurations
- [ ] Final code review and cleanup

## Testing Checklist

### Unit Tests
- [ ] MouseGesture initialization with dict format
- [ ] MouseGesture initialization with list format (backward compat)
- [ ] Distance accumulation logic
- [ ] Directional filtering (only target direction counts)
- [ ] Threshold triggering with remainder handling
- [ ] Accumulator cleanup on button release
- [ ] Negative movement (opposite direction) ignored
- [ ] Zero movement handling
- [ ] Data serialization (list and dict formats)

### Integration Tests
- [ ] Complete flow: press → move → trigger → move → trigger → release
- [ ] Multiple simultaneous staggering gestures
- [ ] Batch + staggering rules active simultaneously
- [ ] Accumulator persistence across movements
- [ ] Notification type detection (incremental vs complete)
- [ ] Rate limiting verification

### Manual Tests
- [ ] Volume control while dragging up/down
- [ ] Horizontal scrolling with staggering
- [ ] Diagonal movements
- [ ] Direction changes mid-gesture
- [ ] Rapid back-and-forth movements
- [ ] Very slow movements (edge case)
- [ ] Configuration save/load/reload
- [ ] UI widget state persistence

### Edge Cases
- [ ] Zero movements (immediate press/release)
- [ ] Movements in opposite direction
- [ ] Diagonal movements with cardinal direction target
- [ ] Device disconnect during gesture
- [ ] Multiple rules with different stagger distances
- [ ] Very large stagger distances (>500px)
- [ ] Very small stagger distances (<10px)
- [ ] Simultaneous button presses
- [ ] Key events during gesture

## Code Locations Quick Reference

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| MouseGesture class | diversion.py | 1031-1084 | Gesture matching logic |
| MouseGesturesXY class | settings_templates.py | 819-884 | Notification generation |
| MouseGestureUI class | rule_conditions.py | 517-616 | UI configuration |
| RawXYProcessing base | settings.py | 774-852 | Base XY handling |
| process_notification() | diversion.py | 1506-1554 | Rule evaluation entry |

## Key Design Decisions

### 1. Marker Value for Incremental Notifications
**Decision:** Use `-1` as marker for incremental notifications  
**Rationale:** `0` already used for movement, `1` for key press. `-1` is unused and clearly distinct.

### 2. Data Format for Staggering Config
**Decision:** Dict format `{"movements": [...], "staggering": True, "distance": 50}`  
**Rationale:** Extensible, backward compatible (support both dict and list), clear semantics.

### 3. Accumulator Cleanup Strategy
**Decision:** Clear on button release  
**Rationale:** Simple, predictable, prevents memory leaks. Periodic cleanup for stale entries.

### 4. Distance Calculation Method
**Decision:** Vector projection onto target direction  
**Rationale:** Accurate for all directions, handles diagonals correctly, ignores perpendicular movement.

### 5. Notification Rate Limiting
**Decision:** Throttle to 50 Hz (20ms minimum interval)  
**Rationale:** Balance responsiveness vs performance. Hardware sends ~100 Hz, halving is acceptable.

### 6. Default Stagger Distance
**Decision:** 50 pixels  
**Rationale:** Based on typical DPI (1000) and comfortable hand movement. User can adjust 10-500.

## Performance Characteristics

### Current System
- **Notifications:** 1 per button press/release cycle
- **Evaluation:** 1 rule evaluation per gesture
- **Memory:** Minimal (no state tracking)

### With Staggering
- **Notifications:** Up to 50 per second (throttled)
- **Evaluation:** Multiple evaluations per gesture
- **Memory:** ~50 bytes per active staggering gesture
- **CPU Impact:** Estimated <2% increase (needs benchmarking)

### Optimization Opportunities
1. **Batch incremental notifications** - Accumulate several before sending
2. **Smart throttling** - Higher rate for fast movements, lower for slow
3. **Accumulator cleanup** - Periodic sweep for stale entries
4. **Direction caching** - Cache vector projections for common directions

## FAQ

**Q: Will this break existing rules?**  
A: No. Existing rules use list format and ignore incremental notifications.

**Q: Can I use staggering with multi-movement gestures?**  
A: Initially, staggering works best with single-direction gestures. Complex sequences may come later.

**Q: What happens if I change direction mid-gesture?**  
A: Only movement in the target direction counts. Perpendicular movement is ignored.

**Q: Can multiple rules have different stagger distances?**  
A: Yes. Each rule maintains its own accumulator.

**Q: What's the minimum/maximum stagger distance?**  
A: Minimum 10 pixels, maximum 500 pixels (UI enforced).

**Q: Does staggering work with all actions?**  
A: Yes. Any action (KeyPress, MouseScroll, etc.) can be triggered by staggering.

**Q: How do I disable staggering?**  
A: Uncheck "Enable Staggering" in the UI, or use list format in YAML.

## Example Configurations

### Volume Control
```yaml
---
- MouseGesture:
    movements: [Mouse Up]
    staggering: true
    distance: 40
- KeyPress: [XF86_AudioRaiseVolume, click]
---
- MouseGesture:
    movements: [Mouse Down]
    staggering: true
    distance: 40
- KeyPress: [XF86_AudioLowerVolume, click]
---
```

### Smooth Scrolling
```yaml
---
- MouseGesture:
    movements: [Mouse Up]
    staggering: true
    distance: 30
- MouseScroll: [0, 3]  # Scroll up 3 units
---
- MouseGesture:
    movements: [Mouse Down]
    staggering: true
    distance: 30
- MouseScroll: [0, -3]  # Scroll down 3 units
---
```

### Zoom Control
```yaml
---
- MouseGesture:
    movements: [Mouse Up]
    staggering: true
    distance: 60
- KeyPress: [[Control_L, plus], click]  # Zoom in
---
- MouseGesture:
    movements: [Mouse Down]
    staggering: true
    distance: 60
- KeyPress: [[Control_L, minus], click]  # Zoom out
---
```

**Note:** The gesture button itself (e.g., side button, back button) is configured separately in Solaar's device settings under "Key/Button Diversion" and set to "Mouse Gestures" mode. The MouseGesture rule only specifies the direction(s) to detect.

## Troubleshooting

### Action triggers too often
- Increase stagger distance (e.g., 40 → 60)
- Check if multiple rules are matching

### Action triggers too rarely
- Decrease stagger distance (e.g., 60 → 40)
- Ensure movement direction matches gesture direction

### Action doesn't trigger at all
- Verify staggering checkbox is enabled
- Check gesture button is configured for "Mouse Gestures" mode
- Ensure movement direction is correct

### Performance issues
- Check notification rate (should be throttled to 50 Hz)
- Reduce number of active staggering rules
- Increase stagger distance to reduce trigger frequency

## Resources

- **Full Implementation Plan:** `STAGGERING_IMPLEMENTATION_PLAN.md`
- **Architecture Diagrams:** `ARCHITECTURE_DIAGRAM.md`
- **Code Comments:** Search for "TODO" and "IMPLEMENTATION PLAN" in modified files
- **Related Issue:** GitHub Issue #2211

## Contact

For questions or clarifications about this implementation plan, refer to:
- GitHub PR discussion thread
- Issue #2211 comments
- Implementation plan documents in this repository

---

**Status:** Planning complete, ready for implementation  
**Next Step:** Review and approve design before coding  
**Estimated Timeline:** 4 weeks for full implementation
