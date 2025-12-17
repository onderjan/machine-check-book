# machine-check-riscv

[**Machine-check-riscv**](https://docs.rs/machine-check-avr/latest/machine_check_riscv/) provides a description of a machine-code system based on the [Renesas R9A02G021CNE](https://www.renesas.com/en/products/r9a02g021) microcontroller (as used in the [Renesas FPB-R9A02G021](https://www.renesas.com/en/design-resources/boards-kits/fpb-r9a02g021)
prototyping board) with a 32-bit RISC-V core.

A compiled binary [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) file must be provided as an argument to  **machine-check-riscv**. DWARF symbols within the binary can also be used within custom property macros. 

## Usage example

Consider the following basic example of C code that can be run on the FPB-R9A02G021 board, which manipulates the port 1 in the following fashion. Pin 0 and 7 (which control LEDs on the board) are set to output. Then, in a loop, the outputs for LEDs are manipulated so that the pin 0 LED is lit (logic high) whether the button is pressed (pin 8 logic low) and pin 7 LED toggles on each iteration.

```c
(...comments...)
// Whether the button has been pressed.
volatile unsigned short button_pressed;

// Port 1 output data register.
volatile unsigned short *PORT1_PODR = (volatile unsigned short*) 0x40040020;

// Port 1 data direction register.
volatile unsigned short *PORT1_PDR = (volatile unsigned short*) 0x40040022;

// Port 1 input data register.
volatile unsigned short *PORT1_PIDR = (volatile unsigned short*) 0x40040026;


int main(void) {
	// make port 1 pin 0 and 7 into an output
    main_start: *PORT1_PDR = *PORT1_PDR | (1 << 7) | 1;

    main_loop: while(1) {
    	// the button is pressed whenever port 1 pin 8 is zero
    	assign1: button_pressed = ((*PORT1_PIDR & (1 << 8)) == 0);

    	// set port 1 pin 0 output value to button_pressed
    	// this makes the LED light up whenever the user button is pressed
    	// on the development kit
        assign2: *PORT1_PODR = (*PORT1_PODR & ~1) | button_pressed;

        // invert the port 1 pin 7 output value
        assign3: *PORT1_PODR = *PORT1_PODR ^ (1 << 7);
    }
    // should never be reached
    main_return: return 0;
}
```

**Machine-check-riscv** can be installed as an executable using cargo:
```console
cargo install machine-check-riscv
  Downloaded machine-check-riscv v0.7.1
  Downloaded 1 crate (95.3KiB) in 0.10s
    Updating crates.io index
  Installing machine-check-riscv v0.7.1
    Updating crates.io index
(...)
   Compiling machine-check v0.7.1
   Compiling machine-check-riscv v0.7.1
    Finished `release` profile [optimized] target(s) in 43.89s
warning: the following packages contain code that will be rejected by a future version of Rust: partitions v0.2.4
note: to see what the problems were, use the option `--future-incompat-report`, or run `cargo report future-incompatibilities --id 1`
  Installing (...)\.cargo\bin\machine-check-riscv.exe
   Installed package `machine-check-riscv v0.7.1` (executable `machine-check-riscv.exe`)
```

Download the file [basic.elf](../data/riscv/basic.elf) which has been compiled from the above code by Renesas E<sup>2</sup> Studio for the FPB-R9A02G021 board using the RV32IC profile (without additional RISC-V extensions). We can verify that the inherent property holds first:

```console
$ machine-check-riscv -E basic.elf --inherent
[2025-12-17T14:41:35Z INFO  machine_check] Starting verification.
[2025-12-17T14:41:35Z INFO  machine_check::verify] Verifying the inherent property.
[2025-12-17T14:41:35Z INFO  machine_check] Verification ended.
+--------------------------------+
|         Result: HOLDS          |
+--------------------------------+
|  Refinements:               0  |
|  Generated states:        564  |
|  Final states:            563  |
|  Generated transitions:   564  |
|  Final transitions:       564  |
+--------------------------------+
```

To do something more interesting, let's verify that the execution always reaches a state where it and all subsequent states are within function `main` (less precisely, reaches `main` and stays within it).

```console 
$ machine-check-riscv -E basic.elf --property 'AF![AG![pc_within_symbol!("main")]]'
[2025-12-17T14:48:26Z INFO  machine_check] Starting verification.
[2025-12-17T14:48:26Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-12-17T14:48:26Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-12-17T14:48:26Z INFO  machine_check::verify] Verifying the given property.
[2025-12-17T14:48:33Z INFO  machine_check] Verification ended.
+--------------------------------+
|         Result: HOLDS          |
+--------------------------------+
|  Refinements:               0  |
|  Generated states:        564  |
|  Final states:            563  |
|  Generated transitions:   564  |
|  Final transitions:       564  |
+--------------------------------+
```

