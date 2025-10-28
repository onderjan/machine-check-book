# Parametric Systems

Often, we do not have complete information about how a system behaves, or we intentionally do not want to describe the behaviour as it could be too complex and not relevant to the properties we want to verify.

For example, we might have a microcontroller program which uses a serial input peripheral where the range of possibly read values depends on some complex configuration. It is not completely correct to treat these
as inputs since the property verification assumes **all** inputs can occur. As such, we might verify that a property holds due to one of the inputs, even though this input never occurs in practice.

We already have one way to handle these kinds of uncertainties: [panic when the result could depend on them](../machine-check/ch3_sec1_instances.html#panics-and-the-inherent-property). This, however, can be too coarse, 
as some properties can hold or not hold independently of this uncertainty. As such, **machine-check** supports *parametric verification*, where instead of verifying whether a property holds in a single system, we verify 
whether it holds in a set of systems. This gives us three possibilities for verification results:
 - Does not hold in any of the systems,
 - Holds in some and does not hold in others,
 - Holds in all of the systems.

If we verify that the property holds or it does not for all systems, it means that we can safely ignore the behaviour hidden by the parameter. If we verify that it only holds in some, it can also be a cause for concern
if this does not match our expectations.

Let's look at the simple [parametric example](https://docs.rs/crate/machine-check/0.7.0/source/examples/parametric.rs) for **machine-check**:

```rust
(...)
    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct Input {
        value: Unsigned<4>,
    }

    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct Param {
        max_value: Unsigned<4>,
    }

    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct State {
        value: Unsigned<4>,
        max_value: Unsigned<4>,
    }

    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct System {}

    impl ::machine_check::Machine for System {
        type Input = Input;
        type State = State;
        type Param = Param;

        fn init(&self, _input: &Input, param: &Param) -> State {
            State {
                max_value: param.max_value,
                value: Unsigned::<4>::new(0),
            }
        }

        fn next(&self, state: &State, input: &Input, _param: &Param) -> State {
            let mut value = input.value;
            if value >= state.max_value {
                value = state.max_value;
            }
            State {
                value,
                max_value: state.max_value,
            }
        }
    }
(...)
```

For understandability, this example only corresponds to 16 actual parametrised systems: the field `max_value` of `Param` is copied to `max_value` in state during initialisation and remains the same throughout time.
The system loads a value from the input during the next state computation, making sure the value stored in the state is at most `max_value`. As such, we have a system where the value stored in the state can only be 0 
(since `max_value` is 0), a system where the value can be 0 or 1 (since `max_value` is 1), and so on until the system where the value can range from 0 to 15. We can now verify some properties on this:

```console
$ cargo run -- --property 'AG![value == 0]'
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 2.47s
warning: the following packages contain code that will be rejected by a future version of Rust: partitions v0.2.4
note: to see what the problems were, use the option `--future-incompat-report`, or run `cargo report future-incompatibilities --id 2`
     Running `target\debug\hello-machine-check.exe --property "AG![value == 0]"`
[2025-10-28T22:26:48Z INFO  machine_check] Starting verification.
[2025-10-28T22:26:48Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-10-28T22:26:48Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-10-28T22:26:48Z INFO  machine_check::verify] Verifying the given property.
[2025-10-28T22:26:48Z INFO  machine_check] Verification ended.
+-----------------------------------+
|   Result: DEPENDS ON PARAMETERS   |
+-----------------------------------+
|  Refinements:                  5  |
|  Generated states:            69  |
|  Final states:                33  |
|  Generated transitions:       96  |
|  Final transitions:           50  |
+-----------------------------------+
```

We can reason about the result as follows: if `max_value` is 0, then the only possible path to take is all-zeros, so it will hold that the value is zero globally on all paths. On the other hand, if `max_value` is larger, 
we can take a path where value is 1 somewhere, so the property does not hold.

Some properties, however, hold or not independently on the parameter:
```console
cargo run -- --property 'AF![as_unsigned(value) > 0]'
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.11s
warning: the following packages contain code that will be rejected by a future version of Rust: partitions v0.2.4
note: to see what the problems were, use the option `--future-incompat-report`, or run `cargo report future-incompatibilities --id 2`
     Running `target\debug\hello-machine-check.exe --property "AF![as_unsigned(value) > 0]"`
[2025-10-28T22:27:03Z INFO  machine_check] Starting verification.
[2025-10-28T22:27:03Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-10-28T22:27:03Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-10-28T22:27:03Z INFO  machine_check::verify] Verifying the given property.
[2025-10-28T22:27:03Z INFO  machine_check] Verification ended.
+--------------------------------+
|     Result: DOES NOT HOLD      |
+--------------------------------+
|  Refinements:              64  |
|  Generated states:        368  |
|  Final states:            151  |
|  Generated transitions:   784  |
|  Final transitions:       287  |
+--------------------------------+
```

No matter the `max_value`, we can always take the path where the state value is zero forever, which disproves the property. See the [parametric example](https://docs.rs/crate/machine-check/0.7.0/source/examples/parametric.rs) for examples of more properties.

It is also possible to use the parameters in the next state computation function: in that case, each time instant has its own "fresh" parameter argument independent of others instead of the single unchanging `max_value` in the example, essentially producing an infinite number of systems fulfilling the description.

>
> &#x1F6E0;&#xFE0F; The ability to verify with parameters is currently somewhat experimental.
> While the abstraction-refinement framework used by **machine-check** is used even for abstracting and refining parameters, uses such as the above example are currently not a very good fit due to the current absence of relational domains more powerful than equality.
>

# System Hierarchy Checking

We can also use parametric systems with a different thought process. Consider that you are given a specification in form of a system which has some unspecified parts, and an implementation that should "fill the spots" while retaining the behaviours of the specification system. Then, you can actually think of the specification system as a system that is parametrised by variables determining the unspecified values.

Let's look at the simple [system hierarchy example](https://docs.rs/crate/machine-check/0.7.0/source/examples/hierarchy.rs) for **machine-check**, where a specification of a system performing an unsigned division of two numbers says that the result of division by zero must be either 0 (as in e.g. some AVR processors) or the unsigned maximum (as in e.g. RISC-V), but leaves the implementation free to choose one of them. The specification result is represented by `specified_value`, the implementation result as `impl_value`, and the relevant computation is as follows (`param.unspecified` is a signed single-bit parameter, which will be extended either to 0 or -1, i.e. the maximum unsigned value):

```rust
            if input.divisor != Unsigned::<4>::new(0) {
                specified_value = input.dividend / input.divisor;
                impl_value = input.dividend / input.divisor;
            } else {
                specified_value = Into::<Unsigned<4>>::into(Ext::<4>::ext(param.unspecified));
                impl_value = self.division_by_zero_result;
            }
```

Running the example normally (add import `clap = { version = "4.4.6", features = ["derive"] }` to your Cargo.toml if you are going to copy the example to your package), the default system implementation has `self.division_by_zero_result` set to 0. Running to check that the values will be the same, we get:

```console
cargo run -- --property 'AG![specified_value == impl_value]'
(...)
[2025-10-28T22:28:20Z INFO  machine_check] Starting verification.
[2025-10-28T22:28:20Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-10-28T22:28:20Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-10-28T22:28:20Z INFO  machine_check::verify] Verifying the given property.
[2025-10-28T22:28:20Z INFO  machine_check] Verification ended.
+-----------------------------------+
|   Result: DEPENDS ON PARAMETERS   |
+-----------------------------------+
|  Refinements:                  9  |
|  Generated states:            44  |
|  Final states:                17  |
|  Generated transitions:     1057  |
|  Final transitions:           34  |
+-----------------------------------+
```

This actually makes sense in the context: if the parameter (unspecified value) is 0, then the implementation and specification system with that parameter are the same. If not, they are different. As such, the specification system is hierarchically "above" the implementation. On the other hand, we can pass the parameter `--system-wrong-div-by-zero` to force the division-by-zero result to 1, which is not allowed:

```console

 cargo run -- --property 'AG![specified_value == impl_value]' --system-wrong-div-by-zero
 (...)
[2025-10-28T22:28:43Z INFO  machine_check] Starting verification.
[2025-10-28T22:28:43Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-10-28T22:28:43Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-10-28T22:28:43Z INFO  machine_check::verify] Verifying the given property.
[2025-10-28T22:28:43Z INFO  machine_check] Verification ended.
+-------------------------------+
|     Result: DOES NOT HOLD     |
+-------------------------------+
|  Refinements:              5  |
|  Generated states:        14  |
|  Final states:             6  |
|  Generated transitions:   71  |
|  Final transitions:       12  |
+-------------------------------+

```

This says that there is no valuation of the parameters such that the implementation would correspond to the specification system. As such, we learned that the implementation hierarchically has no relation with the specification, which violates our expectations.


>
> &#x1F6E0;&#xFE0F; Checking the hierarchical relationship of two parametric systems is currently not supported.
>