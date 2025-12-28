# Spaces (Virtual Desktops) Management - Technical Documentation

This document explains how yabai manages macOS Spaces (virtual desktops), including creation, deletion, focus, and movement operations.

## Overview

macOS Spaces are virtual desktops managed by the window server (SkyLight framework). Yabai interacts with spaces through:

1. **Public APIs**: Limited query operations via `SLSCopy*` functions
2. **Private SkyLight APIs**: Direct space manipulation
3. **Dock Injection (OSAX)**: Advanced operations like create/destroy/move spaces

## Key Source Files

| File | Purpose |
|------|---------|
| `src/space_manager.h` | Space manager struct and function declarations |
| `src/space_manager.c` | Space management implementation (~1150 lines) |
| `src/space.h` | Low-level space query utilities |
| `src/space.c` | Space utility implementations |
| `src/view.h` / `src/view.c` | BSP tree views tied to spaces |
| `src/mission_control.c` | Mission Control event handling |
| `src/sa.m` | Scripting addition client (space operations) |

## Data Structures

### Space Manager (`src/space_manager.h`)

```c
struct space_manager
{
    struct table view;              // Hash table: sid → struct view*
    uint64_t current_space_id;      // Currently active space
    uint64_t last_space_id;         // Previously active space
    bool did_begin;                 // Manager initialized flag
    
    // Default layout settings
    enum view_type layout;
    int top_padding;
    int bottom_padding;
    int left_padding;
    int right_padding;
    int window_gap;
    float split_ratio;
    enum window_node_split split_type;
    enum window_node_child window_placement;
    enum window_insertion_point window_insertion_point;
    bool window_zoom_persist;
    uint32_t auto_balance;
    
    struct space_label *labels;     // User-defined space labels
};

struct space_label
{
    uint64_t sid;
    char *label;
};
```

### Space Operation Errors (`src/space_manager.h`)

```c
enum space_op_error
{
    SPACE_OP_ERROR_SUCCESS              = 0,
    SPACE_OP_ERROR_MISSING_SRC          = 1,
    SPACE_OP_ERROR_MISSING_DST          = 2,
    SPACE_OP_ERROR_INVALID_SRC          = 3,
    SPACE_OP_ERROR_INVALID_DST          = 4,
    SPACE_OP_ERROR_INVALID_TYPE         = 5,  // Not a user space
    SPACE_OP_ERROR_SAME_SPACE           = 6,
    SPACE_OP_ERROR_SAME_DISPLAY         = 7,
    SPACE_OP_ERROR_DISPLAY_IS_ANIMATING = 8,
    SPACE_OP_ERROR_IN_MISSION_CONTROL   = 9,
    SPACE_OP_ERROR_SCRIPTING_ADDITION   = 10,
};
```

### View (per-space layout) (`src/view.h`)

Each space has an associated `struct view` that manages window tiling:

```c
struct view
{
    uint64_t sid;                   // Space ID
    CFStringRef uuid;               // Persistent space UUID
    struct window_node *root;       // BSP tree root
    enum view_type layout;          // VIEW_FLOAT, VIEW_BSP, VIEW_STACK
    // ... padding, gap, split settings
};
```

## Space Types

macOS has three types of spaces, identified by `SLSSpaceGetType()`:

| Type | Value | Description |
|------|-------|-------------|
| User | 0 | Normal desktop created by user |
| System | 2 | Dashboard (legacy) |
| Fullscreen | 4 | Created for fullscreen apps |

```c
// src/space.c
bool space_is_user(uint64_t sid)
{
    return SLSSpaceGetType(g_connection, sid) == 0;
}

bool space_is_fullscreen(uint64_t sid)
{
    return SLSSpaceGetType(g_connection, sid) == 4;
}

bool space_is_system(uint64_t sid)
{
    return SLSSpaceGetType(g_connection, sid) == 2;
}
```

## Space Creation

### Flow: `yabai -m space --create`

