# VFX Forth Coding Style Guidelines

## Horizontal and Vertical Word Definition Formats

Short utility and "access" words must use the horizontal format.

Examples:
`: empty? ( tower -- f ) #items 0= ;`
`: peek ( tower -- disk ) dup empty? if drop 0 else tos @ then ;`
`: tower-push ( disk tower -- ) dup full? if 2drop else push then ;`

Complicated functions must use the vertical format and locals, using the {: <params> | <variables> :} pattern.

Typical Multi-line Word Example:
```forth
: word ( a b -- n )
    {: a b | v :} \ Only one locals declaration pattern allowed, at the top of the word definition.
    123 to v     \ initialize local variable `v`
    a b + v + ;  \ leave return value on stack
```

## Name Clashes

Never name locals after standard Forth and VFXLand5 words as they create the potential for bugs.

Bad: `{: count :} str$ count ` the local `count` will take precedence over Forth's `count`, causing unexpected behavior.
Bad: `{: x :} x @ to x` the local `x` will take precedence over the global variable `x`, causing a crash.
Bad: `{: i :} for i loop` local `i` will take precedence over Forth's `i` loop counter, leading to unexpected behavior. 

Words to avoid naming locals after:

count array stack #items max-items min max i j k x y z

## Array Syntax Hints

`<#items> <size-in-bytes> array <name>`
`<#items> <size-in-bytes> stack <name>`
`cell array[ <name> 1 , 2 , 3 , array]`
`<n> <array> [] @`
