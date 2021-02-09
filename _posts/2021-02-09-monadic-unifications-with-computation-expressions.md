---
layout: post

title: "Monadic unifications with computation expressions"
---

Previously I chained `statements` aka `clauses` using monads and allowed a more powerful logic programming model. Currently it looks like this.

```fsharp
let rule = 
    ?x,?y == 1,2        &&&
    ?x + ?y + ?z == 6   &&&
    ?w == ?z - ?x       &&&
```

I can improve the syntax using the power of [computation expressions](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions). Let's start.

```fsharp
let guid<'a> = Guid.NewGuid() |> string

/// Creates a unique variable
let var<'a> : Expr = Expr.Var <| Var(guid, typeof<obj>)

// Monad type
type M = Substitution option

type LogicBuilder() = 
    member __.Yield x : M = Some Map.empty

let logic = LogicBuilder()
```

The current syntax is a bit ugly so let's prettify it with [custom operations](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions)

```fsharp
type LogicBuilder with
    [<CustomOperation("eq")>] 
    member inline __.Eq(m:M, u:Expr, v:Expr) : M = Option.bind (u == v) m
```
My initial attempt looked like this

```fsharp
let q = logic {
    let x = var
    let y = var
    eq x y
}
```
However this threw the exception `The value or constructor 'x' is not defined.`. Looking at the generated code showed me the issue.

```fsharp
> <@@ logic {
-     let x = var
-     let y = var
-     eq x y
- } @@>
- ;;
val it : Expr =
  Application (Lambda (builder@,
                     Call (Some (builder@), Eq,
                           [Let (x, Call (None, var, []),
                                 Let (y, Call (None, var, []),
                                      Call (Some (builder@), Yield,
                                            [NewTuple (x, y)]))),
                            Coerce (PropertyGet (None, x, []), FSharpExpr),
                            Coerce (PropertyGet (None, y, []), FSharpExpr)])),
             PropertyGet (None, logic, []))
```
As you can see `x` and `y` are being taken from the declaring module and not from the enclosed computation expression. I could start playing around with the `[<ProjectionParameter>]` attributes but at this stage it's easier for me to simply declare the variables outside the expression.

Lastly, for completion, I may as well implement the `walk` function so that I can return some value given the equality clauses.

## Summary
The final implementation looks like this.

```fsharp
let guid<'a> = Guid.NewGuid() |> string
let var<'a> : Expr = Expr.Var <| Var(guid, typeof<obj>)

type LogicBuilder() = 
    member __.Yield _  = Some Map.empty
    [<CustomOperation("eq")>] member inline __.Eq(m, u, v)  = Option.bind (u == v) m
    [<CustomOperation("solve")>] member __.Solve(m, x) = Option.bind (Some << walk x) m

let logic = LogicBuilder()


(*
    let x = var
    let y = var

    > logic {
    -     eq x y
    -     eq x <@ 1 @>
    -     solve y
    - } 
    - ;;
    val it : Expr option = Some Value (1)

    > logic {
    -     eq x y
    -     eq x <@ 1 @>
    -     eq y <@ 2 @>
    -     solve y
    - } 
    - ;;
    val it : Expr option = None
*)
```