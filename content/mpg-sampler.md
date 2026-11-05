---
title: MPG Sampler
date: 2018-01-30
tags:
  - audio
  - p5js
  - web-audio
description: A browser-based drum machine and sampler, inspired by the Akai MPC, built with p5.js.
---

%% TODO: set the real date above, and embed the performance video below if you still have it. %%

The **MPG** is a browser-based sampler and step sequencer I built for the *Sound and Signal* course at Goldsmiths. It's loosely modelled on the Akai MPC. The course provided us with a bare template with a start/stop button and a single working pad, and I built the rest of the instrument from there. A video demonstration of its capabilities can be found [here](https://youtu.be/piW4eBtyXqg?si=lw-T_3pLi8vf9ucw).
![[mpg-sampler-1793867155777.webp]]
## What it does

- **A 3×3 pad grid, a start/stop button and three vertical sliders**, all drawn and handled in p5.js.
- **A 16-step sequencer** that runs at 8 steps per second. A Maximilian phasor drives it, so the timing stays sample-accurate.
- **Six sample pads**, each showcasing a different sequencing technique:
  - The click and hi-hat play on a fixed beat, picked with the modulo of the playhead (every 6 and every 4 steps).
  - The snare follows a simple 16-step pattern of 1s and 0s.
  - The kick, a synth stab and a JFK speech sample use arrays of objects. Each step stores its own note on/off, playback rate (pitch) and volume, which gives the patterns some dynamics and melody.
- **Three modifier pads** that switch what each slider controls, so three sliders can change six parameters:

| Slider  | Default             | With modifier pad held |
| ------- | ------------------- | ---------------------- |
| Vol     | Kick + snare volume | Master volume          |
| Fla/Rev | Synth flanger mix   | Global reverb mix      |
| Dly/Pit | JFK delay time      | JFK pitch              |

## Effects

The audio is processed one sample at a time in a single audio loop, using Maximilian's delay lines and oscillators:

- **Flanger.** A slow sine LFO (0.25 Hz) sweeps the delay time on the synth sample.
- **Delay.** A feedback delay on the JFK sample. Each time the slider moves, the feedback amount is randomised, and at short delay times this sometimes turns into a flanger-like effect.
- **Reverb.** 20 feedback delay lines run in parallel on the master output, each with a random delay time (20–50 ms) and random feedback (0.75–0.94).

## Built with

- [p5.js](https://p5js.org/) for drawing and input
- [Maximilian](https://github.com/micknoise/Maximilian) (maxiLib) for sample playback, timing and DSP

## Revisiting it

Since it was written, browsers have adopted autoplay policies that keep an `AudioContext` suspended until the user interacts with the page. The fix was to call `audio.context.resume()` on the first mouse press.
