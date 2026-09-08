---
draft: "true"
---
A guide for anyone who, like me, would like to directly access the audio signals going in and out of an FPGA (specifically, the Zybo Z7-20). In this post, I will show how you can begin to access the audio signals received and transmitted by the Zybo by implementing a simple audio bypass. In more general terms, we will take an audio signal received via the "Line In" ports of the Zybo, and send it out through the "HPH Out" port. This would be the beginning of any audio processing project that you might want to implement on the Zybo. 

(add diagram of bypass, and where audio processing would sit.)

## Related works
There was once a [blog post](https://setoti-hatenablog-com/entry/zybo_synthesizer_2_Audio_Codec) that explained how to do this, but it has since been taken down, and I can no longer find it on any internet archives for some reason. More recently, Johan Steenkamp also has a post on [Creating Sound on the Zybo board](https://c64onfpga.blogspot.com/2019/02/creating-sound-on-zybo-board.html). His approach avoids the use of any high-level C driver wrappers, and is therefore a more low-level approach to the one I will be using here. Given that I'm only using these wrappers to initialize the audio codec, I believe the overheads incurred are acceptable for the gains in simplicity and stability.

# Overview of running audio on the Zybo
There are three major steps in accessing the audio signal streams on the Zybo Z7-20:
1. Configuring the audio codec
2. Serialising the audio RX/TX streams
3. Generating a block diagram

# Configuring the audio codec
Audio signals are received and transmitted through the audio codec of the FPGA. The audio codec used on the Zybo Z7-20 is the Analog Devices SSM2603. [Here is its data sheet](https://www.analog.com/media/en/technical-documentation/data-sheets/SSM2603.pdf), which includes many useful details on the capabilities of the audio codec. 

What we're interested in for this post is the Register Map on page 19.
![[ssm2603_register_map.webp]]The bits written to these registers allow us to configure the behaviour of the SSM2603. The details of each of these settings are included in the datasheet (pages 20-29).

## Initialising the audio codec
To trigger a software reset, we set R15 to all 0s, which resets all register values to their default settings. From the instructions on page 11 of the data sheet, when activating the digital core of the audio codec, we need to insert some delay between activating the desired power-on blocks from R6 and activating the digital core (R9, D0). They provide us with a formula to approximate the necessary delay time (in $\mu s$): $t = C * 25000/3.5$. Given a capacitor value of $10.1\mu F$, we get $t = 4.7 * 25000 / 3.5 \approx 72143\mu s$. We round this up to $75000\mu s$.

Register R8 determines the sampling rate. From our configuration, given a master clock rate of 12.288MHz (we disabled clock division), our sampling rate has been set to 48kHz (derived from MCLK/256 = 12.288MHz/245 = 48kHz). We then have a bclk of MCLK/4 = 3.072MHz.
# Serialising the audio RX/TX streams
The I2S port is used to transmit data to and from the Zynq SoC on the Zybo. To do this, we'll need a means of accessing the audio streams from the audio codec. Thankfully, Scott_1767 has written an [I2S transceiver](https://forum.digikey.com/t/i2s-transceiver-vhdl/12845) that helps us to achieve this goal. If you're interested in how the transceiver functions, their documentation includes details of how it serializes/deserializes the data streams and the clocks used to drive the circuit.

# Generating a block diagram
We need to connect the I2S transceiver to the audio input and output pins of the Zybo. To do this, we use 4 different IP blocks:
1. Clocking Wizard
2. I2S Transceiver
3. Processing System
4. Processor System Reset (optional..?)
Steps:
- Expose DDR, FIXED_IO, I2C ports on Zynq PS
- Configure clocking wizard
	- 12.288 for I2S transceiver mclk
	- 24.576 for ac_mclk
- Initialize ports in constraints file
- Connect components (include screenshot) - show how to make ports external
- Right-clock block design and select "Create HDL Wrapper" (select "let user", for some reason I can't set it to the top component when I let vivado control updates)
- Set wrapper as top component
- Run synthesis and implementation and generate bitstream
- Export hardware platform (File > Export > Export hardware)
- Launch Vitis
- Create Platform Project
	- Create a new platform from hardware (XSA)
	- Under Hardware Specification, select the exported hardware platform
- Create New Application Project (in platform project)
	- Select platform you just created
	- Select standalone_domain
	- Select Empty C/C++ project
- Under application project, make a new C file to write your audio codec initialization in
	- audio_init_system > audio_init > src > ac_init.c
## Clocking Wizard configuration
We need two clock signals to drive the audio system. One at 12.288 MHz, and the other at 24.576MHz. From the documentation of the I2S transceiver, 

We also need to connect it to a Zynq PS block so that we can configure and initialise the audio codec later.

## Enable I2C communication through Processing System
The processing system is what initializes the audio codec on the Zybo. We enable the I2C interface on the Zynq processing system in order to communicate with the audio codec.


## Processor system reset configuration
Has input ports for "slowest_sync_clk" and "dcm_locked". 
The slowest synchronous clock input should be connected to the slowest synchronous clock used.
DCM stands for Digital Clock Manager, and the DCM lock input port is meant to be connected to the DCM that will be locked last in the system. If there aren't any DCMs in the system, the input should be set to high.
The reset signal was taken from the Zynq processing system (via the "ext_reset_in" port).

According to Xilinx's documentation, the signal generated by the peripheral pin is meant for "all peripherals attached to interconnect that is synchronous with the "slowest_sync_clk".

# Constraints file configuration
Here are the contents of the constraints file.
`# clock`
`set_property -dict { PACKAGE_PIN K17   IOSTANDARD LVCMOS33 } [get_ports { sys_clk }];`
`create_clock -add -name sys_clk_pin -period 8.00 -waveform {0 4} [get_ports { sys_clk }];`
`# switches and LEDs`
`set_property -dict { PACKAGE_PIN G15   IOSTANDARD LVCMOS33 } [get_ports { sw0 }];`
`set_property -dict { PACKAGE_PIN M14   IOSTANDARD LVCMOS33 } [get_ports { led }];`
`set_property -dict { PACKAGE_PIN M15   IOSTANDARD LVCMOS33 } [get_ports { led1 }];`
`set_property -dict { PACKAGE_PIN G14   IOSTANDARD LVCMOS33 } [get_ports { led2 }];`
`# audio codec pins`
`set_property -dict { PACKAGE_PIN R19   IOSTANDARD LVCMOS33 } [get_ports { ac_bclk }];`
`set_property -dict { PACKAGE_PIN R17   IOSTANDARD LVCMOS33 } [get_ports { ac_mclk }];`
`set_property -dict { PACKAGE_PIN P18   IOSTANDARD LVCMOS33 } [get_ports { ac_muten }];`
`set_property -dict { PACKAGE_PIN R18   IOSTANDARD LVCMOS33 } [get_ports { ac_pbdat }];`
`set_property -dict { PACKAGE_PIN T19   IOSTANDARD LVCMOS33 } [get_ports { ac_pblrc }];`
`set_property -dict { PACKAGE_PIN R16   IOSTANDARD LVCMOS33 } [get_ports { ac_recdat }];`
`set_property -dict { PACKAGE_PIN Y18   IOSTANDARD LVCMOS33 } [get_ports { ac_reclrc }];`
`set_property -dict {PACKAGE_PIN N18 IOSTANDARD LVCMOS33} [get_ports iic_scl_io];`
`set_property -dict {PACKAGE_PIN N17 IOSTANDARD LVCMOS33} [get_ports iic_sda_io];`

