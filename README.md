# PulseRain Reindeer - RISCV RV32I[M] Soft CPU
----------------------------------------------
  * # Overview
PulseRain Reindeer is a soft CPU of Von Neumann architecture. It supports RISC-V RV32I[M] instruction set, and features a 2 x 2 pipeline. It strives to make a balance between speed and area, and offers a flexible choice for soft CPU across all FPGA platforms.

## Highlights - 2 x 2 Pipeline
Reindeer's pipeline is composed of 4 stages: 
  + Instruction Fetch (IF)
  + Instruction Decode (ID)
  + Instruction Execution (IE)
  + Register Write Back and Memory Access (MEM)

However, unlike traditional pipelines, Reindeer's pipeline stages are mapped to a 2 x 2 layout, as illustrated below:

![2 x 2 pipeline](https://github.com/PulseRain/Reindeer/raw/master/docs/pipeline_2x2.png "2 x 2 Pipeline")

In the 2 x 2 layout, each stage is active every other clock cycle. For the even cycle, only IF and IE stages are active, while for the odd cycle, only ID and MEM stages are active. In this way, the Instruction Fetch and Memory Access always happen on different clock cycles, thus to avoid the structural hazard caused by the single port memory. 

Thanks to using single port memory to store both code and data, the Reindeer soft CPU is quite portable and flexible across all FPGA platforms. Some FPGA platforms, like the Lattice iCE40 UltraPlus family, carry a large amount of SPRAM with very little EBR RAM. And those platforms will find themselves a good match with the PulseRain Reindeer when it comes to soft CPU.

## Highlights - Hold and Load

Bootstrapping a soft CPU tends to be a headache. The traditional approach is more or less like the following:
  1. Making a boot loader in software
  2. Store the boot loader in a ROM
  3. After power on reset, the boot loader is supposed to be executed first, for which it will move the rest of the code/data into RAM. And the PC will be set to the _start address of the new image afterwards.

The drawbacks of the above approach are:
  1. The bootloader will be more or less intrusive, as it takes memory spaces.
  2. The implementation of ROM is not consistent across all FPGA platforms. For some FPGAs, like Intel Cyclone or Xilinx Artix, the memory initial data can be stored in FPGA bitstream, but other platforms might choose to use external flash for the ROM data. The boot loader itself could be executed in place, or loaded with a small preloader implemented in FPGA fabric. And when it comes to SoC + FPGA, like the Microsemi Smartfusion2, a hardcore processor could also be involved in the boot-loading. In other words, the soft CPU might have to improvise a little bit to work on various platforms.

To break the status quo, the PulseRain Reindeer takes a different approach called "hold and load", which  brings a hardware based OCD (on-chip debugger) into the fore, as illustrated below:

![Hold and Load](https://github.com/PulseRain/Reindeer/raw/master/docs/hold_and_load.png "Hold and Load")

The soft CPU and the OCD can share the same UART port, as illustrated above. The RX signal goes to both the soft CPU and OCD, while the TX signal has to go through a mux. And that mux is controlled by the OCD. 

After reset, the soft CPU will be put into a hold state, and it will have access to the UART TX port by default. But a valid debug frame sending from the host PC can let OCD to reconfigure the mux and switch the UART TX to OCD side, for which the memory can be accessed, and the control frames can be exchanged. A new software image can be loaded into the memory during the CPU hold state, which gives rise to the name "hold-and-load". 

After the memory is loaded with the new image, the OCD can setup the start-address of the soft CPU, and send start pulse to make the soft CPU active. At that point, the OCD can switch the UART TX back to the CPU side for CPU's output.
 
As the hold-and-load gets software images from an external host, it does not need any ROM to begin with. This makes it more portable across all FPGA platforms. If Flash is used to store the software image, the OCD can be modified a little bit to read from the flash instead of the external host.

  * # Folder Structure of the Repository
  
  The folder structure of the [GitHub repository](https://github.com/PulseRain/Reindeer) is illustrated below:
  
![Folder Structure](https://github.com/PulseRain/Reindeer/raw/master/docs/folder_structure.png "Folder Structure")
  
**And here is a brief index of the key items in the repository:**
  * **The soft CPU's HDL code**
  
    The platform dependent top level verilog code can be found in https://github.com/PulseRain/Reindeer/tree/master/source. And the platform independent verilog code can be found in https://github.com/PulseRain/Reindeer/tree/master/submodules/PulseRain_MCU
  
  * **Constraint and other FPGA-related files necessary to produce a binary bitstream for the respective hardware**
  
    Synthesis related constraint files can be found in https://github.com/PulseRain/Reindeer/tree/master/build/synth/constraints, and PAR related constraint files can be found in https://github.com/PulseRain/Reindeer/tree/master/build/par/constraints.
  
    To build bitstream for [**Gnarly Grey UPDuinoV2 board**](http://www.latticesemi.com/en/Products/DevelopmentBoardsAndKits/GnarlyGreyUPDuinoBoard), use Lattice Radiant software to open https://github.com/PulseRain/Reindeer/blob/master/build/par/Lattice/UPDuinoV2/UPDuinoV2.rdf
  
    To build bistream for [**Future Electronics Creative board (SmartFusion2)**](https://www.futureelectronics.com/p/development-tools--development-tool-hardware/futurem2sf-evb-future-electronics-dev-tools-3091560), do the following:
  
    1. Use synplify_pro (part of  Microsemi Libero SOC V11.9) to open https://github.com/PulseRain/Reindeer/blob/master/build/synth/Microsemi/Reindeer.prj, and generate Reindeer.vm

    2. Close synplify_pro and use Libero SOC V11.9 to open https://github.com/PulseRain/Reindeer/blob/master/build/par/Microsemi/creative/creative.prjx, import the Reindeer.vm produced in the previous step, and start the build process to generate bitstream. (In the repository, the Reindeer.vm has already been imported and put in https://github.com/PulseRain/Reindeer/tree/master/build/par/Microsemi/creative/hdl.)
  
  * **Binary version of the bitstream** 
  
    *) The Lattice FPGA bitstream for [**Gnarly Grey UPDuinoV2 board**](http://www.latticesemi.com/en/Products/DevelopmentBoardsAndKits/GnarlyGreyUPDuinoBoard) can be found in https://github.com/PulseRain/Reindeer/raw/master/bitstream_and_binary/Lattice/UPDuinoV2/UPDuinoV2_Reindeer.bin

    *) The Microsemi FPGA bitstream for [**Future Electronics Creative board (SmartFusion2)**](https://www.futureelectronics.com/p/development-tools--development-tool-hardware/futurem2sf-evb-future-electronics-dev-tools-3091560) can be found in https://github.com/PulseRain/Reindeer/raw/master/bitstream_and_binary/Microsemi/creative/creative.stp
  
  * # Simulation with [Verilator](https://www.veripool.org/wiki/verilator)

The PulseRain Reindeer can be simulated with [Verialtor](https://www.veripool.org/wiki/verilator). To prepare the simulation, the following steps (tested on n Ubuntu and Debian hosts) can be followed: 
  1. Install zephyr-SDK, (details can be found in https://docs.zephyrproject.org/latest/getting_started/installation_linux.html)
     
  2. Make sure riscv32-zephyr-elf-  tool chain is in $PATH and is accessible everywhere
     If default installation path is used, the following can be appended to the .profile or .bash_profile
         export ZEPHYR_TOOLCHAIN_VARIANT=zephyr
         
         export ZEPHYR_SDK_INSTALL_DIR=/opt/zephyr-sdk
         
         export PATH="/opt/zephyr-sdk/sysroots/x86_64-pokysdk-linux/usr/bin/riscv32-zephyr-elf":$PATH
         
  3. git https://github.com/PulseRain/Reindeer.git
  
  4. cd Reindeer/sim/verilator
  
  5. Build the verilog code and C++ test bench: make
  
  6. Run the simulation for compliance test: make test_all

BTW, the Makefile has the following targets:
make build    # default target to build the test bench
make test_all  # run compliance test for all 55 cases
make test compliance_case_name  # for example: make test I-ADD-01
              make run elf_file # run sim on an elf file, for example:    make run ~/zephyr/samples/hello_world/build/zephyr/zephyr.elf


