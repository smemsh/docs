Operator precedence
==============================================================================

from https://docs.python.org/3/reference/expressions.html#operator-precedence

highest to lowest

| exponentiation, conditionals: right-left chaining
| everything else: left-right

::

 Operator                  Description

 (expressions...),         binding or parenthesized expression
 [expressions...],         list display
 {key: value...},          dictionary display
 {expressions...}          set display

 x[index],                 subscription
 x[index:index],           slicing
 x(arguments...),          call
 x.attribute               attribute reference

 await x                   Await expression
 **                        exponentiation
 +x, -x, ~x                positive, negative, bitwise NOT
 *, @, /, //, %            integer/matrix multiply, divide, floor, modulo
 +, -                      addition and subtraction
 <<, >>                    shifts
 &                         bitwise AND
 ^                         bitwise XOR
 |                         bitwise OR

 <, <=, >, >=, !=, ==,     comparisons,
 in, not in,               membership,
 is, is not                identity tests

 not x                     boolean NOT
 and                       boolean AND
 or                        boolean OR
 if/else                   conditional expression
 lambda                    lambda expression
 :=                        assignment expression
