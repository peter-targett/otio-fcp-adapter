# FCP XML Transition Support - Test Plan

This document outlines the comprehensive test coverage for transition support in the FCP XML adapter.

## Test Structure

### Unit Tests (TestFcp7XmlElements)

These tests verify individual parsing and building functions work correctly.

#### Reading/Parsing Tests

1. **`test_transition_for_element()`**
   - **Purpose**: Verify basic transition parsing from XML
   - **Validates**:
     - Transition name extraction
     - Transition type mapping (SMPTE_Dissolve)
     - Metadata preservation (alignment, effect parameters)
     - Effect metadata structure
   - **Input**: XML transitionitem with Cross Dissolve effect
   - **Expected**: Transition object with correct properties and metadata

2. **`test_transition_wipe_type_detection()`**
   - **Purpose**: Verify wipe transitions are correctly identified
   - **Validates**:
     - Transition type detection from effect ID
     - Mapping to Custom_Wipe type
   - **Input**: XML with Wipe effect
   - **Expected**: Transition with Custom_Wipe type

3. **`test_transition_fade_type_detection()`**
   - **Purpose**: Verify fade transitions are correctly identified
   - **Validates**:
     - Fade detection from "Dip to Color" or "Fade" effect IDs
     - Mapping to Custom_Fade type
   - **Input**: XML with Dip to Color Dissolve effect
   - **Expected**: Transition with Custom_Fade type

4. **`test_audio_transition_detection()`**
   - **Purpose**: Verify the helper can distinguish audio from video transitions
   - **Validates**:
     - `_is_audio_transition()` function
     - Detection based on mediatype element
     - Detection based on effect ID keywords
   - **Input**: Audio and video transition XML elements
   - **Expected**: True for audio, False for video

5. **`test_transition_cut_point()`**
   - **Purpose**: Verify cut point calculation for different alignment modes
   - **Validates**:
     - center alignment: (start + end) / 2
     - start/start-black alignment: start value
     - end/end-black alignment: end value
   - **Input**: Transition XML with various alignment values
   - **Expected**: Correct RationalTime for cut point

#### Writing/Building Tests

6. **`test_build_transition_item()`**
   - **Purpose**: Verify basic transition XML generation
   - **Validates**:
     - XML structure (transitionitem tag)
     - Start/end values
     - Alignment (center for balanced offsets)
     - Effect element structure
     - Effect ID mapping (SMPTE_Dissolve → Cross Dissolve)
   - **Input**: OTIO Transition with balanced offsets
   - **Expected**: Valid transitionitem XML element

7. **`test_build_transition_item_wipe()`**
   - **Purpose**: Verify wipe transition with custom parameters
   - **Validates**:
     - Custom_Wipe type mapping to Wipe effect ID
     - Custom parameter preservation
     - Effect metadata structure
   - **Input**: OTIO Transition with wipe metadata
   - **Expected**: XML with Wipe effect and parameters

8. **`test_build_transition_item_fade_to_black()`**
   - **Purpose**: Verify fade-out (to black) generation
   - **Validates**:
     - end-black alignment when in_offset is 0
     - Custom_Fade type mapping
   - **Input**: Transition with in_offset=0, out_offset>0
   - **Expected**: XML with end-black alignment and fade effect

9. **`test_build_transition_item_fade_from_black()`**
   - **Purpose**: Verify fade-in (from black) generation
   - **Validates**:
     - start-black alignment when out_offset is 0
   - **Input**: Transition with in_offset>0, out_offset=0
   - **Expected**: XML with start-black alignment

### Integration Tests (AdaptersFcp7XmlTest)

These tests verify the entire read/write pipeline works correctly.

10. **`test_roundtrip_transitions()`**
    - **Purpose**: Full roundtrip test for video transitions
    - **Validates**:
      - Memory → XML string → Memory conversion
      - Multiple transition types in single timeline
      - Transition properties preservation
      - Offsets remain accurate
    - **Timeline Structure**:
      - Clip 1 (100 frames)
      - Cross Dissolve (12 frame in/out)
      - Clip 2 (100 frames)
      - Wipe (15 frame in/out)
      - Clip 3 (100 frames)
    - **Expected**: Identical structure after roundtrip

11. **`test_audio_transition_mediatype()`**
    - **Purpose**: Verify audio transitions have correct mediatype
    - **Validates**:
      - Audio track transitions get mediatype="audio"
      - Effect element contains correct mediatype
      - Roundtrip preserves audio mediatype
    - **Timeline Structure**:
      - Audio track with clips and crossfade
    - **Expected**: mediatype="audio" in generated XML

12. **`test_read_transitions_from_file()`**
    - **Purpose**: Test reading sample file with various transitions
    - **Validates**:
      - Reading from actual FCP XML file
      - Multiple transition types (dissolve, wipe, audio crossfade)
      - Correct item sequence in tracks
      - Audio vs video transition handling
    - **Input File**: `sample_data/transitions_example.xml`
    - **Expected Timeline Structure**:
      - Video track: clip, dissolve, clip, wipe, clip
      - Audio track: clip, crossfade, clip

## Sample Data Files

### transitions_example.xml

A comprehensive test file containing:

- **Video Track**:
  - 3 clips (clip_A.mov, clip_B.mov, clip_C.mov)
  - Cross dissolve between clips A and B (30 frames, center aligned)
  - Wipe transition between clips B and C (30 frames, center aligned, with parameters)

