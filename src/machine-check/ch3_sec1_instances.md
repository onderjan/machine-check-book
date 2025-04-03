# Instances and Panics

It is useful to be able to change the behaviour of the system under verification at runtime, e.g. using command-line arguments, so that the system does not need to be changed and recompiled to change some part of its behaviour. Consider the part of the example from the previous section:

```rust
(...)

    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct System {}

    impl ::machine_check::Machine for System {
        type Input = Input;
        type State = State;

        fn init(&self, _input: &Input) -> State {
            State {
                value: Bitvector::<4>::new(0),
            }
        }

        fn next(&self, state: &State, input: &Input) -> State {
            let mut next_value = state.value;
            if input.increment_value == Bitvector::<1>::new(1) {
                next_value = next_value + Bitvector::<4>::new(1);
            }

            State {
                value: next_value,
            }
        }
    }

(...)

fn main() {
    let system = machine_module::System {};
    machine_check::run(system);
}

```

Let us suppose that we want to specify the maximum `value`, after which it wraps, and supply the value from the main function. We can do this by extending the struct `System` with a new field as follows:

```rust
(...)
    use ::machine_check::Unsigned;
    use ::std::convert::Into;

    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct System {
        pub max_value: Bitvector<4>,
    }

    impl ::machine_check::Machine for System {
        type Input = Input;
        type State = State;

        fn init(&self, _input: &Input) -> State {
            State {
                value: Bitvector::<4>::new(0),
            }
        }

        fn next(&self, state: &State, input: &Input) -> State {
            let mut next_value = state.value;
            if input.increment_value == Bitvector::<1>::new(1) {
                next_value = next_value + Bitvector::<4>::new(1);
            }

            if Into::<Unsigned<4>>::into(next_value) > Into::<Unsigned<4>>::into(self.max_value) {
                next_value = Bitvector::<4>::new(0);
            }

            State { value: next_value }
        }
    }
(...)
fn main() {
    print!("Write the maximum system value: ");
    std::io::stdout()
        .flush()
        .expect("Should flush standard output");
    let mut read_string = String::new();
    std::io::stdin()
        .read_line(&mut read_string)
        .expect("Should read string from standard input");
    let read_max_value: u64 = read_string
        .trim()
        .parse()
        .expect("Input should be an unsigned integer");

    let system = machine_module::System {
        max_value: machine_check::Bitvector::new(read_max_value),
    };
    machine_check::run(system);
}
```

We introduced a new four-bit bit-vector field `max_value` in the `System` struct, and added a condition in the `next` function that converts the next value and maximum value to an unsigned interpretation, and if the next value is larger, it is zeroed. In the main function, we print the command to the user to write the maximum system value, and after we read it, we construct the system accordingly. The `main` function is fully standard Rust. We can now try to verify that there exists an input sequence using which we eventually get to `value` being 10, using a few different values:

```console
$ cargo run -- --property "EF![value == 10]"
   Compiling hello-machine-check v0.1.0 (...)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.14s
     Running `target\debug\hello-machine-check.exe --property "EF![value == 10]"`
Write the maximum system value: 5
[2025-04-03T16:11:59Z INFO  machine_check] Starting verification.
[2025-04-03T16:11:59Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T16:11:59Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T16:11:59Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T16:11:59Z INFO  machine_check] Verification ended.
+-------------------------------+
+-------------------------------+
|  Refinements:              6  |
|  Generated states:        19  |
|  Final states:             6  |
|  Generated transitions:   25  |
|  Final transitions:       13  |
+-------------------------------+
$ cargo run -- --property "EF![value == 10]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.08s
     Running `target\debug\hello-machine-check.exe --property "EF![value == 10]"`
Write the maximum system value: 12
[2025-04-03T16:12:05Z INFO  machine_check] Starting verification.
[2025-04-03T16:12:05Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T16:12:05Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T16:12:05Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T16:12:05Z INFO  machine_check] Verification ended.
+-------------------------------+
|         Result: HOLDS         |
+-------------------------------+
|  Refinements:             10  |
|  Generated states:        33  |
|  Final states:            13  |
|  Generated transitions:   43  |
|  Final transitions:       24  |
+-------------------------------+
$ cargo run -- --property "EF![value == 10]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
     Running `target\debug\hello-machine-check.exe --property "EF![value == 10]"`
Write the maximum system value: 20

thread 'main' panicked at (...)\mck-0.4.0\src\bitvector\concrete\support.rs:15:13:
Machine bitvector value 20 does not fit into 4 bits
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
error: process didn't exit successfully: `target\debug\hello-machine-check.exe --property "EF![value == 10]"` (exit code: 101)
```

For the values of 5 and 12, everything works as intended. Since the four-bit bit-vector cannot be constructed by `Bitvector::new` with the value 20, the program panics.

