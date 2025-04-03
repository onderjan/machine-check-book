# Quickstart

>
> &#x1F393;&#xFE0F; This chapter of the book is still being developed and may change significantly.
> Please come back in a few days.
>


If you are a Rust user, you can [skip to the usage](#using-machine-check). You can also try out [the quick example](https://crates.io/crates/machine-check) from the **machine-check** readme.

## Setting up Rust

**Machine-check** is a tool for verification of systems written in a subset of [the Rust programming language](https://www.rust-lang.org/). In the context of Rust, it works as a library you use in your code. If you do not use Rust yet, you need to [install Rust](https://doc.rust-lang.org/book/ch01-01-installation.html) and [learn its basics](https://doc.rust-lang.org/book/) first. Fortunately, describing finite-state machines does not use that many Rust concepts, so you can just skim over the first few chapters (although preferably 10) of the [Rust Book](https://doc.rust-lang.org/book/), try out a few examples, and return to the book if you do not understand something.

To start using **machine-check**, create a new binary Rust package using Rust's package manager `cargo`:

```console
$ cargo new hello-machine-check      
    Creating binary (application) `hello-machine-check` package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

```

In the created `hello-machine-check` directory, there is a package config file `Cargo.toml` and a hello-world placeholder in `src/main.rs`:

```console
fn main() {
    println!("Hello, world!");
}
```

You can run the program using `cargo run` in the `hello-machine-check` directory:

```console
$ cargo run
   Compiling hello-machine-check v0.1.0 (<your path>\hello-machine-check)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.07s
     Running `target\debug\hello-machine-check.exe`
Hello, world!
```

Let us now move to actually using `machine-check`.

## Using machine-check

In your `Cargo.toml`, add the following dependency:

```toml
[dependencies]
machine-check = "0.4.0"
```

You can start using **machine-check** in Rust now. Let's create a minimal somewhat interesting system to verify. Put this in your `src/main.rs`:

```rust
#[machine_check::machine_description]
mod machine_module {
    use ::machine_check::Bitvector;
    use ::std::{
        clone::Clone,
        cmp::{Eq, PartialEq},
        fmt::Debug,
        hash::Hash,
    };

    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct Input {
        increment_value: Bitvector<1>,
    }

    impl ::machine_check::Input for Input {}

    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct State {
        value: Bitvector<4>,
    }

    impl ::machine_check::State for State {}

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
}

fn main() {
    let system = machine_module::System {};
    machine_check::run(system);
}
```

That is a lot of code. Fortunately, most of it is just boilerplate around the contents of the `Input` and `State` structures and `init` and `next` functions which define a Finite State Machine. We will discuss how the machine is described later; for now, just know that the state contains a four-bit field `value` that is initially zero and after that, each next step of the machine is determined by looking at a single-bit input field `increment_value`, incrementing `value` if `increment_value` is 1 and leaving `value` as-is otherwise. Ready for some verification?

## Verifying

Currently, **machine-check** can verify [Computation Tree Logic](http://en.wikipedia.org/wiki/Computation_tree_logic) properties of the systems. For example, we can determine that from every reachable state of the system, we can, through some sequence of inputs, get to a state where `value` is zero:
```console
cargo run -- --property "AG![EF![value == 0]]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.10s
     Running `target\debug\hello-machine-check.exe --property "AG![EF![value == 0]]"`
[2025-04-03T14:55:21Z INFO  machine_check] Starting verification.
[2025-04-03T14:55:21Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T14:55:21Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T14:55:21Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T14:55:21Z INFO  machine_check] Verification ended.
+-------------------------------+
|         Result: HOLDS         |
+-------------------------------+
|  Refinements:             16  |
|  Generated states:        48  |
|  Final states:            16  |
|  Generated transitions:   64  |
|  Final transitions:       33  |
+-------------------------------+
```

While the example system under verification is very simple, **machine-check** integrates advanced techniques that make verification of more complex systems possible, such as ones combining a processor and a machine-code program. In the following chapters, we will discuss [the properties that can be verified](./ch2_properties.md), [the system descriptions](./ch3_systems.md), [the Command-line Interface](./ch4_cli.md), and [the Graphical User Interface](./ch5_gui.md) that can be used to understand the results of verification or the reasons for the inability to feasibly verify a property against a system.