```
CLI Command
    ↓
message.c: handle_domain_space() parses "--create"
    ↓
space_manager_add_space(sid)
    ↓
Validates:
  - Not in Mission Control
  - Source space exists
  - Display not animating
    ↓
scripting_addition_create_space(sid)
    ↓
Sends SA_OPCODE_SPACE_CREATE to payload
    ↓
payload.m: do_space_create()
    ↓
Creates ManagedSpace object
Calls Dock's internal addSpace function
    ↓
Dock notifies via SLS callback
    ↓
mission_control.c: handles SLS_SPACE_CREATED event
```

### Implementation

**Client side (`src/space_manager.c`):**
```c
enum space_op_error space_manager_add_space(uint64_t sid)
{
    bool is_in_mc = mission_control_is_active();
    if (is_in_mc) return SPACE_OP_ERROR_IN_MISSION_CONTROL;
    if (!sid)     return SPACE_OP_ERROR_MISSING_SRC;

    bool is_animating = display_manager_display_is_animating(space_display_id(sid));
    if (is_animating) return SPACE_OP_ERROR_DISPLAY_IS_ANIMATING;

    return scripting_addition_create_space(sid) 
           ? SPACE_OP_ERROR_SUCCESS 
           : SPACE_OP_ERROR_SCRIPTING_ADDITION;
}
```

**Payload side (`src/osax/payload.m`):**
```c
static void do_space_create(char *message)
{
    uint64_t sid;
    unpack(sid);

    // Get the display for the reference space
    id space_for_display = get_ivar_value(dock_spaces, "_currentDisplaySpace");
    id display_space = space_for_display[space_display_id(sid)];
    id source_space = display_space["_currentSpace"];
    
    // Create new ManagedSpace
    Class managed_space = NSClassFromString(@"ManagedSpace");
    id new_space = [managed_space alloc];
    [new_space init];
    
    // Call Dock's internal addSpace function
    asm__call_add_space(source_space, new_space, add_space_fp);
}
```

## Space Deletion

### Flow: `yabai -m space --destroy`

```
CLI Command
    ↓
message.c: handle_domain_space() parses "--destroy"
    ↓
space_manager_destroy_space(sid)
    ↓
Validates:
  - Not in Mission Control
  - Space exists
  - Is user space (not fullscreen/system)
  - Not the last user space on display
  - Display not animating
    ↓
scripting_addition_destroy_space(sid)
    ↓
Sends SA_OPCODE_SPACE_DESTROY to payload
    ↓
payload.m: do_space_destroy()
    ↓
Calls Dock's internal removeSpace function
    ↓
Dock notifies via SLS callback
    ↓
mission_control.c: handles SLS_SPACE_DESTROYED event
```

### Implementation

**Client side (`src/space_manager.c`):**
```c
enum space_op_error space_manager_destroy_space(uint64_t sid)
{
    bool is_in_mc = mission_control_is_active();
    if (is_in_mc) return SPACE_OP_ERROR_IN_MISSION_CONTROL;

    if (!sid) return SPACE_OP_ERROR_MISSING_SRC;
    if (!space_is_user(sid)) return SPACE_OP_ERROR_INVALID_TYPE;
    if (space_manager_is_space_last_user_space(sid)) return SPACE_OP_ERROR_INVALID_SRC;

    uint32_t did = space_display_id(sid);
    uint64_t first_sid = space_manager_find_first_user_space_for_display(did);

    bool is_animating = display_manager_display_is_animating(did);
    if (is_animating) return SPACE_OP_ERROR_DISPLAY_IS_ANIMATING;

    bool success = scripting_addition_destroy_space(sid);
    if (!success) return SPACE_OP_ERROR_SCRIPTING_ADDITION;

    // Re-tile windows that moved to another space
    if (first_sid) {
        window_manager_validate_and_check_for_windows_on_space(
            &g_space_manager, &g_window_manager, first_sid);
    }

    return SPACE_OP_ERROR_SUCCESS;
}
```

**Payload side (`src/osax/payload.m`):**
```c
static void do_space_destroy(char *message)
{
    uint64_t sid;
    unpack(sid);
    
    id space_for_display = get_ivar_value(dock_spaces, "_currentDisplaySpace");
    id display_space = space_for_display[space_display_id(sid)];
    id source_space = display_space["_currentSpace"];
    
    id monitored_spaces = get_ivar_value(display_space, "_monitoredSpaces");
    for (id space in monitored_spaces) {
        if (space_id(space) == sid) {
            // Call Dock's internal removeSpace function
            asm__call_remove_space(source_space, space, remove_space_fp);
            break;
        }
    }
}
```

