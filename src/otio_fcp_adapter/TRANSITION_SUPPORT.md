# FCP XML Transition Support Improvements

This document describes the enhanced transition support added to the FCP XML adapter for OpenTimelineIO.

## Quick Start

```python
import opentimelineio as otio

# Read a timeline with transitions
timeline = otio.adapters.read_from_file("my_edit.xml")

# Transitions are now fully supported!
for track in timeline.tracks:
    for item in track:
        if isinstance(item, otio.schema.Transition):
            print(f"Found: {item.name}")
```

See [TRANSITION_QUICK_REFERENCE.md](TRANSITION_QUICK_REFERENCE.md) for more examples.

## Overview

The adapter now has comprehensive support for reading and writing transitions in Final Cut Pro 7 XML format, including:

1. **Multiple transition types** (dissolves, wipes, fades)
2. **Audio and video transitions** with proper mediatype handling
3. **Metadata preservation** for roundtripping
4. **Effect parameter support** for complex transitions
5. **Proper alignment handling** (start, center, end, start-black, end-black)

## Reading Transitions

### Enhanced `transition_for_element()` Method

The improved method now:

- **Detects transition types** from effect IDs (wipe, fade, dissolve)
- **Preserves all metadata** for accurate roundtripping
- **Stores effect parameters** in the transition's metadata
- **Maps FCP effect IDs** to OTIO transition types

### Supported Transition Types

| FCP Effect ID | OTIO Transition Type |
|--------------|---------------------|
| Cross Dissolve | SMPTE_Dissolve |
| Wipe, Push, Slide | Custom_Wipe |
| Dip to Color, Fade In/Out | Custom_Fade |
| Others | SMPTE_Dissolve (default) |

## Writing Transitions

### Enhanced `_build_transition_item()` Function

The improved function now:

- **Preserves effect metadata** from input files
- **Maps OTIO transition types** back to FCP effect IDs
- **Handles audio vs video** transitions properly
- **Supports custom effect parameters** from metadata
- **Maintains alignment settings** (center, start-black, end-black)

### Audio Transition Support

The adapter now properly handles audio crossfades:

- Automatically sets `mediatype` to "audio" for transitions on audio tracks
- Uses `_is_audio_transition()` helper to detect audio transitions when reading
- Ensures audio transitions have appropriate effect IDs

## Usage Examples

### Creating a Custom Transition in Python

```python
import opentimelineio as otio
from opentimelineio import opentime

# Create a cross dissolve transition
transition = otio.schema.Transition(
    name="My Dissolve",
    transition_type=otio.schema.TransitionTypes.SMPTE_Dissolve,
    in_offset=opentime.RationalTime(15, 24),  # 15 frames in
    out_offset=opentime.RationalTime(15, 24),  # 15 frames out
)

# Add custom effect metadata for roundtripping
transition.metadata["fcp_xml"] = {
    "effectid": "Cross Dissolve",
    "effect": {
        "effecttype": "transition",
        "mediatype": "video",
        # Add custom parameters here
        "reverse": "FALSE",
    }
}
```

### Creating a Wipe Transition

```python
wipe = otio.schema.Transition(
    name="Wipe Right",
    transition_type=otio.schema.TransitionTypes.Custom_Wipe,
    in_offset=opentime.RationalTime(20, 24),
    out_offset=opentime.RationalTime(20, 24),
    metadata={
        "fcp_xml": {
            "effectid": "Wipe",
            "effect": {
                "parameter": [
                    {"name": "Direction", "value": "0"},  # Right
                    {"name": "Border", "value": "0"},
                ]
            }
        }
    }
)
```

### Creating an Audio Crossfade

```python
audio_fade = otio.schema.Transition(
    name="Audio Crossfade",
    transition_type=otio.schema.TransitionTypes.SMPTE_Dissolve,
    in_offset=opentime.RationalTime(12, 48000),
    out_offset=opentime.RationalTime(12, 48000),
    metadata={
        "fcp_xml": {
            "effectid": "Cross Fade (0 dB)",
            "effect": {
                "effecttype": "transition",
                "mediatype": "audio",
            }
        }
    }
)
```

## Alignment Modes

The adapter supports all FCP XML alignment modes:

- **center**: Transition is centered on the cut point (default)
- **start**: Transition starts at the cut point
- **end**: Transition ends at the cut point
- **start-black**: Fade from black at start
- **end-black**: Fade to black at end

Alignment is automatically determined based on `in_offset` and `out_offset`:
- If `in_offset` is 0: uses "start-black"
- If `out_offset` is 0: uses "end-black"
- Otherwise: uses "center"

## Technical Details

### The -1 Start/End Convention

FCP XML uses `-1` for clip start/end times when a transition affects that boundary. The adapter:

