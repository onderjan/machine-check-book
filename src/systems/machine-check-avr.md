# machine-check-avr

[**Machine-check-avr**](https://docs.rs/machine-check-avr/latest/machine_check_avr/) provides a description of a machine-code system based on the [AVR ATmega328P](https://www.microchip.com/en-us/product/atmega328p) microcontroller, with [some caveats](https://docs.rs/machine-check-avr/latest/machine_check_avr/#known-system-problems). The description was created by transcribing the appropriate parts of the [instruction set manual](https://ww1.microchip.com/downloads/en/DeviceDoc/AVR-InstructionSet-Manual-DS40002198.pdf) and the [non-automotive ATmega328P datasheet](https://ww1.microchip.com/downloads/aemDocuments/documents/MCU08/ProductDocuments/DataSheets/ATmega48A-PA-88A-PA-168A-PA-328-P-DS-DS40002061B.pdf).

The Intel HEX files used to load the program into the microcontroller can be provided as an argument to **machine-check-avr**: [see the docs for details](https://docs.rs/machine-check-avr/latest/machine_check_avr/). This enables machine-code verification using **machine-check**.

For an example discussed in [a presentation about a previous version of **machine-check**](https://avm2024.informatik.uni-freiburg.de/assets/presentations/jan_onderka.pdf) (the program on the slides is simplified for visibility), consider the following program written in C that can be used for calibration using binary search:

```c
#define F_CPU 1000000

#include <avr/io.h>
#include <util/delay.h>

int main(void)
{
	
	DDRC |= 0x01;
	DDRD |= 0xFF;
    while (1) 
    {
		while ((PINC & 0x2) == 0) {}
		
		// signify we are calibrating
		PORTC |= 0x01;
		
		// start with MSB of calibration
		uint8_t search_bit = 7;
		uint8_t search_mask = (1 << search_bit);
		uint8_t search_val = search_mask;
		
		while (1) {
			// wait a bit
			_delay_us(10);
			// write the current search value
			PORTD = search_val;
			// wait a bit
			_delay_us(10);
			
			// get input value and compare it to desired
			uint8_t input_value = PINB;
			
			if ((input_value & 0x80) == 0) {
				// input value lower than desired -> we should lower the calibration value
				search_val &= ~search_mask;	
			}
			
			if (search_bit == 0) {
				// all bits have been set, stop
				break;
			}
			
			search_bit -= 1;
			// continue to next bit
			search_mask >>= 1;
			// update the search value with the next bit set
			search_val |= search_mask;
			
		}
		
		// calibration complete, stop signifying that we are calibrating
		PORTC &= ~0x01;
    }
}
```

We can use **machine-check-avr** to find a verify a property in the compiled machine-code version of the program. We are doing binary search and setting `PORTD` according to `search_val`. Let us consider that, at any point in the program, we should be able to use some sequence of inputs to coerce `PORTD` to be zero.

**Machine-check-avr** can be installed as an executable using cargo:
```console
cargo install machine-check-avr
    Updating crates.io index
  Downloaded machine-check-avr v0.4.0
  Downloaded 1 crate (42.8 KB) in 1.03s
  Installing machine-check-avr v0.4.0
    Updating crates.io index
     Locking 74 packages to latest compatible versions
   Compiling autocfg v1.4.0
   (...)
    Finished `release` profile [optimized] target(s) in 1m 10s
  Installing (...)\.cargo\bin\machine-check-avr.exe
   Installed package `machine-check-avr v0.4.0` (executable `machine-check-avr.exe`)
```

The [Intel HEX](https://en.wikipedia.org/wiki/Intel_HEX) file obtained via compilation (that we can use to program the ATmega328P or provide to **machine-check-avr**) is not very self-explanatory (let us call it `calibration_original.hex`):
```
:100000000C9434000C943E000C943E000C943E0082
:100010000C943E000C943E000C943E000C943E0068
:100020000C943E000C943E000C943E000C943E0058
:100030000C943E000C943E000C943E000C943E0048
:100040000C943E000C943E000C943E000C943E0038
:100050000C943E000C943E000C943E000C943E0028
:100060000C943E000C943E0011241FBECFEFD8E04C
:10007000DEBFCDBF0E9440000C9466000C940000CF
:1000800087B1816087B98AB18FEF8AB9319BFECF82
:1000900088B1816088B980E890E827E033E03A953C
:1000A000F1F700008BB933E03A95F1F700001F99A2
:1000B00003C0392F30958323222321F021509695B8
:1000C000892BECCF88B18E7F88B9E0CFF894FFCF31
:00000001FF
```

We can try to verify the property, which should hold:
```console
machine-check-avr --system-hex-file (...)\calibration_original.hex --property "AG![EF![PORTD == 0]]"
[2025-04-03T20:19:44Z INFO  machine_check] Starting verification.
[2025-04-03T20:19:44Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T20:19:59Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T20:19:59Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T20:19:59Z INFO  machine_check] Verification ended.
+----------------------------------+
|      Result: DOES NOT HOLD       |
+----------------------------------+
|  Refinements:               513  |
|  Generated states:        14090  |
|  Final states:            13059  |
|  Generated transitions:   14603  |
|  Final transitions:       13573  |
+----------------------------------+
```

This is caused by a bug in the program.

The problem is that `PORTD` is not updated to `search_val` at the end of search before the loop breaks, so the lowest bit is always set to one after each calibration. Let us fix this:
```c
(...)
			if (search_bit == 0) {
				// all bits have been set, stop
				
				// FIX BUG FOUND BY TOOL: write the end value
				PORTD = search_val;
				break;
			}
(...)
```

Again, the new hex file does not tell us much, although it contains as much information as the microcontroller needs to execute the program (let's call it `calibration_fixed.hex`):

```
:100000000C9434000C943E000C943E000C943E0082
:100010000C943E000C943E000C943E000C943E0068
:100020000C943E000C943E000C943E000C943E0058
:100030000C943E000C943E000C943E000C943E0048
:100040000C943E000C943E000C943E000C943E0038
:100050000C943E000C943E000C943E000C943E0028
:100060000C943E000C943E0011241FBECFEFD8E04C
:10007000DEBFCDBF0E9440000C9467000C940000CE
:1000800087B1816087B98AB18FEF8AB9319BFECF82
:1000900088B1816088B980E890E827E033E03A953C
:1000A000F1F700008BB933E03A95F1F700001F99A2
:1000B00003C0392F30958323211105C08BB988B136
:1000C0008E7F88B9E3CF21509695892BE7CFF8949E
:0200D000FFCF60
:00000001FF
```

We'll verify that the property now holds:

```console
$ machine-check-avr --system-hex-file (...)\calibration_fixed.hex --property "AG![EF![PORTD == 0]]"   
[2025-04-03T20:20:17Z INFO  machine_check] Starting verification.
[2025-04-03T20:20:17Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T20:20:33Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T20:20:33Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T20:20:33Z INFO  machine_check] Verification ended.
+----------------------------------+
|          Result: HOLDS           |
+----------------------------------+
|  Refinements:               513  |
|  Generated states:        14090  |
|  Final states:            13059  |
|  Generated transitions:   14603  |
|  Final transitions:       13573  |
+----------------------------------+
```

Perfect. Being happy with the result, we can uninstall the executable:
```console
$ cargo uninstall machine-check-avr
    Removing (...)\.cargo\bin\machine-check-avr.exe
```

>
> &#x1F4A1;&#xFE0F; Generally, the major problem with verification of complex systems such as machine-code systems is that **machine-check** may not choose the refinements correctly and the time or memory needed for verification will be unacceptable. This will depend on the program and the cleverness of **machine-check** when using the chosen strategy, and can be investigated [using the GUI](../machine-check/ch5_gui.md).
>


>
> &#x26A0;&#xFE0F; Currently, the ease of use of **machine-check-avr** leaves something to be desired: for example, if we want to verify a property depending on some line in the program source code, we have to obtain the mapping of the source-code line to the the value of the Program Counter (`PC`) ourselves. In the future, **machine-check-avr** could be extended to obtain this information automatically from a file with debug information.
>

