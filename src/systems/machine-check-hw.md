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

To actually verify something, we can obtain a simple Btor2 system, e.g. [`beads.btor2`](https://docs.rs/crate/machine-check-hw/0.5.0/source/examples/beads.btor2) from **machine-check-hw** examples. By pointing machine-check-hw to a Btor2 file, it can verify the safety of the system, as specified in the Btor2 file, with a property `AG![safe == 1]`, which uses a special field `safe`:
```console
$ machine-check-hw verify ./beads.btor2 --property 'AG![safe == 1]'
[2025-06-15T17:12:53Z INFO  machine_check_compile::verify] Transcribing the system into a machine.
[2025-06-15T17:12:53Z INFO  machine_check_compile::verify] Building a machine verifier.
[2025-06-15T17:12:54Z INFO  machine_check_compile::verify] Executing the machine verifier.
[2025-06-15T17:12:54Z INFO  machine_check] Starting verification.
[2025-06-15T17:12:54Z INFO  machine_check::verify] Verifying the inherent property first.
[2025-06-15T17:12:54Z INFO  machine_check::verify] The inherent property holds, proceeding to the given property.
[2025-06-15T17:12:54Z INFO  machine_check::verify] Verifying the given property.
[2025-06-15T17:12:54Z INFO  machine_check] Verification ended.
[2025-06-15T17:12:54Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.000682), build_time: Some(0.9404879), execution_time: Some(0.0755522), prepared: Some(true) }
[2025-06-15T17:12:54Z INFO  machine_check_hw::verify] Used 3 states and 0 refinements.
[2025-06-15T17:12:54Z INFO  machine_check_hw::verify] Reached conclusion: true
```

## Recovery

We can also verify the property of whether the system recovers to its initial inputs (for the state nodes where they are specified). This can be done using the property `AG![EF![eq_init == 1]]`, which uses a special field `eq_init`:
```console
$ machine-check-hw verify ./beads.btor2 --property 'AG![EF![eq_init == 1]]'
(...)
[2025-06-15T17:29:19Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0008441), build_time: Some(0.8279723), execution_time: Some(0.0752981), prepared: Some(true) }
[2025-06-15T17:29:19Z INFO  machine_check_hw::verify] Used 5 states and 5 refinements.
[2025-06-15T17:29:19Z INFO  machine_check_hw::verify] Reached conclusion: true
```

## State properties

Instead of verifying reachability properties in the file, we can verify custom Computation Tree Logic properties on its states. For example, the `beads.btor2` has three states (100, 200, 300) which determine bead positions and a single input. If the input is 0, the beads stay in their positions, if it is 1, they move to the next position. **Machine-check-hw** creates a state variable `node_X` for each state node identifier `X` in the state. This means that we can verify various properties on the states.

```console
$ machine-check-hw verify ./beads.btor2 --property 'EG![node_200 == 1]'
(...)
[2025-06-15T17:19:39Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0006591), build_time: Some(0.8356801), execution_time: Some(0.0754507), prepared: Some(true) }
[2025-06-15T17:19:39Z INFO  machine_check_hw::verify] Used 3 states and 0 refinements.
[2025-06-15T17:19:39Z INFO  machine_check_hw::verify] Reached conclusion: false
```
The conclusion is false, i.e. there does not exist a path where the bead is always in the second position.

```console
$ machine-check-hw verify ./beads.btor2 --property 'EF![EG![node_200 == 1]]'
(...)
[2025-06-15T17:20:42Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0006911), build_time: Some(0.8240966), execution_time: Some(0.0770154), prepared: Some(true) }
[2025-06-15T17:20:42Z INFO  machine_check_hw::verify] Used 6 states and 3 refinements.
[2025-06-15T17:20:42Z INFO  machine_check_hw::verify] Reached conclusion: true
```
The conclusion is true, i.e. there exists a future where the bead moves to second position and then stays there. 

```console
$ machine-check-hw verify ./beads.btor2 --property 'AF![node_200 == 1]'
(...)
[2025-06-15T17:21:16Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0006654), build_time: Some(0.8297027), execution_time: Some(0.0767318), prepared: Some(true) }
[2025-06-15T17:21:16Z INFO  machine_check_hw::verify] Used 5 states and 1 refinements.
[2025-06-15T17:21:16Z INFO  machine_check_hw::verify] Reached conclusion: false
```
The conclusion is false, i.e. we cannot be sure that the bead will ever move to the second position.

```console
$ machine-check-hw verify ./beads.btor2 --property 'EU![node_100 == 1, node_200 == 1]'
(...)
[2025-06-15T17:21:46Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0008906), build_time: Some(0.8443039), execution_time: Some(0.0740382), prepared: Some(true) }
[2025-06-15T17:21:46Z INFO  machine_check_hw::verify] Used 5 states and 1 refinements.
[2025-06-15T17:21:46Z INFO  machine_check_hw::verify] Reached conclusion: true
```
The conclusion is true, i.e. there exists a path where the the bead is in the first position until it is in the second position, and the second position is reached.

```console
$ machine-check-hw verify ./beads.btor2 --property 'AU![node_100 == 1, node_200 == 1]'
(...)
[2025-06-15T17:22:11Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0006671), build_time: Some(0.8300949), execution_time: Some(0.0777126), prepared: Some(true) }
[2025-06-15T17:22:11Z INFO  machine_check_hw::verify] Used 5 states and 1 refinements.
[2025-06-15T17:22:11Z INFO  machine_check_hw::verify] Reached conclusion: false
```
The conclusion is false, i.e. the bead stops being in the first position before being in the second position (which we know is not possible), or the second position is never reached (which is possible if it stays in the first position forever).

```console
$ machine-check-hw verify ./beads.btor2 --property 'AG![!(node_200 == 1) || EX![node_300 == 1]]'
(...)
[2025-06-15T17:25:20Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0006687), build_time: Some(0.8267717), execution_time: Some(0.0761956), prepared: Some(true) }
[2025-06-15T17:25:20Z INFO  machine_check_hw::verify] Used 5 states and 5 refinements.
[2025-06-15T17:25:20Z INFO  machine_check_hw::verify] Reached conclusion: true

$ machine-check-hw verify ./beads.btor2 --property 'AG![!(node_200 == 1) || EX![node_100 == 1]]'
(...)
[2025-06-15T17:25:33Z INFO  machine_check_compile::verify] Stats: Stats { transcription_time: Some(0.0011374), build_time: Some(0.8286665), execution_time: Some(0.0742507), prepared: Some(true) }
[2025-06-15T17:25:33Z INFO  machine_check_hw::verify] Used 5 states and 1 refinements.
[2025-06-15T17:25:33Z INFO  machine_check_hw::verify] Reached conclusion: false
```
The conclusions mean that the bead being in the second position implies that the bead can move to the third position in the next state. However, it cannot move to the first position so quickly.


## Limitations

The Btor2 files must contain only bit-vectors (no arrays) with at most 64 bits. Only safety is supported within Btor2, with no support for fairness, invariant constraints, and justice properties.

It is currently not feasible to verify very complex hardware systems verifiable by state-of-the-art hardware verificaiton tools. However, unlike many such tools, **machine-check** does not focus 
on specific properties such as safety, allowing verification of arbitrary Computation Tree Logic properties.

>
> &#x1F4A1;&#xFE0F; The [HWMCC competition](https://hwmcc.github.io/) compares the state-of-the-art Btor2 verification tools.
>

