# Pipelined MAC Timing Analysis

Implemented three functionally equivalent architectures that accept five numeric inputs `a, b, c, d, e` and compute `(a * b) + (c * d) + e`. This architecture is known as multiply-accumulate (MAC). Each of the three implementations has a different pipeline depth, meaning the calculation is broken into different stages. I compared their timing, area, and latency values by using Yosys to synthesize each design, then using OpenSTA to reveal insights regarding timing.

This project explores the relationship between pipeline placement and critical-path delay, demonstrating that adding pipeline stages does not always improve maximum clock frequency.

In this specific case, adding a pipeline increased the maximum frequency by approximately **13%** at the cost of a **32.9%** increase in hardware area. Adding a second pipeline had no benefit to maximum frequency and increased the design's hardware requirements further by **11.9%**, or **48.7%** over the unpipelined design. 

This project taught me how to use static timing analysis to guide architectural decisions rather instead of relying solely on logically informed assumptions. By tracing the critical path as pipeline boundaries were added, I learned to distinguish between registers that divide timing-limiting logic or just add area and latency without addressing bottlenecks.

## Architecture
Three implementations of the same arithmetic operation were compared:
- `mac_unpipelined` &ndash; MULT &rarr; ADD &rarr; ADD &rarr; REG
- `mac_pipe1` &ndash; MULT &rarr; REG &rarr; ADD &rarr; ADD &rarr; REG
- `mac_pipe2` &ndash; MULT &rarr; REG &rarr; ADD &rarr; REG &rarr; ADD &rarr; REG

```
 UNPIPELINED                                                                               
                                                                                           
                                                ┌───┐                                      
 a ───────────►┌────┐                           │   │                                      
               │MULT├─┐                         │   │                                      
 b ───────────►└────┘ └──►┌───┐      ┌───┐      │   │                                      
                          │ADD├─────►│ADD├──────┤REG├──────► output                        
 c ───────────►┌────┐ ┌──►└───┘      └─┬─┘      │   │                                      
               │MULT├─┘                │        │   │                                      
 d ───────────►└────┘                  │        │   │                                      
                                       │        └───┘                                      
 e ────────────────────────────────────┘                                                   
                                                                                           
                                                                                           
 ──────────────────────────────────────────────────────────────────────────────────────────
 1-STAGE PIPELINE                                                                         
                                                                                           
                           ┌───┐                                                           
 a ───────────►┌────┐      │   │                            ┌───┐                          
               │MULT├──────┤REG├──┐                         │   │                          
 b ───────────►└────┘      │   │  │                         │   │                          
                           └───┘  │   ┌───┐      ┌───┐      │   │                          
                           ┌───┐  ├──►│ADD├─────►│ADD├──────┤REG├──────► output            
 c ───────────►┌────┐      │   │  │   └───┘      └───┘      │   │                          
               │MULT├──────┤REG├──┘                ▲        │   │                          
 d ───────────►└────┘      │   │                   │        │   │                          
                           └───┘                   │        └───┘                          
                           ┌───┐                   │                                       
                           │   │                   │                                       
 e ────────────────────────┤REG├───────────────────┘                                       
                           │   │                                                           
                           └───┘                                                           
                                                                                           
────────────────────────────────────────────────────────────────────────────────────────── 
 2-STAGE PIPELINE                                                                         
                                                                                           
                           ┌───┐                                                           
 a ───────────►┌────┐      │   │                                       ┌───┐               
               │MULT├──────┤REG├──┐              ┌───┐                 │   │               
 b ───────────►└────┘      │   │  │              │   │                 │   │               
                           └───┘  │   ┌───┐      │   │      ┌───┐      │   │               
                           ┌───┐  ├──►│ADD├──────┤REG├─────►│ADD├──────┤REG├──────► output 
 c ───────────►┌────┐      │   │  │   └───┘      │   │      └───┘      │   │               
               │MULT├──────┤REG├──┘              │   │        ▲        │   │               
 d ───────────►└────┘      │   │                 └───┘        │        │   │               
                           └───┘                              │        └───┘               
                           ┌───┐                 ┌───┐        │                            
                           │   │                 │   │        │                            
 e ────────────────────────┤REG├─────────────────┤REG┼────────┘                            
                           │   │                 │   │                                     
                           └───┘                 └───┘                                     
```

Each implementation uses the same ready/valid interface with elastic handshakes and supports consumer stalling as well as backpressure. The implementations differ primarily in the number and placement of internal pipeline registers.

The additional registers divide the combinational arithmetic into progressively smaller paths at the cost of additional latency, sequential logic, and area.

## Synthesis and Static Timing Analysis
The three architectures were synthesized and analyzed under a common flow so their timing and area results could be compared under equivalent assumptions.

### Tools
- **Verilator** &ndash; RTL simulation and verification
- **Yosys** &ndash; synthesis and technology mapping
- **OpenSTA** &ndash; static timing analysis
- **Nangate Open Cell Library** &ndash; standard-cell timing and area models

Each implementation was synthesized against the same standard-cell library and analyzed using the same timing assumptions.

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

## Limitations

The timing results in this project are intended for **relative architectural comparison**, not as predictions of post-layout performance.

The analysis uses technology-mapped synthesis and pre-layout static timing analysis with the Nangate Open Cell Library. It does not represent a complete physical-design flow and does not include effects such as extracted routing parasitics, detailed placement and routing, clock-tree implementation, comprehensive process/voltage/temperature corner analysis, or a complete system-level I/O timing environment.

The input-delay constraints are controlled assumptions used consistently across the three architectures rather than delays derived from a specific upstream block.

As a result, the reported Fmax values should be interpreted as comparative estimates under a common timing model. The relative movement of critical paths and the resulting pipeline tradeoffs are the primary results of the experiment.