---
title: "Returning multiple values from a function via static member constraints"
---

It's becoming cumbersome to constantly write `var`,`var`,`var` etc. for each temporary variable I am creating but I can use [static member constraints](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/generics/statically-resolved-type-parameters) to help me.

I am borrowing the techniques I found in [FSharpPlus](http://fsprojects.github.io/FSharpPlus/) to create a custom type class. 

```fsharp
type 'a expr = Expr<'a>

type Vars = Vars with
    static member Create (_:'a expr) = var  /* (required to prevent over-zelous matching) */
    static member Create ((_,_)) = var,var
    static member Create ((_,_,_)) = var,var,var
    static member Create ((_,_,_,_)) = var,var,var,var
    // etc.

    static member inline (~~) (_)  = 
        let inline call (_:^b) (a: ^a) = 
            ((^a or ^b) : (static member Create : ^a -> ^a) (a))
        call Unchecked.defaultof<Vars> Unchecked.defaultof<_>

let vars = Vars
```

Let's see an example.
```fsharp
let a,b,c = ~~vars
logic {
    eq (%a + %b) %c
    eq %a 1
    eq %b 2
    solve %c
} // ===> 1 + 2 
```
