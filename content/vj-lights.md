---
title: VJ Lights
date: 2018-03-19
tags:
  - audio
  - p5js
  - web-audio
  - fft
  - audio-analysis
description: A virtual lighting rig that reacts to individual instruments in a techno track, using FFT band energy and envelope followers.
draft: "true"
---

**VJ Lights** is an audio-reactive lighting rig I built for the *Sound and Signal* course at Goldsmiths. Four virtual stage lights respond to Carl Craig's *Televised Green Smoke*, and each one tries to follow a different part of the mix. The course gave us a template that played the track and drew its FFT. From there, I chose the frequency bands, did the analysis and tuned each light's response.

## How it works

The track is analysed with a 1024-point FFT (512 bins, 4× overlap). Every frame, I sum the magnitudes of a chosen range of bins for each light, and the mean of that range becomes the light's level. While tuning, I colour-coded each light's bins on the spectrum display and logged the running maximum of each band, so I could calibrate how band energy maps to brightness.

| Light | Target | Bins | Processing |
| --- | --- | --- | --- |
| Red | Kick drum | 0–1 | Envelope follower (attack 0.2, release 0.01) to smooth the response |
| Yellow | Hi-hats (from ~0:30) | 120–230 | Raw band mean. Any smoothing made the light miss the hats. |
| Green | Synth chords (1:17) and claps (3:56) | 20–80 | Mean squared and scaled to emphasise peaks, then a fast envelope follower |
| Blue | Arpeggiated synth (2:38) | 90–103 and 239–245 | Mean of two separate bands |

## What was hard

- **Overlapping sources.** The chords and claps share their frequency range with the kick and hats, which are also percussive. Squaring the band mean before the envelope follower made the light respond to the louder chord and clap hits more than to the background percussion.
- **Spread-out sources.** The arpeggio's energy covered a wide part of the spectrum. I ended up combining the two narrow bands that moved most during the arpeggio and stayed fairly still without it. Because each note of the arpeggio lights up different bins, the light still couldn't follow its rhythm exactly. That limitation is built into fixed-band analysis.
- **Smoothing vs. responsiveness.** Some lights needed an envelope follower to stop flickering, but on the hi-hats any smoothing made the light miss hits.

## Built with

- [p5.js](https://p5js.org/) for drawing the spectrum and lights
- [Maximilian](https://github.com/micknoise/Maximilian) (`maxiFFT`, `maxiSample`, maximEx envelope follower)
