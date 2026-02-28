# FCP XML Transition Architecture

This document provides a visual overview of how transitions flow through the adapter.

## Reading Flow (FCP XML → OTIO)

```
┌─────────────────────────────────────────────────────────────────┐
│                        FCP XML File                              │
│  <transitionitem>                                                │
│    <start>100</start>                                            │
│    <end>130</end>                                                │
│    <alignment>center</alignment>                                 │
│    <effect>                                                      │
│      <effectid>Cross Dissolve</effectid>                         │
│      <mediatype>video</mediatype>                                │
│    </effect>                                                     │
│  </transitionitem>                                               │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ XML parsing
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│               FCP7XMLParser.item_and_timing_for_element()        │
│  - Detects transition element                                    │
│  - Calls transition_for_element()                                │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│            FCP7XMLParser.transition_for_element()                │
│  1. Extract start, end times                                     │
│  2. Calculate cut_point using _transition_cut_point()            │
│  3. Calculate in_offset = cut_point - start                      │
│  4. Calculate out_offset = end - cut_point                       │
│  5. Detect transition type from effectid                         │
│     - "wipe" → Custom_Wipe                                       │
│     - "fade"/"dip" → Custom_Fade                                 │
│     - default → SMPTE_Dissolve                                   │
│  6. Preserve metadata with _xml_tree_to_dict()                   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    schema.Transition                             │
│  name: "Cross Dissolve"                                          │
│  transition_type: SMPTE_Dissolve                                 │
│  in_offset: RationalTime(15, 24)                                 │
│  out_offset: RationalTime(15, 24)                                │
│  metadata:                                                       │
│    fcp_xml:                                                      │
│      alignment: "center"                                         │
│      effect:                                                     │
│        effectid: "Cross Dissolve"                                │
│        mediatype: "video"                                        │
└─────────────────────────────────────────────────────────────────┘
```

## Writing Flow (OTIO → FCP XML)

```
┌─────────────────────────────────────────────────────────────────┐
│                    schema.Transition                             │
│  name: "Cross Dissolve"                                          │
│  transition_type: SMPTE_Dissolve                                 │
│  in_offset: RationalTime(15, 24)                                 │
│  out_offset: RationalTime(15, 24)                                │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│            _build_top_level_track()                              │
│  1. Iterate through track items                                  │
│  2. Get timeline_range for item                                  │
│  3. Calculate transition_offsets from neighbors                  │
│  4. Call _build_item()                                           │
│  5. If audio track + transition:                                 │
│     - Set effect/mediatype to "audio"                            │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   _build_item()                                  │
│  - Detects item is Transition                                    │
│  - Calls _build_transition_item()                                │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              _build_transition_item()                            │
│  1. Create transitionitem element                                │
│  2. Add start/end from timeline_range                            │
│  3. Determine alignment:                                         │
│     - in_offset == 0 → "start-black"                             │
│     - out_offset == 0 → "end-black"                              │
│     - both > 0 → "center"                                        │
│  4. Add rate element                                             │
│  5. Build effect element:                                        │
│     - Get effectid from metadata or map type:                    │
│       * SMPTE_Dissolve → "Cross Dissolve"                        │
│       * Custom_Wipe → "Wipe"                                     │
│       * Custom_Fade → "Dip to Color Dissolve"                    │
│     - Add effecttype="transition"                                │
│     - Add mediatype (video/audio)                                │
│     - Add custom parameters from metadata                        │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ XML generation
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        FCP XML Output                            │
│  <transitionitem>                                                │
│    <start>100</start>                                            │
│    <end>130</end>                                                │
│    <alignment>center</alignment>                                 │
│    <rate>                                                        │
│      <timebase>24</timebase>                                     │
│      <ntsc>FALSE</ntsc>                                          │
│    </rate>                                                       │
│    <effect>                                                      │
│      <name>Cross Dissolve</name>                                 │
│      <effectid>Cross Dissolve</effectid>                         │
│      <effecttype>transition</effecttype>                         │
│      <mediatype>video</mediatype>                                │
│    </effect>                                                     │
│  </transitionitem>                                               │
└─────────────────────────────────────────────────────────────────┘
```

## Timing Calculation

### Reading: XML → OTIO

