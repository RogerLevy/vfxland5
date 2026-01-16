# Oversight v2 Implementation Plan

## Overview
Complete rebuild of the contract-oriented debugging system based on lessons learned from v1. New design eliminates exception handling complexity and uses inline self-healing.

## Phase 1: Core Infrastructure
1. **Define `val:` creation word**
   - Parse validation type (critical/charmful)
   - Create validation function with proper stack signature
   - Store type information for runtime filtering

2. **Contract registry system**
   - Reuse existing contracts dictionary
   - Map word names to contract objects
   - Support `walk>` iteration for global operations

3. **`before`/`after` attachment mechanism**
   - Attach validations to existing words
   - Handle both pre and post validation timing
   - Create word wrappers that call validations

4. **Basic wrapper generation for normal words**
   - Runtime dispatch to validation functions
   - Result code interpretation (0=throw, 1=warn, -1=continue)
   - Preserve original word stack effects

5. **Result code handling**
   - Implement warning vs throwing behavior
   - Integration with logging system

## Phase 2: Control System
1. **Global validation flags**
   - `safety on/off` - master switch
   - `critical safety !` - critical validation flag
   - `charmful safety !` - charmful validation flag

2. **Individual validation control**
   - `disable-validation`/`enable-validation` by validation name
   - Walk contract registry to find all instances
   - Per-instance enabled/disabled flags

3. **Bulk operations**
   - `disable-all-validations`/`enable-all-validations`
   - Batch enable/disable by type

4. **Per-contract enabled flags**
   - `preval`/`postval` mechanism for individual attachment control
   - Enabled flags stored in contract objects

## Phase 3: Self-Healing
1. **Implement `fix?` state-aware word**
   - Returns true on first validation run
   - Returns false on fix verification pass
   - Extensible for per-contract self-healing control

2. **Self-healing retry logic in wrappers**
   - Run validation, attempt fix if failed and fix? is true
   - Re-run validation to verify fix
   - Proper logging of fix attempts and results

3. **Behavior control flags**
   - `self-fixing?` prop for per-validation self-healing control
   - `debug-enabled?` prop for per-validation debug output control

## Phase 4: Utilities & Polish
1. **`warn"`/`info"` with formatting support**
   - Support f" style formatted strings
   - Integration with existing logging system

2. **`.vals` contract inspection**
   - Display all validations attached to a word
   - Show enabled/disabled status and types

3. **Quantified validation helpers**
   - `all-satisfy`, `any-satisfy`, `none-satisfy`, `count-satisfying`
   - Utilities for collection validation in game code

4. **Auto-loading of `-vals.vfx` files**
   - Extend `try`/`include/included/require/required` to
     automatically load validation files
   - `myfile.vfx` loads `myfile-vals.vfx` if it exists

## Phase 5: Deferred Contract Application (Critical Architecture Change)

**Problem:** Current approach requires contracts to be defined before words, creating ordering constraints and preventing validation of words that call other words defined in the same module.

**Solution:** Deferred contract application with automatic wrapping on word definition.

### Implementation Plan

1. **String-based contract storage**
   - Change `>before`/`>after` to take `addr len` instead of `xt`
   - Store contracts by word name string in pending registry
   - `before`/`after` immediate words parse names and store strings

2. **Pending contract registry**
   - Dictionary mapping word names to validation arrays
   - Multiple validations per word name supported
   - Persist across word redefinitions

3. **Redefined `:` and `;` words**
   ```forth
   : : ( <name> )
       next-word-name save-for-later  \ Capture name being defined
       [compile] : ;
   
   : ; 
       current-word-name check-pending-contracts
       apply-matching-validations     \ Auto-wrap if contracts exist
       [compile] ; ; immediate
   ```

4. **Contract file loading workflow**
   ```forth
   \ Step 1: Load validation definitions (no wrapping)
   include module-validations.vfx
   
   \ Step 2: Load source code (auto-wrapping happens)  
   include module.vfx
   ```

### Benefits
- **Clean source separation** - Validation logic completely separate from business logic
- **Universal coverage** - Every word gets validated regardless of internal call patterns
- **Ordering independence** - Contracts can be defined before words exist
- **Interactive development** - Redefined words automatically maintain validation
- **No source contamination** - Core modules stay pure

### File Organization
```
project/
├── src/
│   ├── array.vfx           # Pure business logic
│   ├── math.vfx            # Pure business logic  
│   └── game.vfx            # Pure business logic
└── validation/
    ├── array-contracts.vfx # Validation definitions
    ├── math-contracts.vfx  # Validation definitions
    └── game-contracts.vfx  # Validation definitions
```

## Phase 6: Immediate Words (The Hard Part)
1. **Detection of immediate words at definition time**
   - Identify when target word is immediate
   - Different code path for immediate word handling

2. **Alternative wrapper strategy for immediate words**
   - Compile validation inline vs runtime dispatch
   - Handle stack effects during compilation vs execution

3. **Testing and refinement**
   - Test with critical immediate words like `'s`, `{`, `}`
   - Ensure validation doesn't break immediate word behavior

## Implementation Notes
- **Phase 5 is now critical** - Deferred contract application must be implemented early
- String-based contract definition becomes the primary interface
- XT-based interface becomes secondary for runtime dynamic contracts
- File loading order becomes: validations first, then source modules
- Interactive development workflows benefit greatly from automatic revalidation

## COMPLETED: Define-Before Contract Architecture

### Core Insight
**Contracts must be defined before the words they validate.** This is not a limitation but the optimal architecture for Forth-based systems, as it works with Forth's natural compilation model.

### How It Works
1. **Contract Definition**: `validator before target-word` stores validation in contract registry
2. **Automatic Wrapping**: Redefined `:` and `;` detect when words with contracts are defined and wrap them automatically
3. **Transparent Operation**: Contracted words behave identically to originals but include validation

### Usage Patterns

#### New Words
```forth
\ Define validator first
crucial test: bounds-check ( addr -- addr n:result )
    dup 0<> ;  \ Non-null pointer check

\ Attach to future word
bounds-check before my-function

\ Define word (gets wrapped automatically)
: my-function @ ;
```

#### Existing Words (Core Forth, etc.)
```forth
\ Define validation for existing word
bounds-check before @

\ Redefine word as itself to trigger wrapping
: @ @ ;
```

#### Self-Healing Validators
```forth
crucial test: clamp-index ( n array -- n array n:result )
    over max-items 0 swap within?
    fix? if
        over max-items 1- 0 clamp >r drop r>
        over -1
    then ;

clamp-index before []
```

### Key Benefits
- **Universal coverage**: All word calls get validated, regardless of internal call patterns
- **Clean source separation**: Validation logic separate from business logic
- **Interactive development**: Redefined words automatically maintain validation
- **Zero syntax overhead**: Contracted words look and behave like normal words
- **Performance**: No runtime dispatch overhead when validations pass