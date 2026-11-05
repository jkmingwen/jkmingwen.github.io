---
tags: audio
---
An audiovisual synthesizer is a [[Video synthesizers | video synthesizer]] whose images are affected by an input audio signal.

Here is an overview of the audiovisual synthesizer I wrote for my Bachelor's Capstone Project. It was written in [openFrameworks](https://openframeworks.cc/) and deployed on an [ODROID XU4](https://wiki.odroid.com/odroid-xu4/odroid-xu4). The repository of the project can be found [here](https://github.com/jkmingwen/CapstoneProject).

The video synthesizer had 3 patterns implemented: Circles, Clouds, and Shimmer.

## Circles ([video demo](https://youtu.be/td-MOom6OSc?si=r94SllRCJLfxBYmh))
![[Audiovisual synthesizer-1793787494719.webp]]
Circles is made up of a series of horizontally adjacent ellipses. Each ellipse corresponds to a sample in a buffer storing audio input samples. The amplitude of the sample is mapped to its vertical displacement while the overall intensity of the signal is mapped to the size of the ellipses.
## Clouds ([video demo](https://youtu.be/C0_xnJEDzDE?si=9XL2DBZNF9_4TOYR))
![[Audiovisual synthesizer-1793787507145.webp]]
Clouds is based on a swarming algorithm by Konstantin Makhmutov. Every "particle", represented by an ellipse, experiences an attractive force towards every other particle on the screen above a set proximity threshold. Conversely, a repelling force is applied between particles whose proximity falls below the threshold.

In the Clouds pattern, the intensity of the audio signal is mapped to the magnitude of the force experienced by each particle. The intensity is also mapped to the radii of the ellipses.
## Shimmer ([video demo](https://youtu.be/40OoNjl2jPE?si=-M9_XRpZmqzrbI9o))
![[Audiovisual synthesizer-1793787513832.webp]]
Shimmer is made up of an array of lines that rotate about fixed points to collectively form ripple effects. The angle of each line's rotation is determined by a Perlin Noise function that is increasingly offset with each element. Perlin Noise is also used to determine the opacity of each line, forming vignettes that move across the screen independently of the audio signal. The intensity of the audio signal determines the speed of each line's rotation.

## Re-implementation in JUCE
 I've learnt a lot since I first wrote this video synthesizer back in 2019, and I intend to do a rewrite in the JUCE framework. I intend to implement a more readily modular code architecture, with future optimizations implemented through shaders.