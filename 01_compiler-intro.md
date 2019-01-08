# Compiler Basics

## What is a compiler

Turn source code in one language to target language

```txt
Source program ---compile--> Target program
     |                             |
interpret                       interpret
     v                             v
source answer ---compile--> target answer
```

## Under the hood

1. Source (textual input)
2. ---parser--> syntax tree
3. ---abstraction, simplification--> syntax tree
    * type checking goes here
4. ... ---rewriting, optimization--- ...
5. --code generation--> target

## Syntax trees

* **concrete syntax**: the syntax definition exactly
* **abstract syntax tree**: the program representation