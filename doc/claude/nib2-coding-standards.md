# Nib 2.0 Implementation Coding Standards

Target File: nib2.vfx
Parallel work strategy: Single file with section markers

The target final size for the component is 500 lines.  (Not a hard limit, just an estimate/expectation)

## Phase 0: Standards Document (for review)

### Interactive Testing Protocol
Based on codeprep.md: Use vfxcore.vfx for testing with 5-second timeout:
```bash
(echo "include nib2-core.vfx"; echo "test-word"; echo "bye"; sleep 1) | vfxlin include engineer/vfxcore.vfx
```

### Locals Usage Guidelines

**Use locals when:**
- Function uses passed parameters several times
- Complex calculations with intermediate values needing descriptive names
- Multi-step algorithms where stack juggling becomes error-prone
- When intermediate results improve code clarity

**Avoid locals when:**
- Simple definitions (immediate consumption and/or very simple stack manipulation)
- Direct field access patterns (`'s field @`)
- Short utility functions
- Functions that just call other functions with reordered stack (USER NOTE: AVOID WRITING SUCH "GLUE" FUNCTIONS)

**Examples:**
```forth
\ Avoid locals - too simple
: is-valid? ( classifier - b ) 's valid-bit @ ;

\ Use locals - complex calculation
: allocate-classifier-bits ( count - first-bit )
    {: count | bit-pos :}
    classifier-bit-map @ to bit-pos
    count bits-available? not if
        -1 abort" No classifier bits available"
    then
    bit-pos count reserve-bits
    bit-pos ;
```

### Word Definition Style

**Horizontal format** for simple access/utility words:
```forth
: valid-classifier? ( addr - b ) @ classifier-magic = ;
: >field-offset ( classifier field-name - offset ) 's field-table @ lookup ;
```

**Vertical format** with locals for complex logic:
```forth
: define-protocol ( classifier name-addr name-len - )
    {: classifier name-addr name-len | protocol-ptr :}
    classifier valid-classifier? not if
        -1 abort" Invalid classifier for protocol definition"
    then
    classifier protocols-table @ name-addr name-len lookup to protocol-ptr
    protocol-ptr 0= if
        name-addr name-len create-protocol to protocol-ptr
        classifier protocol-ptr register-protocol
    then ;
```

### Design Document Compliance

**Allowed words**: Only words from nib-upgrades.txt design document:
- Core: `field`, `property`, `var`, `static`, `class:`, `trait:`, `extension:`
- Runtime: `object`, `create-object`, `<object`, `<setup`, `setup:`
- Queries: `is?`, `valid-classifier?`, `defaults`, `>defaults`
- Implementation: `derive`, `is-a`, `protocol:`, `::`

**Private factors allowed**: Internal implementation helpers not in public API

### File Organization Pattern

```forth
\ Nib 2.0 - [Component Name]
\ Author: [Agent Name] 
\ Purpose: [Brief description]

private

\ Internal constants and variables
123 constant INTERNAL-MAGIC
variable internal-state

\ Private helper functions
: internal-helper ( params - result ) ... ;
: ?validate-input ( input - input ) 
    dup valid? not if -1 abort" Invalid input" then ;

public

\ Public API following design document
: public-word ( params - result ) ... ;
```

### String Handling

**CRITICAL**: Strings in Forth are always (addr,len) pairs, never single values:
```forth
\ WRONG - treating string as single value
: bad-example ( filename$ - )
    dup s" phase" search if include-file then ;

\ CORRECT - handling (addr,len) pair properly
: good-example ( filename-addr filename-len - )
    {: filename-addr filename-len :}
    filename-addr filename-len s" phase" search nip nip if
        filename-addr filename-len include-file
    then ;
```

**Common string operations that take (addr,len) pairs:**
- `search`, `compare`, `include-file`, `type`, `evaluate`
- `$create-object` ($ prefix indicates string parameter)

### Stack Comments

Follow stack-comment-conventions.txt:
```forth
: field-access ( object field-name - addr )
: is? ( object classifier - b )  
: setup: ( #params classifier - <code> ; ) ( ... - )
: collision-force ( src-obj dest-obj - x-force. y-force. )
: string-handler ( name-addr name-len - result )  \ Always show strings as addr,len pairs
```

### Error Handling

**Early exits with ?EXIT and -EXIT:**
```forth
: process-classifier ( classifier - )
    dup 0= ?exit  \ exit if 0 
    dup valid-classifier? not if drop exit then
    ... ;

: process-classifier ( classifier - )
    dup -exit  \ exit if 0 (it's just a shorthand for 0= ?exit)
    dup valid-classifier? not if drop exit then
    ... ;
```

**Abort with descriptive messages:**
```forth
classifier-bits-full? if
    -1 abort" Classifier bit pool exhausted"
then
```

### Naming Conventions

**Variables**: `kebab-case` with descriptive context
- `classifier-bit-map`, `field-offset-table`, `protocol-registry`

**Locals**: Short, avoiding dashes when possible
- `class`, `method`, `bit-num`, `xt`, `#nouns`, `name-a`, `name-len`

**Functions**: Action verbs, brief but clear
- `allocate-bits`, `register-protocol`, `validate-stack-diagram`

**Private helpers**: Descriptive, often with ? prefix for validators
- `?valid-field-type`, `?classifier-exists`

**Constants**: SCREAMING_CASE or descriptive
- `CLASSIFIER_MAGIC`, `#max-classifiers`

### Memory Management

**1024-bit Bit table:**
```forth
    
    1024 constant #max-classifiers
    static bit-table
    #max-classifiers 8 / cell- +to /static

```

### Object-Oriented Patterns

**Field access consistency:**
```forth
obj >field-name @          \ for single access / address calculation
obj 's field-name @        \ old single access / address calculation, being phased out
{ field-name @ }         \ for scoped access
```

###  Validation Policy
- NO Oversight validations in nib2.vfx itself
- Validations belong in separate validation files (like nib2-checks.vfx)
- Focus on clean implementation without scattered validation code
- Use simple error handling (abort messages, early exits) instead

### Agent Requirements

**All code-writing agents must:**
1. Include codeprep.md references for testing setup
2. Follow these exact patterns for locals usage
3. Only write words specified in nib-upgrades.txt or private factors
4. Include comprehensive stack comments when doing multi-line stack operations
5. Implement graceful error handling
6. Use consistent naming conventions
7. Structure files with private/public organization

**Quality gates:**
- Interactive testing with vfxcore.vfx
- Stack diagram verification
- Design document compliance check
- Performance benchmarking vs current nib.vfx