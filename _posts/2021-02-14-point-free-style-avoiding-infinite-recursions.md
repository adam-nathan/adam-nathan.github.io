---
layout: post
title: "Point-free style - avoiding infinite recursions"
---

Previously, I had tried to implement the `contains` relation and I arrived at a limited solution:

```fsharp
let inline (!) (x:'a) : Expr<'a> = Expr.Value x |> cast
let contains x vs = List.reduce (|||) [for v in vs -> x == !v]
```

Ideally, I want to build more complicated relations from more primitive relations, such as unify.

## First, a refactor
I am finding that using the custom operations to make my DSL is becoming cumbersome; not only is the code is not so friendly to read but it doesn't give me the same power to compose higher-order relationships. The first thing I will do is to place the logic outside of the computation expressions. I think I will keep the computation expressions because it still allows me to take advantage of auto-quotation using the `ReflectedDefinition` attribute.

Firstly, `disjunction` and `conjunctions` can be made into lists.
```fsharp
let run goal = goal Map.empty
let any (goals:Goal list) : Goal = List.reduce (|||) goals
let all (goals:Goal list) : Goal = List.reduce (&&&) goals
let dnf (goals: Goal list list) : Goal = any (List.map all goals) // disjunctive normal form
```

### Re-implementing the current functionality.
```fsharp
let cons a d l = <@ %a::%d @> == l
let nil xs = xs == <@ [] @>
let car l a = cons a var l
let cdr l d = cons var d l
```

### The 'contains' relation
```fsharp
let rec contains xs x =
    let d = var
    dnf [
        [car xs x]
        [cdr xs d; contains d x]
    ]
```

## Bang!
I get an error! The problem here is the recursive definition of `contains` causes an infinite loop. It's strange because technically the code is point-free and it should not run until receiving the Substitution argument. I need more investigation to see what is happening behind the scenes. The solution was to *pause* the execution using another partial application.

```fsharp
let recurse (f:unit -> Goal) subst = f () subst

let rec contains xs x () =
    let d = var
    dnf [
        [car xs x]
        [cdr xs d; recurse (contains d x)]
    ]
```

## Explanation of contains
Although it might seem unfamiliar at first the code is clear:

```fsharp
    dnf [                                       // basically a list of logical ORs of logical ANDs i.e.  (a & b) | (c & d & e) | f
        [car xs x]                              // the head of the list is definitely in the list
        [cdr xs d; recurse (contains d x)]      // the tail of the list is put into the variable `d` and the routine is repeated
    ]
```

More to come. Stay tuned!