- **Audio Track**:
  - 2 clips (audio_A.wav, audio_B.wav)
  - Audio crossfade between clips (480 samples @ 48kHz, center aligned)

- **Key Features**:
  - Uses `-1` for start/end values on clipped clips
  - Includes custom effect parameters on wipe
  - Audio transition has mediatype="audio"
  - All transitions have proper alignment and rate elements

## Test Execution

### Running All Tests

```bash
# Run entire test suite
python test_fcp7_xml_adapter.py

# Run with verbose output
python -m unittest test_fcp7_xml_adapter -v
```

### Running Specific Test Classes

```bash
# Only utility and element tests
python -m unittest test_fcp7_xml_adapter.TestFcp7XmlUtilities
python -m unittest test_fcp7_xml_adapter.TestFcp7XmlElements

# Only adapter integration tests
python -m unittest test_fcp7_xml_adapter.AdaptersFcp7XmlTest
```

### Running Individual Tests

```bash
# Parsing tests
python -m unittest test_fcp7_xml_adapter.TestFcp7XmlElements.test_transition_for_element
python -m unittest test_fcp7_xml_adapter.TestFcp7XmlElements.test_transition_wipe_type_detection
python -m unittest test_fcp7_xml_adapter.TestFcp7XmlElements.test_audio_transition_detection

# Building tests
python -m unittest test_fcp7_xml_adapter.AdaptersFcp7XmlTest.test_build_transition_item
python -m unittest test_fcp7_xml_adapter.AdaptersFcp7XmlTest.test_build_transition_item_wipe

# Roundtrip tests
python -m unittest test_fcp7_xml_adapter.AdaptersFcp7XmlTest.test_roundtrip_transitions
python -m unittest test_fcp7_xml_adapter.AdaptersFcp7XmlTest.test_audio_transition_mediatype
python -m unittest test_fcp7_xml_adapter.AdaptersFcp7XmlTest.test_read_transitions_from_file
```

## Coverage Matrix

| Feature | Reading | Writing | Roundtrip | Sample File |
|---------|---------|---------|-----------|-------------|
| Cross Dissolve (video) | ✅ | ✅ | ✅ | ✅ |
| Wipe transition | ✅ | ✅ | ✅ | ✅ |
| Fade to/from black | ✅ | ✅ | ✅ | ❌ |
| Audio crossfade | ✅ | ✅ | ✅ | ✅ |
| Center alignment | ✅ | ✅ | ✅ | ✅ |
| Start-black alignment | ✅ | ✅ | ❌ | ❌ |
| End-black alignment | ✅ | ✅ | ❌ | ❌ |
| Custom parameters | ✅ | ✅ | ✅ | ✅ |
| Metadata preservation | ✅ | ✅ | ✅ | ✅ |
| -1 start/end values | ✅ | ✅ | ✅ | ✅ |

## Expected Test Results

All tests should pass with the enhanced transition support. Expected output:

```
test_audio_transition_detection (test_fcp7_xml_adapter.TestFcp7XmlElements) ... ok
test_transition_cut_point (test_fcp7_xml_adapter.TestFcp7XmlElements) ... ok
test_transition_fade_type_detection (test_fcp7_xml_adapter.TestFcp7XmlElements) ... ok
test_transition_for_element (test_fcp7_xml_adapter.TestFcp7XmlElements) ... ok
test_transition_wipe_type_detection (test_fcp7_xml_adapter.TestFcp7XmlElements) ... ok
test_audio_transition_mediatype (test_fcp7_xml_adapter.AdaptersFcp7XmlTest) ... ok
test_build_transition_item (test_fcp7_xml_adapter.AdaptersFcp7XmlTest) ... ok
test_build_transition_item_fade_from_black (test_fcp7_xml_adapter.AdaptersFcp7XmlTest) ... ok
test_build_transition_item_fade_to_black (test_fcp7_xml_adapter.AdaptersFcp7XmlTest) ... ok
test_build_transition_item_wipe (test_fcp7_xml_adapter.AdaptersFcp7XmlTest) ... ok
test_read_transitions_from_file (test_fcp7_xml_adapter.AdaptersFcp7XmlTest) ... ok
test_roundtrip_transitions (test_fcp7_xml_adapter.AdaptersFcp7XmlTest) ... ok

----------------------------------------------------------------------
Ran 12 tests in X.XXXs

OK
```

## Troubleshooting

### Common Test Failures

1. **Metadata mismatch**: If metadata doesn't match exactly, check that:
   - Ignore keys are set correctly in `_xml_tree_to_dict`
   - The `_dict_to_xml_tree` function respects metadata preservation

2. **Timing issues**: If offsets or durations don't match:
   - Verify rate calculations in context
   - Check that rescaling is applied correctly
   - Ensure -1 start/end handling is correct

3. **Missing transitions**: If transitions aren't parsed:
   - Check that `_derefed_element` is called on transitions
   - Verify transition lookahead logic in `track_for_element`

4. **Wrong transition types**: If types don't map correctly:
   - Check the effect ID detection logic
   - Verify the mapping dictionaries in both directions

## Future Test Enhancements

Potential additional tests:

- [ ] Test with drop-frame timecode
- [ ] Test with NTSC rates (29.97, 23.976)
- [ ] Test nested sequences with transitions
- [ ] Test transitions with different rates than track
- [ ] Test edge cases (zero-length transitions, overlapping transitions)
- [ ] Test multi-track synchronized transitions
- [ ] Performance tests with many transitions
- [ ] Test with invalid/malformed transition XML
