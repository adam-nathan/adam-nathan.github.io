---
title: "Running F# Backwards"
---
Finally I am able to start solving complex logic problems. 

## Hello World!
Dubbed the `hello world` of logic programming, `append` is a great way to demonstrate how relations work. I'm going to write a relation that can solve for all unknown arguments. 

Say we have: `append X Y Z`

I should be able to solve all of the below: 

`append [1;2;3] [4;5;6] Z` -- what is `Z`?

`append [1;2;3] Y [1;2;3;4;5;6]` -- what is `Y`?

`append X Y [1;2;3;4;5;6]` -- what is `X` & `Y`?

and even amazingly ...

`append X Y Z` -- what is `X`,`Y` & `Z`?

Before I start, let's make some helper functions.
```fsharp
let rec fold (f:'T -> Expr -> 'T) (state:'T) (q:Expr) = 
    match q with 
    | ES.ShapeVar _ -> f state q
    | ES.ShapeLambda (head,body) -> fold f state body
    | ES.ShapeCombination (o, args) -> List.fold (fold f) state args

let rec foldBack (f:Expr -> 'T -> 'T) (q:Expr) (state:'T) = 
    match q with 
    | ES.ShapeVar _ -> f q state
    | ES.ShapeLambda (head,body) -> foldBack f body state
    | ES.ShapeCombination (o, args) -> List.foldBack (foldBack f) args state

let rename (var:Var) name = Expr.Var (Var(string name, var.Type))

// converts ugly (434ee517-0270-4806-b0a5-b21f3dc59201, e034e355-161b-4335-8037-ffbbd5fe5a03, 434ee517-0270-4806-b0a5-b21f3dc59201) into 
// a prettier form (?0,?1,?0) by traversing the expression tree and replacing unground variables with a number starting from 0.
let prettify q =
    let subst = 
        fold (fun (s:Substitution) q -> 
            match q with 
            | P.Var var when (not << s.ContainsKey) var -> Map.add var (rename var (sprintf "?%d" (Map.count s))) s  // create a substitution that transforms [434ee517-0270-4806-b0a5-b21f3dc59201 --> ?0]
            | _ -> s) (Map.empty) q 
    walk q subst

let map (f:'a -> 'b) (m:'a stream) : 'b stream = bind (singleton << f) m 

let inline run n (qu:_ -> Goal) =
    let x = var // inject a fresh variable
    map ((walk x) >> prettify) (qu x Map.empty) |> take n
```

# Converting a regular function to a relation
Our normal append function looks something like this.

```fsharp
let rec append xs ys = 
    match xs with 
    | [] -> ys
    | a::d -> a :: (append d ys)
```

## Step 1. Make the last argument as the return value, place an imaginary out and change the signature to a boolean
```fsharp
let rec append xs ys (out zs) : bool = 
    (xs = []) && (ys = zs) ||
    (xs == a::d) && (append d ys (out tmp)) && (a::tmp = zs) // notice the need for a tmp value and the removal of non tail-recursive calls.
```
Making the last argument the return value is reminiscent of using an accumulator for non tail-recursive calls.

## Step 2. Convert bool to Goal by replacing all the regular functions with the relational counterparts; i.e. = becomes ==
```fsharp
let rec append xs ys zs : Goal = 
    let a,d,tmp = ~~vars
    ((xs == <@[]@>) &&& (ys == zs)) |||
    ((cons a d xs) &&& (append d ys tmp) &&& (cons a tmp zs)) 
```

## Step 3. Use recurse and unsure the recursive call is the last goal in the sequence.
This needs further explanation but for now trust me that this is true.
```fsharp
let rec append xs ys zs : Goal = 
    recurse (fun () -> 
        let a,d,tmp = ~~vars
        ((xs == <@[]@>) &&& (ys == zs)) |||
        ((cons a d xs) &&& (cons a tmp zs)) &&& (append d ys tmp))

// alternatively
let rec append xs ys zs : Goal = 
    recurse (fun () -> 
        let a,d,tmp = ~~vars
        dnf [
            [nil xs; ys == zs]
            [cons a d xs; cons a tmp zs; append d ys tmp]
        ])
```

I'll include `append` in the DSL 
```fsharp
type LogicBuilder with 
    [<CustomOperation("append")>]
    member this.Append(m:Goal, [<ReflectedDefinition>] xs:Expr<'a list>, [<ReflectedDefinition>]ys:Expr<'a list>, [<ReflectedDefinition>]zs:Expr<'a list>) : Goal = append xs ys zs 
```

## Demonstration
```fsharp
// The equivalent of a regular append - nothing special
run 6 (fun q -> 
    logic {
        append [1;2;3] [4;5;6] %q
    })

// Running backwards
run 6 (fun q -> 
    logic {
        append %q [4;5;6] [1;2;3;4;5;6]
    })
// [1;2;3]

run 6 (fun q -> 
    logic {
        append [1;2;3] %q [1;2;3;4;5;6]
    })
// [4;5;6]
```

## Turning up the power
```fsharp
run 6 (fun q -> 
    logic {
        let xs,ys = ~~vars
        append %xs %ys [1;2;3;4;5;6]
        eq %q (%xs, %ys)
    })
(* 
    ([], [1; 2; 3; 4; 5; 6])
    ([1], [2; 3; 4; 5; 6])
    ([1; 2], [3; 4; 5; 6])
    ([1; 2; 3], [4; 5; 6])
    ([1; 2; 3; 4], [5; 6])
    ([1; 2; 3; 4; 5], [6])
*)
```

## Max power
```fsharp
run 6 (fun q ->
    let xs,ys,zs = ~~vars
    logic {
        append %xs %ys %zs
        eq %q (%xs, %ys, %zs)
    })

(*
    ([],                    ?0, ?0)
    ([?0],                  ?1, ?0::?1)
    ([?0; ?1],              ?2, ?0::?1::?2)
    ([?0; ?1; ?2],          ?3, ?0::?1::?2::?3)
    ([?0; ?1; ?2; ?3],      ?4, ?0::?1::?2::?3::?4)
    ([?0; ?1; ?2; ?3; ?4],  ?5, ?0::?1::?2::?3::?4::?5)
*)
```

The last example shows all the possibilities of the append relation. Since this will run forever I am taking only a few. But this really demonstrates the power of this model and how it can solve logic problems.