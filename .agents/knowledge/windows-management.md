# Window Management - Technical Documentation

This document explains how yabai manages windows, including tracking, tiling with BSP trees, and window operations.

## Overview

Yabai's window management system:
1. Tracks all windows via macOS Accessibility APIs (AXUIElement)
2. Stores windows in hash tables for fast lookup
3. Uses Binary Space Partitioning (BSP) trees for tiling layout
4. Supports floating windows that bypass tiling
5. Integrates with the Dock injection payload for advanced operations

## Key Source Files

| File | Purpose |
|------|---------|
| `src/window_manager.h` | Window manager struct and function declarations (~217 lines) |
| `src/window_manager.c` | Window management implementation (~2500+ lines) |
| `src/window.h` | Window struct and property definitions |
| `src/window.c` | Window object operations |
| `src/view.h` | BSP tree and view structures |
| `src/view.c` | BSP tree implementation |
| `src/application.h` / `src/application.c` | Application tracking |

## Data Structures

### Window (`src/window.h`)

```c
struct window
{
    struct application *application;  // Parent application
    AXUIElementRef ref;               // Accessibility API reference
    uint32_t id;                      // Unique window ID (CGWindowID)
    uint32_t *volatile id_ptr;        // Pointer for async operations
    CFStringRef role;                 // Window role (e.g., "AXWindow")
    CFStringRef subrole;              // Window subrole (e.g., "AXStandardWindow")
    CFStringRef title;                // Window title
    CGRect frame;                     // Current frame (origin + size)
    CGRect windowed_frame;            // Frame before fullscreen
    bool is_root;                     // Is application's main window
    bool is_eligible;                 // Eligible for management
    uint8_t notification;             // AX notification mask
    uint8_t rule_flags;               // Applied rule flags
    uint8_t flags;                    // Window state flags
    float opacity;                    // Current opacity
    int layer;                        // Window layer
    char *scratchpad;                 // Scratchpad label if assigned
};
```

### Window Flags

```c
enum window_flag
{
    WINDOW_SHADOW     = 0x01,  // Has shadow
    WINDOW_FULLSCREEN = 0x02,  // In native fullscreen
    WINDOW_MINIMIZE   = 0x04,  // Is minimized
    WINDOW_FLOAT      = 0x08,  // Is floating (not tiled)
    WINDOW_STICKY     = 0x10,  // Visible on all spaces
    WINDOW_WINDOWED   = 0x20,  // Was windowed before fullscreen
    WINDOW_MOVABLE    = 0x40,  // Can be moved
    WINDOW_RESIZABLE  = 0x80   // Can be resized
};
```

### Window Manager (`src/window_manager.h`)

```c
struct window_manager
{
    AXUIElementRef system_element;          // System-wide accessibility element
    struct table application;               // pid → struct application*
    struct table window;                    // window_id → struct window*
    struct table managed_window;            // window_id → struct view*
    struct table window_lost_focused_event; // Deferred focus events
    struct table application_lost_front_switched_event;
    struct table window_animations_table;
    struct table insert_feedback;
    pthread_mutex_t window_animations_lock;
    
    struct rule *rules;                     // User-defined rules
    struct application **applications_to_refresh;
    
    uint32_t focused_window_id;
    ProcessSerialNumber focused_window_psn;
    uint32_t last_window_id;
    
    // Modes and settings
    bool enable_mff;                        // Mouse follows focus
    enum ffm_mode ffm_mode;                 // Focus follows mouse mode
    enum purify_mode purify_mode;
    enum window_origin_mode window_origin_mode;
    
    // Opacity settings
    bool enable_window_opacity;
    float menubar_opacity;
    float active_window_opacity;
    float normal_window_opacity;
    float window_opacity_duration;
    
    // Animation settings
    float window_animation_duration;
    int window_animation_easing;
    
    struct rgba_color insert_feedback_color;
    struct scratchpad *scratchpad_window;
};
```

### BSP Tree Node (`src/view.h`)

