# REPL-Game Integration: Running Games While Keeping the IDE Responsive

## The Problem

In traditional Forth game development, running the game loop blocks the REPL:

- The game's `go` command runs a blocking loop that processes frames continuously
- While the game runs, the VFX Forth IDE becomes unresponsive
- You cannot type commands, inspect variables, modify code, or use debugging features
- The only way to regain control is to quit the game entirely

This creates a frustrating development cycle where you must constantly stop and restart the game to test changes or inspect state.

## The Goal

Enable the game to run continuously while maintaining **full** VFX Forth IDE functionality:

- Type commands and execute them immediately
- Inspect and modify variables in real-time
- Use mouse selection, copy/paste, and command history
- Access all IDE features (debugger, menus, etc.)
- Modify game code and test changes without restarting

## How VFX Forth's REPL Works

Understanding the REPL's architecture was key to solving this problem:

### The Input Chain

1. **User Input**: The REPL needs to read a line of text from the user
2. **ACCEPT**: VFX Forth calls the deferred word `ACCEPT` to get input
3. **IO Device**: `ACCEPT` is assigned to `richedit-accept` for the IDE console
4. **Blocking Loop**: `richedit-accept` enters a blocking loop waiting for user input:
   ```forth
   : richedit-key ( - char )
     begin
       dup richedit-key? 0=
     while
       2 ms emptyidle    \ Process Windows messages
     repeat
     richedit-Ififo FIFO>(b) drop
   ;
   ```

5. **Polling**: `richedit-key?` checks for available input on every iteration
6. **BusyIdle Hook**: On every poll, `richedit-key?` calls `BusyIdle`:
   ```forth
   : richedit-key? ( sid -- flag )
     dup richedit-flags @ re-quit-mask and
     if drop -1 exit then
     busyidle              \ <-- Called on every poll!
     dup richedit-FLUSHOP drop
     richedit-Ififo FIFO? 0<>
   ;
   ```

### The Key Insight

**`BusyIdle` is called continuously while the REPL waits for input!**

This deferred word is designed to let the system do work while idle. By hooking into `BusyIdle`, we can run game frames during the natural idle periods of the REPL's input polling loop.

## The Solution

### Architecture Overview

Instead of using multitasking or complex threading:

1. **Hook BusyIdle**: Replace `BusyIdle` with our own implementation that runs game frames
2. **Frame Pacing**: Use the existing frame timer and `spin` for proper frame timing
3. **Error Handling**: Catch and report game errors without crashing the IDE
4. **State Management**: Use a flag to control when the game loop is active

### Implementation Components

#### 1. Save Original BusyIdle

```forth
['] BusyIdle >body @ value 'original-busyidle
```

We save the original `BusyIdle` action so we can chain to it after running our frame.

#### 2. Control Flag

```forth
variable repl-game-active
```

This flag indicates whether the REPL game mode is active.

#### 3. Frame Execution

```forth
: repl-mode-frame ( - )
    update-delta
    process-input
    process-tick
    system-controls
    display al_flip_display
    spin
    ;
```

This is identical to the regular `frame` word but separated for clarity. The `spin` call waits for the next frame timer event, providing proper frame pacing.

#### 4. Error-Safe Frame

```forth
: repl-frame ( - )
    ['] repl-mode-frame catch ?dup if
        repl-game-active off
        going @ if post then
        cr ." Game error: " .throw cr
    then ;
```

Wraps frame execution in error handling so game crashes don't crash the IDE.

#### 5. BusyIdle Hook

```forth
: game-busyidle ( - )
    repl-game-active @ if
        going @ if
            repl-frame
        else
            \ Game ended itself (e.g., via system-controls)
            repl-game-active off
            post
            cr ." Game stopped." cr
        then
    then
    'original-busyidle execute ;
```

This is the heart of the system:
- If REPL game mode is active and the game is running, execute a frame
- If the game has stopped itself (e.g., user pressed ESC), clean up
- Always chain to the original `BusyIdle` to maintain normal IDE functionality

#### 6. Initialization

```forth
: repl-pre ( - )
    update-delta update-delta
    need-pump on
    going on
    frame-timer al_resume_timer
    spin ;
```

Similar to the regular `pre` but doesn't call `>display` (which switches focus to the game window). The frame timer is started and we wait for the first frame.

#### 7. User Commands

```forth
: repl-go ( - )
    going @ if
        cr ." Game already running." cr exit
    then
    repl-pre
    repl-game-active on
    cr ." Game started (REPL mode). Type 'repl-stop' to stop." cr ;

: repl-stop ( - )
    repl-game-active @ 0= if
        cr ." Game not running in REPL mode." cr exit
    then
    repl-game-active off
    going @ if post then
    cr ." Game stopped." cr ;

: repl-toggle ( - )
    repl-game-active @ if repl-stop else repl-go then ;
```