## Space Focus

### Flow: `yabai -m space --focus <space>`

```
CLI Command
    ↓
space_manager_focus_space(sid)
    ↓
Validates:
  - Not in Mission Control
  - Not already on target space
  - Display not animating
    ↓
scripting_addition_focus_space(sid)
    ↓
payload.m: do_space_focus()
    ↓
Calls SLSManagedDisplaySetCurrentSpace()
    ↓
If different display: display_manager_focus_display()
```

### Implementation

**Client side (`src/space_manager.c`):**
```c
enum space_op_error space_manager_focus_space(uint64_t sid)
{
    bool is_in_mc = mission_control_is_active();
    if (is_in_mc) return SPACE_OP_ERROR_IN_MISSION_CONTROL;

    uint64_t cur_sid = space_manager_active_space();
    if (cur_sid == sid) return SPACE_OP_ERROR_SAME_SPACE;

    uint32_t cur_did = space_display_id(cur_sid);
    uint32_t new_did = space_display_id(sid);
    bool focus_display = cur_did != new_did;

    bool is_animating = display_manager_display_is_animating(new_did);
    if (is_animating) return SPACE_OP_ERROR_DISPLAY_IS_ANIMATING;

    if (scripting_addition_focus_space(sid)) {
        if (focus_display) {
            display_manager_focus_display(new_did, sid);
        }
    } else {
        return SPACE_OP_ERROR_SCRIPTING_ADDITION;
    }

    return SPACE_OP_ERROR_SUCCESS;
}
```

**Payload side (`src/osax/payload.m`):**
```c
static void do_space_focus(char *message)
{
    uint64_t sid;
    unpack(sid);
    
    uint32_t did = space_display_id(sid);
    CFStringRef uuid = SLSCopyManagedDisplayForSpace(g_connection, sid);
    
    // Hide current space, show target space
    uint64_t current_sid = SLSManagedDisplayGetCurrentSpace(g_connection, uuid);
    CFArrayRef hide = cfarray_of_cfnumbers(&current_sid, sizeof(uint64_t), 1, kCFNumberSInt64Type);
    CFArrayRef show = cfarray_of_cfnumbers(&sid, sizeof(uint64_t), 1, kCFNumberSInt64Type);
    
    SLSHideSpaces(g_connection, hide);
    SLSShowSpaces(g_connection, show);
    SLSManagedDisplaySetCurrentSpace(g_connection, uuid, sid);
    
    CFRelease(hide);
    CFRelease(show);
    CFRelease(uuid);
}
```

## Space Movement

### Moving Space to Another Display

```c
enum space_op_error space_manager_move_space_to_display(struct space_manager *sm, 
                                                        uint64_t sid, uint32_t did)
{
    bool is_in_mc = mission_control_is_active();
    if (is_in_mc) return SPACE_OP_ERROR_IN_MISSION_CONTROL;
    if (!sid)     return SPACE_OP_ERROR_MISSING_SRC;

    uint32_t s_did = space_display_id(sid);
    if (s_did == did) return SPACE_OP_ERROR_INVALID_DST;

    // Can't move last user space off display
    bool last_space = space_manager_is_space_last_user_space(sid);
    if (last_space) return SPACE_OP_ERROR_INVALID_SRC;

    // ... animation checks ...

    bool focus_space = sid == space_manager_active_space();

    if (scripting_addition_move_space_to_display(sid, d_sid, 
            focus_space ? space_manager_prev_space(sid) : 0, 
            focus_space ? 1 : 0)) {
        space_manager_mark_view_invalid(sm, sid);
        if (focus_space) {
            space_manager_focus_space(sid);
        }
        return SPACE_OP_ERROR_SUCCESS;
    }

    return SPACE_OP_ERROR_SCRIPTING_ADDITION;
}
```

### Swapping Spaces

When swapping spaces on the same display, the function uses complex logic to determine the correct sequence of move operations:

