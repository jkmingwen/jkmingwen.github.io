---
title: Space Launch
date: 2018-03-08
tags:
  - audio
  - p5js
  - web-audio
  - synthesis
  - sound-design
description: A fully synthesized soundtrack, timed to a slideshow of a shuttle launch, built from filtered noise, oscillators and envelopes.
draft: "true"
---

%% TODO: the app was hosted on igor.gold.ac.uk, which is probably gone. Consider re-hosting it or adding a screen recording. %%

**Space Launch** is a short piece of sound design I made for the *Sound and Signal* course at Goldsmiths. A slideshow takes a shuttle from the launch pad, through lift-off and booster separation, into orbit and out on a spacewalk. The brief was to score it with synthesis only (samples were banned) and to make something change with every image.

## Sound design

Each layer is a separate synthesized voice with its own envelope. They're summed in one audio loop, and the slideshow's image index triggers, releases and retunes them as the pictures change.

- **Atmosphere (wind).** White noise through a resonant lowpass filter. Two slow sine LFOs sweep the cutoff (40 Hz–2 kHz) and the resonance independently.
- **Ignition.** White noise through a bandpass filter. The centre frequency steps up with each ignition image (20 → 100 → 180 → 330 → 480 Hz) while a line envelope slowly raises the level, so the thrust seems to build.
- **Tension.** A non-realistic, psychological layer: two detuned sawtooth waves (a fifth apart) whose pitch rises exponentially from 50 Hz to 800 Hz over 15 seconds, using `xLine`, while their volume rises too.
- **Lift-off and flight.** Lowpass-filtered noise for the engines. The cutoff drops in later images to suggest the shuttle moving away from the camera. A short burst of filtered noise marks the boosters dropping away.
- **The quiet moment.** After all the noise, a single high sine fades in and out, like the ringing in your ears after a loud sound.
- **Space.** I wrote a brown-noise generator (a clamped random walk) and lowpass filter it for the background of space. The cutoff opens up when the shot moves inside the cabin.
- **Cabin.** Sine and triangle beeps from the control panels, scheduled by a phasor-based clock and a step counter, so different machines beep in different rhythms.
- **Spacewalk.** Breathing, made from white noise through a bandpass filter whose centre frequency is swept by a slow sine wave. The background fades out so the breathing is the focus.

Each envelope is guarded by a flag so it triggers only once, even though the image index is checked every frame.

## Built with

- [p5.js](https://p5js.org/) for the slideshow and timing
- [Maximilian](https://github.com/micknoise/Maximilian) (maxiLib + maximEx) for oscillators, filters and envelopes
