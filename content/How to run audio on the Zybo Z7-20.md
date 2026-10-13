---
tags:
  - hardware
  - fpga
  - audio
---

A guide for anyone who would like to directly access the audio signals going in and out of an FPGA. In this post, I will demonstrate how one might access the audio signals received and transmitted by the Zybo Z7-20 by implementing a simple audio bypass. In more general terms, we will take an audio signal, received via the "Line In" ports of the Zybo, and send it out through the "HPH Out" port. This implementation would be a good starting point for any audio signal processing project that you might want to implement on an FPGA.

(add diagram of bypass, and where audio processing would sit.)

# Related works
There was once a [blog post](https://setoti-hatenablog-com/entry/zybo_synthesizer_2_Audio_Codec) that explained how to access the audio signals in a manner similar to what I describe in this post. Unfortuntately, it has since been taken down, and I can no longer find it on any internet archives. 

More recently, Johan Steenkamp wrote a post on [Creating Sound on the Zybo board](https://c64onfpga.blogspot.com/2019/02/creating-sound-on-zybo-board.html). His approach avoids the use of any high-level C driver wrappers for the audio codec initialisation. It represents a more low-level approach to the one I will be using here. Given that I'm only using these wrappers to initialise the audio codec, I believe the overheads incurred are acceptable for the gains in simplicity and stability.

If you'd simply like to get an audio signal processing program running on an FPGA without having to delve into hardware description language (HDL) details, the [SyFaLa](https://inria-emeraude.github.io/syfala/) project is an excellent tool that automatically compiles [Faust](https://faust.grame.fr/) programs for FPGA platforms.
# Overview of running audio on the Zybo
There are three major steps in accessing the audio signal streams on the Zybo Z7-20 (Zybo, for short):
1. Configure the audio codec
2. Serialise audio RX/TX streams
3. Generate block diagram
4. Program your Zybo with Vitis
I will go through each of these steps and, at the end, you should have a working audio bypass on your Zybo!

# Configure the audio codec
Audio signals are received and transmitted through the audio codec of the FPGA. The audio codec used on the Zybo Z7-20 is the Analog Devices SSM2603. [Here is its data sheet](https://www.analog.com/media/en/technical-documentation/data-sheets/SSM2603.pdf), which includes many useful details on the capabilities of the audio codec.

What we're interested in for this post is the Register Map on page 19.
![[ssm2603_register_map.webp]]The bits written to these registers allow us to configure the behaviour of the SSM2603. The details of each of these settings are included in the datasheet (pages 20-29).

The code below sets the register values for our audio codec configuration. We will only be using it in the final step, where we configure the Zybo using Vitis. Nonetheless, it is clearer to explain the audio codec configuration earlier.
```
int SSMSetConfig()
{
    int Status;

    Status = SSMRegSet(&Iic, 15, 0b000000000); // software reset
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
    Status |= SSMRegSet(&Iic, 7, 0b000001010); // word length 24, I2S mode
    Status |= SSMRegSet(&Iic, 8, 0b000000000); // sampling rate of 48kHZ

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
}
```
You might have noticed that register 4 has an option to enable bypass (D3). While setting this bit to 1 would achieve an audio bypass, it defeats the primary purpose of this guide, which is to show how to expose the audio signal data being received and transmitted by the audio codec. It is therefore set to 0.

I will now briefly explain the configuration set by these register values below.
## Reset all register values to defaults
We start by resetting all register values to their default settings. This behaviour is achieved by triggering a software reset (setting R15 to all 0s).

## Activating the digital core
From the instructions on page 11 of the [SSM2603 data sheet](https://www.analog.com/media/en/technical-documentation/data-sheets/SSM2603.pdf), when activating the digital core of the audio codec, we need to insert a delay between activating the desired power-on blocks from R6 and activating the digital core (R9, D0). Page 11 also provides us with a formula to approximate the necessary delay time (in $\mu s$): $$t = C * 25000/3.5$$Given a capacitor value of $10.1\mu F$, we get: $$t = 10.1 * 25000 / 3.5 \approx 72143\mu s$$For simplicity, we round this up to $75000\mu s$.

## Set Input/Output volume
Registers 0 to 3 allow us to set the volume of the input and output channels. These have been set to 0 dB and -26dB respectively. We set a lower volume on the outputs than the default just to be safe---the last thing we want is to accidentally damage our speakers/headphones.

## Enable DAC
We enable the DAC in Registers 4 and 5, allowing us to write audio data from the Zynq processor to the audio output jacks.

## Configure word length and sampling rate
Register 7 allows us to set the word length of each sample. We leave it at the default of 24 bits. In Register 8, we set our sampling rate to 48kHz.

# Serialise audio RX/TX streams
Audio data is transmitted to and from the Zynq SoC on the Zybo via the I2S port. Reading from and writing to the audio codec therefore requires a means of accessing the audio data from the I2S ports. Thankfully, an I2S transceiver that helps us to achieve this goal already exists. 

The I2S transceiver performs the following functions:
1. Deserialise received (RX) input audio from the audio codec
2. Serialise output audio to be transmitted (TX) to the audio codec
3. Generate clock signals for the audio codec
![[i2s_transceiver_diagram.webp]]
The diagram provided in the [I2S transceiver documentation](https://forum.digikey.com/t/i2s-transceiver-vhdl/12845) situates it in an implementation. The deserialised RX data are exposed via the l/r_data_rx ports. Conversely, data that you wish to serialise for the audio output should be written to the l/r_data_tx ports.

If you're interested in how the transceiver functions, their documentation also includes details of how it serialises/deserialises the data streams and the clocks used to drive the circuit.
## Generating clock signals for the audio codec
 Given that we want a sampling rate ($f_s$) of 48kHz, and that the SSM2603 supports $256f_s$ and $384f_s$ based clocks in normal mode (stated on page 15 of the SSM2603 data sheet), we derive a master clock frequency of $$256*f_s = 256*48kHz = 12.288MHz$$The I2S transceiver is designed to take the same master clock signal as the one sent to the audio codec. This is stated in its documentation and is also implied in the diagram above.

![[samplerate_lookup_table.webp|Lookup table for master clock frequencies included in SSM2603 data sheet]]
Page 25 of the SSM2603 data sheet also provides a handy lookup table to determine the master clock frequency given the sampling rate configuration of the audio codec.

The I2S transceiver takes the master clock signal and generates a serial clock (also known as a bit clock) and a word-select clock (also known as a left-right clock). These are sent to the audio codec, namely the digital audio bit clock (BCLK), DAC sampling rate clock (PBLRC), and the ADC sampling rate clock (RECLRC).

## Receiving and transmitting audio data
The I2S transceiver receives a serialised audio input signal at the sd_rx port, deserialises the data and exposes the audio input samples via the l/r_data_rx ports. For audio processing, you would perform some signal processing on these samples before writing them to the l/r_data_tx ports. The I2S transceiver will then serialise the samples and send them to the audio codec via the sd_tx port.

For the purpose of an audio bypass, all we need to do is to connect the l/r_data_rx ports to the corresponding l/r_data_tx ports. This brings us to the next step, which is generating a block diagram.
# Generate block design
We need to connect the I2S transceiver to the audio input and output pins of the Zybo. To do this, we use 3 IP blocks:
1. Clocking Wizard
2. I2S Transceiver
3. Processing System
They are connected like so:
![[block_design_final.webp]]
The next subsections detail a step-by-step process of the block design generation.
## Instantiate IP blocks
Generate a new block design by clicking on "Generate Block Design" in the Flow Navigator. From here, we can add the IP blocks necessary for our design.

Start by adding the Zynq7 Processing System IP by clicking on the "+" icon in the block design page. Search for "ZYNQ7 Processing System". Do the same for the Clocking Wizard IP. Finally, if you haven't already done so, add the I2S transceiver as a design source and drag it into the block design.
![[add_ip.webp|315]]
### Zynq7 Processing System
Double click on the Zynq7 Processing System IP to customize it. Under the "Peripheral I/O Pins" tab, select the "I2C 0" peripheral.
![[config_zynq7.webp]]
Next, right-click on the output ports for DDR, FIXED_IO, and IIC_0 and select "Make External" from the drop-down menu. Rename the external port of IIC_0 to "iic".
![[make_ext_i2c.webp]]
Finally, connect the "FCLK_RESET0_N" output port of the Zynq7 to the "reset_n" input pin of the I2S transceiver.
### Clocking Wizard
For the clocking wizard, set a single output clock with a frequency of 12.288MHz. This will act as the master clock signal for the I2S transceiver and the audio codec.
![[config_clk_wiz.webp]]
Connect the "clk_out1" output port of the Clocking Wizard to the "mclk" input port of the I2S transceiver.
## Generate external ports
To connect the components of our block design to the components on the Zybo, we generate external ports. This can be done by either selecting "Create Port" from the drop-down menu, or via the hotkey Ctrl+k. 
![[create_port.webp]]
Generate ports with the following names (note their direction!):
- Direction: Input
	- sys_clock
	- ac_recdat
	- sw0
- Direction: Output
	- ac_bclk
	- ac_reclrc
	- ac_pblrc
	- ac_pbdat
	- ac_mclk
	- ac_muten
	- led0
It's OK to leave the Type as "Other". In order to map these ports to the components on the Zybo, however, **it is essential that the names of the ports match those defined in the constraints file.** Here is the constraints file; add it into your Vivado project via Add Sources > Add or create constraints.
### Constraints file configuration
```
# clock
set_property -dict { PACKAGE_PIN K17   IOSTANDARD LVCMOS33 } [get_ports { sys_clk }];
create_clock -add -name sys_clk_pin -period 8.00 -waveform {0 4} [get_ports { sys_clk }];
# switches and LEDs
set_property -dict { PACKAGE_PIN G15   IOSTANDARD LVCMOS33 } [get_ports { sw0 }];
set_property -dict { PACKAGE_PIN M14   IOSTANDARD LVCMOS33 } [get_ports { led0 }];
set_property -dict { PACKAGE_PIN M15   IOSTANDARD LVCMOS33 } [get_ports { led1 }];
set_property -dict { PACKAGE_PIN G14   IOSTANDARD LVCMOS33 } [get_ports { led2 }];
# audio codec pins
set_property -dict { PACKAGE_PIN R19   IOSTANDARD LVCMOS33 } [get_ports { ac_bclk }];
set_property -dict { PACKAGE_PIN R17   IOSTANDARD LVCMOS33 } [get_ports { ac_mclk }];
set_property -dict { PACKAGE_PIN P18   IOSTANDARD LVCMOS33 } [get_ports { ac_muten }];
set_property -dict { PACKAGE_PIN R18   IOSTANDARD LVCMOS33 } [get_ports { ac_pbdat }];
set_property -dict { PACKAGE_PIN T19   IOSTANDARD LVCMOS33 } [get_ports { ac_pblrc }];
set_property -dict { PACKAGE_PIN R16   IOSTANDARD LVCMOS33 } [get_ports { ac_recdat }];
set_property -dict { PACKAGE_PIN Y18   IOSTANDARD LVCMOS33 } [get_ports { ac_reclrc }];
set_property -dict {PACKAGE_PIN N18 IOSTANDARD LVCMOS33} [get_ports iic_scl_io];
set_property -dict {PACKAGE_PIN N17 IOSTANDARD LVCMOS33} [get_ports iic_sda_io];
```
Note: The prefix of the port names of the I2C ports need to match too! This is why we renamed the external port to "iic" after the "Make external" step of setting up the Zynq7 Processing System.

Now, we simply connect all the remaining ports. For clarity, I've added a table of source and destination ports:

| Source port   | Source component | Destination port | Destination component |
| ------------- | ---------------- | ---------------- | --------------------- |
| DDR           | Zynq7 PS         | DDR              | External port         |
| FIXED_IO      | Zynq7 PS         | FIXED_IO         | External port         |
| IIC_0         | Zynq7 PS         | iic              | External port         |
| sys_clk       | External port    | M_AXI_GP0_ACLK   | Zynq7 PS              |
| sys_clk       | External port    | clk_in1          | Clocking Wizard       |
| sw0           | External port    | led0             | External port         |
| sw0           | External port    | ac_muten         | External port         |
| mclk          | Clocking Wizard  | ac_mclk          | External port         |
| mclk          | Clocking Wizard  | mclk             | I2S Transceiver       |
| ac_recdat     | External port    | sd_rx            | I2S Transceiver       |
| FCLK_RESET0_N | Zynq7 PS         | reset_n          | I2S Transceiver       |
| sclk          | I2S Transceiver  | ac_bclk          | External port         |
| ws            | I2S Transceiver  | ac_reclrc        | External port         |
| ws            | I2S Transceiver  | ac_pblrc         | External port         |
| sd_tx         | I2S Transceiver  | ac_pbdat         | External port         |
| l_data_rx     | I2S Transceiver  | l_data_tx        | I2S Transceiver       |
| r_data_rx     | I2S Transceiver  | r_data_tx        | I2S Transceiver       |

At this point, you should have the same block design as the one shown at the beginning of the subsection "Generate block design". Save your block design, right-click on the block design file, and select "Create HDL Wrapper...". For some reason, Vivado will not list the wrapper in Design Sources if the default "Let Vivado manage wrapper and auto-update" is selected. Therefore, in the pop-up window, select "Copy generated wrapper to allow user edits". Finally, set the wrapper as the top-component by selecting "Set as top" in the drop-down menu.

![[create_wrapper.webp]]![[set_as_top.webp]]
## Generate hardware platform
The final step is to create a hardware platform that we can target in Vitis. Start by selecting "Generate Bitstream" under the Flow Navigator. This will also run Synthesis and Implementation on the block design if it is out of date. The entire process can take a few minutes.

Once it is done, select File > Export > Export Hardware... to open the export hardware wizard.
![[export_hardware.webp]]
![[include_bitstream.webp]]
![[export_hardware_location.webp]]
Be sure to select "Include bitstream", then save it in a location of your choice. The XSA file exported will be used as the hardware specification for the Vitis project. This brings us to the final step. 
# Program your Zybo with Vitis
## Create a Vitis project
To initialize the audio codec, we need to create a Vitis project. This can be done in Vivado by clicking on Tools > Launch Vitis IDE. You'll be prompted to "Select a directory as a workspace". I like to set it within the same directory as the Vivado project I'm working on.
![[vitis_workspace_select.webp]]

Select "Create Application project".![[vitis_welcome.webp]]

At the next page, click on the tab labelled "Create a new platform from hardware (XSA)". Under "Hardware Specification", select the XSA file we exported from Vivado. This contains the hardware design we generated in Vivado. 
![[select_hardware_spec.webp]]

Accept the default processor ("ps7_cortexa9_0") and operating system ("standalone").

![[new_application_project.webp]]

![[application_project_domain.webp]]
Next, select "Empty Application (C)".
![[create_empty_application.webp]]

Create a new C file in the "src" folder of the application project to specify the audio codec initialisation in.
![[create_new_file.webp]]
Build the application project. After the build process has completed, you can plug in your Zybo --- remember to turn it on!

Now, finally, run the application project.
![[programming_fpga_progress.webp]]

The "DONE" LED should turn on, indicating that the board has been successfully programmed. Once that is done, you can test if it is working. Plug in your source audio into the port labelled "Line In", and your headphones/speakeres into the port labelled "HPH Out". Turn off the mute switch (SW0); the corresponding LED should turn on as well, and you should now hear audio running through your Zybo!
![[zybo.webp]]
