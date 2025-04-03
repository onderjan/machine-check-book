# Properties

**Machine-check** can verify properties in [Computation Tree Logic](http://en.wikipedia.org/wiki/Computation_tree_logic). Let's start with a very simple property: is the value zero at the start of the computation?

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

Let's not focus on what the messages or numbers mean and just look at the verification result.  **Machine-check** has determined that the property `value == 0` holds, which matches the definition of the init function. Note that if the system had multiple initial states, the convention is to verify whether the property holds in all of them. But since we only have one initial state where `value = 0`, we don't have to think about this. 

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

Wonderful. But we have just started to discover the possibilities. While **AX** and **EX** can be used to reason about the states directly after the current one, CTL also contains operators we can use to reason about unbounded periods of time. **AG**\[*a*\] means that for all inputs, the property *a* will hold *globally*, i.e. in all time instants including and after this one. In our example, for all input sequences, `value` will globally be lesser or equal to 15. (In case **AG** is used as the outer operator, this is usually written as "in all reachable states, `value` will be lesser or equal to 15.") Note that writing this as a **machine-check** property, we have to explicitly say that `value` should be compared as an unsigned number using `as_unsigned` before comparing it:

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

[^1]: The **AR** and **ER** operators are usually not described in the literature on CTL, but can be derived from other operators.

For example, we can make sure that there exists a sequence of inputs where `value` is lesser than 3 until it is 3 (which occurs eventually):

```console
$ cargo run -- --property "EU![as_unsigned(value) < 3, value == 3]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
     Running `target\debug\hello-machine-check.exe --property "EU![as_unsigned(value) < 3, value == 3]"`
[2025-04-03T14:36:52Z INFO  machine_check] Starting verification.
[2025-04-03T14:36:52Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T14:36:52Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T14:36:52Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T14:36:52Z INFO  machine_check] Verification ended.
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

And for all input sequences, `value` being 3 releases the requirement for `value` to be at most 3:

```console
$ cargo run -- --property "AR![value == 3, as_unsigned(value) <= 3]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
     Running `target\debug\hello-machine-check.exe --property "AR![value == 3, as_unsigned(value) <= 3]"`
[2025-04-03T14:38:32Z INFO  machine_check] Starting verification.
[2025-04-03T14:38:32Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T14:38:32Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T14:38:32Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T14:38:32Z INFO  machine_check] Verification ended.
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

Note the differences in the formulation of the properties. In addition to CTL operators, **machine-check** properties support logical not (`!`), and (`&&`), or (`||`) operators, letting us construct more descriptive properties. For example, we can say that in all reachable states, `value` being 5 implies that, for all inputs, it will be 5 or 6 in the next step. Since an implication *a* => *b* can be written as `!(a) || b`, we can verify:


```console
$ cargo run -- --property "AG![!(value == 5) || AX![value == 5 || value == 6]]"
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
     Running `target\debug\hello-machine-check.exe --property "AG![!(value == 5) || AX![value == 5 || value == 6]]"`
[2025-04-03T14:44:02Z INFO  machine_check] Starting verification.
[2025-04-03T14:44:02Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-04-03T14:44:02Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-04-03T14:44:02Z INFO  machine_check::verify] Verifying the given property.
[2025-04-03T14:44:02Z INFO  machine_check] Verification ended.
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

Last but not least, going back to the example from [Quickstart](./ch1_quickstart.md), we can verify a *recovery* property stating that in all reachable states, there exists an input sequence where we can get back to `value` being zero:

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

Note that this property is not possible to verify using formal verification tools that use verification based on traditional Counterexample-based Abstraction Refinement. **Machine-check** can verify this property thanks to [the used Three-valued Abstraction Refinement framework](../under_the_hood/research.md).
