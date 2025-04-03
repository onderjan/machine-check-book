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

Now you can start using **machine-check** in Rust. Let's create a minimal somewhat interesting system to verify. Put this in your `src/main.rs`:

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

Currently, **machine-check** can verify [Computation Tree Logic](http://en.wikipedia.org/wiki/Computation_tree_logic) properties of the systems. Let's start with a very simple property: is the value zero at the start of the computation?
```console
cargo run -- --property "value == 0"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.09s
     Running `target\debug\hello-machine-check.exe --property "value == 0"`
[2025-03-31T17:17:20Z INFO  machine_check] Starting verification.
[2025-03-31T17:17:20Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-03-31T17:17:20Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-03-31T17:17:20Z INFO  machine_check::verify] Verifying the given property.
[2025-03-31T17:17:20Z INFO  machine_check] Verification ended.
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

Let's not focus on what the messages or numbers mean and just look at the verification result. **Machine-check** has determined that the property `value == 0` holds, which matches the definition of the init function. Note that if the system had multiple initial states, the convention is to verify whether the property holds in all of them. But since we only have one initial state where `value = 0`, we don't have to think about this.

Verifying a single state is admittedly quite boring. The real power of the [Computation Tree Logic](http://en.wikipedia.org/wiki/Computation_tree_logic) (CTL) is that we can reason about things *temporally*, looking into states in the future. For example, we can use the *temporal operator* **AX**\[*a*\] which means "for all paths, *a* will hold in the next state" to say "for all paths, `value` will be 0 in the current state". Note that in **machine-check**, since the paths are determined only by the machine inputs, we can also say "for all inputs" instead of "for all paths", which can be more intutive. In **machine-check** property syntax, we write this as `AX![value == 0]`. Note the exclamation mark after `AX`: this is done as the property syntax emulates Rust syntax and it makes a lot of sense to think of the temporal properties as macros, since they change what the variables inside them represent. That aside, let's try to verify.

```console
$ cargo run -- --property "AX![value == 0]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.09s
     Running `target\debug\hello-machine-check.exe --property "AX![value == 0]"`
[2025-03-31T17:34:28Z INFO  machine_check] Starting verification.
[2025-03-31T17:34:28Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-03-31T17:34:28Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-03-31T17:34:28Z INFO  machine_check::verify] Verifying the given property.
[2025-03-31T17:34:28Z INFO  machine_check] Verification ended.
+------------------------------+
|    Result: DOES NOT HOLD     |
+------------------------------+
|  Refinements:             1  |
|  Generated states:        8  |
|  Final states:            5  |
|  Generated transitions:   9  |
|  Final transitions:       7  |
+------------------------------+
```

Of course this does not hold: we can increment the value, so `value` will no longer be zero in **all** of the next states! But what about it being zero in **at least one** next state? That should hold: if the `increment_value` input is 0, we will retain zero `value`. Instead of **AX**\[a\], we will use the **EX**\[*a*\] operator: there exists a path where *a* holds in next state. Let's try to verify.

```console
$ cargo run -- --property "EX![value == 0]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.10s
     Running `target\debug\hello-machine-check.exe --property "EX![value == 0]"`
[2025-03-31T17:38:09Z INFO  machine_check] Starting verification.
[2025-03-31T17:38:09Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-03-31T17:38:09Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-03-31T17:38:09Z INFO  machine_check::verify] Verifying the given property.
[2025-03-31T17:38:09Z INFO  machine_check] Verification ended.
+------------------------------+
|        Result: HOLDS         |
+------------------------------+
|  Refinements:             1  |
|  Generated states:        8  |
|  Final states:            5  |
|  Generated transitions:   9  |
|  Final transitions:       7  |
+------------------------------+
```

Wonderful. But we have just started to discover the possibilities. While **AX** and **EX** can be used to reason about the states directly after the current one, CTL also contains operators we can use to reason about unbounded periods of time. **AG**\[*a*\] means that for all inputs, the property *a* will hold *globally*, i.e. in all time instants including and after this one. In our example, for all inputs, `value` will globally be lesser or equal to 15. Note that we have to explicitly say that `value` should be compared as an unsigned number using `as_unsigned` before comparing it:

```console
$ cargo run -- --property "AG![as_unsigned(value) <= 15]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.08s
     Running `target\debug\hello-machine-check.exe --property "AG![as_unsigned(value) <= 15]"`
[2025-04-03T13:50:16Z INFO  machine_check] Starting verification.
[2025-04-03T13:50:16Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T13:50:16Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T13:50:16Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T13:50:16Z INFO  machine_check] Verification ended.
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

While this would not hold for a constant lesser than 15 (you can try it), **EG**\[*a*\] means that there is an infinite sequence of inputs where *a* holds globally. Reasonably, we can expect that `value` stays to be zero when the `increment_value` input is always kept 0:
```console
$ cargo run -- --property "EG![value == 0]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
     Running `target\debug\hello-machine-check.exe --property "EG![value == 0]"`
[2025-04-03T13:53:09Z INFO  machine_check] Starting verification.
[2025-04-03T13:53:09Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T13:53:09Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T13:53:09Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T13:53:09Z INFO  machine_check] Verification ended.
+------------------------------+
|        Result: HOLDS         |
+------------------------------+
|  Refinements:             1  |
|  Generated states:        8  |
|  Final states:            5  |
|  Generated transitions:   9  |
|  Final transitions:       7  |
+------------------------------+
```

Great. What if we try to verify if it stays to be 3?

```console
$ cargo run -- --property "EG![value == 3]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.09s
     Running `target\debug\hello-machine-check.exe --property "EG![value == 3]"`
[2025-04-03T13:56:23Z INFO  machine_check] Starting verification.
[2025-04-03T13:56:23Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T13:56:23Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T13:56:23Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T13:56:23Z INFO  machine_check] Verification ended.
+------------------------------+
|    Result: DOES NOT HOLD     |
+------------------------------+
|  Refinements:             0  |
|  Generated states:        6  |
|  Final states:            5  |
|  Generated transitions:   6  |
|  Final transitions:       6  |
+------------------------------+
```

It does not, since `value` is 0 at the start. To reason about the future, we can use the **AF** and **EF** operators. **AF**\[*a*\] means that for all input sequences, *a* holds *eventually*. Note that this can be either currently or in the future. **EF**\[*a*\] means that there exists an input sequence for which *a* holds eventually, which is a bit more interesting in the context of our example:
```console
$ cargo run -- --property "EF![as_unsigned(value) == 3]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
     Running `target\debug\hello-machine-check.exe --property "EF![as_unsigned(value) == 3]"`
[2025-04-03T14:05:00Z INFO  machine_check] Starting verification.
[2025-04-03T14:05:00Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T14:05:00Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T14:05:00Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T14:05:00Z INFO  machine_check] Verification ended.
+-------------------------------+
|         Result: HOLDS         |
+-------------------------------+
|  Refinements:              3  |
|  Generated states:        13  |
|  Final states:             6  |
|  Generated transitions:   16  |
|  Final transitions:       10  |
+-------------------------------+
```

The nice thing about CTL is that we can build more interesting reasoning from nesting the operators. Let us suppose we want to make sure that there exists an input sequence which eventually reaches a state where there exists an input sequence where `value` is then globally equal to 3:
```console
cargo run -- --property "EF![EG![value == 3]]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.09s
     Running `target\debug\hello-machine-check.exe --property "EF![EG![value == 3]]"`
[2025-04-03T14:09:30Z INFO  machine_check] Starting verification.
[2025-04-03T14:09:30Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T14:09:30Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T14:09:30Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T14:09:30Z INFO  machine_check] Verification ended.
+-------------------------------+
|         Result: HOLDS         |
+-------------------------------+
|  Refinements:              4  |
|  Generated states:        17  |
|  Final states:             8  |
|  Generated transitions:   21  |
|  Final transitions:       13  |
+-------------------------------+
```
For even more descriptive properties, we can use the **AU**, **EU**, **AR**, and **ER** CTL operators[^1]. These take two arguments. Nothwithstanding the difference in the input sequence requirements, **AU**\[*a*,*b*\] and **EU**\[*a*,*b*\] stipulate that *a* should hold *until* *b* holds, although not necessarily in the position where *b* first holds, and *b* must hold eventually.  **AR**\[*a*,*b*\] and **ER**\[*a*,*b*\] stipulate that *a* *releases* *b*: *b* has to hold before and including the point where *a* first holds, and if *a* never holds, *b* must hold globally. Notice how **AU**/**EU** and **AR**/**ER** are essentially extensions of  **AF**/**EF** and **AG**/**EG**, respectively.

[^1]: The **AR** and **ER** operators are usually not described in the literature on CTL, but can be derived from the other operators in that case.



>
> &#x1F393;&#xFE0F; This chapter of the book is still being developed and may change significantly.
> Please come back in a few days.
>