```c
enum space_op_error space_manager_swap_space_with_space(uint64_t acting_sid, 
                                                        uint64_t selector_sid)
{
    // If different displays, swap windows between spaces
    if (acting_did != selector_did) {
        return space_manager_swap_space_with_space_on_display(...);
    }
    
    // Same display: use scripting addition to reorder
    // Complex logic handles edge cases:
    // - First space in list
    // - Adjacent spaces
    // - Non-adjacent spaces
}
```

## Key SkyLight APIs Used

### Query APIs

```c
// Get spaces for all displays
CFArrayRef SLSCopyManagedDisplaySpaces(int cid);

// Get display UUID for a space
CFStringRef SLSCopyManagedDisplayForSpace(int cid, uint64_t sid);

// Get space type (0=user, 2=system, 4=fullscreen)
int SLSSpaceGetType(int cid, uint64_t sid);

// Get current space for display
uint64_t SLSManagedDisplayGetCurrentSpace(int cid, CFStringRef uuid);

// Get space UUID (persistent identifier)
CFStringRef SLSSpaceCopyName(int cid, uint64_t sid);
```

### Manipulation APIs (via payload)

```c
// Switch active space on display
void SLSManagedDisplaySetCurrentSpace(int cid, CFStringRef display_ref, uint64_t sid);

// Move windows to space
void SLSMoveWindowsToManagedSpace(int cid, CFArrayRef window_list, uint64_t sid);

// Show/hide spaces
void SLSShowSpaces(int cid, CFArrayRef space_list);
void SLSHideSpaces(int cid, CFArrayRef space_list);

// Assign process to space
void SLSProcessAssignToSpace(int cid, pid_t pid, uint64_t sid);
void SLSProcessAssignToAllSpaces(int cid, pid_t pid);
```

## Mission Control Integration

### Event Types (`src/event_loop.h`)

```c
// Space events detected via mission_control.c
SLS_SPACE_CREATED,
SLS_SPACE_DESTROYED,
SPACE_CHANGED,
```

### Mission Control State

```c
// Check if Mission Control is active
bool mission_control_is_active();

// Toggle Mission Control
void space_manager_toggle_mission_control(uint64_t sid)
{
    space_manager_focus_space(sid);
    CoreDockSendNotification(CFSTR("com.apple.expose.awake"), 0);
}

// Toggle Show Desktop
void space_manager_toggle_show_desktop(uint64_t sid)
{
    space_manager_focus_space(sid);
    CoreDockSendNotification(CFSTR("com.apple.showdesktop.awake"), 0);
}
```

## Space Queries

### Get Mission Control Index

```c
int space_manager_mission_control_index(uint64_t sid)
{
    uint64_t result = 0;
    int desktop_cnt = 1;

    CFArrayRef display_spaces_ref = SLSCopyManagedDisplaySpaces(g_connection);
    int display_spaces_count = CFArrayGetCount(display_spaces_ref);

    for (int i = 0; i < display_spaces_count; ++i) {
        CFDictionaryRef display_ref = CFArrayGetValueAtIndex(display_spaces_ref, i);
        CFArrayRef spaces_ref = CFDictionaryGetValue(display_ref, CFSTR("Spaces"));
        int spaces_count = CFArrayGetCount(spaces_ref);

        for (int j = 0; j < spaces_count; ++j) {
            CFDictionaryRef space_ref = CFArrayGetValueAtIndex(spaces_ref, j);
            CFNumberRef sid_ref = CFDictionaryGetValue(space_ref, CFSTR("id64"));
            CFNumberGetValue(sid_ref, CFNumberGetType(sid_ref), &result);
            if (sid == result) goto out;
            ++desktop_cnt;
        }
    }

    desktop_cnt = 0;
out:
    CFRelease(display_spaces_ref);
    return desktop_cnt;
}
```

### Navigate Spaces

```c
uint64_t space_manager_prev_space(uint64_t sid);  // Previous space in order
uint64_t space_manager_next_space(uint64_t sid);  // Next space in order
uint64_t space_manager_first_space(void);         // First space globally
uint64_t space_manager_last_space(void);          // Last space globally
uint64_t space_manager_active_space(void);        // Currently focused space
uint64_t space_manager_cursor_space(void);        // Space under cursor
```

