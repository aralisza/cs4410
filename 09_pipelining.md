# Pipelining

## Tracing

It would be useful to see how the compiler processed each phase in the pipeline for debugging reasons.

```ocaml
type phase =
    | Source of string
    | Parsed of sourcespan expr
    | Anfed of sourcespan expr
    | Tagged of tag expr
    | Result of inst list
;;

type failure = exn list * phase list (* errors, succeeded phases*)

(* 'a says what's in last phase *)
type 'a pipeline = (failure * 'a phase list) either

(* equivalent to f(x), useful to create pipelines *)
(* x |> f *)

(* log = logger that takes expr and creates Parsed of expr... etc. This is because you can't pass constructors as arguments for type reasons *)
(* next actually creates the next phase*)
(* trace is passed in from the |> operator *)
let add_phase (log: 'b -> phase) (next: 'a -> 'b pipeline) (trace: 'a pipeline) -> 'b pipeline
    ...
;;

(* goal: *)
(Right src)
    |> (add_phase tag_parse parse)
    |> (add_phase tag_tagged tag)
    |> (add_phase tag_anfed anf)
```

We are returning errors instead of throwing them so we can report all of one kind of error at a time instead of one at a time. The only time you should use `failwith` for internal compiler errors, but it might be better to just return them anyway to get the compiler trace.

## Function Calls

```ocaml
type 'a expr =
    | ...
    | ECall of string * 'a expr list
    | ...
```

But first, we have to answer: what's the order of evaluation? Eager or lazy? Let's stick with eager.

What else do we need to change?

* ANF
* Codegen
* "resolve the string"
    * Do this in our well-formedness check.
    * In this check, we should also check function arity.