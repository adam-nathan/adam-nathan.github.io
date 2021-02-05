---
layout: post
title: "Understanding unification and substitution"
---

Unification and substitution are the bread and butter of program evaluation and form the basis of logic programming such as Prolog as well as a host of other uses such as pattern matching. Looking at the [wikipedia](https://en.wikipedia.org/wiki/Unification_(computer_science)) entry for unification can be daunting but the basics are actually very simple and intuitive to understand.

## Basic intuition
Let's take a very simple example of logic programming; say I have the following sentences:

```ruby
if ?x :enjoys ?y and ?x :is hungry then ?x :eats ?y.
joe :enjoys pizza.
joe :is hungry.
```
It should be obvious to see that joe will eat pizza, but how is this derived?

### Unification
The first step is to pattern match. So we have:

```ruby
?x          joe
:enjoys     :enjoys
?y          pizza
```

`?x` and `?y` are variables and hold values. Since the keyword `:enjoys` is equal we store the variables in an environment called a `substitution`.

### Substitution
Now we `substitute` the variables in the remaining sentences:

```ruby
?x :eats ?y         [ ?x/joe, ?y/pizza ]

==> joe :eats pizza
```

## Implementing in F#
`?x :eats ?y` is called an `expression` and in F# I can make a an expression by using the quotes <@ ... @>. So for example let's say I want to quote `fun x -> x + 1`.

```fsharp 
> <@ fun x -> x + 1 @> ;;

val it : Expr<(int -> int)> =
  Lambda (x, Call (None, op_Addition, [x, Value (1)]))
```

We see here that the we get an expression tree of type `Expr<int -> int>`.

```fsharp
Lambda (                        // a function that
    x                           // accepts an argument called x 
    ,                           // and evaluates the body (x + 1) by 
        Call (                  // applying
            None,               // a static method named
            op_Addition, [      // + with arguments
                    x,          // variable x 
                    Value (1)   // value 1
                ]))             
```

if `(fun x -> x + 1) 2` equals 3 then what do you think `<@ (fun x -> x + 1) 2 @>` should equal? The answer may suprise you!

```fsharp
> <@ (fun x -> x + 1) 2 @> ;;

val it : Expr<int> =
  Application (Lambda (x, Call (None, op_Addition, [x, Value (1)])), Value (2))
```

So the quote prevents evaluation and preserves the structure of everything inside the quotes. We can simulate the same kind of process that an interpreter would do if this was source code.

```fsharp
open FSharp.Quotations
module P = Patterns
module DP = DerivedPatterns
module ES = ExprShape

// pattern match the quotation
let (P.Application(P.Lambda (head, body), arg)) = <@ (fun x -> x + 1) 2 @>;;
>   val head : Var = x
>   val body : Expr = Call (None, op_Addition, [x, Value (1)])
>   val arg : Expr = Value (2)

// create a substitution map by unifying x/2
let substitution : Map<Var, Expr> = Map.ofList [head, arg]      // map [(x, Value (2))]

// substitute into the body using the map
body.Substitute(substitution.TryFind)           // Call (None, op_Addition, [Value (2), Value (1)]) 

// verify the result is correct
<@ 2 + 1 @>;;
>   val it : Expr<int> = Call (None, op_Addition, [Value (2), Value (1)])

```