```
FCP XML Transition:
├─ start: 100
├─ end: 130
└─ alignment: center

         Cut Point = (100 + 130) / 2 = 115

Timeline: ─────────[─────────────────────]─────────
               start=100            end=130
                    │                 │
                    │←── in_offset ──→│←── out_offset ──→
                    │     (15)        cut      (15)
                    │                (115)

OTIO Transition:
├─ in_offset: 115 - 100 = 15
└─ out_offset: 130 - 115 = 15
```

### Writing: OTIO → XML

```
OTIO Transition:
├─ in_offset: 15
└─ out_offset: 15

Timeline range at index:
├─ start_time: 100
└─ duration: 30

FCP XML:
├─ start: timeline_range.start_time = 100
├─ end: timeline_range.end_time_exclusive() = 130
└─ alignment: "center" (because both offsets > 0)
```

## Adjacent Clip Handling

### With Transition Between Clips

```
                        Timeline
    ┌────────────────────────────────────────────────────┐
    │                                                    │
    │  Clip A          Transition        Clip B          │
    │ ┌─────────┐     ┌─────────┐     ┌─────────┐      │
    │ │         │     │         │     │         │      │
    │ │         │     │         │     │         │      │
    │ │         │     │         │     │         │      │
    │ └─────────┘     └─────────┘     └─────────┘      │
    │  start=0         start=85        start=100        │
    │  end=100         end=115         end=200          │
    │                                                    │
    └────────────────────────────────────────────────────┘
         │                 │                │
         │                 │                │
    source_in=0        in_offset=15     source_in=0
    source_out=100     out_offset=15    source_out=100
         │                 │                │
         ▼                 ▼                ▼

FCP XML Representation:
    <clipitem>
        <start>0</start>
        <end>-1</end>              ← -1 because tail transition exists
        <in>0</in>
        <out>100</out>              ← Extended by transition.in_offset
    </clipitem>
    
    <transitionitem>
        <start>85</start>
        <end>115</end>
        <alignment>center</alignment>
    </transitionitem>
    
    <clipitem>
        <start>-1</start>           ← -1 because head transition exists
        <end>200</end>
        <in>0</in>                  ← Adjusted by transition.in_offset
        <out>100</out>              ← Extended by transition.out_offset
    </clipitem>
```

## Transition Type Detection

### Reading: Effect ID → OTIO Type

```
┌──────────────────────────┬────────────────────────┐
│ FCP Effect ID            │ OTIO Transition Type   │
├──────────────────────────┼────────────────────────┤
│ "Cross Dissolve"         │ SMPTE_Dissolve         │
│ "Cross Fade (0 dB)"      │ SMPTE_Dissolve         │
├──────────────────────────┼────────────────────────┤
│ Contains "wipe"          │ Custom_Wipe            │
│ "Push"                   │ Custom_Wipe            │
│ "Slide"                  │ Custom_Wipe            │
├──────────────────────────┼────────────────────────┤
│ Contains "fade"          │ Custom_Fade            │
│ Contains "dip"           │ Custom_Fade            │
│ "Dip to Color Dissolve"  │ Custom_Fade            │
├──────────────────────────┼────────────────────────┤
│ Other                    │ SMPTE_Dissolve         │
└──────────────────────────┴────────────────────────┘
```

### Writing: OTIO Type → Effect ID

```
┌────────────────────────┬──────────────────────────┐
│ OTIO Transition Type   │ FCP Effect ID            │
├────────────────────────┼──────────────────────────┤
│ SMPTE_Dissolve         │ "Cross Dissolve"         │
│ Custom_Wipe            │ "Wipe"                   │
│ Custom_Fade            │ "Dip to Color Dissolve"  │
└────────────────────────┴──────────────────────────┘

Note: Metadata can override with custom effectid
```

## Alignment Modes

