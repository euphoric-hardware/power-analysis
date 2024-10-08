# Building a power analysis tool from the ground up

- [liberty file format](https://people.eecs.berkeley.edu/~alanmi/publications/other/liberty07_03.pdf)

## Leakage power

- The very first step here is to see if we obtain the static power analysis that Genus provides after synthesis
    - Obtain a gate level netlist from Genus by going through synthesis
    - See if we can map it into a BLIF file by using Yosys & parse it using the blif parser
    - Parse the liberty file of the standard cell library (liberty file is for a particular temp & voltage)
    - Add all the leakage power values from the lib file and compare it with what Genus reports
    - [liberty file example](https://github.com/erihsu/liberty2json/blob/main/examples/latch_cell.lib#L8)
        - If I look into a example liberty file, it seems like the `leakage_power` field actually depends on the input values of a standard cell
        - Can having different input values slightly change the cell circuit characteristics so that the leakage power is altered?
        - If this is truly the case, leakage power estimation may not be as easy as summing numbers from the liberty file?

## Dynamic power

- Dynamic power is induced by the switching of transistors causing capacitors to drain/fill
- The first order approach
    - Parse the VCD from RTL simulation to obtain the switching activity of each signal
    - From the liberty file we can obtain the capacitance of each gate
    - We need to find correlation points in between the gate level netlist and the RTL (to propagate switching activity). This may not be straightforward...
    - Some math to compute the dynamic power (seems like we can just use the `fall_capacitance` and `rise_capacitance`?)

## Glitching power

- Now things become a bit more tricky as we have to perform timing annotated gate level simulation
- This requires a event driven simulation framework to be built (separate section on the event driven simulation framework)
- But assuming that we have the event driven simulation framework and can initialize the simulation using the results from RTL sim, we can now estimate glitching power
- `internal_power` field in the liberty file
    - Describes the power consumed during transitions
    - The `rise_power` and `fall_power` tables provide values for power consumption during rising and falling transitions
    - Indexed by input transition times and output load capacitances

---

## Event driven simulation framework

### Building a reference from VCS

#### First Checkpoint

VCS is an event-driven simulator for Verilog (RTL and gate-level (GL)).
We want to use VCS to perform fast, parallel gate-level simulations given a RTL simulation waveform.
The first step is to get comfortable with the flow of running RTL simulation, running synthesis, and running a full gate-level simulation with VCS.

You can use the instructional machines (`eda-[1-4].eecs.berkeley.edu`).
You may need to add something to your `~/.bashrc` (VCS is located in: `/share/instsww/synopsys-new/vcs/T-2022.06-SP2-9/bin`).

<!--
```bash
export SYNOPSYS_ROOT=/share/instsww/synopsys-new
export VCS_HOME=/share/instsww/synopsys-new/vcs/T-2022.06-SP2-9
export VCS_PATH=$VCS_HOME/bin
export PATH="$VCS_PATH:$PATH"
export LM_LICENSE_FILE="5280@bisc.EECS.Berkeley.EDU:$LM_LICENSE_FILE"
export SNPSLMD_LICENSE_FILE=27005@license-srv.eecs.berkeley.edu
export LM_PROJECT=bwrc_users
```
-->

Next, you can use Hammer to actually run RTL simulation, yosys for synthesis, and VCS for gate-level simulation.
I would reference the [EECS151 ASIC labs](https://inst.eecs.berkeley.edu/~eecs151/fa24/static/asic/lab2/) to do this.
**Start with the lab files themselves, and run through the lab**.
You will notice some PDK files here: `/home/ff/eecs151/fa24`.
Then, try to get the same flow working within [Hammer alone using the e2e directory](https://github.com/ucb-bar/hammer/tree/master/e2e).

All we want to see for this first checkpoint is a waveform from RTL simulation, the synthesized gate-level Verilog, and a gate-level simulation which also produces a waveform.
You can use any RTL you want, for instance, the RTL from the EECS151 labs.
Once you can produce all the collateral above, we can move onto the next step.

#### Injecting State

We propose to speed up gate level simulation by parallelizing it aggressively using an RTL waveform.
This idea isn't new at all (see GATSPI). In fact, it might even be implemented within Joules when it performs RTL stimulus propagation for its internal gate level netlist.
But the problem is the opaqueness of what Joules is doing - we don't trust it at all.
And there is very likely an opportunity to get larger speedups too.

To get started, we need a way to wrap a gate level netlist with a test harness which can read circuit inputs from a file and blast them into the DUT.
We also need a way to force the initial state of registers within the gate level netlist with the values from a given cycle of RTL simulation.
For the former, we need to generate a Verilog testharness which can ingest some text file and drive DUT inputs.
For the latter, we can use UCLI to force register states. This is implemented in Hammer's gate level simulation flow, so we can adopt the same methodology.


### Building the main event driven gate level simulation framework

- Read the following two papers to understand the general concept of gate level sim & power estimation
    - [GATSPI](https://research.nvidia.com/publication/2022-03_gatspi-gpu-accelerated-gate-level-simulation-power-improvement)
    - [Grannite](https://research.nvidia.com/publication/2020-07_grannite-graph-neural-network-inference-transferable-power-estimation)
    - [basic techniques of gate level simulation](https://lume.ufrgs.br/bitstream/handle/10183/126714/000103176.pdf?sequence=1&isAllowed=y)
        - Describes datastructures and considerations when it comes to implementing a timing-annotated gate level simulation framework
            - **Read chapters 7, 9, 10.1, 10.2, 10.3, 10.4**
- Generate a netlist using [yosys](https://github.com/YosysHQ/yosys)
- Parse the generated netlist [blif-parser](https://github.com/ucb-bar/blif-parser)
- Parse liberty files to obtain the gate level delay [liberty-parse](https://crates.io/crates/liberty-parse)
- Implement timing-annotated gate level simulation framework

- Start with a GCD

```
// GCD.sv
// Generated by CIRCT firtool-1.62.0
module GCD(
  input        clock,
               reset,
  input  [1:0] io_value1,
               io_value2,
  input        io_loadingValues,
  output [1:0] io_outputGCD,
  output       io_outputValid
);

  reg [1:0] x;
  reg [1:0] y;
  always @(posedge clock) begin
    if (io_loadingValues) begin
      x <= io_value1;
      y <= io_value2;
    end
    else if (x > y)
      x <= x - y;
    else
      y <= y - x;
  end // always @(posedge)
  assign io_outputGCD = x;
  assign io_outputValid = y == 2'h0;
endmodule
```

- You can install Yosys using conda: [yosys feedstock](https://github.com/conda-forge/yosys-feedstock)
- Some Yosys commands to parse & perform techmapping

```
read_verilog GCD.sv
hierarchy -check -top GCD
proc; opt; memory; opt; fsm; opt; techmap; opt;
write_blif GCD.blif
```

- Parse the `GCD.blif` file and you can obtain a gate level netlist
- Parse the liberty file
- Given a input stimuli (can be a input file containing the input values for a particular cycle), write a simulator that can perform event driven simulation
    - Essentially implement the 2D linked list datastructure in "basic techniques of gate level simulation"
