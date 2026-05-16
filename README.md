# Daisy Seed Bass Multieffects Pedal — Project Documentation

---

## Overview

This project aims to develop a fully customizable bass multieffects pedal using the **Daisy Seed** microcontroller platform. The Daisy Seed supports audio development through two primary libraries: **DaisySP**, a DSP (Digital Signal Processing) library, and **LibDaisy**, a hardware abstraction layer [1]. A known limitation of these libraries is that they are not specifically optimized for bass frequencies, which introduces challenges in low-frequency signal processing.

An additional limitation is that neither DaisySP nor LibDaisy include built-in tuner functionality. To address this, a standalone tuner module is being developed separately using **Max/MSP**, a visual programming environment for audio and multimedia applications [2].

---

## Table of Contents

- [Part 1 — Max/MSP Tuner Module](#part-1--maxmsp-tuner-module)
  - [Definitions](#definitions)
  - [Design Requirements](#design-requirements)
  - [Iteration 1 — Initial Pitch Detection Attempts](#iteration-1--initial-pitch-detection-attempts)
  - [Iteration 2 — Revised Pitch Detection Patch](#iteration-2--revised-pitch-detection-patch)
  - [Main Patch and Display](#main-patch-and-display)
- [Part 2 — Daisy Seed DSP Integration](#part-2--daisy-seed-dsp-integration) *(Coming Soon)*
- [Part 3 — Hardware Design](#part-3--hardware-design) *(Coming Soon)*
- [References](#references)

---

## Part 1 — Max/MSP Tuner Module

**Demo Video:** [https://youtu.be/RtGl-JkB-hE?si=WJLJehn5ctu94i5V](https://youtu.be/RtGl-JkB-hE?si=WJLJehn5ctu94i5V)

---

### Definitions

**Fundamental Frequency (f₀)**
The lowest frequency component of a periodic waveform, generally perceived as the pitch of the signal. In a bass guitar signal, this is the note being played. [3]

**MIDI (Musical Instrument Digital Interface)**
A standardized communication protocol for musical instruments and software. MIDI note numbers are integers ranging from 0–127 that encode pitch, where each integer represents one semitone. [4]

**Pitch**
A perceptual property of sound that allows it to be classified as higher or lower relative to other sounds. In music, pitch is defined by the combination of a note name (note class) and an octave number.

> Example: E (note class) + 2 (octave) = **E2** (pitch)

**Cent**
A unit of measure for musical intervals. One octave contains 1200 cents; one semitone contains 100 cents. Cents are used to express fine intonation deviation. [3]

---

### Design Requirements

Develop a Max/MSP patch that:

1. Accepts a live bass guitar audio signal as input.
2. Detects and outputs the fundamental frequency (f₀) in Hz.
3. Identifies the closest equal-tempered pitch.
4. Calculates the deviation in cents from that pitch.
5. Displays all values in a real-time visual interface.

---

### Iteration 1 — Initial Pitch Detection Attempts

Max 9 provides several built-in objects suitable for pitch detection. Two were evaluated in this iteration: `retune~` and `fzero~`.

#### `retune~`

`retune~` is a Max 9 object designed for pitch detection and pitch shifting. It accepts an audio signal and shifts it by a specified number of cents. It provides five outputs: [5]

- Pitch-shifted audio signal
- Detected frequency (Hz, signal)
- Closest MIDI note number (0–11, where 0 = C)
- Deviation in cents from the closest note
- A list containing the closest note and cent deviation as an integer and float, respectively

**Pros:** Natively provides all outputs required by the design specification.

**Cons:** Due to the natural harmonic overtones present in the open strings of a bass guitar, the fundamental frequency detection was highly unreliable for low-frequency notes. For example, the open E string (E1, ≈41.2 Hz) was consistently misidentified.

**Conclusion:** `retune~` is theoretically well-suited for this application but proved unreliable in practice for low bass frequencies.

---

#### `fzero~`

`fzero~` approximates the fundamental frequency of a mono audio input signal. It accepts one mono signal and outputs an approximated f₀ value in Hz as a float. [6]

**Pros:** Low latency; accurate across the full pitch range of the bass guitar, including low frequencies.

**Cons:** Outputs frequency in Hz only. Additional logic is required to map the detected frequency to a note name, octave, and cent deviation.

**Conclusion:** `fzero~` was selected as the pitch detection method for subsequent iterations. Its accuracy comes at the cost of requiring manual implementation of pitch-mapping logic.

---

#### AI-Assisted Prototyping (Claude Sonnet)

To accelerate development of the pitch-mapping logic around `fzero~`, the Claude Sonnet language model was used to generate an initial Max 9 patch. As the author was still developing familiarity with Max 9's object model and syntax, the intent was to use the generated patch as a structural starting point.

The generated patch was able to receive an audio input, but failed to produce correct output due to data type mismatches between objects. Additionally, a `slide` object was applied prematurely in the signal chain, introducing excessive smoothing that degraded pitch accuracy. The model demonstrated limited knowledge of Max 9's specific object behaviors and conventions.

Despite these issues, analyzing the generated patch's failure points provided a useful foundation for the revised implementation in Iteration 2.

---

### Iteration 2 — Revised Pitch Detection Patch

The revised patch addressed the issues from Iteration 1 by simplifying the signal chain and focusing exclusively on accurate f₀ detection and pitch mapping.

#### Signal Flow Overview

1. Audio input → `fzero~` → f₀ in Hz
2. f₀ (Hz) → frequency-to-MIDI conversion → raw MIDI value
3. Raw MIDI value → `round` → nearest integer MIDI note
4. MIDI note → octave calculation + note name lookup → pitch string

---

#### Frequency to MIDI Conversion

The standard formula for converting a frequency *f* (in Hz) to a MIDI note number *m* is: [4]

```
m = 69 + 12 * log2(f / 440)
```

Where MIDI note 69 corresponds to A4 (440 Hz). The result is rounded to the nearest integer to identify the closest equal-tempered pitch.

---

#### Finding the Octave

MIDI note numbers are organized in groups of 12 (one per octave), spanning 0–127. The octave number for a given MIDI note *m* is calculated as:

```
octave = floor(m / 12) - 1
```

**Example — E1 (MIDI note 28):**
```
floor(28 / 12) = floor(2.333) = 2
2 - 1 = 1  →  Octave 1
```

---

#### Finding the Note Name

The note name is determined using the modulo operation, which returns the remainder after dividing the MIDI note number by 12. This remainder maps to the chromatic scale:

| Remainder | Note     |
|-----------|----------|
| 0         | C        |
| 1         | C# / D♭  |
| 2         | D        |
| 3         | D# / E♭  |
| 4         | E        |
| 5         | F        |
| 6         | F# / G♭  |
| 7         | G        |
| 8         | G# / A♭  |
| 9         | A        |
| 10        | A# / B♭  |
| 11        | B        |

**Example — E1 (MIDI note 28):**
```
28 % 12 = 4  →  E
```

Combined with the octave result above: **E + 1 = E1**

---

### Main Patch and Display

With pitch detection established, the main patch was developed to handle the complete signal path and tuner display.

#### Signal Routing

1. Bass audio input → `pitchDetection` sub-patch → closest pitch (note name + octave)
2. Pitch name + original audio signal → `openString_tuner` sub-patch → deviation in cents

---

#### `openString_tuner` Sub-Patch

This sub-patch determines whether the detected pitch corresponds to one of the four open strings of a standard bass guitar, and if so, calculates the intonation deviation in cents relative to that string's reference frequency.

**Open String Reference Frequencies:**

| String | Pitch | Reference Frequency (Hz) |
|--------|-------|--------------------------|
| E      | E1    | 41.20 Hz                 |
| A      | A1    | 55.00 Hz                 |
| D      | D2    | 73.42 Hz                 |
| G      | G2    | 98.00 Hz                 |

**Note Identification via UTF-32 Encoding:**
Max/MSP does not natively support string comparison between symbol or char objects. As a workaround informed by community research [8], the `atoi` object is used to convert each character of the pitch name to its UTF-32 integer value. The note letter and octave digit are converted separately, producing two integers that together uniquely identify the pitch. These integer pairs are then checked against four parallel conditional branches corresponding to the open strings:

| String | Pitch | UTF-32 (Note) | UTF-32 (Octave) |
|--------|-------|---------------|-----------------|
| E      | E1    | 69            | 49              |
| A      | A1    | 65            | 49              |
| D      | D2    | 68            | 50              |
| G      | G2    | 71            | 50              |

> **Design Note:** Checking only note class (ignoring octave) would allow intonation checking for all E, A, D, and G notes across the fretboard, not just open strings. Whether this is useful for the device is an open question and is noted as a candidate for future testing.

**Cents Deviation Formula:**
For each matched open string, the deviation in cents from the reference frequency is calculated as: [3]

```
cents = 1200 * log2(f_input / f_reference)
```

Where `f_reference` is the equal-tempered reference frequency of the open string. The result is rounded for readability before being returned to the main patch.

---

#### Tuner Display

The visual display is built using three Jitter objects within Max 9: [7]

- **`jit.world`** — creates and manages the OpenGL rendering window
- **`jit.gl.text`** — renders the detected pitch name as text in the center of the display
- **`jit.gl.gridshape`** (×2) — renders two circles: a static yellow circle fixed at center, and a blue circle whose horizontal position is driven by the cents deviation value

The cents deviation value is remapped from the range [−100, 100] to [−3, 3] using the `scale` object. This value drives the horizontal position of the blue circle:

| Position Value | Meaning                  |
|----------------|--------------------------|
| −3             | Off-screen left (flat)   |
| 0              | Centered (in tune)       |
| +3             | Off-screen right (sharp) |

The blue circle's blending mode is set to **exclusion**. When it overlaps the yellow circle at position 0, the two colors combine to produce green — signaling that the note is in tune. This follows standard tuner conventions where a centered green indicator means the pitch is correct.

![Tuner Display — E1 in tune](assets/tuner_display.png)
*Figure 1: The tuner display showing pitch E1 in tune. The blue and yellow circles overlap at center, producing a green result via exclusion blending.*

---

## Part 2 — Daisy Seed DSP Integration

*Coming soon.* This section will cover the implementation of bass-optimized audio effects using the DaisySP and LibDaisy libraries on the Daisy Seed hardware platform.

---

## Part 3 — Hardware Design

*Coming soon.* This section will cover the physical hardware design, enclosure, I/O layout, and integration of the Max/MSP tuner module with the Daisy Seed pedal platform.

---

## References

[1] Electro-Smith, "Daisy Seed," *Electro-Smith Documentation*. [Online]. Available: https://electro-smith.com/daisy

[2] Cycling '74, "Max 9 Documentation," *Cycling '74*. [Online]. Available: https://docs.cycling74.com/max8

[3] Wikipedia contributors, "Cent (music) — Use," *Wikipedia, The Free Encyclopedia*. [Online]. Available: https://en.wikipedia.org/wiki/Cent_(music)#Use

[4] Wikipedia contributors, "General MIDI," *Wikipedia, The Free Encyclopedia*. [Online]. Available: https://en.wikipedia.org/wiki/General_MIDI

[5] Cycling '74, "retune~ Object Reference," *Max 9 Documentation*. [Online]. Available: https://docs.cycling74.com/reference/retune~/

[6] Cycling '74, "`fzero~` Object Reference," *Max 9 Documentation*. [Online]. Available: [https://docs.cycling74.com/max8/refpages/fzero~](https://docs.cycling74.com/reference/fzero~/)

[7] Cycling '74, "Jitter Object API Reference," *Max 9 Documentation*. [Online]. Available: https://docs.cycling74.com/apiref/js/jitterobject/

[8] Cycling '74 Community Forums, "String comparison," *Cycling '74 Forums*. [Online]. Available: https://cycling74.com/forums/string-comparison
