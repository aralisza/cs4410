## Misc

* Control flow operators will mess with anf because anf is about control flow
* If we hoist `if`s and do a *commuting conversion*, it is exponentially bad.

## `<`

new assembly instruction:

```asm
cmp <addr> <const>
```

implementing `a < b`

```asm
    mov eax, [esp - 4] ; load a
    cmp eax, [esp - 8] ; compare with b
    jl if_true
if_false:
    mov eax, 0x00000001
    jmp if_done
if_true :
    mov eax, 0x80000001
if_done:
    ...
```

more efficient version of `a < b` with fewer instructions

```asm
    mov eax, [esp - 4] ; load a
    cmp eax, [esp - 8] ; compare with b
    mov eax, 0x80000001 ; set true
    jl if_done
    mov eax, 0x00000001 ; set false if not true
if_done:
    ...
```

even more efficient version?

```asm
    mov eax, [esp - 4] ; load a
    sub eax, [esp - 8]
    and eax,  0x80000000 ; masking all except sign bit
    or eax, 0x00000001 ; turning number into boolean by adding 1 bit
```

But what about overflows? `min_int - max_int`, i.e., `0x80000000 - 0x7ffffffe` = `0xfffffffe`... which would be wrong. We can be clever about reading  the overflow flag, but that's slower than just using `jl`... because `jl` does the same thing but better.

## `add1`

How can we check if a value is an integer before we try to add1?

```asm
mov eax, _
and eax, 0x00000001 ; won't work because it modifies eax
jnz error
add eax, 1
```

```asm
mov eax, _
test eax, 0x00000001 ; easy fix :)
jnz error
add eax, 1

error:
    ... ; abort the program!
```

### aborting the program

How to abort program? Call a function in our runtime.

```c
// main.c

const int E_NOT_INT = 1;

int error(int code) {
    if (code == E_NOT_INT) {
        printf("Not int");
        exit(1)
    }
    return 0;
}
```

Now, how do we call that function from assembly?

```txt
lo

local2
---------
local1
---------esp
arg1
---------
arg2

hi
```

The caller will push args on backwards, but it makes it easier on the callee.

This is the x86 calling convention only. (cdecl)

But, we are being rude about the stack pointer. We should move it to show how much we are using.

### When calling a function

Caller:

1. push args in reverse order
2. push return address

Callee:

1. push `ebp`
2. save `esp` in `ebp` (base pointer)
3. decrement `esp` by `n`, where `n` is max size of stack needed

```
-----------new esp
...n...
-----------
local1
-----------
old ebp
-----------old esp --> new ebp
return addr
-----------
arg1
-----------
...
```

### When returning from a function call

Callee:

1. restore `esp` to `ebp` (note: maybe backwards?)
2. pop saved `ebp` from stack
3. (answer should be in `eax`)
4. return

Caller:

1. pop off args

### Ending the program

```asm
mov eax, _
test eax, 0x00000001 ; easy fix :)
jnz error
add eax, 1

error:
    push 1
    call error_handler
    add esp, 4
```

### Enhancing error messages

```c
// main.c

const int E_NOT_INT = 1;

int error(int code, VALUE v) {
    if (code == E_NOT_INT) {
        printf("Not int %d", convert(v));
        exit(1)
    }
    return 0;
}
```

```asm
mov eax, _
test eax, 0x00000001 ; easy fix :)
jnz error
add eax, 1

error:
    push eax
    push 1
    call error_handler
    add esp, (4 * 2)
```