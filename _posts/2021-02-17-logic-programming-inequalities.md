---
layout: post
title: "Logic programming inequalities"
---

So far I have only dealt with unification but what about other inequalities, ie. >, =/= < etc. ?

A simple solution is to create `impure` relations. What I mean by this is that only some inputs can be variables, while others must be fixed constants. Take for example the following relation:

```fsharp
let uppercase (s:string expr) (t:string expr) = todo

// t is the uppercase of string s
// t is derived from s but not the other way around
let uppercase_impure (s:string) (t:string expr) = todo
```

Impure goals severely limit the power but are much easier to implement:

```fsharp
// create a goal called pred that accepts any boolean statement
let pred (p:bool expr) a = 
    try if (eval << cast) walk p a) then [a] else []
    with | _ -> []

// customer operator
type LogicBuilder() with
    [<CustomOperation("pred")>] 
    member this.Pred(m:Goal, [<ReflectedDefinition>] p:Expr<bool>) : Goal = m &&& pred p


// example
let u = var<int>

logic {
    eq %u 1
    pred (%u > 0)
    solve (%u)
}  // ==> [1]


logic {
    eq %u 1
    pred (%u < 0)
    solve (%u)
}  // ==> [] fails
```
I'm using an external library called [Unquote](https://github.com/SwensenSoftware/unquote) to evaluate the quotation.

### Drawbacks
As you can see it all works however a major drawback is that I must have unified all the relevant variables before running the predicate. There are simple ways to fix this though.