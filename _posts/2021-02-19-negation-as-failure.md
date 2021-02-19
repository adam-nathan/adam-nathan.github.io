---
layout: post
title: "Negation as failure"
---

Previously, I handled negation by projecting the variables into a predicate; I even handled the case of [non-ground](https://en.wikipedia.org/wiki/Ground_expression) expressions, (unknown variables).

## Early evaluation
However things could be better.

Take the case of the following expression `(x, y, z) =/= (1, 2, 3)`. Say, I do not know what `y` and `z` are but I know that `x` is `3`, the expression is guaranteed to be `true` even though the expression is not ground. 

The expression above can be refactored as: `(x =/= 1) or (y =/= 2) or (z =/= 3)`.

Using [De Morgan's law](https://en.wikipedia.org/wiki/De_Morgan%27s_laws), I can refactor this to be `not (x == 1 &&& y == 2 &&& z == 3)`. In other words, given the `not` goal, I can reuse my current unification logic. How do we implement this `not` goal?

## Negation as Failure
Negation as failure, commonly known as NaF, is used extensively in logic programming, and is relatively powerful but still simple to implement. You can read more about it [here](https://en.wikipedia.org/wiki/Negation_as_failure).

A predicate can be divided into two types:

  1. ¬p (*p is false*)
  2. not p (*p is not true*)

The second type has many interpretations such as `p is unknown`, or `p cannot be shown to be true at this point in time`, or `p may be true in the future`, etc. The difference is subtle but important and is the basis for many forms of logical thinking that are not black and white. NaF uses the [closed-world assumption](https://en.wikipedia.org/wiki/Closed-world_assumption); and what this means is that 1 and 2 are equivalent.

## Implementation
```fsharp
let not (goal:Goal) : Goal = 
    fun a -> 
        match goal a.Substitution with 
        | [] -> [a] // goal failed so continue
        | _ -> []   // goal passed so fail
```



