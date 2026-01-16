# Nib 2.0 Migration Assessment

## Current OOP System Usage

The codebase currently uses the original nib.vfx OOP system with:
- `var` for field definitions (maps to `ofield` underneath)
- `m:` for message definitions
- Manual class management without formal class definitions
- Direct field offsets without dynamic property resolution
- No trait system or protocol definitions

## Component Analysis

### 1. Engineer (engineer/)
**Current state:**
- Uses basic nib.vfx for object context (`me`, `{`, `}`)
- No formal classes defined
- Uses `var` for field definitions sparingly
- array2.vfx already migrated to Nib 2.0 as proof of concept

**Migration effort: LOW**
- Most of Engineer doesn't use OOP heavily
- Main changes: Update includes from nib.vfx to nib2.vfx
- array2.vfx demonstrates successful migration pattern

### 2. Supershow (supershow/)
**Current state:**
- Actor system in stage.vfx uses `var` extensively for actor fields
- Fields: en, x, y, vx, vy, bmp, x1, y1, w1, h1, beha, time, n1-n4, phys, prio, benched
- Uses `m:` for messages (e.g., `m: peeked`)
- Fixed-size actors (512 bytes) in pre-allocated array
- No formal class hierarchy

**Migration effort: MEDIUM**
- Convert actor fields from `var` to properties
- Create formal actor% class with Nib 2.0
- Convert `m:` messages to `::` protocols
- Maintain fixed-size actor constraint (use field-space)
- Update all actor field access patterns

**Key challenges:**
- Performance critical - needs careful field vs property choices
- Fixed memory layout requirement for actor pool
- Extensive field usage throughout game code

### 3. Dark Blue (darkblue/)
**Current state:**
- Built on top of Supershow's actor system
- Uses same actor fields inherited from Supershow
- Game-specific behaviors implemented as actor scripts
- No additional OOP extensions beyond Supershow

**Migration effort: LOW (after Supershow)
- Inherits migration from Supershow
- Update script files to use new field access patterns
- Most changes automatic once Supershow migrated

## Migration Strategy

### Phase 1: Engineer Completion
1. ✅ Complete array2.vfx migration (DONE)
2. Update remaining Engineer components to include nib2.vfx
3. Test core functionality

### Phase 2: Supershow Actor System
1. Design actor% class with Nib 2.0:
```forth
class: actor%
    512 field-space  \ Fixed size requirement
    property en
    property x  property y
    property vx property vy
    property bmp
    property x1 property y1 property w1 property h1
    property beha
    property time
    property n1 property n2 property n3 property n4
    property phys
    property prio
    property benched
class;
```

2. Convert messages to protocols:
```forth
actor% :: peeked ( - )
    \ Default implementation
;
```

3. Update field access patterns:
- `x @` becomes `x @` (no change for properties)
- `'s x` becomes `'s x` (already compatible)

4. Maintain actor pool compatibility:
- Keep fixed 512-byte size
- Ensure array allocation works unchanged

### Phase 3: Dark Blue
1. Update includes to use migrated Supershow
2. Test game-specific behaviors
3. Migrate any game-specific extensions

## Risk Assessment

### Low Risk
- Basic field access patterns remain similar
- Object context system (`me`, `{`, `}`) unchanged
- Performance can be maintained with careful field/property choices

### Medium Risk
- Fixed-size actor constraint needs careful handling
- Message dispatch changes from `m:` to `::`
- Some performance-critical paths may need optimization

### High Risk
- None identified - Nib 2.0 designed for compatibility

## Benefits of Migration

1. **Type Safety**: Formal class definitions with validation
2. **Hot-reload**: Properties support safe hot-reload
3. **Traits**: Can add capabilities via traits (e.g., collidable%, drawable%)
4. **Protocols**: Better method organization and dispatch
5. **Maintenance**: Clearer code structure and documentation

## Array2 Migration (Separate Task)

### Current Array Usage

The codebase uses the original array.vfx in several places:
- **supershow/stage.vfx**: `actors` and `temps` arrays
- **supershow/ui/ui.vfx**: `elements` array  
- **engineer/strout.vfx**: `buffers` array
- **spunk/spunk.vfx**: commented out layer-array
- Various test files

### API Differences

**Old array.vfx:**
- `array:` ... `array;` for named arrays with inline data
- `array[` ... `array]` for unnamed arrays
- `*array` for creating empty arrays
- `each` with different callback signature
- Direct `push`/`pop` without protocol qualification

**New array2.vfx (NIBS-based):**
- `array[` ... `array]` for inline data (different implementation)
- No `array:` ... `array;` syntax
- `array% object` or `array% *object` for empty arrays
- `each` requires callback that handles addresses
- `push`/`pop` are protocols (may need qualification)
- New protocols: `head`/`tail`, `head@`/`tail@`

### Migration Requirements

1. **Basic array creation** - Most uses just create arrays with:
   ```forth
   #actors /actor array actors  \ This syntax stays the same!
   ```
   The `array` word would need updating but usage remains identical.

2. **Array indexing** - Core operation remains unchanged:
   ```forth
   actors []  \ Works the same in both systems
   ```

3. **Iteration** - Would need updates if used:
   - Old: `[: ... ;] array each`
   - New: `' callback array each` (callback gets addresses)

4. **Stack operations** - If any code uses push/pop, needs protocol updates

### Migration Effort: LOW (3-4 hours)

Most array usage is just indexing, which doesn't change. The main work is:
1. Update `array` word definition to use array2.vfx
2. Search and update any `each` iterations (rare)
3. Update any array building code (very rare)
4. Test existing functionality

### Files to Update
- engineer/engineer.vfx - include array2.vfx instead of array.vfx
- Any files using array iteration or building (minimal)

## Estimated Timeline

### Nib 2.0 OOP Migration
- **Engineer**: 30 minutes (just change includes)
- **Supershow**: 2-3 hours (wrap in actor% class)
- **Dark Blue**: 30 minutes (testing only)
- **Total**: 3-4 hours

### Array2 Migration (separate task)
- **Update includes and array word**: 1 hour
- **Find and fix iteration usage**: 1-2 hours  
- **Testing**: 1 hour
- **Total**: 3-4 hours

### Combined Total: 6-8 hours

## Recommendation

Proceed with migration in the suggested phases. The actor system in Supershow is the critical component requiring most attention. Once migrated, it provides a solid foundation for all games built on top.

The migration is worthwhile for:
- Improved maintainability
- Better hot-reload support
- Cleaner code organization
- Foundation for future enhancements