# OCaml Basics

## Declaring a scoped variable block

```ocaml
let x: int = 41 in
    x  + 1
;;
```

## Declaring an unscoped variable

```ocaml
let x = 42;;
```

## Tuples

```ocaml
let xyz = (4, 5, 6) in
    (* pattern matching *)
    let (x, y, z) = xyz in
        x+y+x
;;
```

## Declaring functions

```ocaml
(* implicitly curried *)
let sum1 (x:int) (y:int) (z:int) = x+y+z

(* tuple as arg *)
let sum2 (x:int, y:int, z:int) = x+y+z

(* recursive *)
letrec is_even n =
    if n==0 then true
    else not(is_even(n-1))
;;

(* generic types *)
let do_twice f x = f(f x)
    do_twice: ('a->'a)->'a->'a
```

## Data definitions

```ocaml
(* number list *)
type numlist =
    | End
    | Link of int * numlist
;;

(*
    generic type with 'a
    notice typedefs are like functions except the arguments come first
*)
type 'a list =
    | End
    | Link of 'a * 'a list
;;
```

## Pattern matching custom types

```ocaml
(* using custom types *)
match mylist with
    | End -> ...
    | Link (f, r) -> ...
;;

match reallist with
    | [] -> ...
    | a::b::rest -> ...
    (* error! missing case for list of one element *)
;;
```

## Lists

DO NOT DO THIS:

```ocaml
(* this creates a list of one triple *)
[1, 2, 3]
```

DO THIS:

```ocaml
[1; 2; 3]
```