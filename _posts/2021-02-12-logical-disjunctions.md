---
title: "Logical disjunctions"
---

Currently our little logic language is severely limited because we only return 1 or 0 expressions, but alot of the time many solutions are needed. Take for example the following program:

```fsharp
let x = var
logic {
    contains [1;2;3;4;5] %x
    gt %x 2
    solve %x
}
```
This hypothetical query should return all the values greater than 2.

## Extending the logic monad
We'll start by making the monad type explicit and making things consistent.

```fsharp
type Monad= Substitution -> Substitution option

let inline (==) u v : Monad = unify u v 

type LogicBuilder() = 
    member __.Yield _ : Monad = Some

    [<CustomOperation("solve")>] member __.Solve(m:Monad, [<ReflectedDefinition>]x:Expr<_>)  = fun s -> Option.bind (Some << walk x) (m s)

    [<CustomOperation("eq")>] 
    member __.Eq(m:Monad, [<ReflectedDefinition>] u:Expr<_>, [<ReflectedDefinition>] v:Expr<_>) : Monad = fun s -> Option.bind (u == v) (m s)

    [<CustomOperation("cons")>] 
    member __.Cons(m:Monad, [<ReflectedDefinition>] head:Expr<'a>, [<ReflectedDefinition>]tail:Expr<'a list>, [<ReflectedDefinition>]list:Expr<'a list>) : Monad = __.Eq (m, %head :: %tail, %list)

    [<CustomOperation("car")>] 
    member __.Car(m:Monad, [<ReflectedDefinition>] a:Expr<'a>, [<ReflectedDefinition>]l:Expr<'a list>) : Monad = __.Cons(m, %a, %var, %l)

    [<CustomOperation("cdr")>] 
    member __.Cdr(m:Monad, [<ReflectedDefinition>] d:Expr<'a list>, [<ReflectedDefinition>]l:Expr<'a list>) : Monad = __.Cons(m, %var, %d, %l)
```
The first thing to notice is that the `solve` operation does not return a `Monad` type, so I think I`ll remove this until I can find the proper place for it.

Next I change the `Monad` from `Substitution -> Substitution option` to `Substition -> Substitution list`. Common vernacular is to call this monad a `Goal`.
I need a few minor changes to fix the errors.

```fsharp
type Goal = Substitution -> Substitution list

let inline (==) u v : Goal = fun s -> 
    match unify u v s with 
    | None -> []
    | Some s' -> [s']


type LogicBuilder() = 
    member __.Yield _ : Goal = List.singleton
    member __.Eq(m:Goal, [<ReflectedDefinition>] u:Expr<_>, [<ReflectedDefinition>] v:Expr<_>) : Goal = fun s -> List.collect (u == v) (m s)
```
Knowledge about category theory will tell you that `yield` and `eq` are still both `return` and `bind` respectively.

## Disjunction
Given the above saying [1..5] contains x can be written as:

```fsharp
let inline (!) (x:'a) : Expr<'a> = Expr.Value x |> cast

let x = var<int>
[for i in [1..5] -> (x == !i) Map.empty]

val x : Expr<int> = 2d9f0939-a217-40f2-96cb-e3be345eb7a5
val it : Substitution list list =
  [[map [(2d9f0939-a217-40f2-96cb-e3be345eb7a5, Value (1))]];
   [map [(2d9f0939-a217-40f2-96cb-e3be345eb7a5, Value (2))]];
   [map [(2d9f0939-a217-40f2-96cb-e3be345eb7a5, Value (3))]];
   [map [(2d9f0939-a217-40f2-96cb-e3be345eb7a5, Value (4))]];
   [map [(2d9f0939-a217-40f2-96cb-e3be345eb7a5, Value (5))]]]
```
However this does not have the signature of the `Monad` type above. So let's fix that.

```fsharp
let inline (|||) (m:Goal) (n:Goal) : Goal = 
    fun subst -> m subst @ n subst
```
This is similar to the `&&&` we had before. Basically I'm taking using the current substitution and appending the results from both goals.

```fsharp
let contains x vs = 
    List.reduce (|||) [for v in vs -> x == !v]

let a = var<int>
let q = contains a [1;2;3] (Map.empty)


val a : Expr<int> = ff3ad016-22a7-419a-8201-5bb438b6d8e7
val q : Substitution list =
  [map [(ff3ad016-22a7-419a-8201-5bb438b6d8e7, Value (1))];
   map [(ff3ad016-22a7-419a-8201-5bb438b6d8e7, Value (2))];
   map [(ff3ad016-22a7-419a-8201-5bb438b6d8e7, Value (3))]]
```
It's not exactly perfect because the values have to be constant and the language is diverging from the computation expressions. I'll fix this later.