Simple commands to start, stop, and toggle the game.

## How It Works in Practice

### Execution Flow

1. **User types `repl-go`**:
   - `repl-pre` initializes game state and starts the frame timer
   - `repl-game-active` flag is set to true
   - Control returns to the REPL immediately

2. **REPL waits for next command**:
   - `richedit-accept` enters its polling loop
   - Every iteration calls `richedit-key?`
   - `richedit-key?` calls `BusyIdle` (which is now `game-busyidle`)

3. **`game-busyidle` runs**:
   - Checks if REPL game mode is active (yes)
   - Checks if game is running (yes)
   - Calls `repl-frame` to execute one frame
   - `spin` waits for the frame timer event (proper pacing)
   - Returns to `richedit-key?`
   - Chains to original `BusyIdle` for normal IDE processing

4. **User can type at any time**:
   - When a key is pressed, `richedit-key?` returns true
   - The character is processed normally
   - On the next idle poll, another frame runs
   - The cycle continues

5. **User types `repl-stop`**:
   - `repl-game-active` flag is cleared
   - `post` cleans up game state
   - Game loop stops running
   - REPL continues normally

### Frame Timing

Frame pacing is controlled by the frame timer and `spin`:

- The frame timer generates events at the configured rate (e.g., 60 FPS)
- `spin` waits for the next frame timer event
- This provides accurate frame timing without busy-waiting
- Frames run at the proper rate regardless of REPL activity

## Usage

### Starting the Game in REPL Mode

```forth
repl-go
```

The game window opens and the game runs. The VFX Forth console remains fully functional.

### Interacting While Running

```forth
\ Inspect game variables
player-x @ .
player-y @ .

\ Modify game state
999 player-health !

\ Redefine game words
: player-speed ( - n ) 10 ;  \ Make player faster

\ Call game functions
reset-level
spawn-enemy
```

### Stopping the Game

```forth
repl-stop
```

Or press ESC in the game window (if `esc-quits` is enabled).

### Toggling

```forth
repl-toggle  \ Start if stopped, stop if started
```

## Comparison with Standard `go`

### Traditional `go` Command

```forth
go  \ Blocks until game ends
```

- Starts game with `>display` (switches focus to game window)
- Runs blocking frame loop
- REPL is completely unresponsive
- Must quit game to regain REPL control
- Used for release builds

### REPL-Mode `repl-go`

```forth
repl-go  \ Returns immediately
```

- Starts game without switching focus
- Runs frames during REPL idle periods
- REPL remains fully responsive
- Can inspect/modify game state while running
- Perfect for development and debugging

## Technical Details

### Why This Works

- **Single-threaded**: No multitasking complexity, thread-safety issues, or race conditions
- **Non-invasive**: Doesn't modify core game loop code, just provides an alternate entry point
- **Proper timing**: Uses existing frame timer infrastructure for accurate pacing
- **Error isolation**: Game crashes don't crash the IDE
- **Full integration**: All IDE features remain functional (debugger, history, mouse, etc.)

### Limitations

- Window focus management is manual (game doesn't auto-focus)
- Frame timing depends on REPL polling frequency
- Not suitable for release builds (use standard `go` for that)

### When to Use Each Mode

**Use `repl-go` for**:
- Development and debugging
- Testing code changes without restarting
- Inspecting/modifying game state in real-time
- Experimenting with game parameters
- Learning and exploration

**Use `go` for**:
- Release builds
- Final testing in production mode
- Benchmarking (avoids REPL overhead)
- When you want the game to take full focus

## Implementation History

This solution emerged from exploring VFX Forth's REPL implementation:

1. **Initial approach**: Tried calling processing words from game loop - didn't work because `ACCEPT` is blocking
2. **Alternative idea**: Making game loop an IO device - too invasive and complex
3. **Key discovery**: `richedit-key?` calls `BusyIdle` on every poll
4. **Solution**: Hook `BusyIdle` to run game frames during idle polling
5. **Refinement**: Use `spin` for proper frame pacing instead of manual timing

The final implementation is simple, elegant, and non-invasive - exactly what Forth development should be.

## Files

- **`engineer/repl-game.vfx`**: Complete REPL-Game integration implementation
- **`engineer/engineer.vfx:33`**: Loads the REPL-Game module
- **`engineer/go.vfx`**: Original game loop (unchanged)

## Conclusion

By understanding VFX Forth's REPL architecture and leveraging the `BusyIdle` hook, we achieved what seemed impossible: running a continuous game loop while maintaining full IDE responsiveness. This transforms the development experience, enabling true interactive development where you can modify and test your game in real-time without ever stopping it.

The key insight was recognizing that the REPL already has an idle processing mechanism - we just needed to hook into it.
