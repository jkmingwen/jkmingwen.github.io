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

Here is the relevant code that sets the register values for our audio codec configuration:
`int SSMSetConfig()
{
    int Status;

    Status = SSMRegSet(&Iic, 15, 0b000000000); // Perform Reset
    usleep(75000);

    // Leave crystal oscillator (D5) off as external clock used
    // Power down output (D4) to avoid pops during DAC initialisation
    Status |= SSMRegSet(&Iic, 6, 0b000110000);

    // Audio codec settings
    Status |= SSMRegSet(&Iic, 0, 0b000010111); // L in volume
    Status |= SSMRegSet(&Iic, 1, 0b000010111); // R in volume
    Status |= SSMRegSet(&Iic, 2, 0b001011111); // L out volume
    Status |= SSMRegSet(&Iic, 3, 0b001011111); // R out volume
    Status |= SSMRegSet(&Iic, 4, 0b000010010); // enable DAC, disable mic
    Status |= SSMRegSet(&Iic, 5, 0b000000000); // unmute DAC
    Status |= SSMRegSet(&Iic, 7, 0b000001010); // Word length 24, I2S mode
    Status |= SSMRegSet(&Iic, 8, 0b000000000); // Sampling rate of 48kHZ

    // Delay required before activating digital core
    // Refer to p. 11 of data sheet for delay formula
    // VMID capacitor = 10.1uF
    usleep(75000);

    // Activate the digital core
    Status |= SSMRegSet(&Iic, 9, 0b000000001);

    // Power up the outputs
    Status |= SSMRegSet(&Iic, 6, 0b000100000); // D4=0 (OUT on)

    if (Status != XST_SUCCESS) {
        return XST_FAILURE;
    }

    return XST_SUCCESS;
}`
I'll briefly explain the configuration set by these register values below.
## Reset all register values to defaults
We start by resetting all register values to their default settings. This behaviour is achieved by triggering a software reset (setting R15 to all 0s).

## Activating the digital core
From the instructions on page 11 of the data sheet, when activating the digital core of the audio codec, we need to insert a delay between activating the desired power-on blocks from R6 and activating the digital core (R9, D0). The data sheet provides us with a formula to approximate the necessary delay time (in $\mu s$): $t = C * 25000/3.5$. Given a capacitor value of $10.1\mu F$, we get $t = 4.7 * 25000 / 3.5 \approx 72143\mu s$. We round this up to $75000\mu s$.

## Set Input/Output volume
Registers 0 to 3 allow us to set the volume of the input and output channels. These have been set to 0 dB and -26dB respectively. We set a lower volume on the outputs than the default just to be safe --- the last thing we want is to accidentally damage our headphones.

## Enable DAC
We enable the DAC in Registers 4 and 5, which allow us to write audio data from the Zynq processor to the audio output jacks.

## Configure word length and sampling rate
Register 7 allows us to set the word length of each sample. We leave it at the default of 24 bits. In Register 8, we set our sampling rate to 48kHz. We thus require a master clock rate of 12.288MHz (we disabled clock division).

# Serialising the audio RX/TX streams
Audio data is transmitted to and from the Zynq SoC on the Zybo via the I2S port. To do this, we'll need a means of accessing the audio streams from the audio codec. Thankfully, Scott_1767 has written an [I2S transceiver](https://forum.digikey.com/t/i2s-transceiver-vhdl/12845) that helps us to achieve this goal. If you're interested in how the transceiver functions, their documentation includes details of how it serialises/deserialises the data streams and the clocks used to drive the circuit.

## Generating clock signals for the audio codec
The main thing we need to take away from the documentation for now is how to derive the master clock frequency required by the I2S transceiver. Given that we want a sampling rate ($f_s$) of 48kHz, and the SSM2603 supports $256f_s$ and $384f_s$ based clocks in normal mode (page 15 of the SSM2603 data sheet), we derive a master clock frequency of $256*f_s = 256*48kHz = 12.288MHz$.

The I2S transceiver takes the master clock signal and generates a serial clock (also known as a bit clock) and a word-select clock (also known as a left-right clock). These are sent to the audio codec, namely the digital audio bit clock (BCLK), DAC sampling rate clock (PBLRC), and the ADC sampling rate clock (RECLRC).

## Receiving and transmitting audio data
The I2S transceiver receives a serialised audio input signal at the sd_rx port, deserialises the data and exposes the audio input samples via the l_data_rx and r_data_rx ports. We can process these samples and then write them to the l_data_tx and r_data_rx ports. The I2S transceiver will then serialise the samples and send them to the audio codec via the sd_tx port.

For the purpose of an audio bypass, all we need to do is to connect the l/r_data_rx ports to the corresponding l/r_data_tx ports. This brings us to the next step, which is generating a block diagram.
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
