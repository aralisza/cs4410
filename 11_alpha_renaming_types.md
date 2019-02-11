## Alpha Renaming (Alpha Conversion)

```ocaml
(let x= 1 in  x) + (let x = 2 in x)

(* anfs to *)
let x = 1 in
let x = 2 in
    x + x
```

Not equivalent!

The way to fix this is to do a name resolution pass (alpha renaming).

What step in the pipeline should we put this?
    * After well-formed-ness check, because we could turn a malformed program into a good program?
    * During well-formed-ness check? But now we're tightly coupling behaviors.

```ocaml
let  rec renameE (env : string envt) (e : tag expr) : tag expr =
    | Num -> obvious
    | Prim -> boring
    | If -> boring
    | Let(binds, body, tag) ->
        let help (binds, env) ((name, exp tag) : bind): (bind list * string envt) =
            let new_name = "name_%tag" in
            let new_exp = renameE exp env in
            let new_env = (name, new_name)::env in
            ((new_name, new_exp, tag)::binds, new_env)
        in
        let n_binds, n_env = List.fold_left help ([], env) binds in
        Left(List.rev(n_binds), renameE, n_env, body, tag)
    | Id(name, tag) -> Id(lookup env name, tag)
```

## Type checking

```txt
Axioms (nothing above line)

_________     ___________       _____________
Γ⊢n : Int     Γ⊢true : Bool     `Γ⊢false : Bool


Γ⊢ e1 : Bool   Γ⊢e2 : Bool
__________________________
  Γ⊢ e1 and e2 : Bool


Γ⊢ e1 : Int  Γ⊢ e2 : Int
__________________________
  Γ⊢ e1 + e2 : Int


Γ⊢ e1 : Int  Γ⊢ e2 : Int
______________________
  Γ⊢ e1 < e2 : Bool


 c : Bool  t : Tau  f : Tau
_____________________________
     if c t f : Tau


    e : Tau
________________
isBool e : Bool


    e : Tau
________________
isNum e : Bool


    e : Tau
________________
print e : Tau


e1 : Tau  e2 : Tau
________________
e1 == e2 : Bool

Γ⊢ e : Tau1    Γ, x.Tau1 b : Tau2
__________________________________
    Γ⊢ let x = e in b : Tau2


(x Tau) ∈ Γ
_____________________
Γ⊢ x : Tau


 c : Bool  t : Tau1  f : Tau2   *****"Custom Rule"****
_____________________________
isBool (if c t f) : Bool
```

**Preservation (Confluence)**: If a program is well typed, it won't get stuck. Every subexpression must have a type. It either has a value, or it can take a step towards a value. The "custom rule" may break preservation. This is because we never give `if c t f` a type.

```txt

____________ __________ ___________
⊢ true Bool  ⊢ 1 : Int  ⊢ 2 : Int
___________________________________ _________
⊢ if true then 1 else 2 : Int       ⊢ 3: Int
_____________________________________________
⊢ (if true then 1 else 2) + 3 : Int
```

**Syntax directed**: the shape of our tree directs the shape of our type checking algorithm.

Problem: We first guessed that the whole expression was of type `Int`. We can't just guess and check once we add tuples because there will be infinite types.

Solution1: make the user supply type annotations everywhere. Works great except for `==` and `isbool`.... polymorphic functions.

Fortunately, we can actually do some form of guess and check.

### Hindley-Milner

```txt
let f g h x =  h(g(h(x)))
---------------------------
let f = λg.λh.λx.h(g(h(x)))

f : alpha
alpha = beta -> gamma
g : beta
beta = omega -> chi

gamma = delta -> epsilon
h : delta
delta = mu -> theta

epsilon = mu -> nu
x : mu

chi = mu

f: (theta -> chi) -> (chi -> theta) -> chi -> theta
        g                h              x
```

We stil don't know what theta and chi are. So:

```txt
f: ∀theta, chi (theta -> chi) -> (chi -> theta) -> chi -> theta
```

We allow for alls only on the very outside.

But when we do a type judgement, we still need to know a type not a type schema. To fix this, we just provide it with any new name. Because it will work for all, it should  work with any one.

The good thing a bout Hindley-Milner is that it does at little as possible at each step.