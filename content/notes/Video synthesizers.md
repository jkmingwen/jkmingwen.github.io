---
title: Video Synthesizers
tags:
  - video-synthesis
description: A short history of video synthesizers, summarized from my capstone thesis.
---
A video synthesizer is a tool that takes some form of input and manipulates or utilises said input to produce a visual signal.

Some notable examples of video synthesizer pioneers include:
- John and James Whitney's analogue video synthesizer used repurposed ballistics controllers, famously producing [Lapis (1966)](https://youtu.be/kzniaKxMr2g?si=ZABviqghQQHwGixi)
- Eric Siegel's Electronic Visual Synthesizer (EVS) ([demo](https://mediaburn.org/videos/color-composite-2/))
- Nam June Paik's [Paik-Abe Video Synthesizer](https://njp.ma/video/video-synthesizer/)

More modern visual synthesizers include:
- Critter & Guitari's [ETC](https://youtu.be/9cfNyvWLraQ?si=1kixPy-aY6FtTjix) and [EYESY](https://youtu.be/xIxchxVhVI0?si=zEYthTtudRzYUXY3)
- [Synchronator](https://www.synchronator.com/video.html)
- [TouchDesigner](https://derivative.ca/)
## Audio-to-visual synthesis
A possible input for a video synthesizer is an audio signal. My first exposure to this form of audio-driven video synthesis was in the music visualisation features available in Windows Media Player and iTunes. 

I think that it is notable that video synthesis is commonly accompanied by audio. Visuals that are associated with audio create a feeling of "synchresis", a term that Michel Chion describes in great detail in his book, *Audio-Vision: Sound on Screen (1994)*. Chion's work crystalises so many thoughts about the interplay between the aural and visual senses that I resonated with--I highly recommend giving it a read.

Audio-to-visual synthesis was the focus of my Bachelor's capstone project, in which I wrote an [[Audiovisual synthesizer|audiovisual synthesizer]] in openFrameworks, and ran it on an [ODROID XU4](https://wiki.odroid.com/odroid-xu4/odroid-xu4). Many things have happened since then, and as I've grown as a software engineer, I've found many aspects of my implementation that I think I could improve upon. I'm currently working on a reimplementation of my work in JUCE, with the use of OpenGL to parallelise the computations. 