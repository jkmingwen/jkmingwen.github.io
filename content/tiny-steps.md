---
title: Tiny Steps
date: 2018-04-30
tags:
  - audio
  - max-msp
  - dsp
  - synthesis
  - fft
description: A live-performance patch in Max/MSP with seven polyphonic instruments, including resonant-filter, sample-and-hold, granular and FFT vocoder synths.
---
**Tiny Steps** is a piece of music and a Max/MSP patch I built to perform it, as the final project for *Music Computing 1* at Goldsmiths. It started as a field recording I made on my walk to the grocery shop. I expected chaotic, overwhelming city noise, but the recording sounded playful and relaxed, and I tried to rebuild that feeling. The track opens and closes with traffic noise run through a granular synthesizer. In between, sounds from the same recording, subtractive synths and modulated drums come in and out, with randomly controlled effects adding the kind of ordered chaos you get in a city. Todd Terje's playful sound design was a big influence. An audio recording of the performance can be found [here](https://drive.google.com/file/d/1c_QQpEm_BgmvGelqfELOG38icLuosvmS/view?usp=sharing).

![[tiny-steps-signal-flow.webp]]

## Structure

The patch has seven instruments. Four are driven by sequencers synced to the global `transport`, and three are played live from a MIDI controller. Each synth voice is a `poly~` abstraction, so notes can overlap without cutting each other off. The arrangement moves through eight `preset` states, which I switch between with the numpad as the piece goes on.

## Sequencer-triggered instruments

- **RESO** (`resopoly~`, 16 voices). A short burst of pink noise drives four `reson~` bandpass filters, tuned to the note and to octaves above and below it, which gives an organ-like tone. The filters' Q and gain scale with the cutoff frequency and the note length. The note length also sets an `adsr~` envelope whose attack and decay are proportional to it. A `trapezoid~` waveform added to the output gives a crude distortion, and a `onepole~` lowpass smooths it. The output can be routed as a lead or a rhythm part, through an LFO-swept `lores~` filter and an optional tremolo. The tremolo has a variable rate and note length, plus an optional random delay of 10–560 ms built with `tapin~`/`tapout~`.
- **SAH** (`sampleandholdpoly~`, 12 voices). This emulates a sample-and-hold synth. A selectable saw, square or sine oscillator runs through a lowpass filter (`filtercoeff~`) whose cutoff jumps to a random note at a fixed rate. The range of the random jumps can be set up to five octaves, and an adjustable slew smooths the transitions between cutoffs.
- **GLITCHY**. A 16-step sequencer plays kick, snare and hi-hat samples through `groove~`, with an optional "glitch chorus" effect.
- **RING**. Snare and hi-hat samples are ring modulated for a metallic texture.

## User-triggered instruments

- **GRAN** (`granularpoly~`, 20 voices). A granular synthesizer. Each grain picks a random start point in the buffer, a random length and a random playback rate (with a floor so grains never drop below half speed), and is windowed with a `trapezoid~` envelope. The traffic recording at the start and end of the piece goes through this.
- **PADS**. Four vocal samples from the recording, each run through a spectral frequency shifter (`fbinshift~` inside `pfft~`, 1024-point FFT, 4× overlap).
- **JACOB** (`vocoderpoly~`, 12 voices). An FFT vocoder played from MIDI. Inside `pfft~` (2048-point FFT, 4× overlap), a looping vocal sample and a sawtooth carrier at the MIDI note's pitch are both converted to polar form with `cartopol~`. Their magnitudes are multiplied, then resynthesised with `poltocar~`.

## Control

- MIDI CC and pad messages are parsed and routed to each instrument's parameters by controller number. `funnel` passes on only the values that changed, so moving one knob doesn't resend every parameter.
- All the instruments share one global transport, which keeps the sequencers in time with each other.

## Built with

- [Max/MSP](https://cycling74.com/products/max), mostly `poly~`, `pfft~` and the MSP filter objects
- A MIDI controller with knobs and trigger pads