>
> &#x1F4A1;&#xFE0F; Depending on your terminal, you might have to escape the inner double quotes inside the single quotes.
>

The macro `pc_within_symbol` is a *custom property macro* that **machine-check-riscv** exposes when parsing properties, and which is expanded based on the [DWARF](https://en.wikipedia.org/wiki/DWARF) symbols exposed within the ELF file. Of course, it is not possible to use symbols that are not exposed (or understood by **machine-check-riscv**).

```console
$ machine-check-riscv -E basic.elf --property 'AF![AG![pc_within_symbol!("not_here")]]'
+---------------------------------------------------------------------------------------------------------------------------------------------------+
|   Result: ERROR ("property 'AF![AG![pc_within_symbol!(\"not_here\")]]' could not be constructed: machine-check: Symbol 'not_here' not found\n")   |
+---------------------------------------------------------------------------------------------------------------------------------------------------+
|  Refinements:                                                                                                                                  0  |
|  Generated states:                                                                                                                             0  |
|  Final states:                                                                                                                                 0  |
|  Generated transitions:                                                                                                                        0  |
|  Final transitions:                                                                                                                            0  |
+---------------------------------------------------------------------------------------------------------------------------------------------------+
```

The macro `pc_within_symbol` only works on symbols that have entries for a Program Counter range, e.g. functions. To consider only the start of the range or a symbol with a single Program Counter Address, there is a macro `pc_at_symbol`. For example, we verify whether the previous reach&stay property holds at the `assign1` label.

```console
$ machine-check-riscv -E basic.elf --property 'AF![AG![pc_at_symbol!("assign1")]]'
[2025-12-17T15:00:03Z INFO  machine_check] Starting verification.
[2025-12-17T15:00:03Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-12-17T15:00:03Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-12-17T15:00:03Z INFO  machine_check::verify] Verifying the given property.
[2025-12-17T15:00:03Z INFO  machine_check] Verification ended.
+--------------------------------+
|     Result: DOES NOT HOLD      |
+--------------------------------+
|  Refinements:               0  |
|  Generated states:        564  |
|  Final states:            563  |
|  Generated transitions:   564  |
|  Final transitions:       564  |
+--------------------------------+
```

The property does not hold, since the label corresponds to a single instruction in the loop and it clearly should not be possible to stay there while doing useful work.

For something a little more interesting, let us verify that when we are inside `main`, it should be possible for the variable `button_pressed` to become 1 with some inputs (i.e. being in `main` implies `button_pressed` being 1; note that $a \implies b$ can be written as `!a || b` in classical logic). For viewing the values of variables, a third custom property macro `typed_symbol` is provided, which expands to an expression that gives the value of the variable as a correctly sized bitvector. Note that this is currently limited to variables with a fixed place in the data SRAM (typically globals).

```console
$ machine-check-riscv -E basic.elf --property 'AG![!(pc_at_symbol!("main")) || EF![typed_symbol!("button_pressed") == 1]]'
[2025-12-17T15:09:56Z INFO  machine_check] Starting verification.
[2025-12-17T15:09:56Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-12-17T15:09:56Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-12-17T15:09:56Z INFO  machine_check::verify] Verifying the given property.
[2025-12-17T15:10:01Z INFO  machine_check] Verification ended.
+--------------------------------+
|         Result: HOLDS          |
+--------------------------------+
|  Refinements:               1  |
|  Generated states:        615  |
|  Final states:            604  |
|  Generated transitions:   616  |
|  Final transitions:       606  |
+--------------------------------+
```

To verify the property, a refinement of the state space was necessary to distinguish between the cases of the input pin being 0 and the input pin being 1. We can also see whether `button_pressed` becomes 1 with arbitrary inputs.

```console
$ machine-check-riscv -E basic.elf --property 'AG![!(pc_at_symbol!("main")) || AF![typed_symbol!("button_pressed") == 1]]'
[2025-12-17T15:17:59Z INFO  machine_check] Starting verification.
[2025-12-17T15:17:59Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-12-17T15:17:59Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-12-17T15:17:59Z INFO  machine_check::verify] Verifying the given property.
[2025-12-17T15:18:27Z INFO  machine_check] Verification ended.
+--------------------------------+
|     Result: DOES NOT HOLD      |
+--------------------------------+
|  Refinements:               3  |
|  Generated states:        689  |
|  Final states:            658  |
|  Generated transitions:   692  |
|  Final transitions:       662  |
+--------------------------------+
```

Clearly, it does not, as we can run the system forever without pressing the button (although it might become increasingly tempting). For those interested, a few similar examples and properties can be seen by browsing the source code of **machine-check-riscv**.

>
> &#x1F4A1;&#xFE0F; As for **machine-check-avr**, the verification time and/or memory may become prohibitive very quickly; this is even worse for RISC-V due to the 32-bit registers and usually more complicated initialisation.
>