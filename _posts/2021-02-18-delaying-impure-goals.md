---
layout: post
title: "Delaying impure goals"
---
One of the issues of impure goals is that order of evaluation matters because if the variables have not been unified yet then it's unclear if the goal passed or failed.

A simple way of dealing with this is to delay the goal until more information can come. I can achieve this be simply keeping all the unchecked goals around in the object been passed around, which I will now call a `Package`.

```fsharp
type Package = { Substitution: Substitution; Goals: Goal list }
and Goal = Package -> Package list

/// refactoring not shown

let pred (p:bool expr) a = 
    let p' = walk p a.Substitution
    try if (eval << cast) p') then [a] else []
    with | _ -> 
        // reaching here means that unknown variables still exist and 
        // we must delay the goal till later.
        [{ a with Goals = (pred p')::a.Goals }]
```

## What this means?
So basically I try to substitute as many variables as possible at the beginning and there are two possibilities:

- All variables are removed and I can evaluate the expression and continue accordingly.
- I must wait for the remaining variables to be unified, I append this goal again but with potentially less variables now.

But when does this goal get run? Since I am always waiting for new information it makes sense to rerun the goals after unification.

```fsharp
let inline (==) (u:'a) (v:'a) a =
    match unify u v a.Substitution with 
    | None -> []
    | Some subst' -> 
        let goal:Goal = all a.Goals
        goal { Substitution = subst'; Goals = [] } // run all the goals again with an updated substitution 
```
So basically re-evaluation potentially reduces the number of goals remaining or it will re-add it again. If there are still goals remaining at the end I can throw a `non-determined` exception. 