```c
#define NODE_MAX_WINDOW_COUNT 32

struct window_node
{
    struct area area;                   // Node's screen area
    struct window_node *parent;         // Parent node
    struct window_node *left;           // Left/top child
    struct window_node *right;          // Right/bottom child
    struct window_node *zoom;           // Zoom state storage
    
    uint32_t window_list[NODE_MAX_WINDOW_COUNT];   // Windows in this node
    uint32_t window_order[NODE_MAX_WINDOW_COUNT];  // Stacking order
    int window_count;                   // Number of windows
    
    float ratio;                        // Split ratio (0.0 - 1.0)
    enum window_node_split split;       // SPLIT_X or SPLIT_Y
    enum window_node_child child;       // Which child gets new windows
    int insert_dir;                     // Insertion direction
    struct feedback_window feedback_window;
};

struct area
{
    float x;
    float y;
    float w;
    float h;
};
```

### View (per-space layout) (`src/view.h`)

```c
struct view
{
    CFStringRef uuid;             // Persistent space identifier
    uint64_t sid;                 // Space ID
    struct window_node *root;     // BSP tree root
    uint32_t insertion_point;     // Where new windows go
    enum view_type layout;        // VIEW_BSP, VIEW_STACK, VIEW_FLOAT
    enum window_node_split split_type;
    
    // Layout settings
    int top_padding;
    int bottom_padding;
    int left_padding;
    int right_padding;
    int window_gap;
    uint32_t auto_balance;
    uint64_t flags;
};

enum view_type
{
    VIEW_DEFAULT,  // Inherit from space_manager
    VIEW_BSP,      // Binary space partitioning
    VIEW_STACK,    // All windows stacked
    VIEW_FLOAT     // No automatic tiling
};
```

## Hash Table Storage

Windows and applications are stored in hash tables for O(1) lookup:

```c
// Window lookup by ID
struct window *window_manager_find_window(struct window_manager *wm, uint32_t window_id)
{
    return table_find(&wm->window, &window_id);
}

// Application lookup by PID
struct application *window_manager_find_application(struct window_manager *wm, pid_t pid)
{
    return table_find(&wm->application, &pid);
}

// Check if window is managed (tiled)
struct view *window_manager_find_managed_window(struct window_manager *wm, struct window *window)
{
    return table_find(&wm->managed_window, &window->id);
}
```

## Window Discovery and Tracking

### Application Observation

When an application launches, yabai creates an AXObserver to watch for window events:

```c
// From application.c
bool application_observe(struct application *application)
{
    AXObserverCreate(application->pid, &observer_handler, &application->observer_ref);
    
    // Watch for window creation, destruction, focus, etc.
    application_observe_notification(application, application->ref, 
                                     kAXWindowCreatedNotification);
    application_observe_notification(application, application->ref, 
                                     kAXFocusedWindowChangedNotification);
    // ... more notifications
}
```

### Window Creation Flow

```
Application launches
    ↓
process_manager detects via NSWorkspace
    ↓
application_create() - Create application object
    ↓
window_manager_add_application() - Add to hash table
    ↓
application_observe() - Set up AX notifications
    ↓
window_manager_add_application_windows() - Enumerate existing windows
    ↓
For each window:
    window_manager_create_and_add_window()
        ↓
    window_create() - Create window object
        ↓
    window_manager_add_window() - Add to hash table
        ↓
    window_observe() - Set up window notifications
        ↓
    Apply rules (window_manager_apply_rules_to_window)
        ↓
    If should manage: space_manager_tile_window_on_space()
```

### Window Creation Implementation

```c
struct window *window_manager_create_and_add_window(
    struct space_manager *sm, 
    struct window_manager *wm, 
    struct application *application, 
    AXUIElementRef window_ref, 
    uint32_t window_id, 
    bool one_shot_rules)
{
    // Create window object
    struct window *window = window_create(application, window_ref, window_id);
    if (!window) return NULL;
    
    // Add to window hash table
    window_manager_add_window(wm, window);
    
    // Set up AX notifications
    if (window_observe(window)) {
        // Apply user rules
        char *title = window_title_ts(window);
        char *role = window_role_ts(window);
        char *subrole = window_subrole_ts(window);
        
        window_manager_apply_rules_to_window(sm, wm, window, 
                                             title, role, subrole, one_shot_rules);
        
        // Tile if should manage
        if (window_manager_should_manage_window(window) && 
            !window_check_flag(window, WINDOW_FLOAT)) {
            uint64_t sid = window_space(window->id);
            space_manager_tile_window_on_space(sm, window, sid);
        }
    }
    
    return window;
}
```

