---
title: "Monadic unifications with computation expressions improved"
---
Previously I showed how it's possible to create a simple logic program using computation expressions. There a few things that bother me though.

- I prefer `eq x 1` instead of `eq x <@ 1 @>`.
- Currenty it's impossible to do `eq (x,y) <@ 1,2 @>` because the lhs argument is now `Expr*Expr` instead of `Expr`.

## Auto Quotations
Currently I need to write code like this:
```fsharp
    let x = var
    let y = var

    logic {
        eq <@ %x, %y @> <@ 1, 2 @>
        eq x <@ 1 @>
        solve y
    } 
```

There is a really powerful metaprogramming feature that came out in F# 4.0, that allows [auto-quotation of arguments of method calls](https://github.com/fsharp/fslang-design/blob/master/FSharp-4.0/AutoQuotationDesignAndSpec.md). All that is required is to add the `ReflectedDefinition` attribute and mark the type as an Expr<'a> instead of 'a.

```fsharp
// ORIGINALLY: [<CustomOperation("eq")>] member inline __.Eq(m, u, v)  = Option.bind (u == v) m     
[<CustomOperation("eq")>] member inline __.Eq(m, [<ReflectedDefinition>] u:Expr<_>, [<ReflectedDefinition>] v:Expr<_>) = Option.bind (u == v) m
```

Now this gives me:
```fsharp
    let x = var
    let y = var

    logic {
        eq (%x,%y) (1,2)
        eq %x 1
        solve y
    } 
```

One thing that pleasantly surprised me was the ability to use the splice operator `%` even though it's not surrounding by `<@ @>` quotes. I actually really like this now because having to write `%x` makes it clear that `x` is a variable.