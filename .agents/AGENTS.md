# AGENTS.md - LLM Agent Instructions for yabai

This file provides guidance for LLM agents working with the yabai codebase.

## Project Overview

yabai is a tiling window manager for macOS that extends the built-in window manager. It uses:
- Binary Space Partitioning (BSP) algorithm for window layout
- macOS Accessibility APIs for window control
- Private SkyLight framework for advanced features
- Mach-based code injection into Dock.app for privileged operations

## Knowledge Base

The `.agents/knowledge/` directory contains detailed technical documentation:

| Document | Topics Covered |
|----------|---------------|
| [`dock-injection.md`](knowledge/dock-injection.md) | OSAX system, Mach injection, payload architecture, SkyLight APIs |
| [`spaces-management.md`](knowledge/spaces-management.md) | Space creation/deletion, focus, movement, Mission Control integration |
| [`windows-management.md`](knowledge/windows-management.md) | Window tracking, BSP trees, tiling, floating, operations |

**Always consult these documents before making changes to related code.**

## Build Commands

```bash
# Debug build
make

# Production build  
make install

# Clean
make clean

# Run tests
cd tests && make all
```

## Architecture Overview

### Global Manager Instances (src/yabai.c)

```c
struct window_manager g_window_manager;  // Windows and applications
struct space_manager g_space_manager;    // Spaces and layouts
struct display_manager g_display_manager; // Displays
struct process_manager g_process_manager; // Running processes
struct event_loop g_event_loop;          // Event dispatch
struct mouse_state g_mouse_state;        // Mouse tracking
int g_connection;                        // SkyLight connection ID
```

### Key Source Files

| File | Purpose | Lines |
|------|---------|-------|
| `src/yabai.c` | Entry point, socket server, initialization | ~500 |
| `src/message.c` | IPC command parser (largest file) | ~4000 |
| `src/window_manager.c` | Window operations | ~2500 |
| `src/space_manager.c` | Space operations | ~1150 |
| `src/view.c` | BSP tree implementation | ~1200 |
| `src/event_loop.c` | Event handling | ~800 |
| `src/sa.m` | Scripting addition client | ~500 |
| `src/osax/payload.m` | Injected payload | ~800 |

### Data Flow

```
User Command (yabai -m ...)
    ↓
message.c: parse command
    ↓
*_manager.c: validate and execute
    ↓
window.c/space.c: low-level operations
    ↓
SkyLight API or Scripting Addition
    ↓
Event callbacks update state
```

## Coding Conventions

### Style

- C11 standard
- 4-space indentation
- Opening brace on same line
- `snake_case` for functions and variables
- `UPPER_SNAKE_CASE` for constants and macros
- Prefix global manager functions with manager name: `window_manager_*`, `space_manager_*`

### Memory Management

- Manual memory management (no ARC for Objective-C: `-fno-objc-arc`)
- Custom memory pools for frequent allocations (`src/misc/memory_pool.h`)
- Hash tables for fast lookups (`src/misc/hashtable.h`)
- Dynamic arrays via `buf_*` macros from `sbuffer.h`

### Error Handling

- Return enum error codes from operations
- Check return values immediately
- Use `goto err;` pattern for cleanup

### Example Function Pattern

```c
enum space_op_error space_manager_some_operation(uint64_t sid)
{
    // Early validation
    if (!sid) return SPACE_OP_ERROR_MISSING_SRC;
    
    bool is_in_mc = mission_control_is_active();
    if (is_in_mc) return SPACE_OP_ERROR_IN_MISSION_CONTROL;
    
    // Get dependencies
    uint32_t did = space_display_id(sid);
    bool is_animating = display_manager_display_is_animating(did);
    if (is_animating) return SPACE_OP_ERROR_DISPLAY_IS_ANIMATING;
    
    // Perform operation
    if (!scripting_addition_do_thing(sid)) {
        return SPACE_OP_ERROR_SCRIPTING_ADDITION;
    }
    
    return SPACE_OP_ERROR_SUCCESS;
}
```

## Important Considerations

### SIP (System Integrity Protection)

Many features require SIP to be partially disabled:
- `CSR_ALLOW_UNRESTRICTED_FS` - Write to `/Library/ScriptingAdditions`
- `CSR_ALLOW_TASK_FOR_PID` - Inject into Dock.app

### Apple Silicon

- Requires `-arm64e_preview_abi` boot argument
- Pointer Authentication (PAC) must be handled correctly
- Different byte patterns for each architecture

### macOS Version Compatibility

- Minimum: macOS 11.0 (Big Sur)
- Payload contains version-specific patterns in:
  - `src/osax/arm64_payload.m`
  - `src/osax/x64_payload.m`

When Apple releases new macOS versions, patterns may need updating.

### Thread Safety

- Main event loop runs on main thread
- Window animations run on background thread
- Use `pthread_mutex_t` for shared state
- `window_animations_lock` protects animation table

## Common Development Tasks

### Adding a New Window Command

1. Add command parsing in `src/message.c` `handle_domain_window()`
2. Implement operation in `src/window_manager.c`
3. If needed, add scripting addition opcode in `src/osax/common.h`
4. Implement payload handler in `src/osax/payload.m`
5. Add client call in `src/sa.m`

### Adding a New Space Command

1. Add parsing in `src/message.c` `handle_domain_space()`
2. Implement in `src/space_manager.c`
3. Add error code to `enum space_op_error` if needed
4. May need scripting addition for privileged operations

### Modifying BSP Tree Behavior

1. Understand `struct window_node` in `src/view.h`
2. Core operations in `src/view.c`:
   - `window_node_split()` - Splitting nodes
   - `window_node_update()` - Area calculation
   - `view_add_window_node()` - Adding windows
   - `view_remove_window_node()` - Removing windows

### Updating for New macOS Version

1. Reverse engineer Dock.app with Ghidra/Hopper
2. Find byte patterns for internal functions/objects
3. Update pattern functions in `*_payload.m` files
4. Update `OSAX_VERSION` in `src/osax/common.h`
5. Test all scripting addition features

## Testing Changes

```bash
# Build and test
make && cd tests && make all

# Manual testing
./bin/yabai &
yabai -m query --windows
yabai -m window --focus next
```

## Debugging Tips

1. **Console.app** - Check for `[yabai]` and `[yabai-sa]` logs
2. **SIP status** - `csrutil status`
3. **Socket exists** - `ls -la /tmp/yabai_*.socket`
4. **Scripting addition** - `ls -la /Library/ScriptingAdditions/yabai.osax`
5. **Build with sanitizers**:
   ```bash
   make asan  # Address sanitizer
   make tsan  # Thread sanitizer
   ```

## Resources

- [yabai wiki](https://github.com/koekeishiya/yabai/wiki)
- [SkyLight headers](https://github.com/nicklockwood/SkyLight) (unofficial)
- [Accessibility API docs](https://developer.apple.com/documentation/applicationservices/axuielement_h)
