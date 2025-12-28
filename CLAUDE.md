# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a single-file HTML application for real-time violin intonation analysis. It uses the Web Audio API to capture microphone input, detect pitch via autocorrelation, and visualize both current pitch and historical intonation accuracy on a musical staff and heatmap.

## Running the Application

Open `intonation.html` directly in a modern web browser (Chrome, Firefox, Edge). The browser will request microphone permissions when you click "Start Analyzer".

No build process, dependencies, or server required - this is a standalone HTML file.

## Architecture

### Three Main Components

1. **Pitch Detection Engine** (lines 148-213)
   - Uses Web Audio API `AnalyserNode` with autocorrelation algorithm
   - Implements smoothing via circular buffer (`pitchBuffer`) to reduce jitter
   - Volume gate (`threshold`) prevents noise from triggering detection
   - Converts frequency → MIDI note number → cents offset from equal temperament

2. **Musical Staff Visualization** (lines 103-146)
   - Canvas-based staff notation with treble clef range (G3-B5, MIDI 55-83)
   - Notes plotted left-to-right, wrapping when reaching canvas edge
   - Color-coded by intonation: green (<5 cents), orange (5-15 cents), red (>15 cents)
   - Smart note plotting: Only plots when MIDI changes OR after silence (prevents repeated dots for sustained notes)

3. **Intonation Heatmap** (lines 77-87, 189-202)
   - Right panel showing all 29 notes in playing range
   - Each row displays: note name, diverging bar chart (min/avg/max cents), hit count
   - Time-windowed data (`winDur`) - older data automatically expires
   - Visual feedback: bars show intonation spread, dot shows average offset

### Key State Variables

- `history[midi]`: Object mapping MIDI numbers to arrays of `{t: timestamp, c: cents}` observations
- `lastPlottedMidi`: Tracks last note plotted on staff (prevents repeated plotting)
- `wasSilent`: Flag that triggers new note plotting after silence gap
- `pitchBuffer`: Circular buffer for temporal smoothing of pitch detection
- `staffX`: Current x-position on staff canvas (increments with each note)

### Pitch Detection Flow

1. `render()` loop reads audio buffer via `analyser.getFloatTimeDomainData()`
2. Calculate RMS volume to check against `threshold`
3. If volume sufficient, call `autoCorrelate()` to detect fundamental frequency
4. Add frequency to `pitchBuffer` and compute moving average
5. Convert to MIDI note + cents offset
6. Update history, UI displays, and conditionally plot on staff
7. Prune history entries older than `winDur`

### Critical Behaviors

- **Smart Note Plotting** (lines 173-181): The `wasSilent` flag ensures that holding a note doesn't create multiple dots on the staff, but playing the same note twice with a gap between creates two dots.

- **Autocorrelation** (lines 205-213): YIN-style algorithm that finds period of waveform. Returns -1 if signal is too noisy (correlation < 50% of zero-lag).

- **MIDI to Staff Y Mapping** (lines 68-75): Hardcoded lookup table defining vertical pixel positions for each MIDI note on the canvas. Below middle C (60), ledger lines are drawn dynamically.

## Configuration Controls

Three sliders in header adjust real-time parameters:
- **Gate** (5-100): Volume threshold for pitch detection
- **Smooth** (1-20): Size of pitch averaging buffer
- **History** (1-20 seconds): Time window for heatmap data retention
