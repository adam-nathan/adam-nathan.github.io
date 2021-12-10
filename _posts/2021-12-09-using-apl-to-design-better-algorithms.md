---
layout: post
title: "Using APL to Design Better Algorithms"
usemathjax: true
---

# Using APL to Design Better Algorithms

A while ago I became interested in the APL language. I don't quite remember how I got into it but there is something magical about it. It's hotly debated if APL is readable with it's cryptic symbols, I can vouch that it does become easier with time and part of me feels like I'm Neo in the Matrix. 

### Conway's Game of Life
````apl
life←{                                  ⍝ John Conway's "Game of Life".
    ↑1 ⍵∨.∧3 4=+/,¯1 0 1∘.⊖¯1 0 1∘.⌽⊂⍵  ⍝ Expression for next generation.
}
````
> Some people will marvel at the mathemetical elegance, others will recoil in horror.

## SQUIGOL
One thing that struck me was its similarity to John Bird's squigol notation, otherwise known as Bird-Meertens formalism (BMF).

In short, the goal is to take an inefficient program that is easy to explain and transform it by way of proven transformations.

---
### Maximum subarray problem
In computer science, the maximum sum subarray problem is the task of finding a contiguous subarray with the largest sum, within a given one-dimensional array $$A[1...n]$$ of numbers. Formally, the task is to find indices $${\displaystyle i}$$ and $${\displaystyle j}$$ with $${\displaystyle 1\leq i\leq j\leq n}$$, such that the sum

$${\displaystyle \sum _{x=i}^{j}A[x]}$$

is as large as possible.

---
Written in plain English, we have: 
> maximum of the sum of all the segments of the array A

I first create readable helper functions in APL (APL goes from right-to-left).

```apl
prefixes ← ,\           ⍝ generates all possible prefixes of an array i.e. (1 2 3) → ((1) (1 2) (1 2 3))
suffixes ← ¯1∘+∘⍳∘≢↓¨⊂  ⍝ generates all possible suffixes of an array i.e. (1 2 3) → ((1 2 3) (2 3) (3))
flatten ← ⊃,/
max ← ⌈/
sum ← +/
map ← ¨                 ⍝ foreach element apply function f
segs ← (flatten suffixes map ∘ prefixes)    ⍝ all possible contiguous subarrays
```

The solution then is
```apl
mss ← (max sum map ∘ segs)
```

The `segs` function is $$O(n^2)$$ followed by sum and map, which are both $$O(n)$$. In total the brute-force solution is $$O(n^3)$$.

### Transformation

```apl
max sum map segs nums                               ⍝ original              O(n^3)
max sum map concat suffixes map prefixes nums       ⍝ inline definition    
max concat sum map map suffixes map prefixes nums   ⍝ map-promotion
max max map sum map map suffixes map prefixes nums  ⍝ fold-promotion
max (max (sum map) ∘ suffixes) map prefixes nums    ⍝ map once
max 0 (0⌈+) foldl map prefixes nums                 ⍝ Horners rule          O(n^2) 
max 0 (0⌈+) scanl nums                              ⍝ scan lemma            O(n)
```

As you can see we very easily discovered an algorithm that runs in O(n) time, by simply following a few pre-known rules.


## Loop-fusion Rule 
#### `(f¨g¨) ≡ (f∘g¨)`
> Combine operations and only loop once

## Map-promotion Rule 
#### `(f¨⊃,/) ≡ (⊃,/f¨¨)`
> Changing an array before flattening is the same as changing it after flattening

## Fold-promotion Rule
#### `(f/⊃,/) ≡ (f/f/¨)`
> The sum of sums is just the sum

## Horner's Rule 
#### `(f/x,y) g (f/y) ≡ (x g f/⍬) f y`
> Factor out suffixes by treating the prefix as an empty array

This reduces the complexity from $$O(n^2)$$ to $$O(n)$$ and it's a bit harder to explain, so an example is would be the summation of multiplying suffixes of an array.


$$
\begin{aligned}
xyz + yz + z 
  &= x\cdot y\cdot z 
  \\ &+ 1\cdot y\cdot z 
  \\ &+ 1\cdot 1\cdot z 
\end{aligned}
$$

$$
((((x+1) \cdot y) + 1) \cdot z) \cdots
$$

> `Note`
This requires implies a left fold, which unfortunately APL's operator `/` isn't really due to right-most associativity. Most of the time it's not a problem because operators that are distributive tend to be associative.

## Scan Lemma 
#### `f foldl map prefixes ≡ scanl`
Again this is a problem since APL scan `\` doesn't [behave][1] exactly like a left-scan, and it's not very [efficient either][2]. I've purposefully left out the implementation of both foldl and scanl for clarity.


## Putting it all together
```apl
⌈/ +/ ¨ ⊃,/ (¯1∘+∘⍳∘≢↓¨⊂) ¨ ,\ nums  ⍝ definition
⌈/ ⊃,/ +/¨¨ (¯1∘+∘⍳∘≢↓¨⊂) ¨ ,\ nums  ⍝ map-promotion
⌈/ ⌈/¨ +/¨¨ (¯1∘+∘⍳∘≢↓¨⊂) ¨ ,\ nums  ⍝ fold-promotion
⌈/ (⌈/+/¨∘(¯1∘+∘⍳∘≢↓¨⊂)) ¨,\ nums    ⍝ loop-fusion
⌈/ 0⌈+ / ¨,\ nums                    ⍝ Horner's rule
⌈/ 0⌈+ \ nums                        ⍝ scan-lemma
```
---

## In Conlusion
It's possible to refactor a brute-force algorithm to an optimized one that runs in $$O(n)$$ by recognizing patterns, or idioms, APL is a great tool to blow away a lot of the chaff and see the idioms clearly.
One downside that I found was that `foldl` & `scanl` do not behave as expected, it's not clear if that would be a problem when operators are, but it's hard to argue how beautiful the result is! 

# --- `⌈/0⌈+\` ---

<sub>*Adapted from the paper [Algebraic Identities for Program Calculation][3]*</sub>

[1]: https://stackoverflow.com/q/70272288
[2]: https://stackoverflow.com/q/70273199
[3]: http://comjnl.oxfordjournals.org/content/32/2/122.full.pdf