## BSP Tree Tiling Algorithm

### Overview

The BSP tree recursively divides screen space. Each leaf node contains one or more windows (stacked).

```
                    root
                   /    \
               left      right
              /    \        |
          left   right   [window3]
            |       |
       [window1] [window2]
```

### Adding a Window

```c
struct window_node *view_add_window_node_with_insertion_point(
    struct view *view, 
    struct window *window, 
    uint32_t insertion_point)
{
    if (!view->root) {
        // First window - create root node
        view->root = window_node_create(window->id);
        view_update(view);
        return view->root;
    }
    
    // Find where to insert (at insertion point or focused window)
    struct window_node *leaf = view_find_window_node(view, insertion_point);
    if (!leaf) leaf = window_node_find_first_leaf(view->root);
    
    // Stack layout - add to existing node
    if (view->layout == VIEW_STACK) {
        view_stack_window_node(leaf, window);
        return leaf;
    }
    
    // BSP layout - split the node
    return window_node_split(view, leaf, window);
}
```

### Splitting a Node

```c
struct window_node *window_node_split(
    struct view *view, 
    struct window_node *node, 
    struct window *window)
{
    // Determine split direction
    enum window_node_split split = view->split_type;
    if (split == SPLIT_AUTO) {
        // Split along longer axis
        split = node->area.w > node->area.h ? SPLIT_X : SPLIT_Y;
    }
    
    // Create new nodes
    struct window_node *left = window_node_create(node->window_list[0]);
    struct window_node *right = window_node_create(window->id);
    
    // Set up tree structure
    node->left = left;
    node->right = right;
    left->parent = node;
    right->parent = node;
    
    node->split = split;
    node->ratio = 0.5f;  // Default 50/50 split
    
    // Clear window from parent (now intermediate node)
    node->window_count = 0;
    
    return right;
}
```

### Calculating Node Areas

```c
void window_node_update(struct view *view, struct window_node *node)
{
    if (window_node_is_leaf(node)) {
        return;  // Leaf nodes use parent's calculation
    }
    
    struct area *area = &node->area;
    int gap = view_check_flag(view, VIEW_ENABLE_GAP) ? view->window_gap : 0;
    
    if (node->split == SPLIT_Y) {
        // Vertical split - left/right
        float width = area->w * node->ratio;
        
        node->left->area.x = area->x;
        node->left->area.y = area->y;
        node->left->area.w = width - gap/2;
        node->left->area.h = area->h;
        
        node->right->area.x = area->x + width + gap/2;
        node->right->area.y = area->y;
        node->right->area.w = area->w - width - gap/2;
        node->right->area.h = area->h;
    } else {
        // Horizontal split - top/bottom
        float height = area->h * node->ratio;
        
        node->left->area.x = area->x;
        node->left->area.y = area->y;
        node->left->area.w = area->w;
        node->left->area.h = height - gap/2;
        
        node->right->area.x = area->x;
        node->right->area.y = area->y + height + gap/2;
        node->right->area.w = area->w;
        node->right->area.h = area->h - height - gap/2;
    }
    
    // Recurse
    window_node_update(view, node->left);
    window_node_update(view, node->right);
}
```

## Floating vs Tiled Windows

### Determining if Window Should Be Managed

```c
bool window_manager_should_manage_window(struct window *window)
{
    // Check rule flags first
    if (window_check_rule_flag(window, WINDOW_RULE_MANAGED)) {
        return window_check_rule_flag(window, WINDOW_RULE_MFF_VALUE);
    }
    
    // Must be movable and resizable
    if (!window_can_move(window)) return false;
    if (!window_can_resize(window)) return false;
    
    // Must be standard window type
    if (!window_is_standard(window)) return false;
    
    // Must be at normal level
    if (!window_level_is_standard(window)) return false;
    
    return true;
}
```

### Making Window Float

