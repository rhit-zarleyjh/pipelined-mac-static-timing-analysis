# Pipelined MAC Timing Analysis

Implemented three functionally equivalent architectures that accept five numeric inputs `a, b, c, d, e` and compute `(a * b) + (c * d) + e`. This architecutre is known as multiply-accumulate (MAC). Each of the three implementations has a different pipeline depth, meaning the calculation is broken into different stages. I compared their timing, area, and latency values by using Yosys to synthesize each design, then using OpenSTA to reveal insights regarding timing.

This project explores the relationship between pipeline placement and critical-path delay, demonstrating that adding pipeline stages does not always improve maximum clock frequency. It 

## Architecture
Three implementations of the same arithmetic operation were compared:
- `mac_unpipelined` &ndash; MULT &rarr; ADD &rarr; ADD &rarr; REG
- `mac_pipe1` &ndash; MULT &rarr; REG &rarr; ADD &rarr; ADD &rarr; REG
- `mac_pipe2` &ndash; MULT &rarr; REG &rarr; ADD &rarr; REG &rarr; ADD &rarr; REG

Each implementation uses the same ready/valid interface with elastic handshakes and supports consumer stalling as well as backpressure. The implementations differ primarily in the number and placement of internal pipeline registers.

The additional registers divide the combinational arithmetic into progressively smaller paths at the cost of additional latency, sequential logic, and area.

## Verification
Each architecture was tested using a self-checking SystemVerilog testbench to verify arithmetic correctness and ready/valid behavior. AI assistance was used to generate and refine portions of the SystemVerilog verification testbench, including stimulus and protocol checks. The generated verification code was reviewed and exercised through simulation and regression.

The RTL architecture, synthesis/STA flow, timing constraints, critical-path analysis, and performance analysis were developed and evaluated separately.

Verification included:
- Correct arithmetic results
- Input acceptance only when `in_valid && in_ready`
- Output consumption only when `out_valid && out_ready`
- Consumer stalls and backpressure
- Bubbles between transactions
- Consecutive transactions
- Reset behavior
- Randomized traffic with reproducible seeds

A regression script was used to run repeated randomized simulations before synthesis and STA.

## Synthesis and Static Timing Analysis
The three architectures were synthesized and analyzed under a common flow so their timing and area results could be compared under equivalent assumptions.

### Tools
- **Verilator** &ndash; RTL simulation and verification
- **Yosys** &ndash; synthesis and technology mapping
- **OpenSTA** &ndash; static timing analysis
- **Nangate Open Cell Library** &ndash; standard-cell timing and area models

Each implementation was synthesized against the same standard-cell library and analyzed using the same timing assumptions.

### Timing Constraints
The clock period is supplied to the timing flow as a configurable parameter rather than being fixed in the SDC file.

The arithmetic inputs use the following interface timing assumptions:

```tcl
set_input_delay -clock clk -max 0.7 [get_ports {a b c d e}]
set_input_delay -clock clk -min 0.1 [get_ports {a b c d e}]
```
The `-max` input delay models the latest input arrival for setup analysis, while the `-min` input delay models the earliest arrival for hold analysis.

The 0.7 ns and 0.1 ns values are controlled interface assumptions for this experiment rather than measurements from a specific upstream block. These assumptions are applied to all three architectures.

### Timing Reports
The STA flow separately reports:
- Overall worst setup path
- Input-to-register setup path
- Register-to-register setup path

This distinction allows the design-level critical path to be identified while also allowing individual pipeline stages to be examined.

Full STA reports are saved for detailed inspection, while the terminal output is filtered for the startpoint, endpoint, arrival time, required time, and slack of the worst paths.

### Minimum Clock Period Search
`scripts/find_clkmin.sh` performs a binary search for the approximate minimum clock period that satisfies setup timing.

The search maintains:
- A variable lower bound that must fail timing
- A variable upper bound that must pass timing
- A variable search precision

For each candidate clock period, the script runs STA and examines the overall worst setup slack, moving the passing bound downward for a nonnegative slack and opposite for a negative slack.

This produces a bounded estimate of the minimum passing clock period and its corresponding estimated maximum frequency.

## Timing and Area Results

| Metric | Unpipelined | Pipe1 | Pipe2 |
|---|---:|---:|---:|
| Min. passing period (ns) | 2.07 | 1.83 | 1.84 |
| Estimated Fmax (MHz) | 483 | 546 | 543 |
| Cell count | 942 | 1123 | 1149 |
| DFF count | 19 | 69 | 104 |
| Total area | 1153 | 1532 | 1715 |
| Sequential area | 86 | 312 | 470 |
| Latency (cycles) | 1 | 2 | 3 |
| Critical path class | Input &rarr; Reg | Input &rarr; Reg | Input &rarr; Reg |

### Relative Tradeoffs

| Metric | Pipe1 vs. Unpipelined | Pipe2 vs. Unpipelined | Pipe2 vs. Pipe1 |
|---|---:|---:|---:|
| Fmax change | 13.0% | 12.4% | -0.5% |
| Total area change | 32.9% | 48.7% | 11.9% |
| Sequential area change | 262.8% | 446.5% | 50.6% |
| DFF count change | 263.2% | 447.4% | 50.7% |
| Latency change | +1 cycle | +2 cycles | +1 cycle |