1. **When reading**: Detects `-1` values and calculates actual timing from the transition's cut point
2. **When writing**: Sets start/end to `-1` when adjacent transitions have non-zero offsets
3. **Adjusts source ranges**: Properly offsets clip in/out points to account for transition overlap

### Transition Timing Calculations

```python
# For a centered transition:
cut_point = (start + end) / 2
in_offset = cut_point - start
out_offset = end - cut_point

# For clips adjacent to transitions:
if has_head_transition:
    clip_start = -1
    source_in -= transition.in_offset
    
if has_tail_transition:
    clip_end = -1
    source_out += transition.out_offset
```

## Metadata Preservation

All transition metadata is preserved in the `fcp_xml` namespace:

```python
transition.metadata["fcp_xml"] = {
    "@id": "transitionitem-42",  # Original ID
    "alignment": "center",
    "effect": {
        "effectid": "Cross Dissolve",
        "effecttype": "transition",
        "mediatype": "video",
        "effectcategory": "dissolve",
        "parameter": [...],  # Custom parameters
    }
}
```

## Testing

### Manual Testing

To test transition support manually:

```python
import opentimelineio as otio

# Read an FCP XML file with transitions
timeline = otio.adapters.read_from_file("timeline_with_transitions.xml")

# Examine transitions
for track in timeline.tracks:
    for item in track:
        if isinstance(item, otio.schema.Transition):
            print(f"Transition: {item.name}")
            print(f"  Type: {item.transition_type}")
            print(f"  In offset: {item.in_offset}")
            print(f"  Out offset: {item.out_offset}")
            print(f"  Metadata: {item.metadata.get('fcp_xml', {})}")

# Write back to FCP XML
otio.adapters.write_to_file(timeline, "output.xml")
```

### Automated Test Suite

The adapter includes comprehensive test coverage for transitions in `test_fcp7_xml_adapter.py`:

#### Unit Tests

**Element Parsing Tests:**
- `test_transition_for_element()` - Basic transition parsing
- `test_transition_wipe_type_detection()` - Wipe transition type detection
- `test_transition_fade_type_detection()` - Fade transition type detection
- `test_audio_transition_detection()` - Audio vs video transition detection
- `test_transition_cut_point()` - Transition cut point calculation with different alignments

**Element Building Tests:**
- `test_build_transition_item()` - Basic transition XML generation
- `test_build_transition_item_wipe()` - Wipe transition with metadata
- `test_build_transition_item_fade_to_black()` - Fade out (end-black alignment)
- `test_build_transition_item_fade_from_black()` - Fade in (start-black alignment)

#### Integration Tests

**Roundtrip Tests:**
- `test_roundtrip_transitions()` - Full roundtrip of video transitions (memory → XML → memory)
- `test_audio_transition_mediatype()` - Audio transition mediatype preservation
- `test_read_transitions_from_file()` - Reading sample file with multiple transition types

**Sample Files:**
- `sample_data/transitions_example.xml` - Sample FCP XML with video dissolves, wipes, and audio crossfades

#### Running the Tests

```bash
# Run all FCP XML adapter tests
python test_fcp7_xml_adapter.py

# Run only transition-related tests
python -m unittest test_fcp7_xml_adapter.TestFcp7XmlElements.test_transition_for_element
python -m unittest test_fcp7_xml_adapter.TestFcp7XmlElements.test_transition_wipe_type_detection
python -m unittest test_fcp7_xml_adapter.AdaptersFcp7XmlTest.test_roundtrip_transitions

# Run with verbose output
python -m unittest test_fcp7_xml_adapter -v
```

#### Test Coverage

The test suite verifies:

✅ **Reading transitions:**
- Cross dissolves, wipes, and fades
- Audio and video transitions
- Different alignment modes (center, start-black, end-black)
- Metadata preservation
- Effect parameters

✅ **Writing transitions:**
- Transition type mapping to FCP effect IDs
- Offset calculation and -1 start/end values
- Audio vs video mediatype
- Custom effect parameters
- Alignment determination from offsets

✅ **Roundtripping:**
- Full timeline with multiple transitions
- Metadata preservation across read/write cycles
- Audio and video tracks separately
- Mixed transition types in single sequence

## Known Limitations

1. **Custom parameters**: While the adapter preserves custom effect parameters through metadata, it doesn't interpret or validate them
2. **Complex transitions**: Multi-track or compound transitions may require additional metadata
3. **Transition types**: Only common transition types are mapped; others default to SMPTE_Dissolve

## Future Enhancements

Potential improvements for future versions:

- [ ] Support for custom transition effects with full parameter interpretation
- [ ] Better handling of multi-track synchronized transitions
- [ ] Validation of transition timing constraints
- [ ] Support for motion path transitions
- [ ] Border and texture parameters for wipes
