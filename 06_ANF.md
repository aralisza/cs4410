# A Normal Form

## Hw overview

* Keywords should fail if found as a loose identifier
* Bindings should have at least one element
* Pattern matching lines up with our expression shapes well, and allows us to recurse over it easily

## Conditionals, continued

We will tag nodes using function with signature

```ocaml
let rec tag (e : 'a expr, t : tag) -> (tag expr * tag) ...
```

Our job will be to thread the tag through the tree

## Binary ops

Let's add the ability to evaluate arith exps:

```txt
(2 + 3) + 4
```

We wll need to add binops to our AST

```ocaml
type 'a expr  =
    | Num of int
    | Id of string
    | Let of binding list * expr
    | Prim1 of prim1 * expr (* renamed to prim1 *)
    | Prim2 of prim2 * expr * expr (* added *)
```

What would the assembly look like?

```asm
; (2 + 3) + 4
mov EAX, 2
add EAX, 3
add EAX, 4

; ((4 - 2) + 7) * 5
mov EAX, 4
add EAX, -2
add EAX, 7
mul EAX, 5

; (2 + 3) * (5 - 4)
mov EAX, 2
add EAX, 3
; ...?
```
Where do we store the result to calculate the other half?Storing it on the stack or register doesn't make much sense because we want the translation to be always consistent. The intrinsic difference between  assembly and AST is that assembly doesn't support tree shape, only list shape. In the other two expressions, each expression was in tail position--it was the last thing to do in evaluation order (left-right)

If only we have some way to translate this expression to use the stack? We have `let` expressions!

```ocaml
(* we use $ because it's just some symbol that isn't allowed as a character in our identifiers in our language. This guarantees no collisions *)
let $temp1 = 2 + 3 in
let $temp2 = 5 - 4 in
    $temp1 * $temp2

(* or really, *)
let $temp11 = 2 in
let $temp12 = 3 in
let $temp1 = $temp11 + $temp12 in
...
(* because now the definition of evaluating infix operations is now store the left and right in a let *)
```

This is good beacuse we don't have to worry about stack and stuff again and can just reuse what we implemented in let. We are relying on the assumption that our expressions are evaluated eagerly.

## ANF

How can we tell if our program is immediately ready? The right side needs to be immediately ready and the left side needs to be ready enough. It's called *administrative normal form* or *A normal form*. A program is in ANF if it fits this shape.

A (restrictive) version of ANF is in A normal form if on the right side of any binding there is only one operation.

```ocaml
let is_immediate e =
    match e with
        | (Num | Id) -> true (* immediates *)
        | _ -> false
;;

let rec isANF e =
    match e with
        | Let (binds, body) ->
            List.for_all(fun (_, bind) -> isiANF bind)) binds && isANF body
        | Prim1 (_, e) -> is_immediate e
        | Prim2 (_, e1, e2) -> is_immediate e1 && is_immediate e2
        | _ -> is_immediate e
```

This version is easy to implement but doesn't recognize `((4 - 2) + 7) * 5`.

How can we produce an ANF expression?

## Producing ANF

```ocaml
(* this signature isn't powerful enough. *)
let rec anf (e : tag expr) -> unit expr =
    ...
```

We can write a helper that that returns an immediate and a context to evaluate that immediate (e.g., if it was an identifier)

```ocaml
let rec anf (e : tag expr) -> unit expr =
    let rec  helper (e  : tag expr) : (unit expr * binding list) =
        match e with
            | Num (n, _) -> (Num (n, ()), [])
            (* translating this didn't create any new bindings*)
            | Id (x, _) -> (Id (x, ()), [])
            | Prim2 (op, e1, e2, tag) ->
                let e1_ans, e1_ctx = helper e1
                and e2_ans, e2_ctx = helper e2
                (* has to return identifier because only int and id are immediates, so let's make up a new identifier *)
                and temp = sprintf "$temp_%d" tag in
                    (Id (temp, ()),
                        e1_ctx
                        @ e2_ctx
                        @ [(temp, Prim2 (op, e1_answer, e2_answer))])
            | .......
```

This list of bindings we are producing is not the same as the environment in our previous step. After calling this function, we can just create a let with the bindings returned. Also we can't shadow with unique identifiers.

However, we still need `e1_ct`x before `e2_ctx` because our language is leftmost, innermost. We are transforming the program in a way that encodes the execution order that we want. Right now, that doesn't matter much because we're only doing arithmetic, but this order is very important once we have side effects.

Now, what about `if`s?

## Normalizing if statements

Obviously we want to normalize all the parts. Obviously.

We want each of our bindings to do one step (primitive operation). However, `if`s are conditioning based on the argument. Therefore, for it to make any sense, the condition must be an *immediate*. Only the condition is eagerly evaluated. Only the arms are lazily evaluated.

What about the bodies? It's straightforward way to say that we just want to normalize the bodies... but what about nested ifs? If we naively translate `if`s, there will be a lot of code duplication.