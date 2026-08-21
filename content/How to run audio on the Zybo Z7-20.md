---
draft: "true"
---
A guide for anyone who, like me, would like to directly access the audio signals going in and out of an FPGA. This guide specifically targets the Zybo Z7-20 as it is the only one I have on hand.

There are three major steps in accessing the audio signal streams on the Zybo Z7-20:
1. Serialising the audio RX/TX streams
2. Generating a block diagram
3. Initialise the audio codec

There was once a blog post (link to archived webpage) about how to do this, but it has since been taken down.

# Serialising the audio RX/TX streams
To do this, we'll need a means of accessing the audio streams from the audio codec. Thankfully, Scott_1767 has written an [I2S transceiver](https://forum.digikey.com/t/i2s-transceiver-vhdl/12845) that helps us to achieve this goal. I don't think it's necessary to go into the details of how it deserialises/serialises the input/output data streams. Perhaps for a future post.

# Generating a block diagram
We need to connect the I2S transceiver to the audio input and output pins of the Zybo. We also need to connect it to a Zynq PS block so that we can configure and initialise the audio codec later.

# Initialising the audio codec
In our case, its the SSM2603. [Here's the data specsheet.](https://www.analog.com/media/en/technical-documentation/data-sheets/SSM2603.pdf) What we're interested in is the Register Map on page 19: (insert image/table of register map here).