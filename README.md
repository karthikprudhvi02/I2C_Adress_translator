# FPGA-Based I²C Address Translator

An FPGA/RTL project developed in response to an **FPGA Developer technical task from Vicharak Technologies**.

The original problem asks for an FPGA-based I²C address translator that allows two devices with the same default I²C address to coexist on the same system by exposing one device through a different virtual address.

## Problem Statement

Many I²C peripherals ship with a fixed or limited set of slave addresses. If two identical devices use the same address, an I²C master cannot uniquely select one of them on the same bus.

Example target use case:

- Device 1 default address: `0x48`
- Device 2 default address: `0x48`
- Device 1 remains accessible at `0x48`
- Device 2 is intended to be exposed through a virtual address such as `0x49`
- The FPGA sits between the system master and target device and handles the I²C transaction while preserving the transferred data

This project explores the required RTL building blocks for that translator using Verilog.

## Features Implemented in the Uploaded RTL

- I²C master finite-state machine
- I²C slave finite-state machine
- START and STOP handling
- 7-bit address capture/transmission
- Read/write transaction support
- ACK handling
- Multi-byte write support
- Shared open-drain SDA behavior using tri-state control
- Selectable I²C SCL generation
  - 100 kHz mode
  - 400 kHz mode
- Testbench-based verification
- Test traffic using both `0x48` and `0x49`

## Important Implementation Note

The current uploaded RTL contains the I²C master, slave, clock-generation, integration, and testbench logic required for the design.

The current `I2C_Slave.v` accepts both `0x48` and `0x49`, and the supplied testbench exercises transactions using those addresses. However, the uploaded `I2C_Translator.v` does **not yet contain an explicit upstream-to-downstream address-remapping block such as `0x49 -> 0x48` on separate I²C buses**.

Therefore, this repository should currently be described as an implementation of the **core I²C master/slave and transaction-handling architecture for the Vicharak address-translator task**, unless explicit address-remapping logic is added.

## Repository Structure

```text
.
├── I2C_Master.v
├── I2C_Slave.v
├── I2C_Translator.v
├── SCL_Generator.v
├── tb_clock.v
└── README.md
```

### `I2C_Master.v`

Implements the I²C master-side transaction FSM.

Main operations include:

- START generation
- Address + R/W transmission
- Address ACK sampling
- Write-data transmission
- Read-data sampling
- Multi-byte write sequencing
- STOP generation

The master shifts output data on SCL falling-edge events and samples incoming information around SCL rising-edge events.

### `I2C_Slave.v`

Implements the I²C slave-side transaction FSM.

Main operations include:

- START detection
- Address reception
- R/W-bit capture
- ACK generation
- Write-data reception
- Read-data transmission
- Multi-byte data collection
- STOP detection

In the current source, address matching accepts:

```verilog
7'b1001000   // 0x48
7'b1001001   // 0x49
```

### `SCL_Generator.v`

Generates the I²C serial clock used by the master.

`SCL_select` selects between the two supported operating modes:

- `0` -> 100 kHz target mode
- `1` -> 400 kHz target mode

### `I2C_Translator.v`

Top-level integration module connecting the I²C master and slave blocks.

It exposes:

- write data
- read data
- target address
- read/write control
- byte count
- selectable SCL mode

### `tb_clock.v`

Testbench for the integrated design.

The supplied test sequence includes:

- Reset
- Multi-byte write transaction
- Address `0x48`
- A later transaction using address `0x49`
- 32-bit write-data stimulus

## High-Level Architecture

```text
                  +----------------------+
                  |   I2C_Translator     |
                  |                      |
 Inputs --------> |   +-------------+    |
                  |   | I2C Master  |    |
                  |   +------+------+    |
                  |          | SDA/SCL    |
                  |   +------v------+    |
 Outputs <--------|   | I2C Slave   |    |
                  |   +-------------+    |
                  +----------------------+
```

For a complete physical address translator, the architecture should ultimately use separate upstream and downstream I²C interfaces:

```text
 System I²C Master
        |
        | Upstream I²C
        v
+-----------------------+
| FPGA Address          |
| Translator            |
|                       |
| Virtual 0x49          |
|      -> Physical 0x48 |
+-----------------------+
        |
        | Downstream I²C
        v
 Target Device @ 0x48
```

## FSM Concept

The master-side flow is broadly:

```text
IDLE
  -> START
  -> ADDRESS_SHIFTING
  -> ADDRESS_ACK
  -> WRITE / READ
  -> DATA_ACK
  -> STOP
```

The design uses FPGA-friendly structures such as:

- finite-state machines
- counters
- shift registers
- synchronized SCL edge detection
- tri-state SDA control

## Multi-Byte Transfers

The master accepts a 32-bit write-data input and uses `byte_count` to determine how many bytes are transmitted.

The slave stores received bytes into a multi-byte output register.

This extends the basic single-byte task toward multi-byte I²C communication.

## Simulation

The project is intended to be simulated using a Verilog simulator such as:

- Xilinx Vivado Simulator
- Icarus Verilog
- EDA Playground

Example with Icarus Verilog:

```bash
iverilog -o i2c_sim \
    I2C_Master.v \
    I2C_Slave.v \
    I2C_Translator.v \
    SCL_Generator.v \
    tb_clock.v

vvp i2c_sim
```

For waveform debugging:

```bash
gtkwave dump.vcd
```

A `$dumpfile` / `$dumpvars` block can be added to the testbench if waveform output is required.

## Original Vicharak Task Requirements

The technical task requested an FPGA module that:

- behaves as an I²C slave toward the bus master
- behaves as an I²C master toward the target device
- translates one target device address while preserving data
- supports bidirectional read/write communication
- maintains standard 100 kHz / 400 kHz I²C timing
- uses FPGA-friendly FSM/counter/shift-register logic
- is verified using simulation
- ideally supports multi-byte transactions

## Skills Demonstrated

- Verilog HDL
- RTL design
- FPGA design
- finite-state machines
- I²C protocol
- serial communication
- synchronous digital design
- counters and shift registers
- protocol-level debugging
- testbench development
- hardware verification

## Future Improvements

- Add two physically separate I²C interfaces:
  - upstream slave interface
  - downstream master interface
- Implement explicit address mapping, for example:
  - upstream `0x49`
  - downstream `0x48`
- Preserve the R/W bit while rewriting only the 7-bit address
- Add runtime-configurable virtual and physical address registers
- Complete repeated-START support across translated transactions
- Extend read-side multi-byte handling
- Add self-checking testbench assertions
- Add waveform screenshots
- Add Vivado utilization and timing reports
- Verify the design on FPGA hardware

## Background

This project was developed from a technical problem statement provided as part of the **Vicharak Technologies FPGA Developer application process**. The task focuses on solving I²C address conflicts using FPGA-based protocol handling.

## Author

**Karthik-Prudhvi**  
B.Tech, Electronics and Communication Engineering  
Indian Institute of Technology Patna

- GitHub: [Karthik-Prudhvi](https://github.com/karthikprudhvi02)
