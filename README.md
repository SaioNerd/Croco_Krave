# Croc System-on-Chip: SEC-DED Memory Protection

A reliable memory protection extension for the educational Croc SoC, implementing hardware Single Error Correction, Double Error Detection (SEC-DED). Developed as part of the VLSI 2 coursework at ETH Zürich, this repository includes all SystemVerilog RTL, testbenches, and scripts necessary to synthesize and simulate the protected memory architecture in [IHP's open-source 130nm technology](https://github.com/IHP-GmbH/IHP-Open-PDK/tree/main).

**📄 [Read the Final Project Report (PDF)**](https://www.google.com/search?q=doc/final_report.pdf)

Building upon the PULP project's Croc SoC, this design introduces robust error mitigation for the SRAM banks without compromising the readability of the RTL and scripts.

## Architecture

The SoC is composed of the standard Croc domains, with the addition of the memory protection layer:

* The `croc_domain` containing a CVE2 core, an OBI crossbar, and standard peripherals.
* The **SEC-DED Module**, situated between the OBI crossbar and the SRAM banks. It encodes data during write operations (generating parity/syndrome bits) and decodes/corrects data during read operations.
* The `user_domain` where further custom accelerators or open-source designs can be integrated.

The main interconnect is OBI, detailed in [the official specification](https://github.com/openhwgroup/obi/blob/072d9173c1f2d79471d6f2a10eae59ee387d4c6f/OBI-v1.6.0.pdf). The various IPs are managed by [Bender](https://github.com/pulp-platform/bender), with the reliable memory protection code implemented entirely in SystemVerilog under `rtl/sec_ded/`.

## Configuration

The main SoC configurations, including memory protection parameters, are defined in `rtl/croc_pkg.sv`:

| Parameter | Default | Function |
| --- | --- | --- |
| `PulpJtagIdCode` | `32'h1C0C_5DB3` | Debug module ID code |
| `EnableSecDed` | `1` | Toggle hardware error correction and detection module |
| `NumSramBanks` | `2` | Number of memory banks |
| `SramBankNumWords` | `512` | Number of 32-bit words in a memory bank |
| `SyndromeBits` | `7` | Number of parity/syndrome bits allocated per 32-bit data word |
| `BootAddr` | `32'h1000_0000` | Default boot address set in 'soc_ctrl' register |

The SRAMs are instantiated via a technology wrapper (`tc_sram_impl`). In this SEC-DED implementation, the wrapper is extended to allocate the necessary physical memory width to accommodate both the 32-bit data payload and the corresponding syndrome bits for error correction.

## Bootmodes

Currently, the only way to boot is via JTAG.

## Memory Map

The memory map remains fully compatible with [Cheshire's memory map](https://pulp-platform.github.io/cheshire/um/arch/#memory-map). Accesses to the SRAM banks automatically route through the SEC-DED encoder/decoder logic transparently to the core.

| Start Address | Stop Address | Description |
| --- | --- | --- |
| `32'h0000_0000` | `32'h0004_0000` | Debug module (JTAG) |
| `32'h0200_0000` | `32'h0200_4000` | Bootrom |
| `32'h0204_0000` | `32'h0208_0000` | CLINT peripheral |
| `32'h0300_0000` | `32'h0300_1000` | SoC control/info registers |
| `32'h0300_2000` | `32'h0300_3000` | UART peripheral |
| `32'h0300_5000` | `32'h0300_6000` | GPIO peripheral |
| `32'h0300_A000` | `32'h0300_B000` | Timer peripheral |
| `32'h1000_0000` | `+SRAM_SIZE` | Memory banks (Protected by SEC-DED) |
| `32'h2000_0000` | `32'h8000_0000` | Passthrough to user domain |
| `32'h2000_0000` | `32'h2000_1000` | reserved for user ROM text |

## Flow

```mermaid
graph LR;
  Bender-->Yosys;
  Yosys-->OpenRoad;
  OpenRoad-->KLayout;

```

1. Bender provides the list of SystemVerilog files, including the SEC-DED logic.
2. Yosys parses, elaborates, optimizes, and maps the design to the IHP 130nm technology cells.
3. The netlist, constraints, and floorplan are loaded into OpenRoad for Place & Route.
4. The design def is read by KLayout, merging the geometry of the standard cells, SRAM macros, and the synthesized correction logic.

## Requirements

Synthesis and simulation rely on the open-source toolchain container maintained by Harald Pretl. Please refer to the [Tool Repository](https://github.com/iic-jku/IIC-OSIC-TOOLS). Supported version: 2025.12.

### ETHZ Systems

For development on ETHZ Design Center infrastructure, use the integrated `icdesign` cockpit to access the internal IHP PDK. Initialize the workspace directly inside the project directory:

```sh
# Ensure you are inside the checked-out repository
icdesign ihp13 -nogui

```

Alternatively, invoke the pre-installed OSIC tools container:

```sh
oseda -2025.12 bash

```

## Getting Started

A software example and fault-injection testbench are provided to verify the SEC-DED functionality.

To run the synthesis and place & route flow:

```sh
git submodule update --init --recursive
cd yosys && ./run_synthesis.sh --synth
cd ../openroad && ./run_backend.sh --all
cd ../klayout && ./run_finishing.sh --gds

```

To simulate the design (including error injection and correction logging):

```sh
cd sw && make all
cd ../verilator && ./run_verilator.sh --build --run ../sw/bin/helloworld.hex

```

For Questasim/Modelsim execution:

```sh
cd vsim && ./run_vsim.sh --build --run ../sw/bin/helloworld.hex

```

### Modifying the SEC-DED Logic

The core memory protection files are located in `rtl/sec_ded/`. If you alter the pipeline stages or syndrome generation matrix, re-generate the default synthesis file-list to ensure Bender captures the architectural changes:

```sh
cd yosys && ./run_synthesis.sh --flist
cd ../verilator && ./run_verilator.sh --flist

```

## Bender

This project relies on [Bender](https://github.com/pulp-platform/bender) for IP dependency management. The `Bender.lock` file contains the resolved dependency tree for the Croc SoC and the respective PULP components.

* `bender checkout`: Downloads specified commits into `.bender`.
* `bender update`: Re-evaluates the dependency tree and generates a new lock file. Always re-simulate the SEC-DED testbenches if you update the lock file.
* `bender vendor`: Used to manage local IP patches under `rtl/patches`.

## License

Unless specified otherwise in the respective file headers, hardware sources and tool scripts are licensed under the Solderpad Hardware License 0.51 (see `LICENSE.md`). Software sources are licensed under Apache 2.0.
