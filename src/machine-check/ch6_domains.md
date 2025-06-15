# Abstraction Domains

The ability of **machine-check** to verify complex systems is powered by two concepts: *abstraction* that lets us reason about more than just concrete states 
and *refinement* that reduces the amount of abstraction if we are not able to prove the property using it. You do not need to know about these domains
for verification, but it is useful for inspecting the state spaces, which can be visualised using the GUI.

The abstraction technique used is *existential abstraction*, where we represent each state or input in an *abstraction domain* that limits the
concrete possibilities. In **machine-check**, it is guaranteed at least one of these possibilities definitely exists.

For abstraction, **machine-check** currently uses the three-valued bit-vector (TVBV) domain, already seen in the [chapter on GUI](ch5_gui.md). This can be 
combined with an experimental *dual-interval* domain using machine-check feature `Zdual_interval`. This feature is also available in the systems provided
in the next chapters.

## Three-valued Bit-vector Domain

Already shown in the [chapter on GUI](ch5_gui.md), the idea of this domain is fairly simple: represent every bit of a state (or an input) by one of three values:
 - `'0'`, meaning the bit is definitely 0,
 - `'1'`, meaning the bit is definitely 1,
 - `'X'`, meaning the bit may be 0 or 1, perhaps both (but we do not know if it actually can be both).

The bits are then displayed in the same manner as numbers in binary (most significant bit on the left, least significant bit on the right). This means that
e.g. `"1X01"` admits two possible concrete values, expressed in binary as 1001 and 1101 or in decimal as 9 or 13.

This domain is useful especially because it plays well with bitwise operations and, in particular, bitmasking. The main reason for introducing it first
in **machine-check** was that for embedded machine-code systems, reading input pins may be encoded in an instructions that reads a whole (e.g. 8-bit) *port*,
then masks it. Using the three-valued domain and reading e.g. bit 2 of some port, we can mask read `"XXXXXXXX"` to `"00000X00"`, which will dramatically
shrink our state space and need for refinements.

## Dual-Interval Domain

Intervals are a very straightforward abstraction: for example, if we have an interval [0,3], this means that the underlying possibilities are 0, 1, 2, and 3.
Again, since this is an abstraction, not all of them must be actually present, we might have just e.g. 1 and 3.

The problem with normal intervals is that they do not handle signedness nicely: if we want to represent signed 0 and -1 using an unsigned interval, we can only 
use the full unsigned interval, making the abstraction fairly useless. This is problematic with digital systems since a lot of the time, the signedness
is determined by the actual instruction or operation, not the data location itself.

We could use both signed and unsigned intervals, but this is rather awkward, as e.g. representing a 4-bit bitvector containing 4 and 12 (-4 in two's complement)
would give us [4,12] ∩ [-4,4] (or, when represented as unsigned, [12,4]). Instead, we can use a *dual-interval* abstraction where we separately consider 
the possibilities for both values of sign bit, obtaining [4,4] ∪ [12,12]. This is done in **machine-check**, and the bounds are represented as if unsigned.
Specially, if the dual-interval abstraction forms just a one unsigned interval, this interval is shown. For example, the abstract value for a 4-bit bitvector 
where the possibilities range from 3 to 9, [3,9] will be shown instead of [3,7] ∪ [8,9]. Specially, if only one value is possible, it will be shown instead.

While the dual-interval domain is not particularly useful for bitwise operations, it can precisely track arithmetic bounds. 


>
> &#x1F6E0;&#xFE0F; The increased precision brought by the dual-interval domain can lead to larger state spaces and longer verification times, 
> which currently tends to be the case for most used test cases when the domain is used. 
> Therefore, it is currently disabled by default until more clever strategies for state space generation and refinement are introduced.
>


## Combining the Domains

Both domains are used together when the `Zdual_interval` feature flag is used. This leads to bit-vector variables having values such as 
 - `"00001000" ⊓ 8` (where only one value is possible),
 - `"0000000100XXXX" ⊓ [71, 72]` (where there exists a single unsigned interval),
 - `"XXXXXXXX" ⊓ ([0, 126] ∪ [152, 255])` (where there exist two intervals with different sign bit, which do not combine to a single unsigned interval).

The symbol ⊓ stands for a *join* operation between the two domains. In essence, only the values admitted by **both** domains are possible. For example,
`"0000000100XXXX" ⊓ [71, 72]` admits only the possibilities 71 and 72.


 