## Critical-Path Analysis
The timing reports show how pipeline placement divided the arithmetic process across separate timing paths by introducing additional sequential boundaries.

### Unpipelined

The worst setup path was an input-to-register path beginning at `d[1]` and terminating at a bit of `abcde_s1`.

Conceptually:

```text
input → MULT / ADD arithmetic → result register
```

The arithmetic is completed before reaching the first datapath register, leaving the longest combinational chain of the three implementations.

### Pipe1

The worst setup path began at `b[2]` and terminated at `ab_s1[15]`, a register holding part of the multiplication result.

Conceptually:

```text
input → MULT → pipeline register
```

The first pipeline boundary therefore removed the downstream additions from the initial timing path.

The worst internal register-to-register path ran from a first-stage arithmetic register to the final arithmetic result register:

```text
pipeline register → ADD + ADD → result register
```

Despite this internal arithmetic stage, the input-to-register multiplication path remained the design-level critical path near the minimum passing clock period.

### Pipe2

The worst input-to-register path similarly began at `b[2]` and terminated at `ab_s1[14]`.

Conceptually:

```text
input → MULT → pipeline register
```

The additional pipeline stage split the downstream additions:

```text
pipeline register → ADD → pipeline register → ADD → result register
```

However, this did not shorten the already-critical input-to-first-register multiplication path. The additional pipeline stage did not materially improve the maximum clock frequency.

## Design Tradeoffs

The first pipeline stage produced a meaningful timing improvement. `mac_pipe1` increased estimated Fmax by approximately 13.0% relative to the unpipelined implementation, reducing the approximate minimum passing clock period from 2.07 ns to 1.83 ns.

This improvement required additional hardware. Relative to the unpipelined architecture, Pipe1 increased total area by 32.9%, increased DFF count from 19 to 69, and added one cycle of latency.

Pipe2 further divided the downstream addition logic, but the design-level critical path had already moved to the input-to-first-register multiplication stage. Because the additional pipeline boundary did not divide this limiting path, Pipe2 provided no meaningful Fmax improvement over Pipe1.

Instead, relative to Pipe1, Pipe2:

- Increased total area by 11.9%
- Increased sequential area by 50.6%
- Increased DFF count by 50.7%
- Added one additional cycle of latency
- Reduced estimated Fmax by approximately 0.5% in this synthesis/STA result

The small measured Fmax difference between Pipe1 and Pipe2 should not be interpreted as evidence that the additional stage inherently makes the design slower. The important result is that the two designs have approximately the same timing limit because they share essentially the same critical input-to-register stage.

Under the assumptions of this experiment, Pipe1 therefore provides the strongest performance/area/latency tradeoff of the three implementations. It captures the useful timing benefit of pipelining without paying for an additional pipeline boundary that does not shorten the limiting path.

More generally, the experiment demonstrates that pipeline depth alone does not determine maximum frequency. Pipeline registers improve timing only when their placement divides logic on a path that limits, or would otherwise limit, the clock period. Once the critical path moves elsewhere, further pipelining of noncritical logic may increase latency, area, and clocked state without increasing Fmax.

## Running the Flow

### Simulation and Synthesis

Run the standard flow for a particular implementation:

```bash
./scripts/run_flow.sh mac_pipe2 999
```
The optional second argument can specify a seed, and will default to 1.

### Regression Testing

Run randomized regression testing with the corresponding regression script.

```bash
./scripts/run_regression.sh mac_pipe2 10
```
The optional second argument determines the number of times the module will be tested, with a default value of 1.

### Static Timing Analysis

Run synthesis and STA at a specified clock period:

```bash
./scripts/run_timing.sh mac_pipe2 2.0
```

The second argument specifies the clock period in nanoseconds. If omitted, the timing script uses a fixed default of 5 ns.

The timing flow reports the worst overall, input-to-register, and register-to-register setup paths. Complete synthesis and STA reports are retained under `results/`.

### Minimum Clock Period Search

Search for the approximate minimum passing period:

```bash
./scripts/find_clkmin.sh mac_pipe1 0.01 1.0 3.0
```

Arguments are:

```text
find_clkmin.sh DESIGN [PRECISION] [LOW] [HIGH]
```

where:

- `DESIGN` selects the RTL implementation.
- `PRECISION` specifies the desired clock-period search resolution in nanoseconds.
- `LOW` specifies a clock period expected to fail timing.
- `HIGH` specifies a clock period expected to pass timing.

The script verifies the supplied bounds before performing the binary search.

Example output identifies the final failing and passing bounds, interval width, approximate minimum passing period, and estimated Fmax.

## Limitations

The timing results in this project are intended for **relative architectural comparison**, not as predictions of post-layout performance.

The analysis uses technology-mapped synthesis and pre-layout static timing analysis with the Nangate Open Cell Library. It does not represent a complete physical-design flow and does not include effects such as extracted routing parasitics, detailed placement and routing, clock-tree implementation, comprehensive process/voltage/temperature corner analysis, or a complete system-level I/O timing environment.

The input-delay constraints are controlled assumptions used consistently across the three architectures rather than delays derived from a specific upstream block.

As a result, the reported Fmax values should be interpreted as comparative estimates under a common timing model. The relative movement of critical paths and the resulting pipeline tradeoffs are the primary results of the experiment.