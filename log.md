## Progress 1 (7-Dec-2020 to 15-Jan-2021)

- Background research about FPGAs in general, Xilinx 7 Series FPGAs, their Frame
and CLB structures, Partial Dynamic Reconfiguration, AXI protocol and AXI
specific IPs, etc.
- Got familiar with Xilinx Vivado toolchain for FPGA development; the challenge
was to access all Vivado features using TCL commands and not use the GUI, for
better automation and reproduction
- Acquired the required proficiency in VHDL language and learned about general
workflow in hardware description (various steps involved like synthesis,
implementation etc.)
- Wrote an AXI Lite slave IP (with some registers) in VHDL. This is the current
method for managing register sets in FPGA. This test setup can be used later
for comparison with the new method.
- Code and simulation/verification of the AXI Lite slave:
https://github.com/Swaraj1998/axi-lite-slave
- Full test setup project for implementation:
https://github.com/Swaraj1998/axi-lite-slave-prj

- Currently, trying to find a way to read back the bitstream from Xilinx FPGA
through PCAP or ICAP interface (by reading/writing to certain registers) using a
python script

## Progress 2 (16-Jan-2021 to 31-Jan-2021)

- I was able to write a python script which can write/readback a full/partial
bitstream to the FPGA in Xilinx's Zynq SoC, using Xilinx's "devcfg" interface (a
wrapper around the PCAP interface), through direct register access
- Due to sparse documentation, it was a bit difficult/time taking to figure out
how exactly to directly access this "devcfg" interface, without using the Xilinx
SDK or their linux based driver (which is the usual way to access them)
- Studied the various open source tools for FPGA development (mostly under
project "Symbiflow"), which will come in use later
- Link to code: https://github.com/Swaraj1998/xilinx-devcfg

- My next task is to generate a test bitstream with a Look-Up Table (LUT), and
try to change the behaviour of the LUT through partial bitstream reconfiguration
in runtime (DPR)

## Progress 3 (1-Feb-2021 to 15-Feb-2021)

- I managed to successfully upload a live partial bitstream to the FPGA, using
my "devcfg" based python script, to change the logic of a targeted LUT
- For this I had to create a bitstream with a LUT at a specific location on the
FPGA, create another bitstream with only the logic of the LUT changed, and
generate a diff of both the bitstreams to find the frames with bits changed
- I then wrote a script which could extract specified frames from a full
bitstream and generate a loadable partial bitstream file (which I can upload to
the FPGA) 
- To achieve this I had to study the structure of the bitstream files in detail,
and there is still a lot of info to find that do not make sense to me at this
moment, which Xilinx does not disclose that easily
- Links to the test project and the partial bitstream generator script:
https://github.com/Swaraj1998/lut-test
https://github.com/Swaraj1998/xilinx-devcfg/blob/master/scripts/gen_partial_bitstream.py

- Next, I have to find a way to somehow 'mark' specific components (like LUTs,
or registers), and find the location (or the frame address) where they are mapped
to in the FPGA (which I can then use to update them through DPR)

## Progress 4 (16-Feb-2021 to 1-Mar-2021)

- I was able to reconfigure a LUT 'without' fixing it to a particular location
- I got the location of the test LUT from Vivado through Tcl commands, in a
form for e.g. SLICE_X26Y123/A6LUT, wherein the slice location and the
associated LUT (A out of {A,B,C,D}) are mentioned
- The mapping between this location and its frame address used in the bitstream
is proprietary information, but fortunately 'Project X-Ray' documents these in
the form of a database, which I used to get the mappings with a few scripts
- Link to scripts I wrote to get the 'location to address' mapping: 
https://github.com/Swaraj1998/prjxray/commit/aa8df8f272bda0a490df2211bd3ed86f0ebf25e0
- I also read a few research papers which mentioned some faster ICAP
implementations to do DPR, which will come in use later

- This was an important milestone. My next task is to implement a test 32-bit
register with a bunch of LUTs, and find a suitable way to reconfigure them in
runtime without potential side effects (i.e. different clock domains, undefined
outputs during reconfiguration, etc.).

## Progress 5 (2-Mar-2021 to 31-Mar-2021)

- I implemented a test 32-bit register (address generator) in VHDL, using 16
6-input-2-output LUTs, along with a bunch of flip flops to store the 32-bit
output so that outputs are defined during partial reconfiguration
- I tested this setup by setting and receiving the signals through the debug 
registers (FTM) present in Xilinx Zynq, for which I had to write another script
- Links to register setup and FTM script:
https://github.com/Swaraj1998/reg-test
https://github.com/Swaraj1998/zynq-misc-scripts/blob/master/zynq_ftm.py

- Next, I am trying to put constraints on the mappings of the register
structure (using RLOC attribute), so I can easily identify the slices that can
be targeted to be reconfigured

## Progress 6 (1-Apr-2021 to 13-Apr-2021)

- I added the appropriate RLOC constraints to arrange the LUTs in a specific
order which is suitable and efficient for reconfiguration (specifically mapped
to slices in a single column)
- Worked on a script/tool which can read/modify the LUT INIT values (which
defines a LUT's logic) directly in a Zynq bitstream, given a slice location 
- This script can now be used to simply read/write these INIT values to change
the register's behaviour/output
- The frames associated with the register are read from FPGA in runtime, the INIT
values can be changed and the frames are written back to the FPGA using PDR
- Link to the bitstream modifier script:
https://github.com/Swaraj1998/reg-test/blob/master/scripts/bitmod_init.py

- The basic proof of concept of the project is now complete. But a lot still
needs to be done/improved for the actual implementation. This is what I will
mostly be working on for the rest of the internship.

## Progress 7 (15-Apr-2021 to 30-Apr-2021)

- Much of my time was spent on debugging some issues with the bitmod_init.py
script, one such was the need to re-calculate CRC values for each frame while
modifying a full bitstream
- Generalized the structure of VHDL code to allow the creation of multiple custom
register instances of different types (boolean, constant etc.) depending on
the number of select signals and output width
- Used a custom attribute "REG_NAME" in HDL to identify each register uniquely 
- Modified the Vivado build script to write out the slice information
corresponding to each register instance name to a file "reg_slice_map.db", which
will be used by other scripts to help modify the correct bits in a bitstream

- Next, I have to figure out a way to use only SLICELs and not SLICEMs for the
implementation of register instances. SLICEMs may contain dynamic memory resources
and reconfiguring the frames associated with them will interfere with other
SLICEMs which are configured by the same set of frames.

## Progress 8 (1-May-2021 to 7-May-2021)

- I used PBLOCK constraints in Vivado to map the register resources to only
SLICELs and not SLICEMs, this took some time to setup correctly as an essential
PBLOCK property needed to be tweaked which was not properly documented
(IS_SOFT property)
- By modifying and combining all other scripts, I wrote the main interface
script "reg.py" which can read/write register values to any of the custom
register instances
- Command example: {python reg.py REG_32_CONST 0 -w 0xdeadbeef}, would write the
32-bit value "0xdeadbeef" to the "REG_32_CONST" named register instance at the
0'th register index (ranging [0-31] for each possible unique register) 
- Similarly, '-r' option will read the specified register's value from the FPGA 
- Link to the main register interface:
https://github.com/Swaraj1998/reg-test/blob/master/scripts/reg.py
- With this the final implementation of the project is now complete, it can be
used to define/read/write various custom register sets in the Zynq FPGA based on
PDR, using minimal FPGA resources 
- The remaining work is mostly optimization, which I will continue to work on
after the internship
