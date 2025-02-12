
# Lab 2

# Getting Started

- Start by __forking__ this repository, __not__ cloning it. This will create a copy that you own.
- Then, clone your forked copy into the `workspace`  folder. In other words, `cd` into `asic-tools/workspace`, and then `git clone <your forked copy>`
>#### Forking:
>![alt text](docs/fork.png)

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

We can also use the `dir::` preprocessor command to use wildcard matching. For example, `dir::rtl/*.sv` goes into the `rtl` directory and matches all files ending in `.sv`.

<table><tr><td>

```Json
"VERILOG_FILES": ["dir::rtl/*.sv",
 "dir::rtl2/*.sv"]
```

</td><td>

```Yaml
VERILOG_FILES:
- dir::include/*.svh
- dir::rtl/*.sv
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
        - The most important warnings are timing violations.

# Treasure Hunt

With your design successfully passed through the OpenLane flow, it is time to find some important statistics. Find and format a report on the following:

- Design and Core Area
    - What is the difference?
- 