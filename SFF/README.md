# Combining VUnit tests with cocotb components

## Introduction

The dialogues in [VUnit/vunit#651](https://github.com/VUnit/vunit/issues/651) and [cocotb/cocotb#2381](https://github.com/cocotb/cocotb/issues/2381) lead to two possible approaches for combining VUnit and cocotb:

- Let cocotb borrow some of VUnit's classes and overload/modify them in cocotb's repository or in another location external to VUnit. In this context, cocotb would replace VUnit's HDL runner; therefore, cocotb's regression management features would be preserved. Code borrowed from VUnit would be used for compiling HDL sources in the correct order and incrementally, thus replacing the Makefiles that cocotb's users do currently rely on. Other than that, regular HDL testbenches used by VUnit users would be independent and not compatible with cocotb testbenches.
- Treat cocotb as an optional feature in VUnit, which allows enhancing regular HDL testbenches. In this context, VUnit would be kept almost untouched. Existing options would be used for configuring cocotb as any other VHPI/VPI module written in C/C++ or in any other language. cocotb's regression manager or report generation features would not be used, because VUnit's HDL runner would be used. As a result, testbenches would naturally combine HDL and cocotb (Python) features, which would require cocotb users to understand some HDL used in the testbenches.

@ktbarret is working on the first approach in [ktbarrett/vunit-cocotb](https://github.com/ktbarrett/vunit-cocotb). This example is a variation of the content in that repository, for exploring the second approach.

## Proof of concept

- [src/dff.vhd](src/dff.vhd) is the UUT, which should remain unmodified, regardless of the verification approach. The regular workflow in VUnit requires the designer to instantite the UUT in an HDL testbench, which is the top level unit in the simulation. With cocotb, a Python testbench is used for driving the ports of the UUT, which is the top level unit in the simulation. That is, in a regular cocotb workflow, no HDL testbench exists.
- [test/tb_dff.vhd](test/tb_dff.vhd) is a regular VUnit testbench, where the UUT is instantiated and corresponding signals are declared in the testbench. Two named tests are defined. Test *VUnit* is a regular HDL test procedure, using some of VUnit's check VHDL library. On the other hand, test *cocotb* is a placeholder with a `5ms` delay/latency (`wait`).
- [test/tb_dff_cocotb.vhd](test/tb_dff_cocotb.vhd) is a copy of `test/tb_dff.vhd`, but test *cocotb* is defined only, and all the testbench signals are undriven.
- [run.py](run.py) is a regular VUnit run script, where `set_sim_option` and `set_pre_config` features are used for configuring cocotb modules in specific tests only (the ones named *cocotb*).
- [test/dff_cocotb.py](test/dff_cocotb.py) is the cocotb co-simulation logic. This is unmodified (yet).

There is a CI workflow for executing all the tests (2 in `tb_dff.vhd` and 1 in `tb_dff_cocotb.vhd`). See [test.yml](../.github/workflows/test.yml) and [github.com/umarcor/osvb/actions](https://github.com/umarcor/osvb/actions).

### To Do

- cocotb can force signals by wrapping values in `Force()`. See [docs.cocotb.org: Quickstart Guide > Assigning values to signals](https://docs.cocotb.org/en/stable/quickstart.html#assigning-values-to-signals). Therefore, it should be possible to override the HDL description of signal `c` in `tb_dff.vhd`. I.e., to drive the simulation time through cocotb, while preserving other behavioural descriptions in the HDL testbench.

## Conclusions and future work

- It seems that cocotb is unaware of the difference between the port of a UUT or the signal of a testbench. That is `dut.c` works either when `c` is a port in `dff.vhd` as the top level unit in the simulation (as seen in regular cocotb executions), or when `c` is a signal in `tb_dff.vhd`/`tb_dff.vhd`. This is shown in the tests executed with GHDL, thus, using the VPI interface. Results might be different with other implementations of VPI or with simulators using VHPI.
  - Nonetheless, a relevant conclusion is that vanilla VUnit and cocotb can already be combined. **cocotb can be used for driving some or all the signals in an otherwise regular HDL testbench**. As long as the port names of the UUT are copied as signal names in the testbench, existing cocotb Python scripts might be preserved unmodified.
- Simulation termination synchronisation is not possible yet. VUnit requires `test_runner_cleanup` to be executed at the end of the HDL testbench, in order to cleanup and terminate the simulation. At the same time, cocotb forces the simulator to shut down through VPI/VHPI. Therefore, either cocotb finalises the execution and VUnit produces a failure, or VUnit finalises the execution before cocotb is done.
  - In the current examples VUnit's private procedure `runner_pkg.p_disable_simulation_exit` is used. This prevents `test_runner_cleanup` from finalising the simulation, and cocotb is ensured to finish. However, this is just a workaround for having green CI runs, because the termination status of cocotb is not passed to VUnit.
  - **Synchronization might be solved by making *cocotb* tests in HDL testbenches wait for an specific signal to change**. Then, cocotb would set that signal through VHPI/VPI, instead of sending the shut down order to the simulator. However, **cocotb does currently not allow waiting for the simulator to finish by itself or registering custom shutdown callbacks**. It assumes to be in charge of the top level unit, and of the simulation.
  - Should cocotb allow setting a signal, **the value of that signal might be used for reporting test exit codes to VUnit**. The type of the signal might be an array, for reporting multiple tests at the same time.
- The current example is for describing a single test using cocotb. However, cocotb supports defining multiple tests in the same script. Since VUnit identifies each test with a string in the HDL testbench, it would be interesting to investigate whether cocotb can read the name of the active test through VHPI/VPI. Then, users might decide to describe one or multiple cocotb tests per each VUnit test, and/or to skip some of them.
  - Therefore, having a single HDL test named `cocotb` might suffice for running any numbers of cocotb tests.
  - Furthermore, VUnit's Python interface allows registering tests programmatically. That is, any number of custom tests with specific generics might be defined in `run.py`, which would then be matched by cases in `dff_cocotb.py`.
- Although instantiating the UUT in an HDL testbench is natural for VUnit users, it might be cumbersome for cocotb only users. Unfortunately,cocotb cannot declare additional signals in the testbench (which are not declared in HDL). It can neither instantiate itself as a component or instantiate some other component that exists in a previously analised library, or connect the signals to the component. cocotb runs after elaboration, and GHDL's VPI works after elaboration only. Therefore, any solution for adding content to an existing HDL testbench needs to be done by code-generation, which is not the nicest solution given the VHDL parsers currently available in Python. That is, have a script read the ports of the UUT, their modes and types. Then, generate a testbench with the signals, the instantiated UUT and the port map between them.
  - [ghdl/ghdl#1449](https://github.com/ghdl/ghdl/pull/1449) is a recent exercise for reading the ports, modes and types using GHDL's Python interface to `libghdl`. See [ghdl/ghdl: pyGHDL](https://github.com/ghdl/ghdl/blob/master/pyGHDL).
- cocotb generates reports indepently from VUnit. It would be interesting to investigate how to make VUnit pass the location of the output, or to support cocotb using VUnit's logging library through VHPI/VPI.

## References

- [jwprice100/vcst](https://github.com/jwprice100/vcst)
