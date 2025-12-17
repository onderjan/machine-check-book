# machine-check-hw

[**Machine-check-hw**](https://docs.rs/machine-check-hw/latest/machine_check_hw/) is a tool for verifying hardware systems in the [Btor2](https://fmv.jku.at/papers/NiemetzPreinerWolfBiere-CAV18.pdf) specification format. It translates the system to the subset of Rust supported by **machine-check**, compiles the resulting Rust package using Cargo or rustc together with verification logic, and runs the executable.

To install **machine-check-hw**, execute
```console
$ cargo install machine-check-hw
```

The next step is optional, but useful for reasonable performance. To avoid downloading all of the libraries for every machine compilation, first run
```console
$ machine-check-hw prepare
```
This will create a new directory `machine-check-preparation` in the executable installation directory which contains the needed libraries, speeding up compilation and avoiding subsequent downloads, allowing offline verification.

In case something goes wrong with preparation, you can revert back to no-preparation mode by running
```console
$ machine-check-hw prepare --clean
```

## Verification

It is possible to verify various properties of Btor2 systems using **machine-check-hw**. The inherent property is also verified during property verification: there are no explicit panics created from the Btor2 files, but division and remainder by zero violate the inherent property.

## Safety

To actually verify something, we can obtain a simple Btor2 system, e.g. [`beads.btor2`](https://docs.rs/crate/machine-check-hw/0.7.1/source/examples/beads.btor2) from **machine-check-hw** examples. By pointing machine-check-hw to a Btor2 file, it can verify the safety of the system, as specified in the Btor2 file, with a property `AG![safe == 1]`, which uses a special field `safe`:
```console
$ machine-check-hw verify ./beads.btor2 --property 'AG![safe == 1]'
[2025-10-28T22:38:52Z INFO  machine_check_compile::verify] Transcribing the system into a machine.
[2025-10-28T22:38:52Z INFO  machine_check_compile::verify] Building a machine verifier.
[2025-10-28T22:38:58Z INFO  machine_check_compile::verify] Executing the machine verifier.
[2025-10-28T22:38:58Z INFO  machine_check] Starting verification.
[2025-10-28T22:38:58Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-10-28T22:38:58Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-10-28T22:38:58Z INFO  machine_check::verify] Verifying the given property.
[2025-10-28T22:38:58Z INFO  machine_check] Verification ended.
[2025-10-28T22:38:58Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0008279), build_time: Some(6.1309139), execution_time: Some(0.0952816), prepared: Some(true) }
[2025-10-28T22:38:58Z INFO  machine_check_hw::verify] Used 3 states and 0 refinements.
[2025-10-28T22:38:58Z INFO  machine_check_hw::verify] Reached conclusion: true
```

## Recovery

We can also verify the property of whether the system recovers to its initial inputs (for the state nodes where they are specified). This can be done using the property `AG![EF![eq_init == 1]]`, which uses a special field `eq_init`:
```console
$ machine-check-hw verify ./beads.btor2 --property 'AG![EF![eq_init == 1]]'
(...)
[2025-10-28T22:39:13Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0007483), build_time: Some(5.8962471), execution_time: Some(0.0995338), prepared: Some(true) }
[2025-10-28T22:39:13Z INFO  machine_check_hw::verify] Used 5 states and 5 refinements.
[2025-10-28T22:39:13Z INFO  machine_check_hw::verify] Reached conclusion: true
```

## State properties

Instead of verifying reachability properties in the file, we can verify custom Computation Tree Logic properties on its states. For example, the `beads.btor2` has three states (100, 200, 300) which determine bead positions and a single input. If the input is 0, the beads stay in their positions, if it is 1, they move to the next position. **Machine-check-hw** creates a state variable `node_X` for each state node identifier `X` in the state. This means that we can verify various properties on the states.

```console
$ machine-check-hw verify ./beads.btor2 --property 'EG![node_200 == 1]'
(...)
[2025-10-28T22:39:50Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0006642), build_time: Some(5.8038741), execution_time: Some(0.0949872), prepared: Some(true) }
[2025-10-28T22:39:50Z INFO  machine_check_hw::verify] Used 3 states and 0 refinements.
[2025-10-28T22:39:50Z INFO  machine_check_hw::verify] Reached conclusion: false
```
The conclusion is false, i.e. there does not exist a path where the bead is always in the second position.

```console
$ machine-check-hw verify ./beads.btor2 --property 'EF![EG![node_200 == 1]]'
(...)
[2025-10-28T22:39:27Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.000709), build_time: Some(5.914955), execution_time: Some(0.0975522), prepared: Some(true) }
[2025-10-28T22:39:27Z INFO  machine_check_hw::verify] Used 6 states and 3 refinements.
[2025-10-28T22:39:27Z INFO  machine_check_hw::verify] Reached conclusion: true
```
The conclusion is true, i.e. there exists a future where the bead moves to second position and then stays there. 

```console
$ machine-check-hw verify ./beads.btor2 --property 'AF![node_200 == 1]'
(...)
[2025-10-28T22:40:05Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0008921), build_time: Some(6.03603), execution_time: Some(0.0960045), prepared: Some(true) }
[2025-10-28T22:40:05Z INFO  machine_check_hw::verify] Used 5 states and 1 refinements.
[2025-10-28T22:40:05Z INFO  machine_check_hw::verify] Reached conclusion: false
```
The conclusion is false, i.e. we cannot be sure that the bead will ever move to the second position.

```console
$ machine-check-hw verify ./beads.btor2 --property 'EU![node_100 == 1, node_200 == 1]'
(...)
[2025-10-28T22:40:19Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0010942), build_time: Some(5.7909552), execution_time: Some(0.0959956), prepared: Some(true) }
[2025-10-28T22:40:19Z INFO  machine_check_hw::verify] Used 5 states and 1 refinements.
[2025-10-28T22:40:19Z INFO  machine_check_hw::verify] Reached conclusion: true
```
The conclusion is true, i.e. there exists a path where the the bead is in the first position until it is in the second position, and the second position is reached.

```console
$ machine-check-hw verify ./beads.btor2 --property 'AU![node_100 == 1, node_200 == 1]'
(...)
[2025-10-28T22:40:37Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.001045), build_time: Some(5.9079046), execution_time: Some(0.0968339), prepared: Some(true) }
[2025-10-28T22:40:37Z INFO  machine_check_hw::verify] Used 5 states and 1 refinements.
[2025-10-28T22:40:37Z INFO  machine_check_hw::verify] Reached conclusion: false
```
The conclusion is false, i.e. the bead stops being in the first position before being in the second position (which we know is not possible), or the second position is never reached (which is possible if it stays in the first position forever).

```console
$ machine-check-hw verify ./beads.btor2 --property 'AG![!(node_200 == 1) || EX![node_300 == 1]]'
(...)
[2025-10-28T22:40:51Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0006929), build_time: Some(5.9525331999999995), execution_time: Some(0.096231), prepared: Some(true) }
[2025-10-28T22:40:51Z INFO  machine_check_hw::verify] Used 5 states and 5 refinements.
[2025-10-28T22:40:51Z INFO  machine_check_hw::verify] Reached conclusion: true

$ machine-check-hw verify ./beads.btor2 --property 'AG![!(node_200 == 1) || EX![node_100 == 1]]'
(...)
[2025-10-28T22:41:05Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0006606), build_time: Some(5.8645651), execution_time: Some(0.0946764), prepared: Some(true) }
[2025-10-28T22:41:05Z INFO  machine_check_hw::verify] Used 5 states and 1 refinements.
[2025-10-28T22:41:05Z INFO  machine_check_hw::verify] Reached conclusion: false
```
The conclusions mean that the bead being in the second position implies that the bead can move to the third position in the next state. However, it cannot move to the first position so quickly.


## Limitations

The Btor2 files must contain only bit-vectors (no arrays) with at most 64 bits. Only safety is supported within Btor2, with no support for fairness, invariant constraints, and justice properties.

It is currently not feasible to verify very complex hardware systems verifiable by state-of-the-art hardware verificaiton tools. However, unlike many such tools, **machine-check** does not focus 
on specific properties such as safety, allowing verification of arbitrary Computation Tree Logic properties.

>
> &#x1F4A1;&#xFE0F; The [HWMCC competition](https://hwmcc.github.io/) compares the state-of-the-art Btor2 verification tools.
>

