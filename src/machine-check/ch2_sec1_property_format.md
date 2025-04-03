# Property Format

In this section, the format of properties will be introduced slightly more formally (but ignoring the precise details) using the [Extended Backus-Naur Form](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form). Whitespace is assumed to be stripped between tokens.

>
> &#x26A0;&#xFE0F; The current property format is a placeholder to approximately emulate Rust syntax. This may change to a proper subset of Rust in the future.
>
> The given EBNF form is only intended for understanding and may not be completely precise.
>

```ebnf
letter = "A" | "B" | "C" | "D" | "E" | "F" | "G"
       | "H" | "I" | "J" | "K" | "L" | "M" | "N"
       | "O" | "P" | "Q" | "R" | "S" | "T" | "U"
       | "V" | "W" | "X" | "Y" | "Z" | "a" | "b"
       | "c" | "d" | "e" | "f" | "g" | "h" | "i"
       | "j" | "k" | "l" | "m" | "n" | "o" | "p"
       | "q" | "r" | "s" | "t" | "u" | "v" | "w"
       | "x" | "y" | "z" ;
digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
hex_digit = digit | "A" | "B" | "C" | "D" | "E" | "F" | 
        "a" | "b" | "c" | "d" | "e" | "f" ;

number = digit, {digit} | "-", digit, {digit} | "0x", hex_digit, {hex_digit};

equality_operator = "==" | "!=";
comparison_operator = "==" | "!=" | "<" | "<=" | ">" | ">=";
ident = letter, { letter, digit, "_" };
macro = "AX!" | "EX!" | "AG!" | "EG!" | "AF!" | "EF!" 
        | "AU!" | "EU!" | "AR!" | "ER!"; 

indexing = "[", number, "]";
left_side = ident, [indexing],
atomic_property = left_side, equality_operator, number |
    "as_unsigned", "(", left_side, ")", comparison_operator, number |
    "as_signed", "(", left_side, ")", comparison_operator, number;
negation = "!", "(", property, ")";
property_part = atomic_property 
        | negation 
        | macro, "[", property, {",", property} , "]";
property = property_part, {"&&", property_part} 
        | property_part, {"||", property_part};
```

