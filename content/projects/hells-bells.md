---
title: Hells Bells
date: 2018-02-13
tags:
  - audio
  - p5js
  - web-audio
  - synthesis
description: Nine bells, alarms and sirens recreated in the browser using additive, AM and FM synthesis only, with no samples.
draft: "true"
---

**Hells Bells** is a browser instrument I built for the *Sound and Signal* course at Goldsmiths. It plays nine bells and alarms, from a gong to a digital alarm clock, each recreated with synthesis only. Samples weren't allowed. The brief asked for five sounds, at least one additive and at least one AM or FM. I did all nine, because the interesting part was working out what makes each one sound like itself: how harmonic it is, where its energy sits in the spectrum, and how fast it decays.

## Approach

Every bell is a bank of ten oscillators. Each oscillator gets its own randomised frequency, amplitude and attack/release envelope, and they're summed in a sample-by-sample audio loop. Several of the bells also have a separate "strike" layer: a very short burst of tones that recreates the hammer or clapper hitting the metal.

## The sounds

| Sound | Technique | What I was going for |
| --- | --- | --- |
| Gong | Additive, random **MIDI** pitches (50–60) | The most inharmonic of the set. Picking random MIDI notes instead of random frequencies stops the partials from bunching up at the top of the range. Also has a loud, low strike layer. |
| Church bell | Additive, random **frequencies** (150–400 Hz) | Slightly more harmonic than the gong, with a softer attack and the longest decay (3–7 s) to suggest a large, resonant bell. |
| Handbell | Additive, partials spread ±20 Hz around 2 kHz | Much more harmonic. The strike layer is pitched an octave above the bell, which sounded more convincing than random noise. |
| Service bell | Additive + low sawtooth strike | Like the handbell, but the attack is more of a thud, so the strike layer uses low saw waves. |
| Bike bell | **AM** at 10 Hz | Amplitude modulation recreates the hammer striking repeatedly. It triggers on both press and release, like a real bike bell. |
| Doorbell | Additive, triangle waves | Two tones: one on press, and one about a fourth lower on release. |
| Alarm bell | Additive + phasor clock | A phasor running at 35 Hz works as a metronome and retriggers the envelopes for as long as the button is held. |
| Siren | **FM**, sawtooth | A slow sine LFO sweeps the pitch up and down, with some random jitter so it sounds hand-cranked. It's held on an ADSR for as long as the button is down. |
| Digital alarm | Triangle waves + step sequencer | The same phasor clock, but stepping through an 8-step on/off pattern to get the classic "four beeps, rest" rhythm. |

## Built with

- [p5.js](https://p5js.org/) for the image-button grid and input
- [Maximilian](https://github.com/micknoise/Maximilian) (maxiLib + maximEx) for oscillators, envelopes and the audio loop
