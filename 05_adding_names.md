# Adding more features

## Generating assembly
```ocaml
type 'a expr =
    | Const of int  * 'a (* Mov(Reg(EAX), num) *)
    | ID of string * 'a
    | Add1 of 'a expr *a
    | Let  of string * 'a expr * 'a expr * 'a

match 'a expr with
    | Const n  * a -> Mov(Reg(EAX), num)
    | ID s -> ...
    | Add1 expr * a -> (translate expr) @ [Add(Reg(EAX), Const(1))]
    | Let s * expr * expr * a ->
        lets slot = lookup name in
            (* save it in EAX *)
            (translate bind)
            @
            (* save EAX on stack *)
            [(mov (RegOffset(ESP, slot), Reg(EAX)))]
            @
            (translate body (with name somehow?))
```

```nasm
[ESP - 4] ; get value of ESP, minus 4 (1 stack slot), dereference to get the memory slot
```

## Example assembly generation

```ocaml
let x = (let x = 10 in add1 x) in
let y = add1(x) in y
```

```asm
; (let x = 10 in add1 x)
mov EAX, 10
mov [ESP - 4], EAX
mov EAX, [ESP - 4]
add EAX, 1
mov [ESP - 4], EAX
mov EAX, [ESP - 4]

; (let x = (...) in)
mov [ESP - 4], EAX

; let y = add1(x) in y
mov EAX, [ESP - 4]
add EAX, 1
mov [ESP - 8], EAX
mov EAX, [ESP - 8]
```

The code is awful, but it is simple and easy to see which syntax came from where. It is compositional.

Clearly, the `(let x = 10 in add1 x)` isn't being used, and we can make it more efficient, but let's not worry about that now.

This `let` only supports one binding. If we want multiple, we can add another phase called desugaring so we don't have to implement it twice. Macros are an extensions of desugaring.

## New problem: unbound identifiers

```ocaml
let x = (let x = 10 in add1 x) in
let y = add1(z) in y (* z is unbound *)
```

When we do lookup, it will error with a key not found error. We could just try/catch it... but that requires the compiler writer to be very careful. A better approach is to add a layer before the actual compiler so you guarantee that the code *should* compile when we tried to compile. We then can wrap the entire compiler in a try catch and any errors will be Internal Compiler Errors (ICE) that should never happen.

## Conditionals

```ocaml
if 5 then 6 else 7
```

```nasm
mov EAX, 5
cmp EAX, 0  ; sets some flag
je if_false ; checks some flag

if_true:
    mov EAX, 6
    jmp if_done

if_false:
    mov EAX, 7

if_done:
```

But what if we have 2 ifs? We will have duplicate labels.

We can just add another pass that will number every node. Then we can make our labels `${node_number}_true`. Alternatively, we can use a global counter (gensym)... but that's hard to test.