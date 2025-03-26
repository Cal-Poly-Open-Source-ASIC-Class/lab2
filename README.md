
# Lab 2

# Getting Started

- Start by __using this template__, __not__ cloning it. This will create a copy that you own.
- Then, clone __your copy__ into the `workspace`  folder. In other words, make sure you are in `~/workspace`, and then `git clone <your forked copy.git>`
>#### Using as Template:
>![alt text](docs/template.png)

>#### Cloning:
>![alt text](docs/clone.png)

# Lab 2 Specification

In this lab, you will take your accelerator design from lab 1 and take it from design (RTL) to manufacturable chip (GDSII) using the OpenLane toolflow. You will then run post-layout simulation to ensure your laid out chip matches your behavioral verilog.

## OpenLane

Openlane contains every tool needed to take your system verilog to a completed chip, provided you configure it with a `json` or `yaml` file. Create a `config.json` or `config.yaml`. (Most documentation is in `json`, but `yaml` allows comments). Here is an example:

<table><tr><td> Json </td> <td> Yaml </td></tr><tr><td>

```Json
{
    "DESIGN_NAME": "name",
    "VERILOG_FILES": "dir::rtl/*.sv",
    "CLOCK_PERIOD": 10,
    "CLOCK_PORT": "clk",
    "FP_PDN_VOFFSET": 7,
    "FP_PDN_HOFFSET": 7,
    "FP_PDN_SKIPTRIM": true,
    "RUN_POST_GRT_RESIZER_TIMING": true,
    "RUN_POST_GRT_DESIGN_REPAIR": true
}
```

</td><td>

```Yaml
---
DESIGN_NAME: name
VERILOG_FILES: dir::rtl/*.sv
CLOCK_PERIOD: 10
CLOCK_PORT: clk
# Power Distribution Stuff
FP_PDN_VOFFSET: 7
FP_PDN_HOFFSET: 7
FP_PDN_SKIPTRIM: true
RUN_POST_GRT_RESIZER_TIMING: true
RUN_POST_GRT_DESIGN_REPAIR: true
```

</td></tr></table>

