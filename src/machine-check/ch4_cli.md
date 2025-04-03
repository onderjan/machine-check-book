# Command-line Interface

In the previous chapters, the Command-Line Interface was used for controlling **machine-check**. When [`machine_check::run`](https://docs.rs/machine-check/0.4.0/machine_check/fn.run.html) is called, it parses the program arguments and executes the verification accordingly. It is also possible to augment the arguments using [`machine_check::parse_args`](https://docs.rs/machine-check/latest/machine_check/fn.parse_args.html) and [`machine_check::execute`](https://docs.rs/machine-check/latest/machine_check/fn.execute.html), although the actual way this works may change.

The full list of arguments can be [found here](https://docs.rs/machine-check/latest/machine_check/struct.ExecArgs.html) or be printed using `--help`. The usually useful arguments are:
 - `--inherent`: Orders **machine-check** to verify the inherent property.
 - `--property <PROPERTY>`: Orders **machine-check** to verify the given property. The inherent property is verified beforehand by default.
 - `--assume-inherent`: Does not verify the inherent property before the given property. If the inherent property does not hold, the result of verifying the given property will be meaningless.
 - `--strategy <STRATEGY>`: The strategy **machine-check** should take to verify the property. The choice of the strategy can have a drastic impact on verification time and memory used. There are currently three strategies:
   - `naive`: Essentially constructs the state space of the system by brute force. Useless for most non-trivial systems as the state space is too big to use this strategy.
   - `default`: Uses [Three-valued Abstraction Refinement](../under_the_hood/research.md) with *input precision* only. This is the default as it is a reasonable choice for many systems.
   - `decay`: Uses a more aggressive abstraction and refinement approach with *decay* of computed states using *step precision*. This can verify some systems more easily than the default strategy, and is recommended if the default strategy does not produce a result in reasonable time.

The command-line interface is currently completely automatic: it starts verifying and returns the result once it is done (which, in many cases, may be after an infeasibly long time). Some progress information can be shown using the `-v` / `--verbose` flag (usable multiple times for more verbosity), but that is intended more as a debug tool and may slow down verification considerably, especially if the output is not rerouted to a file.

For a better insight into the workings of **machine-check**, [use the Graphical User Interface](./ch5_gui.md).
