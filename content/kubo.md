---
title: Kubo
date: 2017-04-21
tags:
  - audio
  - hardware
  - arduino
  - instrument-design
description: An augmented electric guitar with hand-wound duophonic pickups that split the strings into three signal paths, plus wireless servo control of effects pedals.
---
**Kubo** is an augmented electric guitar I built with Nathaniel and Abhinav for *Interface Design for Time-based Art Forms* at Yale-NUS. The augmentation consists of duophonic pickups and wireless control of effects pedals. Our main reference point was the modern MIDI keyboard. A keyboard can apply different sounds to different ranges of keys, so a player can have a rhythm and a lead tone at the same time, and its effect knobs stay within reach while playing. Kubo brings both of those ideas to the guitar. A video demonstration can be found [here](https://youtu.be/GVYjbwj_Ycw?si=IovsiobI4n_wm00o).

The name of the project came from a movie I had recently watched, *Kubo and the Two Strings*, referencing the pairs of strings that the signal would be split into. That said, here is a picture of the final product:
![[kubo-guitar.webp]]
Looking closely at the pickups, notice how the middle pickup only has two poles beneath the middle pair of strings, while the bridge pickup has poles under the lowest and highest pair of strings. The neck pickup is left as-is for the player to use the electric guitar with a unified signal.

There are two control boxes: one in black and one in white. The knobs on the black box control wireless effect parameters while the knobs of the white box controls the gain for each pair of pickups. I'll describe the design of the duophonic pickups and the wireless effects control in turn.
## Splitting the signal

Instead of one pickup that hears all six strings, Kubo has three **duophonic pickups**, one for each pair of strings. Each pair gets its own signal path, so an effect can be applied to just the low strings, for example, while the high strings stay clean. To put effects on all six strings, you place them after the summing circuit instead.

- **Pickups.** We wound the pickups ourselves: using 0.1 mm enamelled copper wire on a 3D-printed bobbin with two pole pieces each. An Arduino-driven stepper motor turned the bobbin and counted the number of full rotations so we could keep the coils consistent across each pickup. The finished coils were potted in a mineral oil, paraffin wax, and beeswax mix to avoid microphonic feedback.
- **Output unit (preamps).** An enclosure mounted on the guitar holds three LM358 op-amp preamp stages, one per pickup, each with its own gain knob, plus a mute switch. It runs on a 9V battery and has three ¼" outputs, which can go to three separate effects chains.
- **Summing circuit.** A second LM358 stage mixes the three paths, either before or after their effects, back into a single output for the amplifier.
![[kubo-potting-pickup.webp|335x251]]![[kubo-pickup.webp|335x251]]
## Wireless effects control

The second part of Kubo lets the guitarist turn the knobs on effects pedals from the guitar itself.

- **Transmitter.** An Arduino Uno in an enclosure on the guitar reads three potentiometers and sends their values over a 315 MHz RF module.
- **Receiver.** A second Arduino Uno with an RF receiver drives three servo motors.
- **Servo mounts.** Each servo sits in a 3D-printed housing that clamps onto a bolt over the pedal, and its position along the bolt can be adjusted. A custom 3D-printed arm extension (pictured below) couples the servo to the pedal knob it should turn.
![[kubo-wireless-control.webp]]
## Playing it

Kubo has no fixed way of being played, but its features suggest a few:

1. As a normal electric guitar, with more flexible effects routing.
2. As a drone instrument, like a sitar, with separate effects on different string pairs producing contrasting droning notes.
3. Like an organ, adjusting effect parameters as notes ring to change their timbre.

## What I'd change

- **Master volume.** Each string pair has its own gain knob, but there's no single control for the overall level. The output box needs a master volume knob.
- **Wireless reliability and latency.** The servo connection kept dropping, possibly because of heavy RF interference where we tested. Adding an antenna helped. Our power supply also wasn't robust enough for the current spikes when the servos moved, so we deliberately slowed down the control code, and a knob turn took about a second to reach the pedal. That's far too slow for responsive real-time control. A proper power supply and a better radio link would be the first fixes.
- **Live use.** There are too many cables, signal chains and controls to manage in front of an audience. One idea is footswitch-triggered effect presets, along the lines of [this project ](http://hackaday.com/2015/10/09/controlling-guitar-amps-with-servos/) that uses servos to switch amp presets.

## Built with

- Arduino Uno ×2, 315 MHz RF transmitter/receiver modules, [RadioHead](https://www.airspayce.com/mikem/arduino/RadioHead/)
- LM358 op-amps for the preamps and summing stage
- SM-S2309S servos with 3D-printed housings, clamps and arm extensions (printed with MakerBot Print)
- Hand-wound pickups on 3D-printed bobbins, wound with an Arduino-driven stepper motor
