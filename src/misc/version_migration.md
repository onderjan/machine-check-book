# Version Migration

This page details the steps necessary to migrate the systems and properties to new versions of **machine-check**, starting from version 0.4.0.

## 0.5.0 → 0.6.0

The properties now [support μ-calculus](../advanced/adv2_mu_calculus.md), but the properties from version 0.5.0 
can be used as-is without any changes. The systems must be modified as follows.

The `machine_check::Input` and `machine_check::State` traits are now blanket-implemented. Remove the following:

```rust
    impl ::machine_check::Input for Input {}

    impl ::machine_check::State for State {}
```

Parameters were added in version 0.6.0 to support [parametric systems](../advanced/adv1_parametric_systems.md).
For a non-parametric system (behaving as previously), add the following structure into the machine module:

```rust
    #[derive(Clone, PartialEq, Eq, Hash, Debug)]
    pub struct Param {}
```

Then, for the `machine_check::Machine` implementation, apply the changes as written in the comments:

```rust
    impl ::machine_check::Machine for System {
        type Input = Input;
        type Param = Param; // <-- add this
        type State = State;

        // add the function argument '_param: &Param' as follows
        fn init(&self, input: &Input, _param: &Param) -> State {
            (...)
        }

        // add the function argument '_param: &Param' as follows
        fn next(&self, state: &State, input: &Input, _param: &Param) -> State {
            (...)
        }
    }
```



## 0.4.0 → 0.5.0

No changes necessary.