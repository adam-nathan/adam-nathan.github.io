---
title: "Life is the sum of our choices"
mathjax: true
---

Functions accept and emit values
$$f:: \star \rightarrow \star$$
Imagine composing functions like a pipe; taking values and transforming them at each step in time.
$$
x \xrightarrow{f_1} x_1 \xrightarrow{f_2} \dots \xrightarrow{f_{n}} x_n
$$
A `monad` is the same, just with extra metadata (`ϵ`) riding along and accumulating at each step
>Think of `ϵ` as 'environment' or  'effect'
>
$$
\begin{align*}
x \to \epsilon_1 x_1 \to \epsilon_2 x_2 \to \epsilon_3 x_3 \to \dots \to \epsilon_nx_n \\
\epsilon_1 x_1 \to \epsilon_2 x_2 \to \epsilon_3 x_3 \to \dots \to \epsilon_nx_n \\
\epsilon_1.\epsilon_2 x_2 \to \epsilon_3 x_3 \to \dots \to \epsilon_nx_n \\
\epsilon_1.\epsilon_2.\epsilon_3 x_3 \to \dots \to \epsilon_nx_n \\
\vdots \\ 
\epsilon_1.\epsilon_2.\epsilon_3 \dots \epsilon_nx_n \\
\equiv\\
\epsilon_{1\dots_n} x_n
\end{align*}
$$
$\epsilon_{..i}$ represents the historical past that led up to the value $x_i$

As the title suggests monads can be summed/multiplied, or to be more fancy are **monoids in the category of endofunctors**. We only need an ***initial*** $\epsilon_0$ and a method to ***combine*** $ϵ \otimes ϵ$.

## Example - Middleware
An HTTP message is just a `(header . body)`. The header starts with an empty dictionary that is read/written to via the middleware `body -> (header . body)`, chained together thus

`(header . body) -> (header . body) -> (header . body) -> (header . body)`

### Authentication
Authentication is metadata stored in the header. Authenticated requests pass through, however unauthenticated requests can be diverted to a new login-middleware and redirected back to the original path with an updated header.