## View Management

Each space has an associated view for window tiling:

```c
// Get or create view for space
struct view *space_manager_find_view(struct space_manager *sm, uint64_t sid)
{
    struct view *view = table_find(&sm->view, &sid);
    if (!view) {
        view = view_create(sid);
        table_add(&sm->view, &sid, view);
    }
    return view;
}

// Query view (doesn't create if missing)
struct view *space_manager_query_view(struct space_manager *sm, uint64_t sid)
{
    if (sm->did_begin) return space_manager_find_view(sm, sid);
    return table_find(&sm->view, &sid);
}

// Refresh view layout
void space_manager_refresh_view(struct space_manager *sm, uint64_t sid)
{
    struct view *view = space_manager_find_view(sm, sid);
    if (view->layout == VIEW_FLOAT) return;

    view_update(view);
    view_flush(view);
}
```

## Moving Windows Between Spaces

```c
void space_manager_move_window_to_space(uint64_t sid, struct window *window)
{
    // Try direct API first
    if (!workspace_use_macos_space_workaround()) {
        CFArrayRef window_list_ref = cfarray_of_cfnumbers(
            &window->id, sizeof(uint32_t), 1, kCFNumberSInt32Type);
        SLSMoveWindowsToManagedSpace(g_connection, window_list_ref, sid);
        CFRelease(window_list_ref);
    } 
    // Fallback: use scripting addition
    else if (!scripting_addition_move_window_to_space(sid, window->id)) {
        // Final fallback: workspace ID trick
        SLSSpaceSetCompatID(g_connection, sid, 0x79616265);
        SLSSetWindowListWorkspace(g_connection, &window->id, 1, 0x79616265);
        SLSSpaceSetCompatID(g_connection, sid, 0x0);
    }
}
```

## Initialization

```c
void space_manager_begin(struct space_manager *sm)
{
    // Set defaults
    sm->layout = VIEW_FLOAT;
    sm->split_ratio = 0.5f;
    sm->auto_balance = SPLIT_NONE;
    sm->split_type = SPLIT_AUTO;
    sm->window_placement = CHILD_SECOND;
    sm->window_insertion_point = INSERT_FOCUSED;
    sm->window_zoom_persist = true;
    sm->labels = NULL;
    
    // Initialize view hash table
    table_init(&sm->view, 23, hash_view, compare_view);

    // Create views for all existing spaces
    int display_count;
    uint32_t *display_list = display_manager_active_display_list(&display_count);
    if (!display_list) return;

    for (int i = 0; i < display_count; ++i) {
        int space_count;
        uint64_t *space_list = display_space_list(display_list[i], &space_count);
        if (!space_list) continue;

        for (int j = 0; j < space_count; ++j) {
            struct view *view = view_create(space_list[j]);
            table_add(&sm->view, &space_list[j], view);
        }
    }

    sm->current_space_id = space_manager_active_space();
    sm->last_space_id = sm->current_space_id;
    sm->did_begin = true;
}
```

## Space Labels

Users can assign labels to spaces for easier reference:

```c
// Set label for space
void space_manager_set_label_for_space(struct space_manager *sm, 
                                       uint64_t sid, char *label);

// Get label for space ID
struct space_label *space_manager_get_label_for_space(struct space_manager *sm, 
                                                      uint64_t sid);

// Get space ID for label
struct space_label *space_manager_get_space_for_label(struct space_manager *sm, 
                                                      char *label);

// Remove label
bool space_manager_remove_label_for_space(struct space_manager *sm, uint64_t sid);
```

## Known Limitations

1. **Mission Control**: Most space operations fail if Mission Control is active
2. **Animation**: Operations blocked while display is animating space transitions
3. **Last Space**: Cannot destroy the last user space on a display
4. **Fullscreen Spaces**: Cannot directly manipulate fullscreen app spaces
5. **System Spaces**: Dashboard space (legacy) cannot be manipulated
6. **Scripting Addition**: Create/destroy/move require the payload to be loaded