```
CENTER ALIGNED (default)
─────────[transition]─────────
     ^        ^        ^
  start    cut_point  end
     │        │        │
     ├──in_offset──────┤
              ├──out_offset──┤


START-BLACK (fade from black)
─────────[transition]─────────
     ^                 ^
  start=cut_point     end
     │                 │
     ├──in_offset=0────┤
                  ├──out_offset──┤


END-BLACK (fade to black)
─────────[transition]─────────
     ^                 ^
   start          end=cut_point
     │                 │
     ├──in_offset──────┤
                  ├──out_offset=0─┤


START ALIGNED (not start-black)
─────────[transition]─────────
     ^   ^             ^
 start cut_point      end
     │   │             │
     ├─in─┤            │
         ├──out_offset──────┤


END ALIGNED (not end-black)
─────────[transition]─────────
     ^             ^   ^
   start      cut_point end
     │             │   │
     ├──in_offset──────┤
                   ├out┤
```

## Media Type Handling

```
┌─────────────────────────────────────────────────────────┐
│                  Track Processing                        │
└─────────────────────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌──────────────┐        ┌──────────────┐
│ Video Track  │        │ Audio Track  │
└──────────────┘        └──────────────┘
        │                       │
        ├─ Clip ─────────────┬─├─ Clip
        ├─ Transition        │ ├─ Transition
        │   mediatype:       │ │   mediatype:
        │   "video" ◄────────┤ │   "audio" ◄───┐
        ├─ Clip              │ ├─ Clip         │
        │                    │ │                │
        └────────────────────┘ └────────────────┘
                                        │
                            Auto-set in _build_top_level_track()
```

## Metadata Preservation

```
┌─────────────────────────────────────────────────────────┐
│                   Reading Phase                          │
└─────────────────────────────────────────────────────────┘
                    │
                    ▼
        _xml_tree_to_dict(transition_element)
                    │
                    ├─ Ignores: "name", "start", "end", 
                    │           "alignment", "rate", "effect"
                    │
                    ├─ Converts effect subtree separately
                    │
                    ▼
        transition.metadata["fcp_xml"] = {
            "alignment": "center",
            "@id": "transitionitem-1",
            "effect": {
                "effectid": "Cross Dissolve",
                "effecttype": "transition",
                "mediatype": "video",
                "wipecode": "0",
                ...
            }
        }

┌─────────────────────────────────────────────────────────┐
│                   Writing Phase                          │
└─────────────────────────────────────────────────────────┘
                    │
                    ▼
        _element_with_item_metadata("transitionitem", transition)
                    │
                    ├─ Creates base element from metadata
                    │
                    ▼
        _build_transition_item()
                    │
                    ├─ Adds required elements (start, end, rate)
                    │
                    ├─ Checks for preserved effect metadata
                    │   └─ If exists: Use parameters from metadata
                    │   └─ If not: Generate from transition_type
                    │
                    ▼
        Final XML with preserved + computed elements
```

## Key Functions Reference

| Function | Purpose | Location |
|----------|---------|----------|
| `transition_for_element()` | Parse XML to OTIO Transition | fcp_xml.py:~1120 |
| `_build_transition_item()` | Build XML from OTIO Transition | fcp_xml.py:~1310 |
| `_transition_cut_point()` | Calculate cut point from alignment | fcp_xml.py:~452 |
| `_is_audio_transition()` | Detect audio vs video transition | fcp_xml.py:~475 |
| `_build_top_level_track()` | Process track with transition handling | fcp_xml.py:~1530 |

## Testing Architecture

```
┌────────────────────────────────────────────────────────────┐
│              Test Suite Organization                        │
└────────────────────────────────────────────────────────────┘
                    │
        ┌───────────┴────────────┐
        ▼                        ▼
┌──────────────────┐    ┌──────────────────┐
│   Unit Tests     │    │ Integration Tests│
│ (Elements)       │    │ (Full Adapter)   │
└──────────────────┘    └──────────────────┘
        │                        │
        ├─ Parsing               ├─ Roundtrip
        │  ├─ Dissolve           │  ├─ Video transitions
        │  ├─ Wipe               │  └─ Audio transitions
        │  ├─ Fade               │
        │  └─ Audio              ├─ File I/O
        │                        │  └─ Read sample file
        ├─ Building              │
        │  ├─ Basic              └─ Audio mediatype
        │  ├─ With params           verification
        │  ├─ Fade out
        │  └─ Fade in
        │
        └─ Helpers
           ├─ Cut point
           └─ Audio detection
```

This architecture ensures comprehensive coverage at both the element level and the full adapter level.
