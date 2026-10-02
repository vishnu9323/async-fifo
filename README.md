# Asynchronous FIFO — Verilog RTL

A parameterized **Asynchronous FIFO** implemented in synthesizable Verilog for reliable data transfer between two independent clock domains.

The design uses **Gray-coded read/write pointers** and **two-flop synchronizers** to safely communicate FIFO status information across asynchronous clock domains.

---

## Overview

An asynchronous FIFO is useful when a producer and consumer operate using different and unrelated clocks.

### Key Problem

Directly transferring multi-bit binary counters between asynchronous clock domains can lead to:

- Metastability
- Invalid intermediate values
- Incorrect full/empty detection
- Data corruption

This design addresses these issues using standard CDC techniques.

---

## Architecture

```text
                    ASYNCHRONOUS FIFO
        ┌─────────────────────────────────────┐
        │                                     │
        │          Dual-Port Memory           │
        │                                     │
        │    ┌──────────────────────────┐     │
        │    │                          │     │
wclk ───┼───►│       FIFO Memory        │◄────┼─── rclk
        │    │                          │     │
        │    └──────────────────────────┘     │
        │          ▲                ▲         │
        │          │                │         │
        │      Write Addr       Read Addr     │
        │          │                │         │
        │     ┌────┴─────┐    ┌────┴─────┐   │
        │     │  Write   │    │   Read   │   │
        │     │ Pointer  │    │  Pointer │   │
        │     │  + Full  │    │ + Empty  │   │
        │     └────┬─────┘    └────┬─────┘   │
        │          │                │         │
        │          ▼                ▼         │
        │      Gray Pointer      Gray Pointer │
        │          │                │         │
        │       ┌──┴────────────────┴──┐      │
        │       │   2-FF Synchronizers  │      │
        │       └───────────────────────┘      │
        │                                     │
        └─────────────────────────────────────┘
```

---

## Design Features

- Independent write and read clocks
- Parameterized data width
- Parameterized FIFO depth
- Gray-code pointer implementation
- Two-flop clock-domain synchronization
- Full and empty flag generation
- Separate read and write control logic
- Synthesizable RTL
- Self-checking simulation testbench
- Suitable for FPGA/ASIC-oriented RTL study

---

## RTL Structure

```text
Async_FIFO/
│
├── Verilog_code/
│   ├── FIFO.v
│   ├── FIFO_memory.v
│   ├── two_ff_sync.v
│   ├── rptr_empty.v
│   ├── wptr_full.v
│   └── FIFO_tb.v
│
├── README.md
└── ...
```

### Module Responsibilities

| Module          | Responsibility                       |
| --------------- | ------------------------------------ |
| `FIFO.v`        | Top-level integration                |
| `FIFO_memory.v` | FIFO storage / dual-port memory      |
| `two_ff_sync.v` | Clock-domain pointer synchronization |
| `rptr_empty.v`  | Read pointer and empty detection     |
| `wptr_full.v`   | Write pointer and full detection     |
| `FIFO_tb.v`     | Functional verification              |

---

## How It Works

### Write Side

The write logic operates entirely using `wclk`.

```text
Write Request
     │
     ▼
Is FIFO Full?
   │       │
  YES     NO
   │       │
 Ignore   Write Data
           │
           ▼
     Increment Pointer
           │
           ▼
       Gray Encode
```

A write is accepted only when the FIFO is not full.

---

### Read Side

The read logic operates entirely using `rclk`.

```text
Read Request
     │
     ▼
Is FIFO Empty?
   │       │
  YES     NO
   │       │
 Ignore   Read Data
           │
           ▼
     Increment Pointer
           │
           ▼
       Gray Encode
```

A read is accepted only when the FIFO is not empty.

---

## Why Gray Code?

Binary counters can change multiple bits during a single increment.

For example:

```text
Binary:

0111 → 1000
```

Four bits change simultaneously.

If this value is sampled by another asynchronous clock domain, the receiving logic may observe an inconsistent intermediate value.

Gray code changes only **one bit per increment**.

```text
Gray:

0100 → 1100
```

This makes Gray-coded pointers much safer to synchronize across clock domains.

---

## Clock-Domain Crossing

The write pointer is generated in the write-clock domain and synchronized into the read-clock domain.

Similarly, the read pointer is generated in the read-clock domain and synchronized into the write-clock domain.

```text
          WRITE DOMAIN
               │
        Write Gray Pointer
               │
               ▼
          ┌─────────┐
          │  FF1    │
          └────┬────┘
               │
               ▼
          ┌─────────┐
          │  FF2    │
          └────┬────┘
               │
               ▼
          READ DOMAIN
```

The two-flop synchronizer reduces the probability of metastability propagating into the destination logic.

