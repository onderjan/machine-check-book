# Systems

>
> &#x1F393;&#xFE0F; This chapter of the book is still being developed and may change significantly.
> Please come back in a few days.
>

Let us return to the [Quickstart](ch1_quickstart) example:

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

That is a lot of code. Fortunately, most of it is just boilerplate around the contents of the `Input` and `State` structures and `init` and `next` functions which define a Finite State Machine. Let us unpack it bit by bit. Do not worry if you do not understand everything.

To enable formal verification of properties of digital systems using more intelligent means than brute-force computation, **machine-check**  transforms them in a Rust macro which is called by placing an attribute `#[machine_check::machine_description]` on an inner module, in this case called `machine_module`. The code that goes through the macro is enriched with verification analogues for fast verification. As the macro only supports a small subset of Rust code, it will produce a (hopefully useful) error if it does not understand something. However, you can still write any Rust code you want outside the module.

Since the `machine_description` macro cannot read the outside of the module, it does not assume anything about it, such as that the identifier `Clone` refers to the trait[`::std::clone::Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html) which is usually imported in the [Rust prelude](https://doc.rust-lang.org/std/prelude/index.html): you might have changed `Clone` to mean something else outside. As such, you need to import it in the module using the [absolute path](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html) `::std::clone::Clone`. As referring to `::std::clone::Clone` each time it is used is annoying, we bring it in scope with a [use declaration](https://doc.rust-lang.org/book/ch07-04-bringing-paths-into-scope-with-the-use-keyword.html), so we can just write `Clone` again and `machine_description` knows it really is `::std::clone::Clone`.

Onto the more interesting part, the structure input:

```rust
    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct Input {
        increment_value: Bitvector<1>,
    }
```

This defines how the input of the Finite-State Machine (FSM) looks like. Do not worry about the `derive` line; this is needed simply to ensure that it is sensible to manipulate using `machine-check`. We define a structure `Input` that has exactly one field `increment_value` of type `Bitvector<1>`. [Bitvector](https://docs.rs/machine-check/latest/machine_check/struct.Bitvector.html) is a type provided by `machine-check` that has no signedness information and wrapping arithmetic operations. The number in angled brackets determines the bit-width, i.e. `increment-value` has only a single bit. So, basically, the input of the FSM is just a single bit. The next line is:

```rust
    impl ::machine_check::Input for Input {}
```

This just states that our input structure can be used as an input to `machine-check` machines. Do not worry about it. The state structure is more interesting:

```rust
   #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct State {
        value: Bitvector<4>,
    }

    impl ::machine_check::State for State {}
```

There is just one field `value` in the state, a four bits wide bit-vector. What's next?

```rust

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
```

This is where the fun begins. We define a structure `System` that has no fields and implement a trait `::machine_check::Machine` on it that will determine the actual FSM.

```rust
        type Input = Input;
        type State = State;
```

The type declarations say that the input type of the FSM (left-side `Input`) is the structure `Input` we defined previously. Similarly, the FSM state type is the structure `State`. To describe the FSM, we also need is a function that creates the initial states of the FSM and a function that transforms a state to a next state, and this is exactly what we have.

```rust
    fn init(&self, _input: &Input) -> State {
            State {
                value: Bitvector::<4>::new(0),
            }
        }
```

The init function takes the system structure (which has no fields in our case, so is quite useless in this situation) and an input, creating an initial machine state based on that. Our init function just creates a state where `value` is a four-bit bit-vector with the value zero.

>
> &#x26A0;&#xFE0F; Currently, the initialisation function also takes the input structure as an argument, possibly creating multiple initial states. This will most likely change in the future when parametric systems are implemented.
>

The state function is a bit more interesting:

```rust

        fn next(&self, state: &State, input: &Input) -> State {
            let mut next_value = state.value;
            if input.increment_value == Bitvector::<1>::new(1) {
                next_value = next_value + Bitvector::<4>::new(1);
            }

            State {
                value: next_value,
            }
        }
```

We create a mutable variable `next_value` based on the value of the current state, and if the input field `increment_value` is equal to one, we add 1 to the value. Note that despite the `Bitvector` not having any signedness, addition is the same for unsigned and signed numbers in wrap-around arithmetic, so we can do this nicely here. We then return the next value.

So, we have defined a machine that contains a four-bit bit-vector `value` that is incremented in each step if `increment_value` is one, otherwise, it is left as-is.
