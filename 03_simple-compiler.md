# Simple compiler

## The desired result

We have a scaffold of a C program:

```c
#include <stdio.h>

int main(int argc, char** argv) {
    return 0;
}
```

How can we call our program from this scaffold? We will be using this scaffold to bootstrap our programs. This will be our standard library.

We need to declare that our function exists somewhere:

```c
// scaffold.c

#include <stdio.h>

// use label in assembly to locate code
extern in our_code_starts_here() asm("our_code_starts_here");

int main(int argc, char** argv) {
    int result = our_code_starts_here();
    printf("%d\n", result);
    return 0;
}
```

How to write something useful in assembly? Let's return 42 as the answer.

We need to store our result since we can't just "return" an answer. "The answer goes in EAX" register.

```txt
; file 42.asm
section .text
global our_code_starts_here

our_code_starts_here:
    mov EAX, 42
    ret
```

To compile:

```bash
# compile to intermediate object files
nasm -f elf32 -o our_code.o 42.asm
# linking the object file together
clang -g -n32 -o main scaffold.c our_code.o
```

## Let's create a compiler to do this for us

We want our compiler to take in a file with a number in it and output a program that prints that number.

```ocaml
open Printf

(* A very sophisticated compiler - insert the given integer into the mov instruction at the correct place *)
let compile (program : int) : string =
    sprintf "
section .text
global our_code_starts_here
our_code_starts_here:
    mov eax, %d
    ret\n" program;;

(* Some OCaml boilerplate for reading files and command-line arguments *)
let () =
    let input_file = (open_in (Sys.argv.(1))) in
    let input_program = int_of_string (input_line input_file) in
    let program = (compile input_program) in
    printf "%s\n" program;;
```

This program will print the assembly code to standard out. If we pipe it into a file, it will be runnable assembly.

## If we want to grow the language, which part of the pipeline will it affect?

First, we need a better input language. Let's define an AST for our input language.

```ocaml
type value =
    | VInt of int
    (* | VBool of bool *)

(* bind (the result of) the program to an identifier *)
type binding = string * prog

type op = Plus | Minus | Times

type prog =
    | Int of VInt
    (* | Bool of VBool *)
    | Binop of op * prog * prog
    | Ident of string

    (* the binding is in scope of the given prog *)
    | Let of binding * prog

letrec eval (p:prog) (env: env) =
    match prog with
        | Int i ->  i
        | Binop (op, left, right) ->
            let lval = eval left env in
            let rval = eval right env in
                (match op with
                    | Plus -> lval + rval
                    | ...)
        | Ident name -> lookup env name
        | Let (bindings, body) -> fold_left(fun env bind =
            (name, (eval prog env))::env, env, bindings)
```

But how do you store strings in assembly? Let's not think about that it's too hard. What we really want is an identifier. It doesn't have to be a string. Let's map names in program to addresses in memory. An address is just a thing that uniquely identifies words in memory. This is analogous to identifiers that uniquely identify values in our program.

But what about shadowing? Add one more layer of indirection! Keep a map of identifiers to addresses, then find the right one when we need them.

## Programmers are less smart than they think

Suppose that the programmer screwed up and you want to present the error message to the programmer. How do you tell them where they screwed up? We're not keeping track of this in our AST.

We can add source info in AST to remedy this.

```ocaml
type pos = int * int * int * int (* line number, char, etc *)
```

However, this sucks because the source location isn't inherent to being a binop. And we have to keep track of this on eval.

```ocaml
type prog =
    | Int of VInt * α
    | Binop of op * prog * prog * α
    | Ident of string * α
    | Let of binding * prog * α

(* α is a free slot that can be used for positions, unique identifiers,  whatever that is needed at the moment *)
```