```c
void window_manager_make_window_floating(
    struct space_manager *sm, 
    struct window_manager *wm, 
    struct window *window, 
    bool should_float, 
    bool force)
{
    if (should_float) {
        // Untile from BSP tree
        struct view *view = window_manager_find_managed_window(wm, window);
        if (view) {
            space_manager_untile_window(view, window);
            window_manager_remove_managed_window(wm, window->id);
        }
        window_set_flag(window, WINDOW_FLOAT);
    } else {
        // Add back to tiling
        window_clear_flag(window, WINDOW_FLOAT);
        if (window_manager_should_manage_window(window)) {
            uint64_t sid = window_space(window->id);
            struct view *view = space_manager_tile_window_on_space(sm, window, sid);
            window_manager_add_managed_window(wm, window, view);
        }
    }
}
```

## Window Operations

### Swap Windows

```c
enum window_op_error window_manager_swap_window(
    struct space_manager *sm, 
    struct window_manager *wm, 
    struct window *a, 
    struct window *b)
{
    if (a->id == b->id) return WINDOW_OP_ERROR_SAME_WINDOW;
    
    struct view *a_view = window_manager_find_managed_window(wm, a);
    struct view *b_view = window_manager_find_managed_window(wm, b);
    
    if (!a_view) return WINDOW_OP_ERROR_INVALID_SRC_VIEW;
    if (!b_view) return WINDOW_OP_ERROR_INVALID_DST_VIEW;
    
    struct window_node *a_node = view_find_window_node(a_view, a->id);
    struct window_node *b_node = view_find_window_node(b_view, b->id);
    
    if (!a_node) return WINDOW_OP_ERROR_INVALID_SRC_NODE;
    if (!b_node) return WINDOW_OP_ERROR_INVALID_DST_NODE;
    
    // Swap window lists between nodes
    window_node_swap_window_list(a_node, b_node);
    
    // Update managed window table
    window_manager_remove_managed_window(wm, a->id);
    window_manager_remove_managed_window(wm, b->id);
    window_manager_add_managed_window(wm, a, b_view);
    window_manager_add_managed_window(wm, b, a_view);
    
    // Flush changes to screen
    window_node_flush(a_node);
    window_node_flush(b_node);
    
    return WINDOW_OP_ERROR_SUCCESS;
}
```

### Warp Window (Move to Different Position)

```c
enum window_op_error window_manager_warp_window(
    struct space_manager *sm, 
    struct window_manager *wm, 
    struct window *a, 
    struct window *b)
{
    // Remove from current position
    struct view *a_view = window_manager_find_managed_window(wm, a);
    space_manager_untile_window(a_view, a);
    window_manager_remove_managed_window(wm, a->id);
    
    // Insert at new position (next to window b)
    struct view *b_view = window_manager_find_managed_window(wm, b);
    struct window_node *b_node = view_find_window_node(b_view, b->id);
    
    struct window_node *new_node = view_add_window_node_with_insertion_point(
        b_view, a, b->id);
    
    window_manager_add_managed_window(wm, a, b_view);
    
    // Flush
    view_update(b_view);
    view_flush(b_view);
    
    return WINDOW_OP_ERROR_SUCCESS;
}
```

### Stack Windows

```c
enum window_op_error window_manager_stack_window(
    struct space_manager *sm, 
    struct window_manager *wm, 
    struct window *a, 
    struct window *b)
{
    if (a->id == b->id) return WINDOW_OP_ERROR_SAME_WINDOW;
    
    struct view *a_view = window_manager_find_managed_window(wm, a);
    struct view *b_view = window_manager_find_managed_window(wm, b);
    struct window_node *b_node = view_find_window_node(b_view, b->id);
    
    if (b_node->window_count >= NODE_MAX_WINDOW_COUNT) {
        return WINDOW_OP_ERROR_MAX_STACK;
    }
    
    // Remove from current position
    space_manager_untile_window(a_view, a);
    window_manager_remove_managed_window(wm, a->id);
    
    // Add to b's node stack
    b_node->window_list[b_node->window_count] = a->id;
    b_node->window_order[b_node->window_count] = a->id;
    b_node->window_count++;
    
    window_manager_add_managed_window(wm, a, b_view);
    window_node_flush(b_node);
    
    return WINDOW_OP_ERROR_SUCCESS;
}
```

### Move Window

```c
void window_manager_move_window(struct window *window, float x, float y)
{
    // Try scripting addition first (more reliable)
    if (scripting_addition_move_window(window->id, x, y)) {
        return;
    }
    
    // Fall back to AX API
    AXValueRef position_ref = AXValueCreate(kAXValueTypeCGPoint, 
                                            &(CGPoint){x, y});
    AXUIElementSetAttributeValue(window->ref, kAXPositionAttribute, position_ref);
    CFRelease(position_ref);
}
```