For a more complete example, check out [Openlane's Example config.json](https://github.com/efabless/openlane2-ci-designs/blob/da5ed2cae9da72290c6fc016b2d19cd2b8914bae/spm/config.json)

### Including Verilog Files

Verilog Files can be discretely named one by one as a list. 

<table><tr><td>

```Json
 "VERILOG_FILES": ["rtl/file1.sv",
  "rtl/file2.sv"],
```

</td><td>

```Yaml
VERILOG_FILES:
- rtl/file1.sv
- rtl/file2.sv
```

</td></tr></table>

We can also use the `dir::` preprocessor command to use wildcard matching. For example, `dir::rtl/*.sv` goes into the `rtl` directory and matches all files ending in `.sv`. Only use this if you have a flat directory structure, meaning all of your verilog files are in one folder.

<table><tr><td>

```Json
"VERILOG_FILES": "dir::rtl/*.sv",
```

</td><td>

```Yaml
VERILOG_FILES: dir::rtl/*.sv
```

</td></tr></table>

### Clocking

The `CLOCK_PERIOD` variable controls the speed target of your design by setting the target period in nanoseconds. 
- For the Skywater 130nm PDK, a typical starting value would be between 100 and 10 ns, or between 10 and 100 MHz. 
- Start your clock period conservatively and lower it until you find your maximum frequency, the frequency at which you start to get warnings about setup time.


### Variables
When configuring the openlane flow, nearly everything is configurable.   
- [Common Variables](https://openlane2.readthedocs.io/en/latest/reference/common_flow_vars.html)
    - Most Common Variables
- [PDK-Specific Variables](https://openlane2.readthedocs.io/en/latest/reference/common_pdk_vars.html)
    - Some variables are specific to the PDK being used (Skywater 130, Global Foundries 180, etc.)
- [All of the Variables](https://github.com/The-OpenROAD-Project/OpenLane/blob/master/docs/source/reference/configuration.md)
    - Can't find a variable? It is probably described here

## Running the Flow
With your configuration done, it is time to run the tools. 

```
openlane --flow Classic <config.json or config.yaml> 
```

Or using the provided Makefile:
`make openlane`
- The makefile creates a link to the most recent run in `runs/recent` to make results easier to find.

### Interpreting Results

- Openlane outputs the results of the flow in the `runs` into a folder tagged with the time/date
- Each step of the flow is numbered
    - `runs/<runDate>/01-verilator-lint` has the results from the first step, linting with verilator
- A sucessful flow will create a `final` directory with output results
    - `runs/<runDate>/error.log` should be empty
    - `runs/<runDate>/warning.log` ideally should be empty, but often is not
        - The most important warnings are timing violations. Keep your eyes out for `Setup violations found` or similar.
- A failure will populate the aformentioned `error.log`, and no `final` directory will be 
    - Check the last (highest-numbered) step for more specific logs to see where things went wrong.

### Viewing the Chip

There are many ways to view your design, including:
- `make openroad` to auto view your last design
- `openroad`: launch with `openroad -gui` and open `runs/<run>/final/odb/<design>.odb`
- `magic runs/<run>/final/mag/<design>.mag` to launch magic
    - Select your design by drawing a box around it. Then type `expand all` in the magic console to see all the cells.
- `klayout runs/<run>/final/klayout_gds/<design>.klayout.gds` to view in KLayout

### Static Timing Analysis (STA)

1. Create the variable `SYNTH_AUTONAME` and set it to true to your config. Then Re-Run the flow.
    - This ensures logic and wires are named based on your verilog instead of arbitrarily numbered. 
    - Important for when we start tracing out timing paths.
    - Not recomended to have this turned on all the time, can cause instability in large designs.
2. Re-Run the flow (`make openlane` or likewise)
3. Open the timing results:
    - `openroad-staprepnr` (step 12) has before-layout timing
    - `openroad-stapostpnr` (step 56) has after-layout timing
    - Within each, `summary.rpt` has a summary of the worst slack (`WS`) and total negative slack (`TNS`)

>[!NOTE] 
>Timing analysis uses __corners__ to cover the slowest and fastest possible performance of your chip, based on fabrication variation and conditions.
>  
>| Process Variation | Temperature   | Voltage   |
>| ----------------- | ------------- | --------- |
>| ss (slow)         | 100 C         | 1.6 V     |
>| tt (typical)      | 40 C          | 1.8 V     |  
>| ff (fast)         | 25 C          | 1.95 V    |
> 
> - __Setup Violations__ occur when your chip's logic is too slow to meet the target frequency
>   - Most likely in slowest corner, `ss_100C_1.6V`
>   - Concerned with "Maxs Paths", longest and slowest paths between registers
> - __Hold Violations__ occur when your chip's logic is too fast, causing a register's input not to be stable.
>   - Most likely in fastest corner, `ff_25C_1.95V`
>   - Concerned with "Min Paths", shortest and fastest paths between registers

4. View the Max Path results in the slowest corner
    - `runs/<run>/56-openroad-stapostpnr/max_ss_100C_1v60/max.rpt` shows the worst max paths.

5. Open your design in `openroad` to visually view the worst max paths.
    - `make openroad` will launch openroad with timing on your most recent run.
    - Navigate to the timing report tab on the right and then click update.
    - Click on your worst paths to see them visualized on your laid out chip.
    - To use openroad manually, launch `openroad` from the command line and input the following commands:
    ```
    read_db runs/recent/final/odb/<name>.odb
    read_lib /foss/pdks/sky130A/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__ss_100C_1v60.lib
    read_sdc runs/recent/final/sdc/<name>.sdc
    gui::show
    ```

### Gate Level Testing

Once your design has completed through the flow (or at the very least passed synthesis), you will receive a __gate-level netlist__, which is a verilog file of your design using only standard cell gates. 
- It can be found in the `pnl` folder of your final results
- In other words, look in `runs/recent/final/pnl/`

This file provides a more accurate model of your design as it exists on hardware. It is important to make sure our designs work not just in the simulator but also after synthesis, which is why we often re-run our tests against the gate-level netlist model, called gate-level simulation. For gate-level simulation, we have to use Icarus (iverilog) instead of verilator, since verilator does not support certain gate-level features.

#### Powering
This model also needs to be powered to function correctly. Add the following to the global scope all of your testbenches, so that Power and Ground wires are added when necessary. 

```verilog
`ifdef USE_POWER_PINS
    wire VPWR;
    wire VGND;
    assign VPWR=1;
    assign VGND=0;
`endif

```
#### Automatic Gate Level Tests
To automatically run all tests against your most recent openlane run's gate-level model, use: 
```
make gl_tests
```
After running this, you may run a specific test with
```
GL=1 make tests/<testname>
```
#### Manual Gate Level Tests
To manually run tests against your gate-level model:
- Create a directory called `gl` (try `mkdir gl`) and copy your model file `<design>.pnl.v` into that folder. 
- Then, add the following line to the top of the netlist (`<design>.pnl.v`):
```verilog
`define UNIT_DELAY #1
`define USE_POWER_PINS
`define FUNCTIONAL
`include "libs.ref/sky130_fd_sc_hd/verilog/primitives.v"
`include "libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v"
```
- This tells the simulator where to look for definitions of standard cells.
- Finally, run tests using `GL=1 make tests` or `GL=1 make tests/<testname>`


# Deliverables

With your design successfully passed through the OpenLane flow, it is time to find some important statistics. Find and format a report on the following:

- Design Pictures
- Maximum Frequency
    - Iterate over your design, lowering period until you start to hit timing warnings.
- Critical Path
    - After inspecting the post place-and-route (pnr) timing analysis (sta), what signals were involved in your critical path?
    - What line(s) of your verilog code created this critical path? This may take some thinking.
- Gate Level Tests
    - In-lab demo of successful tests
- Design Area and Core Area 
    - Check the `final/metrics.json` of your run. Note the units are square microns / square um.
    - What % of a [Tiny Tapeout](https://tinytapeout.com/) tile is this? Look for dimensions in `um`.
    - What % of an [Efabless Chipignite](https://efabless.com/products) die is this? Look for area in `sq mm`.
