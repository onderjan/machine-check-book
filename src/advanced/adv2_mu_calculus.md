# μ-calculus Properties

While the Computation Tree Logic is relatively simple to use, there are interesting properties that cannot be expressed using it. **Machine-check** supports a much more powerful temporal logic: [propositional (also called modal) μ-calculus](https://en.wikipedia.org/wiki/Modal_%CE%BC-calculus). In fact, even the CTL properties are internally translated to μ-calculus before they are used for model checking, and the only thing needed to unlock the power of μ-calculus are two new macro-style operators.


## Local Reasoning

Compared to CTL, μ-calculus properties are described using *local* reasoning, which is converted to global reasoning using the operators **μ** and **ν**. These correspond to computing the least and the greatest *fixed points*, respectively, so the machine-check macros for them are `lfp!` and `gfp!`, respectively. Let's look at what this means by examining the CTL operators first.

The CTL operator `AG![a]`, meaning "on all paths, globally, *a* holds", can be rewritten to `gfp![Z, a && AX![Z]]`. This makes the identifier `Z` a *fixpoint variable* within the second argument of `gfp!`. We can think of `Z` as a variable that has some value for every state. The computation will proceed as follows:
 - `Z` will be initialised to be true in all states.
 - In the first computation iteration, since `Z` is true everywhere, `AX![Z]` is also be true everywhere and we only need to consider `a`. Thus, at the end of the iteration, `Z` will become false for all states where `a` does not hold. 
 - In subsequent computation iterations, `Z` will change in the states directly preceding the states where `Z` changed to false.
 - This will propagate further and further until there are no longer any states where the iteration changes the valuation of `Z` in any state.

The computation will only end when we reach the *fixed point*, where the iteration produces nothing new; then, `Z` is the computed valuation of states. This will produce the same result as `AG![a]` should: all states which have some (arbitrarily far-away) successor where `a` does not hold will have the valuation false, while all others will have the valuation true since nothing has happened to change it.

For another example, the operator `AF![a]`, meaning "on all paths, eventually (now or in the future), *a* holds", `lfp![Z, a || AX![Z]]`. This is the *least* fixed point (**μ**), which sets `Z` to false in all states at first, then sets it true in the states where `a` holds, and then propagates the true valuations backward. 

The full table of translations for CTL operators is:

```
EG![a] = gfp![Z, a && EX![Z]]
AG![a] = gfp![Z, a && AX![Z]]

EF![a] = lfp![Z, a || EX![Z]]
AF![a] = lfp![Z, a || AX![Z]]

ER![a, b] = gfp![Z, b && (a || EX![Z])]
AR![a, b] = gfp![Z, b && (a || AX![Z])]

EU![a, b] = lfp![Z, b || (a && EX![Z])]
AU![a, b] = lfp![Z, b || (a && AX![Z])]
``` 

Note how the `E`/`A` path quantifier only affects the choice of `EX!`/`AX!`, while the choice between `G`/`F` or `R`/`U` selects the type of the fixed point. As the CTL operators may be used as-is in **machine-check**, the most useful thing about the translations above is that we can get some intuition about how to build properties **not** expressible in CTL.

## Example: Holds in Even Positions

The example [mu_even](https://docs.rs/crate/machine-check/0.6.1/source/examples/mu_even.rs) in **machine-check** demonstrates a system with an interesting property not expressible in Computation Tree Logic, nor the related Linear Tree Logic and CTL*: a property holding at even time instants. The value is computed as follows:

```rust
(...)
            let currently_odd_position = state.odd_position + Bitvector::<1>::new(1);

            // if the current position is even, set the value to 0
            // if the current position is odd, set the value to the input value
            let mut value = Bitvector::<8>::new(0);
            if currently_odd_position == Bitvector::<1>::new(1) {
                value = input.value;
            }
(...)
```

We want to know if the value is definitely zero in even time instants, while it can be whatever in odd time instants. The solution in μ-calculus is simple: use the property `gfp![Z, value == 0 && AX![AX![Z]]]`. This means that even if the value is nonzero in an odd time instant, this will only be propagated backwards two system steps at a time, so it cannot impact the valuation of the property at time zero. Running verification:

```console
$ cargo run -- --property 'gfp![Z, value == 0 && AX![AX![Z]]]'
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.10s
warning: the following packages contain code that will be rejected by a future version of Rust: partitions v0.2.4
note: to see what the problems were, use the option `--future-incompat-report`, or run `cargo report future-incompatibilities --id 2`
     Running `target\debug\hello-machine-check.exe --property "gfp![Z, value == 0 && AX![AX![Z]]]"`
[2025-10-28T22:29:40Z INFO  machine_check] Starting verification.
[2025-10-28T22:29:40Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-10-28T22:29:40Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-10-28T22:29:40Z INFO  machine_check::verify] Verifying the given property.
[2025-10-28T22:29:40Z INFO  machine_check] Verification ended.
+------------------------------+
|        Result: HOLDS         |
+------------------------------+
|  Refinements:             0  |
|  Generated states:        3  |
|  Final states:            2  |
|  Generated transitions:   3  |
|  Final transitions:       3  |
+------------------------------+
```

Variations of the local reasoning could be used to construct other properties of interest, e.g. checking only every fifth state from some point, checking using existential next operators, etc.

## Example: Infinitely Often

The example [mu_infinitely_often](https://docs.rs/crate/machine-check/0.6.1/source/examples/mu_infinitely_often.rs) demonstrates the ability of the μ-calculus to verify a property that cannot be expressed in Computation Tree Logic, but can be in Linear Time Logic: that something happens *infinitely often*. This can be expressed as **FG** in LTL. The similar property **AFAG** in CTL describes a different concept, that something happens *eventually forever*. The example gives the standard system that reveals the difference between the two: in state 0 where *p* holds, we have a choice between looping and going to state 1 where *p* does not hold. From state 1, the only possible transition is to state 2 where *p* holds again, and where the system loops forever.

Let us first verify the CTL property *eventually forever*:

```console
$ cargo run -- --property 'AF![AG![p == 1]]'
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.99s
warning: the following packages contain code that will be rejected by a future version of Rust: partitions v0.2.4
note: to see what the problems were, use the option `--future-incompat-report`, or run `cargo report future-incompatibilities --id 2`
     Running `target\debug\hello-machine-check.exe --property "AF![AG![p == 1]]"`
[2025-10-28T22:30:26Z INFO  machine_check] Starting verification.
[2025-10-28T22:30:26Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-10-28T22:30:26Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-10-28T22:30:26Z INFO  machine_check::verify] Verifying the given property.
[2025-10-28T22:30:26Z INFO  machine_check] Verification ended.
+------------------------------+
|    Result: DOES NOT HOLD     |
+------------------------------+
|  Refinements:             1  |
|  Generated states:        8  |
|  Final states:            3  |
|  Generated transitions:   9  |
|  Final transitions:       5  |
+------------------------------+
```

This does not hold. The problem is that while we are looping in state 0, we are threatened by the possibility of going to the state 1, where *p* does not hold.

The property *infinitely often* allows us to suspend our disbelief for the moment we will be in the state 1. The translation of the property from LTL to μ-calculus is non-trivial but well-known, `'lfp![X,gfp![Y, AX![X] || (p == 1 && AX![Y])]]'`. Verifying with **machine-check**:

```console
$ cargo run -- --property 'lfp![X,gfp![Y, AX![X] || (p == 1 && AX![Y])]]'
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.10s
warning: the following packages contain code that will be rejected by a future version of Rust: partitions v0.2.4
note: to see what the problems were, use the option `--future-incompat-report`, or run `cargo report future-incompatibilities --id 2`
     Running `target\debug\hello-machine-check.exe --property "lfp![X,gfp![Y, AX![X] || (p == 1 && AX![Y])]]"`
[2025-10-28T22:30:45Z INFO  machine_check] Starting verification.
[2025-10-28T22:30:45Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-10-28T22:30:45Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-10-28T22:30:45Z INFO  machine_check::verify] Verifying the given property.
[2025-10-28T22:30:45Z INFO  machine_check] Verification ended.
+------------------------------+
|        Result: HOLDS         |
+------------------------------+
|  Refinements:             1  |
|  Generated states:        8  |
|  Final states:            3  |
|  Generated transitions:   9  |
|  Final transitions:       5  |
+------------------------------+
```

As such, *p* holds *infinitely often* but not *eventually forever* in this system.


>
> &#x26A0;&#xFE0F; While μ-calculus properties must be by definition monotone, this is not enforced by the current version of **machine-check** in favour of richness of properties, and it is possible to e.g. force the verification into an infinite loop by verifying `lfp![X,!X]`, which will toggle the values of the fixed-point variable between zero and one infinitely. Monotonicity can be trivially ensured by only using AND and OR to manipulate the variable within its fixed point, without any negations or other applied functions/operators.
>