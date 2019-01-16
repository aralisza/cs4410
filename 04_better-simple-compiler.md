# Better simple compiler

## Making it more extensible

Our compiler so far...

This code generates code that will vary based off the program. Runtime varies based off the machine it is run on. So, we don't need to worry about generating the runtime here.

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

We were writing our asm as a string... stringly typed. Let's create an AST for asm and then convert the AST to string so we don't do this.

```ocaml
open Printf

(* our program's type *)
type prog =
    | ENum of int

type reg =
    | EAX

type arg =
    | Const of int
    | Reg of reg

type instruction =
    | IRet
    | IMov of arg * arg

let compile (program : prog) : instruction list =
    match program with
        | ENum value ->
            [(IMov(Reg(EAX), Const(program)))]
;;

let instructions_to_string (instrs : instruction list) : string = "TODO"

let string_to_prog code = ENum(int_of_string code)

let compile_prog (e : expr) : string =
    let instrs = compile e in
    let asm = instructions_to_string instrs in
    let prelude = "
section .text
global our_code_starts_here
our_code_starts_here:" in
    let suffix in "ret" in
    prelude ^ "\n" ^ asm ^ "\n" ^ suffix
;;

let () =
    let input_file = (open_in (Sys.argv.(1))) in
    let input_program = int_of_string (input_line input_file) in
    let program = (compile_prog input_program) in
    printf "%s\n" program;;
```

So far, we've jut moved things around into data types and immediately deconstruct them, but this sets us up for growth. We can now easily extend the program to do other things.

## Increment and decrement

Let's add the increment and decrement functionality to our program

What do we need to add?

```ocaml
open Printf

(* our program's type *)
type prog =
    | ENum of int
    (* added *)
    | EInc of prog
    | EDec of prog

type reg =
    | EAX

type arg =
    | Const of int
    | Reg of reg

type instruction =
    | IRet
    (* added *)
    | IMov of arg * arg (* move dest, src *)
    | IAdd of arg * arg (* add dest, src *)

let compile (program : prog) : instruction list =
    match program with
        | ENum value ->
            [(IMov(Reg(EAX), Const(program)))]
        | (* todo *)
;;

let instructions_to_string (instrs : instruction list) : string = "TODO"  (* this would need to be modified too *)

let string_to_prog code = ENum(int_of_string code)

let compile_prog (e : expr) : string =
    let instrs = compile e in
    let asm = instructions_to_string instrs in
    let prelude = "
section .text
global our_code_starts_here
our_code_starts_here:" in
    let suffix in "ret" in
    prelude ^ "\n" ^ asm ^ "\n" ^ suffix
;;

let () =
    let input_file = (open_in (Sys.argv.(1))) in
    let input_program = int_of_string (input_line input_file) in
    let program = (compile_prog input_program) in
    printf "%s\n" program;;
```

Rest of todo will be on hw.

## Variables

How to store variables? Given expression, how to allocate stack slots to names?

### Memory structure

```txt
+----------+ - low registers
    Code
+----------+
    Global
+----------+
    Heap
             - grows v


+----------+


             - grows ^
    Stack
+----------+ - higher registers
```

How do we know where we are in the call stack? `ESP` register. It refers to where the current stack begins. If you modify the stack pointer, you have to put it back where you found it after you return or else. The callee is responsible for saving and restoring.

### De bruijn Indices

```ocaml
let  x (* 1 *) = 5 in
    add1(x (* 1 *))
;;

let x (* 1 *) =  5 in
    let y (* 2 *) = 10
        in add1(x (* 1 *))

let x (* 1 *) =
    let y (* 1 *) = 3
        in y (* 1 *)
    in x (* 1 *)
```

```ocaml
type 'a expr =
    | EConst of int * 'a (* store the index in 'a*)
    | EIdent of string * 'a
    | Elet of string * 'a expr * 'a

```