---
title: Ambient Sampler
date: 2018-02-19
tags:
  - audio
  - max-msp
  - dsp
description: A MIDI-controlled sampler instrument in Max/MSP for performing an ambient piece from guitar, vocal and drum samples.
---
%% Upload video to Youtube %%
The **Ambient Sampler** is a MaxMSP patch I built for *Music Computing 1* at Goldsmiths, along with an ambient piece performed on it. I wanted something that felt ambient but kept moving from start to finish. The volume swells of the piano and guitar in Jay Som's *Lipstick Stains* were the main inspiration. The piece begins and ends with a droning guitar. Its pitch shifts over the course of the piece to act as a melody, while oscillating filters and delays on the guitar and vocals add texture. A video demo can be found [here](https://drive.google.com/open?id=1ovyAWYkksEiAv8qS3SGu_qamXeiO0AYR).

![[ambient-sampler-1793886306652.webp]]
## Structure

The patch has four sections. Three are instruments, each with its own sequencer and signal processing. The fourth maps a MIDI controller onto every playable parameter. Each instrument stores its patterns in `preset` objects, and a single pad press on the controller switches all three instruments to the same preset at once.

## Instruments

- **Ambient guitar.** A guitar recording is loaded into a `buffer~` and played back by several `groove~` objects at different playback rates (including an octave up, an octave down and reversed). The copies are summed into one dense tone, an idea that came from the "Analyzing Algorithms" exercise in Cipriani and Giri's *Electronic Music and Sound Design*. The result goes through a resonant lowpass filter (`lores~`) with a modulated cutoff frequency. From the controller, I can change the modulation rate and depth and the filter's resonance, which turns the sweeps into rhythmic pulses.
- **Vocal phrases.** I picked out a set of start and end points in a vocal sample and stored them in a `coll`. `play~` uses those to slice the sample into different phrases. The output goes through a feedback delay with a lowpass filter inside the feedback loop, so high frequencies don't build up as the signal repeats. Changing the delay or ramp-time parameters modulates the delay length through `line~`, which bends the pitch of the repeats.
- **Drums.** Using the same slicing method, I cut a kick, snare, closed hi-hat and open hi-hat out of a drum loop. They go through a Gigaverb (BEAP) reverb so they sit together and with the rest of the piece. Holding a trigger pad switches individual sequencer steps on, and releasing it switches them off. That makes it possible to vary the drum pattern live without changing the preset (about 2:15 in the demo).

## MIDI mapping

A `midicontroller` subpatch parses knob messages (`ctlin`) and pad messages (`midiin`) and routes them by controller number to each instrument. A `funnel` on its output means only the parameter that actually changed gets updated. That matters on a hardware controller, where you don't want every parameter to jump to the current knob positions whenever you touch one.

## Built with

- [Max/MSP](https://cycling74.com/products/max) (`groove~`, `play~`, `coll`, `tapin~`/`tapout~`, `lores~`, BEAP Gigaverb)
- Audacity for preparing the samples
- A MIDI controller with knobs and trigger pads
