# Arbitrary functions

## Environment

We need to keep track of arity in our environment.

With this, we also need to decide shadowing rules. We can:

1. (Dis)allow function/value shadowing
1. (Dis)allow builtin function/value shadowing
1. (Dis)allow function/value within its own function body usage
1. (Dis)allow function/value shadowing in function argument

etc.

There are so many. Every single place you add a name the environment is a potential opportunity for shadowing. Design your test cases concurrently with implementing adding names to the environment. The structure of your code inform structure of your tests.

## ANF

We can use the type system to ensure our `anf` function returns a type that has to be anfed.

```ocaml
type immexpr =
    | ImmNum of int
    | ImmBool of bool
    | ImmId of string
```

Now we don't need `is_immediate` anymore.

How do I define `aexpr`?

```ocaml
type cexpr = (* compound expr, can be on the rhs of a let *)
    | CPrim1 of prim1 * immexpr
    | CPrim2 of prim2 * immexpr * immexpr
    | CIf of immexpr * aexpr a cexpr
    | CImmediate of immexpr
    (* note: no ELet *)

let aexpr =
    | ALet1 of string cexpr * aexpr
    | ACexpr of cexpr

let anf e =
    let rec helpI (e : expr) : (immexpr * (string * cexpr) list) =
        match e with
            | EPrim1(op, e) ->
                let ans, ctx = helpI e in
                let tmp = "..." in
                (temp, ctx @ [(temp, CPrim1(op, ans))])
            | ...
    and helpAC (e : expr) : (cexp * (string * cexp) list) =
        match e with
            | EPrim1(op, e) ->
                let (ans, ctx) = helpI e in
                (CPrim1 (op, ans), ctx)
            | ...
```

CPS, Static Single Assignment Form (SSA), and ANF are all analogous.

If the shadowing is correct, ANF shouldn't introduce any additional shadowing.

What about shadowing across files? This requires a module system. This is a mess.

```ocaml
type aprogram =
    | AProgram of defn list * expr
```

## Factorial

```python
def fact(n):
    if n < 1: 1
    else: n  * fact(n-1)

fact(2)

"""
______________<-ESP (2)
______________
______________

______________
true
______________
EBP
______________
saved return
______________<-ESP (1)
2             (1 * 2)
______________
1             (saved result from recurring)
______________
1             (2 - 1)
______________
false         (because 2 is not less than 1)
______________<-EBP
EBP
______________
saved return
______________
2
______________<-our_code_starts_here
"""
```

Everything between ESP (1) and ESP (2) is repeated over and over again for every recursive call.

```python
def facttail(n, acc):
    if n<=1: acc
    else: facttail(n * acc, n-1)

facttail(2, 1)

"""
______________<-ESP (1)
???
______________
1
______________
2
______________
false         (because 2 is not less than 1)
______________<-EBP
EBP
______________
saved return
______________
1
______________
2
______________<-our_code_starts_here
"""
```

Now we know what the stack frame for facttail looks like. We can just replace the arguments for the original facttail call with the new arguments. Instead of implementing tail recursive calls as a `call`, implement as a `jmp`. To account for the first two instructions of a function being push `EBP` and decrement `ESP`, we can either reset ESP or have two entry points into the function, one with the setup, one without.

There is a really nasty edge case:

```python
def min(x, y):
    if x < y: x
    else: min(y, x)

min(12, 10)

"""
______________
false
______________
EBP
______________
ret
______________
12-> ...12
______________
10-> 12
______________
"""
```

When we swap arguments, let's set 10 to 12, then 12 to... 12? Womp womp

To fix this, push to stack temporarily to preserve order.

Tail calls turns your recursion into `while` loops.

`Printstack` is just printing those stack diagrams. The challenge is stopping. A hack is to declare a pointer in C and then the first thing the assembly does is save EBP to that pointer.

**Deforestation** transforms syntax trees to syntax trees that convert into better assembly. (because it's eliminating data trees)

## Why don't a lot of languages have tail calls?

Because the stack trace is exposed as a value that can be used to make decisions in your program. For example, Java security monitor. You can also just throw an exception and catch the error and regex on that string.