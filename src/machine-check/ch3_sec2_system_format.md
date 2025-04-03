# System Format

The systems are written in a subset of Rust. Generally, the `machine_description` macro will emit the code that was written enriched by internal modules and implementations, so if the original code is not compilable by Rust without errors, it will not become compilable. **Machine-check** will also emit errors if it cannot translate the system code to the verification equivalents.

The systems generally should conform to the structure presented previously. The [examples](https://docs.rs/crate/machine-check/0.4.0/source/examples/) and [the source code of the **machine-check-avr** description](https://docs.rs/crate/machine-check-avr/0.4.0/source/src/system.rs) can be used for inspiration. As such, only the general principles of defining the structures and writing the functions to achieve the desired system behaviour will be detailed here.

>
> &#x26A0;&#xFE0F; The system format and accepted Rust constructs may change in the future for so that the descriptions can be written more easily.
>

## Structures and Fields

Structures can be defined as follows:

```rust
    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct ExampleStruct {
        pub bitvector_field: ::machine_check::Bitvector<4>,
        pub unsigned_field: ::machine_check::Unsigned<13>,
        pub signed_field: ::machine_check::Signed<32>,
        pub bitvector_array_field: ::machine_check::BitvectorArray<14, 12>,
    }
```

The currently supported types are all provided by **machine-check**. There are three bit-vector types [Bitvector](https://docs.rs/machine-check/0.4.0/machine_check/struct.Bitvector.html), [Unsigned](https://docs.rs/machine-check/0.4.0/machine_check/struct.Unsigned.html), and [Signed](https://docs.rs/machine-check/0.4.0/machine_check/struct.Signed.html) for variables that contain a binary value with the width determined by the generic constant *W*, with 2^*W* values possible. All of the types offer wrapping arithmetic, and they differ in the interpretation of the contained value. In [Unsigned](https://docs.rs/machine-check/0.4.0/machine_check/struct.Unsigned.html), the values are considered to be unsigned, and in [Signed](https://docs.rs/machine-check/0.4.0/machine_check/struct.Signed.html), they are considered signed in two's complement. This determines the behaviour of width extension, right shift and comparison operations. In [Bitvector](https://docs.rs/machine-check/0.4.0/machine_check/struct.Bitvector.html), these operations are not provided, although the type still is weakly unsigned when being constructed via `new` and used to index arrays. The  [Bitvector](https://docs.rs/machine-check/0.4.0/machine_check/struct.Bitvector.html) type is useful when constructing systems such as processor-based systems, where the memory cells do not necessarily have any signedness information, and the signedness is provided by the instruction currently operating on the cells. This can greatly reduce description errors in selecting the wrong operation signedness.

The bit-vector types can be converted into each other by using the standard library [Into](https://doc.rust-lang.org/std/convert/trait.Into.html). In addition to supporting standard operators (excluding division, remainder, and operator-assignment), they also support width extension / narrowing via the [Ext](https://docs.rs/machine-check/0.4.0/machine_check/trait.Ext.html) trait. The [Signed](https://docs.rs/machine-check/0.4.0/machine_check/struct.Signed.html) type is currently not possible to construct via `new` like the other bit-vector types.

The type [BitvectorArray](https://docs.rs/machine-check/0.4.0/machine_check/struct.BitvectorArray.html) is a power-of-2 bit-vector array where the first generic constant determines the index width (its power of two gives the array length) and the second one determines the element width. This makes it possible to ensure there are no possible panics when indexing. The array is retained using a sparse representation, ensuring that it is possible to construct e.g. a 48-bit-length array filled with zeros in constant time before assigning the values just to the cells of interest.

## Functions

The current expressivity in functions is fairly restricted. In addition to the `init` and `next` functions that are implemented for the trait `::machine_check::Machine`, it is also possible to write functions in the inherent implementation of the struct:

```rust
    impl System {
        fn my_func() -> Bitvector<4> {
            Bitvector::<4>::new(3)
        }
    }
```

All functions must have a return type that is one of the types known by **machine-check** (the basic provided types or defined structs) and can take an arbitrary numbers of parameters that are either of these types or are immutable references to them. This can be used for composition of complex behaviours, such as in the [machine-check-avr description](https://docs.rs/machine-check-avr/0.4.0/src/machine_check_avr/system.rs.html)

As for the statements inside the functions, please consult the [examples](https://docs.rs/crate/machine-check/0.4.0/source/examples/) and the [machine-check-avr description](https://docs.rs/machine-check-avr/0.4.0/src/machine_check_avr/system.rs.html) to build an intuition on what is supported. Notably, the type inference is currently much weaker than that of Rust, so explicit type declarations might be necessary to get rid of failed type inference errors from machine-check. It is currently not possible to use method calls, so e.g. converting to an four-bit unsigned type must be performed as:
```rust
::std::convert::Into::<Unsigned<4>>::into(variable)
```
Similarly, bit extension or narrowing to eight bits (which must be done on [Unsigned](https://docs.rs/machine-check/0.4.0/machine_check/struct.Unsigned.html) or [Signed](https://docs.rs/machine-check/0.4.0/machine_check/struct.Signed.html) so the signedness is known) must be performed as:
```rust
::machine_check::Ext::<8>::ext(variable)
```

In addition to panic macros discussed previously, the [`bitmask_switch` macro](https://docs.rs/machine-check/0.4.0/machine_check/macro.bitmask_switch.html) can be used for easier implementation of e.g. processor instructions. Examples of it can be found in [`simple_risc`](https://docs.rs/crate/machine-check/0.4.0/source/examples/simple_risc.rs) and the [machine-check-avr description](https://docs.rs/machine-check-avr/0.4.0/src/machine_check_avr/system.rs.html).

In case **machine-check** gives cryptic errors when the description is being compiled, you can try to return the description to the previous state where it compiled and carefully change things to pinpoint what breaks the translation, or comment out the `machine_description` macro temporarily to see if the code is invalid Rust, as the Rust compiler usually gives more understandable error messages. 


