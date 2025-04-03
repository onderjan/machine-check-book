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

Let's not focus on what the messages or numbers mean and just look at the verification result.  **Machine-check** has determined that the property `value == 0` holds, which matches the definition of the init function. 

>
> &#x26A0;&#xFE0F; Currently, if the system has multiple initial states, **machine-check** follows the convention that the property holds exactly if it holds in all of the initial states.
>
> This behaviour may change in the future once system parametrisation is implemented.
>

Verifying a single state is admittedly quite boring. The real power of the [Computation Tree Logic](http://en.wikipedia.org/wiki/Computation_tree_logic) (CTL) is that we can reason about things *temporally*, looking into states in the future. For example, we can use the *temporal operator* **AX**\[*φ*\] which means "for all paths, *φ* will hold in the next state" to say "for all paths, `value` will be 0 in the current state". In **machine-check**, since the paths are determined only by the machine inputs, we can also say "for all input sequences" instead of "for all paths", which can be more intutive. In **machine-check** property syntax, we write this as `AX![value == 0]`. The exclamation mark after `AX`: this is done as the property syntax emulates Rust syntax and it makes a lot of sense to think of the temporal properties as macros, since they change what the variables inside them represent. That aside, let's try to verify.

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

>
> &#x1F4A1;&#xFE0F; In this handbook, the verifier executable is built in debug mode, and uses the default strategy, using `cargo run -- --property <PROPERTY>`.
>
> If verification becomes too slow for your system, try compiling in release mode:
>
> `cargo run --release -- --property <PROPERTY>`
>
> This will slow down compilation but produce a more optimised executable that will be able to verify faster.
> You can also try to use **machine-check** with the [decay strategy](./ch4_cli.md) (this can make verification faster or slower based on the system):
>
> `cargo run --release -- --property <PROPERTY> --strategy decay`
>
> Note that the `--` argument that separates the arguments for the build tool cargo from the arguments that will be passed to the built program and processed by **machine-check**.
>

Of course this does not hold: we can increment the value, so `value` will no longer be zero in **all** of the next states! But what about it being zero in **at least one** next state? That should hold: if the `increment_value` input is 0, we will retain zero `value`. Instead of **AX**\[*φ*\], we will use the **EX**\[*φ*\] operator: there exists a path where *φ* holds in next state. Let's try to verify.

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

Wonderful. But we have just started to discover the possibilities. While **AX** and **EX** can be used to reason about the states directly after the current one, CTL also contains operators we can use to reason about unbounded periods of time. **AG**\[*φ*\] means that for all input sequences, the property *φ* will hold *globally*, i.e. in all time instants including and after this one. In our example, for all input sequences, `value` will globally be lesser or equal to 15. (In case **AG** is used as the outer operator, this is usually written as "in all reachable states, `value` will be lesser or equal to 15.") Writing this as a **machine-check** property, we have to explicitly say that `value` should be compared as an unsigned number using `as_unsigned` before comparing it:

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

While this would not hold for a constant lesser than 15 (you can try it), **EG**\[*φ*\] means that there is an infinite input sequence where *φ* holds globally. Reasonably, we can expect that `value` stays to be zero when the `increment_value` input is always kept 0:

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

It does not, since `value` is 0 at the start. To reason about the future, we can use the **AF** and **EF** operators. **AF**\[*φ*\] means that for all input sequences, *φ* holds *eventually*. This can be either currently or in the future. **EF**\[*φ*\] means that there exists an input sequence for which *φ* holds eventually, which is a bit more interesting in the context of our example:

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
For even more descriptive properties, we can use the **AU**, **EU**, **AR**, and **ER** CTL operators[^1]. These take two arguments. Nothwithstanding the difference in the input sequence requirements, **A**\[*φ*\]**U**\[*ψ*\] and **E**\[*φ*\]**U**\[*ψ*\] stipulate that *φ* should hold *until* *ψ* holds, although not necessarily in the position where *ψ* first holds, and *ψ* must hold eventually.  **A**\[*φ*\]**R**\[*ψ*\] and **E**\[*φ*\]**R**\[*ψ*\] stipulate that *φ* *releases* *ψ*: *ψ* has to hold before and including the point where *φ* first holds, and if *φ* never holds, *ψ* must hold globally. Notice how **AU**/**EU** and **AR**/**ER** are essentially extensions of  **AF**/**EF** and **AG**/**EG**, respectively.

[^1]: The **AR** and **ER** operators are usually not described in the literature on CTL, but can be derived from other operators.

In **machine-check** properties, we write these operators as macros with two arguments. For example, we can make sure that there exists an input sequence where `value` is lesser than 3 until it is 3 (which occurs eventually):

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

Note the differences in the formulation of the previous two properties.

In addition to CTL operators, **machine-check** properties support logical not (`!`), and (`&&`), or (`||`) operators, letting us construct more descriptive properties. For example, we can say that in all reachable states, `value` being 5 implies that, for all input sequences, it will be 5 or 6 in the next step. Since an implication *φ* ⇒ *ψ* [can be written](https://en.wikipedia.org/wiki/Material_conditional#Semantics) as ¬*φ* ∨ *ψ*, we can verify:


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

Notably, this property is not possible to verify using formal verification tools that use verification based on the traditional Counterexample-based Abstraction Refinement. **Machine-check** can verify this property thanks to [the used Three-valued Abstraction Refinement framework](../under_the_hood/research.md).
