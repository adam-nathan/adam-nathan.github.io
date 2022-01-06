---
title: "Unification using F# quotations"
---

Previously I explained the concept of unification and substitution. I also showed how to apply substitution to an expression using a substitution map.

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

The problem is that I am manually creating the substitution map instead of algorithmically using unification.

## Unification
Let's see how to write such a function we shall call `unify` using the example from the previous post.

```ruby
?x :enjoys ?y.
?x :is hungry.
joe :enjoys pizza.
mary :is hungry.
```
Two expressions `A` and `B` *unify* if and only if:
 1) `A` and `B` have the same structure.
 2) Terms `a` and `b` which are in the same relative position in expressions `A` and `B` respectively must be equal if both are non-variables.
 3) Variables are resolved using a substitution map and must satisfy 1 and 2; non-resolved variables are added to the substitution map.

```ruby
(?x :enjoys ?y) =?= (joe :enjoys pizza)  ===> OK [x/joe; y/pizza]
(joe :is hungry) =?= (joe :is tired)    ===> FAILS 
(?x :is hungry) =?= (mary :is hungry)    ===> FAILS because mary != joe
```

Now time to implement.

```fsharp
// first I improve my original definition of substitution by recursively substituting sub-expressions. I call this `walk`.
let rec walk (e:Expr) s = 
    e.Substitute(fun var -> 
        match s.TryFind(var) with 
        | Some v -> Some (walk v s)
        | _ -> None)

let rec unify u v s = 
    match walk u s, walk v s with                                 // case 3: resolve variables by recursively walking (aka substituting)
    | x, y when x = y -> Some s                                   // case 1: equal terms unify
    | P.Var x, expr | expr, P.Var x -> Map.add x expr s |> Some   // case 3: substitution map is extended with [variable/term] pair
    | P.Application(x, argx), P.Application(y, argy) ->           // case 2: matching structure
        match unify x y s with             // try to unify the sub-expression 
        | None -> None  
        | Some s' -> unify argx argy s'    // if successful then s' contains the extended substitution map so we continue trying to unify until completion or failure 
    | P.Lambda(xhead,xbody), P.Lambda(yhead,ybody) -> // like before
    // match all patterns
    // ...
    // ...
    | _,_ -> None     // cannot match so FAIL
```

Matching all the possible combinations of expressions is a lot of work. Fortunately there is a general pattern called ExpressionShape.

```fsharp
module P = Patterns
module ES = ExprShape

let rec unify u v s = 
    match walk u s, walk v s with 
    | x, y when x = y -> Some s
    | P.Var x, expr | expr, P.Var x -> Map.add x expr s |> Some
    | ES.ShapeCombination(_, args1), ES.ShapeCombination(_,args2) when u.Type = v.Type && args1 <> [] && args2 <> [] -> 
        let folder s u v = 
            match s with 
            | None -> None
            | Some s' -> unify u v s'
        List.fold2 folder (Some s) args1 args2      // zip the sub-expressions [u1; u2; u3 ...] with [v1; v2; v3 ...] gives [u1,v1; u2,v2; u3,v3 ...]
                                                    // and fold with Option.bind, so: (unify u1 v1) >=> (unify u2 v2) >=> (unify u3 v3) etc.
    | _,_ -> None
```

Let's try this out.

```fsharp
// define some variables
let x = Expr.Var(Var("x", typeof<int>)) |> Expr.Cast<int>
let y = Expr.Var(Var("y", typeof<int>)) |> Expr.Cast<int>
let z = Expr.Var(Var("z", typeof<int>)) |> Expr.Cast<int>

let inline (==) u v = 
  assert (Option.isSome (unify u v Map.empty))

x == x                            // ok
x == y                            // x/y
x == <@ 1 @>                      // x/1 
<@ 1 @> == x                      // x/1
<@ 1 @> == <@ 1 @>                // ok
<@ 2 @> == <@ 1 @>                // fails
<@ (%x,%y) @> == <@ (1,2) @>      // x/1, y/2
<@ (%x,%x) @> == <@ (%y,%z) @>    // x/y, y/z
<@ (%x,%x) @> == <@ (1,2) @>      // fails
```

Perfect!