---

## Full and Empty Detection

### Empty

The FIFO is empty when the next read pointer matches the synchronized write pointer.

```text
Read Pointer == Synchronized Write Pointer
                         │
                         ▼
                    EMPTY = 1
```

### Full

The FIFO is full when the next write pointer reaches the corresponding wrapped position of the synchronized read pointer.

The additional pointer bit distinguishes:

```text
Same address + same wrap count
        → EMPTY

Same address + different wrap state
        → FULL
```

---

## Verification

The testbench verifies the FIFO under multiple conditions.

### Test 1 — Normal Operation

```text
Write Data
    ↓
FIFO
    ↓
Read Data
    ↓
Compare
```

The testbench checks that the data read from the FIFO matches the data previously written.

### Test 2 — FIFO Full

The FIFO is filled until the `full` flag becomes active.

Additional write attempts are then performed to verify that the FIFO prevents writes when full.

### Test 3 — FIFO Empty

The FIFO is emptied until the `empty` flag becomes active.

Additional read attempts are performed to verify that invalid reads are prevented.

---

## Verification Flow

```text
             Reset
               │
               ▼
       Generate wclk/rclk
               │
               ▼
        Write Test Data
               │
               ▼
        Check FULL Flag
               │
               ▼
         Read Test Data
               │
               ▼
        Check EMPTY Flag
               │
               ▼
        Compare Data
               │
               ▼
       Simulation PASS
```

---

## Expected Behavior

| Condition             |    Write |     Read |
| --------------------- | -------: | -------: |
| FIFO Empty            |  Allowed |  Blocked |
| FIFO Partially Filled |  Allowed |  Allowed |
| FIFO Full             |  Blocked |  Allowed |
| Reset Active          | Disabled | Disabled |

---

## Simulation

The project can be simulated using tools such as:

- AMD Vivado Simulator
- Questa/ModelSim
- Other Verilog-compatible simulators

### Typical Vivado Flow

```text
Create/Open Project
        ↓
Add RTL Sources
        ↓
Add Simulation Sources
        ↓
Set FIFO_tb as Simulation Top
        ↓
Run Behavioral Simulation
        ↓
Inspect Waveforms
        ↓
Verify FULL / EMPTY / Data
```

---

## Important Waveform Signals

During simulation, the following signals are useful for debugging:

```text
wclk
rclk

winc
rinc

wdata
rdata

wptr
rptr

wfull
rempty

waddr
raddr
```

The most important checks are:

1. Data must not be written when `wfull = 1`.
2. Data must not be read when `rempty = 1`.
3. Write pointer must advance only on valid writes.
4. Read pointer must advance only on valid reads.
5. Gray pointers must cross clock domains through synchronizers.
6. Read data must preserve FIFO ordering.

---

## Key RTL Concepts Demonstrated

This project demonstrates practical RTL concepts including:

- Clock Domain Crossing (CDC)
- FIFO architecture
- Gray-code counters
- Pointer generation
- Dual-port memory
- Synchronizer design
- Full/empty detection
- Parameterized Verilog
- Testbench development
- Waveform-based debugging

This project can be used to discuss several important digital-design topics:

### 1. Why asynchronous FIFO?

To transfer data safely between two independent clock domains.

### 2. Why Gray code?

To minimize the possibility of incorrect pointer sampling during CDC.

### 3. Why two flip-flops?

To reduce the probability that metastability propagates into the destination clock domain.

### 4. Why an extra pointer bit?

To distinguish between the FIFO being completely empty and completely full when the memory addresses appear equal.

### 5. What happens when FIFO is full?

Further write operations are prevented until space becomes available.

### 6. What happens when FIFO is empty?

Further read operations are prevented until new data is written.

---

## Limitations

Functional simulation can verify logical behavior, but it cannot fully reproduce physical metastability behavior in hardware.

For hardware implementation, CDC constraints, timing constraints, reset strategy, and technology-specific memory inference should also be considered.

---

## Future Improvements

Possible extensions include:

- Add programmable almost-full flag
- Add programmable almost-empty flag
- Add occupancy monitoring
- Add stronger constrained-random verification
- Add SystemVerilog assertions
- Add functional coverage
- Add formal verification
- Implement on FPGA hardware
- Add synthesis and timing reports

---

## References

- AMD/Xilinx FPGA design documentation
- Verilog/SystemVerilog RTL design references

---

## Author

**Vishnu Prajapati**

Electronics & Communication Engineering
RTL Design | Verilog | FPGA | Digital Design | CDC

---

### Project Goal

The primary goal of this project is to demonstrate a practical understanding of **asynchronous data transfer, CDC-safe pointer synchronization, FIFO control logic, and RTL verification**.
