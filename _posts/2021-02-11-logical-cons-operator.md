---
title: "Logical cons operator"
---

One of the most useful operators in logical programming is matching the list pattern via `cons` operator. In F# typically you would do `let hd::tl = ...`. How can this be done in logic programming using quotations?
Now that I have the unify operator implemented it's actually quite easy to build up new constructs.

## Extending the logic builder
```fsharp
type LogicBuilder with 
    [<CustomOperation("cons")>] 
    member __.Cons(m, [<ReflectedDefinition>] head:Expr<'a>, [<ReflectedDefinition>]tail:Expr<'a list>, [<ReflectedDefinition>]list:Expr<'a list>) = 
        __.Eq (m, %head :: %tail, %list)
```
Notice the same pattern as before; we use the ReflectedDefinition to auto-quote the parameters, and we call the member method. (**note:** I'm splicing outside any visible quotes -- very cool!)

## Implementing the remaining operators
The complement to cons is `car` (head), and `cdr` (tail); the names here are from `LISP`. 

```fsharp
    [<CustomOperation("car")>] 
    member __.Car(m, [<ReflectedDefinition>] a:Expr<'a>, [<ReflectedDefinition>]l:Expr<'a list>) = 
        let ___ = var
        __.Cons(m, %a, %___, %l)
    [<CustomOperation("cdr")>] 
    member __.Cdr(m, [<ReflectedDefinition>] d:Expr<'a list>, [<ReflectedDefinition>]l:Expr<'a list>) = 
        let ___ = var
        __.Cons(m, %___, %d, %l)
```

Notice here that I am creating temporary disposable variables, unification with these will succeed but I don't care to know the value.

## Cons in action

```fsharp
let fst = var
let tl = var
let snd = var
let lst = var

logic {
    car %fst [1;2;3]        // let fst::tl = [1;2;3]
    cdr %tl [1;2;3]         
    car %snd %tl            // let snd::_ = tl
    cons %fst [%snd] %lst   // let lst = fst::[snd]
    solve %lst
}


val it : Expr option =
  Some
    NewUnionCase (Cons, Value (1),
              NewUnionCase (Cons, Value (2), NewUnionCase (Empty)))

    // [1;2]
```