# Booleans

## Misc

* Don't forget to test overflows
* CPS and ANF are very similar

## Introducing booleans

* `true` and `false`
* What happens when the program is just `true`?
    * should be able to output `true`

Add bool to our `expr` type. Seems simple, but now all of our matches are broken because there's an extra case.

More importantly, how do we represent the values in our pipeline so that it still makes sense in assembly? We can't just use 1 and 0 because they already represent integers.

We seem to need to return both the value and the type of the answer. We are only putting answer in EAX right now. We could put value in one register and type in another but it doesn't fit in that model.

We can also statically type our language, but it still doesn't resolve the problem of how do we tell the caller what type our program returns.

## Solution

We can reserve values to help us. Let's reserve 2 billion values :). The lowest bit will tell whether it's a number or boolean. We can't use the highest bit because that's the sign bit.

We're throwing away half of our integers to get 2 booleans. But, we are going to have more types of data in the future, so we can use the rest of the values in the future.

This is what languages actually do, except C, but C loses out on a lot of features because it can't tell the difference between an int and a pointer.

What if we have more than a million types? :^) Here we need to distinguish primitive vs constructed types.

We need a function `translate` such that: `translate(n_1 + n_2) = translate(n_1) + translate(n_2)`

```txt
1 => 2
5 => 10
n => 2n (i.e., n lsl 1)

6 => 12
```

We can just translate numbers as twice itself, then addition/subtraction/comparison just carries over. For multiplication, we need to divide by 2 (shr doesn't  preserve sign, but asr does). For division, we need to multiply by 2. In order to make this work, the lowest bit must be `0`.

`0` means number,  `1` means boolean.

## Operating on booleans

How can we encode for logical `and` & `or`?

```txt
1 = true = 0x80000001
0 = false = 0x00000001

and => bitwise and
or => bitwise or
not => xor 0x80000000
```

## Interpreting the return value

Now, our answer is in EAX. How should we pass this info to the C program?

What is the signature of `our_code_starts_here`? Right now it returns an `int`. However, it should actually be interpreting the word that is returned by our program. We can't just `printf ...%d` anymore. We need to write a translator from our encoded form.

## Runtime errors

What do we do if the program attempts to add `true + 1`? We could write a static typechecker, which should be after the names checking in the pipeline.

Or, we can be like python and throw a runtime error. This would occur in assembly. Let's do that for now. How?