### Resize Window

```c
void window_manager_resize_window(struct window *window, float width, float height)
{
    AXValueRef size_ref = AXValueCreate(kAXValueTypeCGSize, 
                                        &(CGSize){width, height});
    AXUIElementSetAttributeValue(window->ref, kAXSizeAttribute, size_ref);
    CFRelease(size_ref);
}
```

### Set Window Frame (with animation)

```c
void window_manager_set_window_frame(struct window *window, 
                                     float x, float y, 
                                     float width, float height)
{
    // If animation enabled, animate
    if (g_window_manager.window_animation_duration > 0.0f) {
        struct window_capture capture = {
            .window = window,
            .x = x, .y = y, .w = width, .h = height
        };
        window_manager_animate_window(capture);
        return;
    }
    
    // Otherwise set immediately
    window_manager_move_window(window, x, y);
    window_manager_resize_window(window, width, height);
    
    // Update cached frame
    window->frame.origin.x = x;
    window->frame.origin.y = y;
    window->frame.size.width = width;
    window->frame.size.height = height;
}
```

## Focus Management

### Focus Window

```c
void window_manager_focus_window_with_raise(
    ProcessSerialNumber *psn, 
    uint32_t window_id, 
    AXUIElementRef window_ref)
{
    // Bring application to front
    _SLPSSetFrontProcessWithOptions(psn, window_id, kCPSUserGenerated);
    
    // Set as main/key window
    AXUIElementSetAttributeValue(window_ref, kAXMainAttribute, kCFBooleanTrue);
    AXUIElementPerformAction(window_ref, kAXRaiseAction);
    
    // Update tracking
    g_window_manager.focused_window_id = window_id;
    g_window_manager.focused_window_psn = *psn;
}

void window_manager_focus_window_without_raise(
    ProcessSerialNumber *psn, 
    uint32_t window_id)
{
    // Focus without raising - uses scripting addition
    scripting_addition_focus_window(window_id);
}
```

### Find Windows

```c
// Find window by direction from current
struct window *window_manager_find_closest_managed_window_in_direction(
    struct window_manager *wm, 
    struct window *window, 
    int direction);

// Find window under cursor
struct window *window_manager_find_window_below_cursor(struct window_manager *wm);

// Find window at point
struct window *window_manager_find_window_at_point(struct window_manager *wm, CGPoint point);

// Navigate managed windows
struct window *window_manager_find_prev_managed_window(sm, wm, window);
struct window *window_manager_find_next_managed_window(sm, wm, window);
struct window *window_manager_find_first_managed_window(sm, wm);
struct window *window_manager_find_last_managed_window(sm, wm);

// Navigate within stack
struct window *window_manager_find_prev_window_in_stack(sm, wm, window);
struct window *window_manager_find_next_window_in_stack(sm, wm, window);
```

## Event Handling

### Window Events (from AXObserver)

| Event | Handler |
|-------|---------|
| kAXWindowCreatedNotification | Create and add window |
| kAXUIElementDestroyedNotification | Remove window from tracking |
| kAXWindowMiniaturizedNotification | Set WINDOW_MINIMIZE flag |
| kAXWindowDeminiaturizedNotification | Clear WINDOW_MINIMIZE flag |
| kAXFocusedWindowChangedNotification | Update focused_window_id |
| kAXWindowMovedNotification | Update window frame cache |
| kAXWindowResizedNotification | Update window frame cache |

### Event Loop Integration (`src/event_loop.h`)

```c
// Window-related event types
WINDOW_CREATED,
WINDOW_DESTROYED,
WINDOW_FOCUSED,
WINDOW_MOVED,
WINDOW_RESIZED,
WINDOW_MINIMIZED,
WINDOW_DEMINIMIZED,
WINDOW_TITLE_CHANGED,
```

## Rules System

Users can define rules to automatically handle windows:

```c
struct rule
{
    char *label;
    regex_t app_regex;
    regex_t title_regex;
    regex_t role_regex;
    regex_t subrole_regex;
    bool app_regex_valid;
    bool title_regex_valid;
    bool role_regex_valid;
    bool subrole_regex_valid;
    bool one_shot;
    struct rule_effects effects;
};

struct rule_effects
{
    uint64_t sid;           // Send to space
    uint32_t did;           // Send to display
    float opacity;          // Set opacity
    int layer;              // Set layer
    uint32_t grid[6];       // Grid placement
    bool manage;            // Force manage/unmanage
    bool sticky;            // Make sticky
    bool mff;               // Mouse follows focus
    bool scratchpad;        // Make scratchpad
    // ... more effects
};
```

### Applying Rules

```c
void window_manager_apply_rules_to_window(
    struct space_manager *sm, 
    struct window_manager *wm, 
    struct window *window, 
    char *title, char *role, char *subrole, 
    bool one_shot_rules)
{
    for (int i = 0; i < buf_len(wm->rules); ++i) {
        struct rule *rule = &wm->rules[i];
        
        if (window_manager_rule_matches_window(rule, window, title, role, subrole)) {
            window_manager_apply_rule_effects_to_window(sm, wm, window, &rule->effects);
            
            if (rule->one_shot && one_shot_rules) {
                // Remove one-shot rule after matching
                buf_del(wm->rules, i);
                --i;
            }
        }
    }
}
```

## Initialization

```c
void window_manager_begin(struct space_manager *sm, struct window_manager *wm)
{
    // Get all running applications
    NSArray *applications = [[NSWorkspace sharedWorkspace] runningApplications];
    
    for (NSRunningApplication *app in applications) {
        if ([app activationPolicy] == NSApplicationActivationPolicyRegular) {
            pid_t pid = [app processIdentifier];
            
            // Create application object
            struct application *application = application_create(app);
            if (!application) continue;
            
            // Add to tracking
            window_manager_add_application(wm, application);
            
            // Set up observation
            if (application_observe(application)) {
                // Add existing windows
                window_manager_add_application_windows(sm, wm, application);
            }
        }
    }
    
    // Initial space validation
    uint64_t sid = space_manager_active_space();
    window_manager_validate_and_check_for_windows_on_space(sm, wm, sid);
}
```

## Key Functions Reference

### Window Lookup
- `window_manager_find_window(wm, window_id)` - Find by ID
- `window_manager_find_managed_window(wm, window)` - Get view for managed window
- `window_manager_focused_window(wm)` - Get currently focused window

### Window Operations
- `window_manager_move_window(window, x, y)` - Move window
- `window_manager_resize_window(window, w, h)` - Resize window
- `window_manager_set_window_frame(window, x, y, w, h)` - Set frame (with animation)
- `window_manager_swap_window(sm, wm, a, b)` - Swap two windows
- `window_manager_warp_window(sm, wm, a, b)` - Move window next to another
- `window_manager_stack_window(sm, wm, a, b)` - Stack window on another

### Window State
- `window_manager_make_window_floating(sm, wm, window, float, force)` - Toggle floating
- `window_manager_make_window_sticky(sm, wm, window, sticky)` - Toggle sticky
- `window_manager_set_opacity(wm, window, opacity)` - Set opacity
- `window_manager_set_window_layer(window, layer)` - Set layer

### Focus
- `window_manager_focus_window_with_raise(psn, wid, ref)` - Focus and raise
- `window_manager_focus_window_without_raise(psn, wid)` - Focus without raise

### BSP Tree
- `view_add_window_node(view, window)` - Add window to tree
- `view_remove_window_node(view, window)` - Remove window from tree
- `view_find_window_node(view, window_id)` - Find node for window
- `view_update(view)` - Recalculate all node areas
- `view_flush(view)` - Apply node areas to actual windows

## Integration with Scripting Addition

For operations requiring elevated privileges:

```c
// Window positioning (more reliable than AX)
scripting_addition_move_window(wid, x, y);

// Opacity control
scripting_addition_set_opacity(wid, opacity, duration);

// Layer control
scripting_addition_set_layer(wid, layer);

// Sticky (appear on all spaces)
scripting_addition_set_sticky(wid, sticky);

// Shadow control
scripting_addition_set_shadow(wid, shadow);

// Focus without raise
scripting_addition_focus_window(wid);

// Move to space
scripting_addition_move_window_to_space(sid, wid);
```
