VFXLand5 has its own dialect, deviating slightly from standard Forth

## Engineer redefinitions
`not` - Boolean negation.  Equivalent to `0=`.
`2+` `2-` - Add/subtract integer/fixed coordinates ( x y x y -- x y )
`{` `}` - Object scoping.  Usage: `<object> { ... } `
`2@` `2!` - Fetch/store coordinates ( a -- x y ) ( x y a -- ) stored in memory in this order: [ x , y ].  Only for integers and fixed-point values.
`$+` - Append string to counted string ( $ a n -- $ ) 

## Engineer important words to know
`\`` `\`\`` `\`\`\`` `\`\`\`\` - No-ops.  Syntax suger to aid Forth stack management
`for` - `( n -- )` Equivalent to `0 ?do`.  Usage: `<n> for ... loop`

## Deprecated words
`1+` - use `1 +`
`1-` - use `1 -`
`2*` - use `2 *`
`2/` - use `2 /`

## VFX Forth-specific words and patterns
`'c'` - equivalent to `[char] c`