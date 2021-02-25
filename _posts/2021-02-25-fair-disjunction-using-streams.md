---
layout: post
title: "Fair disjunction using streams"
---

## Fair disjunction
An important feature of logic programming is the ability to handle divergent search paths. To understand this better, take the following example.

```fsharp
let rec nonstop () = 
  recurse nonstop
```

This creates an infinite loop. Of course, this is a silly example but take the following:

```fsharp
let three = 
  dnf [
    [(* divergent sequence *)]
    [number 3]
  ]

take 1 three // [3]
take 2 three // infinite-loop
```

In this example as I'm searching the problem space I should be able to find the number 3, however if I am greedily taking a depth-first-search approach, I will never reach the solution.
*NOTE: taking two causes an infinite loop as it never finds a second answer*

## Streams
A solution is to balance the search strategy by taking a limited amount from each branch - it's a kind of cross between depth-first and breadth-first searching.

There are many approaches to implement this, such as found in [Backtracking, Interleaving, and Terminating Monad Transformers](http://okmij.org/ftp/papers/LogicT.pdf), but one implementation I like is the based on [miniKanren](http://minikanren.org/).

The idea is to simply use a [thunk](https://en.wikipedia.org/wiki/Thunk) to postpone evaluation of the tail.

```fsharp
// notice the similarity to the List type
type List<'T> = Empty | Cons of 'T * List<'T>
type Stream<'T> = Empty | Cons of 'T * (unit -> Stream<'T>)
```
To thunk the whole stream I add an additional `incomplete` term:
```fsharp
type Stream<'T> = 
  | Empty 
  | Cons of 'T * (unit -> Stream<'T>)
  | Inc of Stream<'T>
and 'a stream = Stream<'a>
```

Rather than appending two streams, they are instead interleaved together.

```fsharp
let rec interleave m n = 
  match m with 
  | Empty -> n
  | Cons (a,z) -> Cons (a, fun () -> interleave n (z ()))  // NOTE: the order is swapped
  | Inc z -> Inc (fun () -> interleave n (z ())) // postpone interleaving
```
Similarly the `bind` operator is modified

```fsharp
  let rec bind (f:'a -> 'b stream) (m:'a stream) = 
      match m with 
      | Empty -> Empty
      | Cons (a,z) -> interleave (f a) (bind f (z ()))
      | Inc z -> bind f (z ())
```
Lastly, I need a way to lazily take `n` number of items.

```fsharp
let rec take n m = 
  if n <= 0 then [] else
  match m with 
  | Empty -> []
  | Cons (a,z) -> a :: take (n-1) (z ())
  | Inc z -> take n (z ())
```
For completeness I create a computation expression:

```fsharp
let ofSeq (xs:'a seq) : 'a stream = 
  Seq.foldBack (fun n acc -> Cons(n, fun () -> acc)) xs Empty

let inline (>>=) m f = bind f m
let inline (<|>) m n = interleave m n 

type StreamBuilder() = 
  member __.Zero() : _ stream = Empty
  member __.Return x : _ stream = Cons(x,Empty)
  member __.ReturnFrom m : _ stream = m
  member __.Bind (m:_ stream, f) : _ stream = m >>= f
  member __.Bind (xs:_ seq, f) : _ stream = (ofSeq xs) >>= f // helper to be compatible with lists etc.
  member __.Combine (m:_ stream, n:_ stream) = m <|> n
  member __.Delay f = Inc f

let stream = StreamBuilder()
```

## Examples
We can demonstrate the distribution of streams:

```fsharp
let triples = stream {
    let! a = [1..10]
    let! b = [1..10]
    let! c = [1..10]
    return (a,b,c)

(*
> take 1000 triples;;
val it : (int * int * int) list =
  [(1, 1, 1); (2, 1, 1); (1, 2, 1); (3, 1, 1); (1, 1, 2); (2, 2, 1); (1, 3, 1);
   (4, 1, 1); (1, 1, 3); (2, 1, 2); (1, 2, 2); (3, 2, 1); (1, 1, 4); (2, 3, 1);
   (1, 4, 1); (5, 1, 1); (1, 1, 5); (2, 1, 3); (1, 2, 3); (3, 1, 2); (1, 1, 6);
   (2, 2, 2); (1, 3, 2); (4, 2, 1); (1, 1, 7); (2, 1, 4); (1, 2, 4); (3, 3, 1);
   (1, 1, 8); (2, 4, 1); (1, 5, 1); (6, 1, 1); (1, 1, 9); (2, 1, 5); (1, 2, 5);
   (3, 1, 3); (1, 1, 10); (2, 2, 3); (1, 3, 3); (4, 1, 2); (1, 2, 6);
   (2, 1, 6); (1, 4, 2); (3, 2, 2); (1, 2, 7); (2, 3, 2); (1, 3, 4); (5, 2, 1);
   (1, 2, 8); (2, 1, 7); (1, 6, 1); (3, 1, 4); (1, 2, 9); (2, 2, 4); (1, 3, 5);
   (4, 3, 1); (1, 2, 10); (2, 1, 8); (1, 4, 3); (3, 4, 1); (1, 3, 6);
   (2, 5, 1); (1, 5, 2); (7, 1, 1); (1, 3, 7); (2, 1, 9); (1, 4, 4); (3, 1, 5);
   (1, 3, 8); (2, 2, 5); (1, 7, 1); (4, 1, 3); (1, 3, 9); (2, 1, 10);
   (1, 4, 5); (3, 2, 3); (1, 3, 10); (2, 3, 3); (1, 5, 3); (5, 1, 2);
   (1, 4, 6); (2, 2, 6); (1, 6, 2); (3, 1, 6); (1, 4, 7); (2, 4, 2); (1, 5, 4);
   (4, 2, 2); (1, 4, 8); (2, 2, 7); (1, 8, 1); (3, 3, 2); (1, 4, 9); (2, 3, 4);
   (1, 5, 5); (6, 2, 1); (1, 4, 10); (2, 2, 8); (1, 6, 3); (3, 1, 7); ...]
*)
} 
```
It now handles infinite streams gracefully
```fsharp
let rec inf n = stream {
    return n 
    return! (inf n)
}

take 10 (stream {
    return! inf 1
    return! inf 2
    return! inf 3
})

val it : int list = [1; 2; 1; 3; 1; 2; 1; 3; 1; 2]
```

```fsharp
let rec nonstop () = stream {
    return! nonstop ()
}

let s123 = stream {
    return 1
    return! nonstop ()
    return 2
    return! nonstop ()
    return 3
}

> Stream.take 3 s123;;
val it : int list = [1; 2; 3]

> Stream.take 4 s123;; // error!
```