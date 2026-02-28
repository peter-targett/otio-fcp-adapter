# FCP XML Transition Support - Quick Reference

Quick reference for working with transitions in the FCP XML adapter.

## Basic Usage

### Read Transitions
```python
import opentimelineio as otio

timeline = otio.adapters.read_from_file("my_edit.xml")
```

### Write Transitions
```python
otio.adapters.write_to_file(timeline, "output.xml")
```

## Creating Transitions

### Cross Dissolve (Video)
```python
from opentimelineio import schema, opentime

dissolve = schema.Transition(
    name="Cross Dissolve",
    transition_type=schema.TransitionTypes.SMPTE_Dissolve,
    in_offset=opentime.RationalTime(15, 24),
    out_offset=opentime.RationalTime(15, 24)
)
```

### Audio Crossfade
```python
crossfade = schema.Transition(
    name="Audio Crossfade",
    transition_type=schema.TransitionTypes.SMPTE_Dissolve,
    in_offset=opentime.RationalTime(480, 48000),
    out_offset=opentime.RationalTime(480, 48000),
    metadata={
        "fcp_xml": {
            "effect": {
                "effecttype": "transition",
                "mediatype": "audio"
            }
        }
    }
)
```

### Wipe
```python
wipe = schema.Transition(
    name="Wipe Right",
    transition_type=schema.TransitionTypes.Custom_Wipe,
    in_offset=opentime.RationalTime(20, 24),
    out_offset=opentime.RationalTime(20, 24),
    metadata={
        "fcp_xml": {
            "effectid": "Wipe",
            "effect": {
                "parameter": {
                    "name": "Direction",
                    "value": "0"
                }
            }
        }
    }
)
```

### Fade to Black
```python
fade_out = schema.Transition(
    name="Fade Out",
    transition_type=schema.TransitionTypes.Custom_Fade,
    in_offset=opentime.RationalTime(0, 24),     # 0 = from cut point
    out_offset=opentime.RationalTime(30, 24)    # 30 frames to black
)
```

### Fade from Black
```python
fade_in = schema.Transition(
    name="Fade In",
    transition_type=schema.TransitionTypes.Custom_Fade,
    in_offset=opentime.RationalTime(30, 24),    # 30 frames from black
    out_offset=opentime.RationalTime(0, 24)     # 0 = to cut point
)
```

## Transition Types

| OTIO Type | FCP Effect ID | Description |
|-----------|---------------|-------------|
| `SMPTE_Dissolve` | "Cross Dissolve" | Standard dissolve |
| `Custom_Wipe` | "Wipe" | Wipe transition |
| `Custom_Fade` | "Dip to Color Dissolve" | Fade to/from color |

## Adding Transitions to Tracks

```python
track = schema.Track(kind=schema.TrackKind.Video)
track.extend([
    clip1,
    transition,
    clip2
])
```

## Inspecting Transitions

```python
for item in track:
    if isinstance(item, schema.Transition):
        print(f"Name: {item.name}")
        print(f"Type: {item.transition_type}")
        print(f"In: {item.in_offset}")
        print(f"Out: {item.out_offset}")
        
        # Check metadata
        if "fcp_xml" in item.metadata:
            effect = item.metadata["fcp_xml"].get("effect", {})
            print(f"Effect ID: {effect.get('effectid')}")
            print(f"Media Type: {effect.get('mediatype')}")
```

## Common Patterns

### Balanced Dissolve (Center Aligned)
```python
# Equal in and out offsets = centered on cut point
in_offset = out_offset = opentime.RationalTime(12, 24)
```

### Fade from Black at Timeline Start
```python
# Place at start of track with in_offset > 0, out_offset = 0
fade_in = schema.Transition(
    name="Fade In",
    transition_type=schema.TransitionTypes.Custom_Fade,
    in_offset=opentime.RationalTime(24, 24),  # 1 second @ 24fps
    out_offset=opentime.RationalTime(0, 24)
)

track.insert(0, fade_in)  # At the beginning
```

### Fade to Black at Timeline End
```python
# Place at end with in_offset = 0, out_offset > 0
fade_out = schema.Transition(
    name="Fade Out",
    transition_type=schema.TransitionTypes.Custom_Fade,
    in_offset=opentime.RationalTime(0, 24),
    out_offset=opentime.RationalTime(24, 24)  # 1 second @ 24fps
)

track.append(fade_out)  # At the end
```

## Metadata Keys

### Top-Level Transition Metadata
```python
transition.metadata["fcp_xml"] = {
    "@id": "transitionitem-1",          # Unique ID
    "alignment": "center",               # start, center, end, start-black, end-black
    "effect": { ... }                    # Effect parameters
}
```

