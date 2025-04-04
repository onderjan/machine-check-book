<img src="images/logo.svg" width="250" style="margin: 3em; margin-left: auto; margin-right: auto; display:block">
<div style="text-align: center">
<h1>User Guide to machine-check</h1>
Formal verification of digital systems made easier
</div>

# Preface

This is the user guide for **machine-check**, a tool/library for formal verification of arbitrary digital systems described in the Rust programming language using finite-state machines.

Unlike testing, which determines if a specification holds for a small set of system executions, formal verification can determine whether it holds for all possible executions. While only tiny systems can be verified by brute force, **machine-check** uses an intelligent approach to make verification of larger and more capable systems possible.

Unlike some other verifiers, **machine-check** is designed to be *sound*, never returning a wrong verification result[^1], and *complete*, theoretically always returning a result in finite time and memory. While arbitrary finite-systems can be represented, the current focus is on verification of systems comprised of machine-code programs executed on embedded processors.

Interested? [Read the Quickstart.](./machine-check/ch1_quickstart.md)

> &#x26A0;&#xFE0F; Currently, **machine-check** is in active development, with the [version 0.4.0](https://crates.io/crates/machine-check/0.4.0) described in this guide. While the basic building blocks are in place, some details may still change considerably in the future.
>


## Useful links
 - [The main website](https://machine-check.org/)
 - [The blog](https://machine-check.org/blog/)
 - [Latest stable version at crates.io](https://crates.io/crates/machine-check)
 - [Latest stable API documentation at docs.rs](https://docs.rs/machine-check/)
 - [Development repository at GitHub](https://github.com/onderjan/machine-check)

[^1]: In other words, if the wrong verification result is returned, this is always considered a bug in **machine-check**. If you encounter a bug, [please report it](https://github.com/onderjan/machine-check/issues).

Just like **machine-check**, this book is available under the Apache 2.0 or MIT licence. Its original sources [are available at GitHub](https://github.com/onderjan/machine-check-book).