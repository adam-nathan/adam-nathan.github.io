---
layout: post
title: "Monadic unifications"
---

Previously I implemented unification and gave examples such as the following:

```fsharp
    <@ (%x,%y) @> == <@ (1,2) @>      // x/1, y/2
```
This is nice but not so powerful, the real benefit comes from chaining these into a rule and is the heart of logic programming. Looking at our current signatures.

```fsharp
type Substitution = Map<Var,Expr>

val walk : e:Expr -> s:Substitution -> Expr

> let inline (==) u v = unify u v ;;
val inline ( == ) : u:Expr -> v:Expr -> (Substitution -> Substitution option)
```

We define the term `statement` and create a list of statements as such:

```fsharp
let statements = [
    ?x,?y == 1,2
    ?x + ?y + ?z == 6
    ?w == ?z - ?x
]

val statements : (Substitution -> Substitution option) list
```

The careful reader will observe that we need a function to convert `(Substitution -> Substitution option) list` to `Substitution -> Substitution option` which is commonly known as `reduce`. 

```fsharp
val reduce : ('T -> 'T -> 'T) -> 'T list -> 'T

// let's derive this reducer
val reducer : (Substitution -> Substitution option) -> (Substitution -> Substitution option) -> (Substitution -> Substitution option);;

// 1. parameterize the Substitution type and implement
val reducer : ('S -> 'S option) -> ('S -> 'S option) -> ('S -> 'S option);;

let reducer m n = 
    fun subst -> 
        match m subst with 
        | None -> None
        | Some subst' -> n subst'  // apply to next argument 

```

This is essentially the [Kliesli operator](https://en.wikipedia.org/wiki/Kleisli_category) in the [Maybe monad](https://en.wikipedia.org/wiki/Monad_(functional_programming)) category. It is beyond the scope of this post to explain what this actually means but it's sufficient to inspect the signature to verify that this is correct.


```fsharp
// >=> is the common notation aka fish operator
let inline (>=>) f g a = f a >>= g // >>= is also known as the bind operator

let inline (>>=) (m:'a option) (f:'a -> 'b option) : 'b option = Option.bind f m

// putting it together
val inline (>=>) (f:'a option) (g:'a option) (a:'a) : 'a option;;
```

## Finally putting it together
The last thing I want to do is rename the fish operator to something more intuitive looking like `&&&`. Now we can start to write logic rules by constantly chaining statements together in a very neat looking way.

```fsharp
let rule = 
    ?x,?y == 1,2        &&&
    ?x + ?y + ?z == 6   &&&
    ?w == ?z - ?x       &&&
```