It is possible to use the command-line arguments 


>
> &#x26A0;&#xFE0F; It is possible to use command-line arguments for the construction of systems, although the current form is highly unstable.
>
> See the source code of [machine-check-avr](../systems/machine-check-avr.md) for details.
>


## Panics and the Inherent Property

Sometimes, we want to disallow verification of a system containing some instance data. For example, we are implementing a processor and did not describe some instruction yet, or the instruction is disallowed by the processor specification itself. We can use the Rust macros [panic!](https://doc.rust-lang.org/std/macro.panic.html), [todo!](https://doc.rust-lang.org/std/macro.todo.html), and [unimplemented!](https://doc.rust-lang.org/std/macro.unimplemented.html) to preclude verification of properties in some cases. For example, suppose that we did not have the time to describe the behaviour of `max_value` within the introduced condition, so we wrote:

```console
            if Into::<Unsigned<4>>::into(next_value) > Into::<Unsigned<4>>::into(self.max_value) {
                ::std::todo!("Zero the next value when it is greater than max value");
            }
```

Now, the verification will return an error when verifying a property and a panic can eventually occur with some input sequence (i.e. **EF**\[*panic*\]):
```console
$ cargo run -- --property "EF![value == 10]"    
   Compiling hello-machine-check v0.1.0 (...)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.14s
     Running `target\debug\hello-machine-check.exe --property "EF![value == 10]"`
Write the maximum system value: 5
[2025-04-03T16:22:02Z INFO  machine_check] Starting verification.
[2025-04-03T16:22:02Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T16:22:02Z ERROR machine_check] Verification returned an error.
+----------------------------------------------------------------------------------------------------------+
|                                      Result: ERROR (inherent panic)                                      |
+----------------------------------------------------------------------------------------------------------+
|  Refinements:                                                                                         6  |
|  Generated states:                                                                                   22  |
|  Final states:                                                                                        9  |
|  Generated transitions:                                                                              28  |
|  Final transitions:                                                                                  16  |
|  Inherent panic message:   "not yet implemented: Zero the next value when it is greater than max value"  |
+----------------------------------------------------------------------------------------------------------+
```

However, in case the panic cannot occur, the verification will proceed normally:
```console
$ cargo run -- --property "EF![value == 15]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.08s
     Running `target\debug\hello-machine-check.exe --property "EF![value == 15]"`
Write the maximum system value: 15
[2025-04-03T16:23:35Z INFO  machine_check] Starting verification.
[2025-04-03T16:23:35Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T16:23:35Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T16:23:35Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T16:23:35Z INFO  machine_check] Verification ended.
+-------------------------------+
|         Result: HOLDS         |
+-------------------------------+
|  Refinements:             15  |
|  Generated states:        47  |
|  Final states:            17  |
|  Generated transitions:   62  |
|  Final transitions:       33  |
+-------------------------------+
```

This is the meaning of the messages when verifying: first, the inherent property (no panics can be reached) is verified, then, the given property (`EF![value == 15]` in our case) is verified. We can also verify just the inherent property itself:
```console
$ cargo run -- --inherent
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.09s
     Running `target\debug\hello-machine-check.exe --inherent`
Write the maximum system value: 10
[2025-04-03T16:27:53Z INFO  machine_check] Starting verification.
[2025-04-03T16:27:53Z INFO  machine_check::verify] Verifying the inherent property.
[2025-04-03T16:27:53Z INFO  machine_check] Verification ended.
+----------------------------------------------------------------------------------------------------------+
|                                          Result: DOES NOT HOLD                                           |
+----------------------------------------------------------------------------------------------------------+
|  Refinements:                                                                                        11  |
|  Generated states:                                                                                   36  |
|  Final states:                                                                                       14  |
|  Generated transitions:                                                                              47  |
|  Final transitions:                                                                                  26  |
|  Inherent panic message:   "not yet implemented: Zero the next value when it is greater than max value"  |
+----------------------------------------------------------------------------------------------------------+

$ cargo run -- --inherent
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
     Running `target\debug\hello-machine-check.exe --inherent`
Write the maximum system value: 15
[2025-04-03T16:28:14Z INFO  machine_check] Starting verification.
[2025-04-03T16:28:14Z INFO  machine_check::verify] Verifying the inherent property.
[2025-04-03T16:28:14Z INFO  machine_check] Verification ended.
+------------------------------+
|        Result: HOLDS         |
+------------------------------+
|  Refinements:             0  |
|  Generated states:        6  |
|  Final states:            5  |
|  Generated transitions:   6  |
|  Final transitions:       6  |
+------------------------------+
```

It is also possible to assume the inherent property holds by using the `--assume-inherent` command-line argument. In this case, if the inherent property actually does not hold, any verification results obtained while assuming it will be meaningless.
