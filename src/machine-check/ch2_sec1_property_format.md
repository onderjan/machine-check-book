# Property Format

The properties of **machine-check** are described in a subset of Rust expressions, augmented with temporal logic operators and some syntax sugar for easily expressing values.

## Available Variables

All fields of the struct `State` of the machine under verification are available to use just using their struct field names. Together, these variables represent the current state. The fields must be of **machine-check** types [Unsigned](https://docs.rs/machine-check/0.7.1/machine_check/struct.Unsigned.html) and [Signed](https://docs.rs/machine-check/0.7.1/machine_check/struct.Signed.html), [Bitvector](https://docs.rs/machine-check/0.7.1/machine_check/struct.Bitvector.html), or [BitvectorArray](https://docs.rs/machine-check/0.7.1/machine_check/struct.BitvectorArray.html).

>
> &#x1F6E0;&#xFE0F; Currently, the signedness of bit-vectors in the system is not known by the property, i.e. [Unsigned](https://docs.rs/machine-check/0.7.1/machine_check/struct.Unsigned.html) and [Signed](https://docs.rs/machine-check/0.7.1/machine_check/struct.Signed.html) fields of `State` are seen by the property as [Bitvector](https://docs.rs/machine-check/0.7.1/machine_check/struct.Bitvector.html) variables.
>

## Standard Expressions and Syntax Sugar

Standard Rust operators that are supported by the types of fields are also available in property expressions. This notably includes [BitvectorArray](https://docs.rs/machine-check/0.7.1/machine_check/struct.BitvectorArray.html) read (`array_field[bitvector_index]`) and short-circuiting AND (`&&`) and OR (`||`) for Booleans obtained from equality, comparison, or temporal logic operators.

Also supported are constructor functions for the types. The functions must be used by their full paths. For example, `::machine_check::Unsigned::<8>::new(0) != ::machine_check::Unsigned::<8>::new(1)` is a valid type. Of course, this is a bit unwieldy, so literals in equalities/comparisons are coerced to the appropriate **machine-check** type if it is possible to infer that, e.g. `value == 0` is a valid property expression if `value` is a bitvector field.

Also supported are the functions `as_signed` and `as_unsigned`, demonstrated previously. Note that the syntax sugar for literal coercion and `as_signed`/`as_unsigned` are specific to properties only and cannot be used when writing systems.

## Temporal Logic Operators

The temporal logic operators are written as Rust macros. There are two basic types of them: Computation Tree Logic operators and μ-calculus fixed-point operators. Generally, they expect property expressions given as their arguments to evaluate to Boolean values, and they return a Boolean value. The argument expressions are essentially evaluated in new contexts, using the field values of the state they operate on instead of the outer values.

The CTL operators `EX!`, `AX!`, `EF!`, `AF!`, `EG!` and `AG!` take a single property expression. The CTL operators `EU!`, `AU!`, `ER!`, and `AR!` take two property expressions, in the order of arguments usual for CTL. The μ-calculus fixed-point operators [described in an advanced chapter](../advanced/adv2_mu_calculus.md) take two arguments, where the first one is an identifier of a new fixed-point variable that can be used within any nesting of the temporal operators (unless shadowed) and the second one is the property expression to evaluate.