### Effect Metadata
```python
effect_metadata = {
    "effectid": "Cross Dissolve",        # FCP effect identifier
    "effecttype": "transition",          # Always "transition"
    "mediatype": "video",                # "video" or "audio"
    "effectcategory": "Dissolve",        # Category (optional)
    "parameter": [...]                   # Custom parameters (optional)
}
```

## Alignment Modes

| Alignment | In Offset | Out Offset | Use Case |
|-----------|-----------|------------|----------|
| `center` | > 0 | > 0 | Standard transition between clips |
| `start-black` | > 0 | 0 | Fade from black |
| `end-black` | 0 | > 0 | Fade to black |
| `start` | > 0 | > 0* | Transition starts at cut |
| `end` | > 0* | > 0 | Transition ends at cut |

*When manually set in metadata

## Rate Handling

Transitions inherit rate from their context:
```python
# Track rate
track = schema.Track(kind=schema.TrackKind.Video)

# Transition uses same rate as track
transition = schema.Transition(
    name="Dissolve",
    transition_type=schema.TransitionTypes.SMPTE_Dissolve,
    in_offset=opentime.RationalTime(15, 24),   # 24fps
    out_offset=opentime.RationalTime(15, 24)   # 24fps
)
```

## Testing

### Run Transition Tests
```bash
# All transition tests
python -m unittest test_fcp7_xml_adapter.TestFcp7XmlElements.test_transition_*
python -m unittest test_fcp7_xml_adapter.AdaptersFcp7XmlTest.test_*transition*

# Specific test
python -m unittest test_fcp7_xml_adapter.AdaptersFcp7XmlTest.test_roundtrip_transitions
```

### Verify Transition in Output
```python
# After writing
timeline = otio.adapters.read_from_file("output.xml")
transition = timeline.tracks[0][1]  # Assuming transition at index 1

assert isinstance(transition, schema.Transition)
assert transition.name == "Cross Dissolve"
```

## Troubleshooting

### Transition Not Written
**Problem**: Transition missing in output XML

**Solution**:
- Ensure transition is between two clips (not at start/end unless fade)
- Check that `in_offset` and `out_offset` are > 0 (unless fade)
- Verify track contains transition in sequence

### Wrong Alignment
**Problem**: Transition has incorrect alignment

**Solution**:
- For center: Both offsets > 0
- For start-black: `in_offset` > 0, `out_offset` = 0
- For end-black: `in_offset` = 0, `out_offset` > 0
- Or manually set in metadata: `metadata["fcp_xml"]["alignment"] = "center"`

### Audio Transition Shows as Video
**Problem**: Audio transition has `mediatype="video"`

**Solution**:
- Ensure transition is on audio track
- Or explicitly set in metadata:
```python
transition.metadata["fcp_xml"] = {
    "effect": {
        "mediatype": "audio"
    }
}
```

### Timing Off After Roundtrip
**Problem**: Transition timing doesn't match after read/write

**Solution**:
- Check rates match between clips and track
- Use `rescaled_to()` to normalize rates:
```python
in_offset = opentime.RationalTime(15, 24)
normalized = in_offset.rescaled_to(track_rate)
```

### Custom Parameters Lost
**Problem**: Custom effect parameters not preserved

**Solution**:
- Store in effect metadata:
```python
transition.metadata["fcp_xml"]["effect"]["parameter"] = [
    {"name": "Border", "value": "10"},
    {"name": "Direction", "value": "0"}
]
```

## Best Practices

1. **Use Consistent Rates**: Keep transitions at same rate as track
2. **Preserve Metadata**: When reading, keep `fcp_xml` metadata for roundtrip
3. **Set Names**: Give transitions descriptive names
4. **Test Roundtrips**: Always verify your edits can roundtrip correctly
5. **Check Adjacent Clips**: Ensure clips have enough media for overlaps

## Common Errors

### ValueError: Unsupported item
```python
# Missing: Transition not between clips
track = schema.Track()
track.append(transition)  # ❌ No clip before

# Correct:
track.extend([clip1, transition, clip2])  # ✅
```

### Rate Mismatch
```python
# Wrong: Different rates
clip = schema.Clip(source_range=opentime.TimeRange(
    opentime.RationalTime(0, 24), ...
))
transition = schema.Transition(
    in_offset=opentime.RationalTime(15, 30)  # ❌ Different rate
)

# Correct: Same rates
transition = schema.Transition(
    in_offset=opentime.RationalTime(15, 24)  # ✅ Same as clip
)
```

## References

- [TRANSITION_SUPPORT.md](TRANSITION_SUPPORT.md) - Full documentation
- [TRANSITION_TEST_PLAN.md](TRANSITION_TEST_PLAN.md) - Test details
- [TRANSITION_ARCHITECTURE.md](TRANSITION_ARCHITECTURE.md) - Implementation details
- [FCP XML Spec](https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/FinalCutPro_XML/)
