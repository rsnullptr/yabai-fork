# Dock Code Injection (OSAX) - Technical Documentation

This document explains how yabai injects code into macOS's Dock.app process to gain access to private APIs for advanced window and space management.

## Overview

Yabai uses a **Mach-based code injection system** to inject a payload into the `Dock.app` process. This allows yabai to access private macOS APIs and manipulate windows/spaces in ways that would otherwise be impossible through public APIs.

The system is structured as a macOS "Scripting Addition" (.osax) - a legacy mechanism that allows code to be loaded into other processes.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              YABAI MAIN BINARY                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Embedded Binaries (compiled into yabai via xxd -i):                 │   │
│  │    - __src_osax_loader (loader binary)                               │   │
│  │    - __src_osax_payload (payload dylib)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                    scripting_addition_install()                             │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  /Library/ScriptingAdditions/yabai.osax/                            │   │
│  │    Contents/                                                         │   │
│  │      MacOS/loader              ← Mach injection binary               │   │
│  │      Resources/payload.bundle/                                       │   │
│  │        Contents/MacOS/payload  ← Dylib loaded into Dock.app         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    mach_loader_inject_payload()
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DOCK.APP PROCESS                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Injected payload.bundle (via dlopen from shellcode):                │   │
│  │    - Creates Unix socket at /tmp/yabai-sa_<USER>.socket             │   │
│  │    - Listens for commands from yabai                                 │   │
│  │    - Has access to Dock's internal objects & private SkyLight APIs   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │
                         Unix Socket IPC
                                    │
┌─────────────────────────────────────────────────────────────────────────────┐
│                              YABAI (CLIENT)                                  │
│  scripting_addition_*() functions send commands via socket                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Key Source Files

| File | Purpose |
|------|---------|
| `src/osax/common.h` | Shared protocol definitions, opcodes, version |
| `src/osax/loader.m` | Mach injection binary that injects payload into Dock |
| `src/osax/payload.m` | Main payload code running inside Dock.app |
| `src/osax/arm64_payload.m` | arm64-specific patterns and assembly macros |
| `src/osax/x64_payload.m` | x86_64-specific patterns and assembly macros |
| `src/sa.h` | Client-side API header |
| `src/sa.m` | Client-side implementation (yabai to payload communication) |

## Protocol Definition (common.h)

### Version and Capability Flags

```c
#define OSAX_VERSION "2.1.23"

// Capability flags returned during handshake
#define OSAX_ATTRIB_DOCK_SPACES     0x01  // Can access dock_spaces object
#define OSAX_ATTRIB_DPPM            0x02  // Has dp_desktop_picture_manager
#define OSAX_ATTRIB_ADD_SPACE       0x04  // Can add spaces
#define OSAX_ATTRIB_REM_SPACE       0x08  // Can remove spaces
#define OSAX_ATTRIB_MOV_SPACE       0x10  // Can move spaces
#define OSAX_ATTRIB_SET_WINDOW      0x20  // Can focus windows via private API
#define OSAX_ATTRIB_ANIM_TIME       0x40  // Animation time patch applied
```

### Command Opcodes

```c
enum sa_opcode {
    SA_OPCODE_HANDSHAKE             = 0x01,
    SA_OPCODE_SPACE_FOCUS           = 0x02,
    SA_OPCODE_SPACE_CREATE          = 0x03,
    SA_OPCODE_SPACE_DESTROY         = 0x04,
    SA_OPCODE_SPACE_MOVE            = 0x05,
    SA_OPCODE_WINDOW_MOVE           = 0x06,
    SA_OPCODE_WINDOW_OPACITY        = 0x07,
    SA_OPCODE_WINDOW_OPACITY_FADE   = 0x08,
    SA_OPCODE_WINDOW_LAYER          = 0x09,
    SA_OPCODE_WINDOW_STICKY         = 0x0A,
    SA_OPCODE_WINDOW_SHADOW         = 0x0B,
    SA_OPCODE_WINDOW_FOCUS          = 0x0C,
    SA_OPCODE_WINDOW_SCALE          = 0x0D,
    SA_OPCODE_WINDOW_SWAP_PROXY     = 0x0E,
    SA_OPCODE_WINDOW_ORDER          = 0x0F,
    SA_OPCODE_WINDOW_TO_SPACE       = 0x10,
};
```

## Loader (loader.m)

The loader is a standalone binary that performs Mach-based code injection into Dock.app.

### Key Functions

1. **`get_dock_pid()`** - Finds Dock.app's PID using `NSRunningApplication`

2. **`shell_code[]`** - Architecture-specific shellcode that:
   - Creates a new thread in the remote process via `pthread_create_from_mach_thread`
   - Calls `dlopen()` to load the payload bundle

### Injection Sequence (main function)

```c
// 1. Get Dock's task port (requires SIP disabled)
task_for_pid(mach_task_self(), pid, &task);

// 2. Allocate stack in remote process
mach_vm_allocate(task, &stack, stack_size, VM_FLAGS_ANYWHERE);

// 3. Allocate code segment for shellcode
mach_vm_allocate(task, &code, sizeof(shell_code), VM_FLAGS_ANYWHERE);

// 4. Patch shellcode with actual addresses
// - pthread_create_from_mach_thread address
// - dlopen address  
// - payload path string
memcpy(shell_code + offset, &pcfmt_address, sizeof(uint64_t));
memcpy(shell_code + offset, &dlopen_address, sizeof(uint64_t));
memcpy(shell_code + offset, payload_path, strlen(payload_path));

// 5. Write shellcode to remote process
mach_vm_write(task, code, (vm_address_t)shell_code, sizeof(shell_code));

// 6. Make code executable
vm_protect(task, code, sizeof(shell_code), 0, VM_PROT_EXECUTE | VM_PROT_READ);

// 7. Create and run thread with shellcode as entry point
// x86_64: thread_create_running()
// arm64:  thread_create() + thread_convert_thread_state() + thread_resume()

// 8. Poll for completion (checks for magic return value 0x79616265 = "yabe")
for (int i = 0; i < 10; ++i) {
    thread_get_state(thread, ...);
    if (thread_state.__rax == 0x79616265) // success
    usleep(20000);
}

// 9. Terminate injection thread
thread_terminate(thread);
```

### arm64 Special Handling

- Uses `ptrauth_sign_unauthenticated()` for PAC (Pointer Authentication) on Apple Silicon
- Uses `thread_convert_thread_state()` for proper thread state handling
- Different code paths for macOS 14.4+/15.0+ vs earlier versions

## Payload (payload.m)

This shared library runs inside Dock.app and provides the core functionality.

### Entry Point (Constructor)

```c
__attribute__((constructor))
void load_payload(void) {
    const char *user = getenv("USER");
    char socket_file[255];
    snprintf(socket_file, sizeof(socket_file), "/tmp/yabai-sa_%s.socket", user);
    start_daemon(socket_file);
}
```

### Initialization - Pattern Scanning

The `init_instances()` function uses pattern scanning to locate internal Dock.app objects:

| Object/Function | Purpose |
|----------------|---------|
| `dock_spaces` | Global object managing spaces |
| `dp_desktop_picture_manager` | Desktop picture manager (for space moves) |
| `add_space_fp` | Function pointer to internal addSpace |
| `remove_space_fp` | Function pointer to internal removeSpace |
| `move_space_fp` | Function pointer to internal moveSpace |
| `set_front_window_fp` | Function pointer to set front window |
| `animation_time_addr` | Address to patch space-switching animation |

### Animation Patch

Patches instruction to disable space transition animation:

```c
if (vm_protect(mach_task_self(), page_align(animation_time_addr), 
               vm_page_size, 0, VM_PROT_READ | VM_PROT_WRITE | VM_PROT_COPY) == KERN_SUCCESS) {
#ifdef __x86_64__
    *(uint64_t *) animation_time_addr = 0x660fefc0660fefc0;  // XOR xmm0,xmm0 x2
#elif __arm64__
    *(uint32_t *) animation_time_addr = 0x2f00e400;  // movi d0, #0
#endif
}
```

### Command Handler

```c
static void handle_message(int sockfd, char *message) {
    enum sa_opcode op = *message++;
    switch (op) {
    case SA_OPCODE_HANDSHAKE:      do_handshake(sockfd); break;
    case SA_OPCODE_SPACE_FOCUS:    do_space_focus(message); break;
    case SA_OPCODE_SPACE_CREATE:   do_space_create(message); break;
    case SA_OPCODE_SPACE_DESTROY:  do_space_destroy(message); break;
    case SA_OPCODE_SPACE_MOVE:     do_space_move(message); break;
    case SA_OPCODE_WINDOW_MOVE:    do_window_move(message); break;
    case SA_OPCODE_WINDOW_OPACITY: do_window_opacity(message); break;
    // ... etc
    }
}
```

### Private SkyLight APIs Used

```c
extern int SLSMainConnectionID(void);
extern CGError SLSSetWindowAlpha(int cid, uint32_t wid, float alpha);
extern OSStatus SLSMoveWindowWithGroup(int cid, uint32_t wid, CGPoint *point);
extern void SLSManagedDisplaySetCurrentSpace(int cid, CFStringRef display_ref, uint64_t sid);
extern void SLSMoveWindowsToManagedSpace(int cid, CFArrayRef window_list, uint64_t sid);
extern void SLSShowSpaces(int cid, CFArrayRef space_list);
extern void SLSHideSpaces(int cid, CFArrayRef space_list);
```

## Architecture-Specific Patterns (arm64_payload.m / x64_payload.m)

These files contain version-specific byte patterns for each macOS version:

```c
// Example from arm64_payload.m
const char *get_dock_spaces_pattern(NSOperatingSystemVersion os_version) {
    if (os_version.majorVersion == 15) {
        return "?? 12 00 ?? ?? ?? ?? 91 ?? 02 40 F9 ?? ?? 00 B4 ?? ?? ?? ??";
    } else if (os_version.majorVersion == 14) {
        return "36 16 00 ?? D6 ?? ?? 91 ?? 02 40 F9 ?? ?? 00 B4 ?? 03 14 AA";
    }
    // ... patterns for each macOS version
}
```

### Inline Assembly for Calling Dock Functions

```c
// arm64
#define asm__call_add_space(v0,v1,func) \
    __asm__("mov x0, %0\n""mov x20, %1\n" : :"r"(v0), "r"(v1) :"x0", "x20"); \
    ((void (*)())(func))();

// x86_64
#define asm__call_add_space(v0,v1,func) \
    __asm__("movq %0, %%rdi;""movq %1, %%r13;""callq *%2;" \
            : :"r"(v0), "r"(v1), "r"(func) :"%rdi", "%r13");
```

## Client-Side API (sa.h / sa.m)

### Key Functions

```c
// Installation/Loading
int scripting_addition_load(void);      // Install & inject payload
int scripting_addition_uninstall(void); // Remove .osax

// Space Operations
bool scripting_addition_focus_space(uint64_t sid);
bool scripting_addition_create_space(uint64_t sid);
bool scripting_addition_destroy_space(uint64_t sid);
bool scripting_addition_move_space_to_display(...);

// Window Operations
bool scripting_addition_move_window(uint32_t wid, int x, int y);
bool scripting_addition_set_opacity(uint32_t wid, float opacity, float duration);
bool scripting_addition_set_layer(uint32_t wid, int layer);
bool scripting_addition_set_sticky(uint32_t wid, bool sticky);
bool scripting_addition_set_shadow(uint32_t wid, bool shadow);
bool scripting_addition_focus_window(uint32_t wid);
bool scripting_addition_move_window_to_space(uint64_t sid, uint32_t wid);
```

### Message Protocol

```c
// Message format: [length:2][opcode:1][data:variable]
#define sa_payload_init() char bytes[0x1000]; int16_t length = 1+sizeof(length)
#define pack(v) memcpy(bytes+length, &v, sizeof(v)); length += sizeof(v)
#define sa_payload_send(op) *(int16_t*)bytes = length-sizeof(length), \
                            bytes[sizeof(length)] = op, \
                            scripting_addition_send_bytes(bytes, length)

// Example usage:
bool scripting_addition_focus_space(uint64_t sid) {
    sa_payload_init();
    pack(sid);
    return sa_payload_send(SA_OPCODE_SPACE_FOCUS);
}
```

## Build Process (makefile)

The build embeds loader and payload binaries into yabai:

```makefile
$(OSAX_SRC): $(OSAX_PATH)/loader.m $(OSAX_PATH)/payload.m
    # 1. Compile payload as shared library (Universal Binary, arm64e for PAC)
    xcrun clang $(OSAX_PATH)/payload.m -shared -fPIC -O3 \
        -mmacosx-version-min=11.0 -arch x86_64 -arch arm64e \
        -o $(OSAX_PATH)/payload \
        -framework SkyLight -framework Foundation -framework Carbon
    
    # 2. Compile loader
    xcrun clang $(OSAX_PATH)/loader.m -O3 \
        -mmacosx-version-min=11.0 -arch x86_64 -arch arm64e \
        -o $(OSAX_PATH)/loader -framework Cocoa
    
    # 3. Convert to C arrays using xxd
    xxd -i -a $(OSAX_PATH)/payload $(OSAX_PATH)/payload_bin.c
    xxd -i -a $(OSAX_PATH)/loader $(OSAX_PATH)/loader_bin.c
```

## Complete Injection Flow

### Phase 1: Installation (`scripting_addition_load()`)

1. **SIP Check**: Verify CSR_ALLOW_UNRESTRICTED_FS and CSR_ALLOW_TASK_FOR_PID are disabled
2. **arm64 Check**: Verify boot-args include `-arm64e_preview_abi`
3. **Create .osax Bundle**: Write embedded binaries to `/Library/ScriptingAdditions/yabai.osax/`
4. **Sign Binaries**: Ad-hoc codesign both loader and payload
5. **Restart Dock**: Kill Dock.app so it restarts fresh

### Phase 2: Injection (`mach_loader_inject_payload()`)

1. Execute loader via `popen()`
2. Loader finds Dock PID via `NSRunningApplication`
3. Get Dock's task port via `task_for_pid()`
4. Allocate remote memory for stack and shellcode
5. Inject patched shellcode with dlopen address and payload path
6. Create remote thread at shellcode address
7. Shellcode creates pthread and calls `dlopen(payload_path)`
8. Poll for magic return value `0x79616265` ("yabe")
9. Terminate injection thread

### Phase 3: Payload Initialization

1. Constructor `load_payload()` starts socket server
2. Pattern scan Dock binary for internal objects/functions
3. Patch animation duration instruction
4. Listen for commands on `/tmp/yabai-sa_<USER>.socket`

### Phase 4: Runtime Communication

1. Yabai packs command (opcode + parameters) and sends via socket
2. Payload receives and dispatches to handler
3. Handler uses private APIs/internal Dock functions
4. Response sent back to yabai

## Capabilities Provided

### Space Operations (not possible via public APIs)
- Focus space without animation
- Create new desktops programmatically
- Destroy desktops
- Move spaces between displays

### Window Operations (enhanced control)
- Pixel-perfect window positioning
- Window transparency with fade animations
- Z-order (layer) control
- Sticky windows (appear on all spaces)
- Shadow control
- Programmatic window focus
- Move windows to specific spaces

## Security Requirements

### Required SIP Flags (must be disabled)
```c
#define CSR_ALLOW_UNRESTRICTED_FS 0x02  // Write to /Library/ScriptingAdditions
#define CSR_ALLOW_TASK_FOR_PID    0x04  // Get task port for Dock.app
```

### Apple Silicon Additional Requirements
- Boot-arg: `-arm64e_preview_abi`
- All function pointers must be PAC-signed

### Privilege Requirements
- Root access required for installation and injection
- Socket file has `chmod 0600` for user-only access

## Maintenance for New macOS Versions

When a new macOS version is released:

1. Reverse engineer Dock.app using Ghidra/Hopper
2. Find new byte patterns for internal functions/objects
3. Update offset functions in `arm64_payload.m` and `x64_payload.m`
4. Update pattern functions with new byte sequences
5. Test all features on the new version
6. Update `OSAX_VERSION` in `common.h`

## Debugging Tips

1. Check Console.app for `[yabai-sa]` log messages
2. Verify SIP status: `csrutil status`
3. Check socket exists: `ls -la /tmp/yabai-sa_*.socket`
4. Test handshake - the attrib bitmap shows what features initialized
5. Pattern failures show in console logs

## Key Implementation Patterns

### Pattern Scanning
```c
static uint64_t hex_find_seq(uint64_t baddr, const char *c_pattern) {
    // Pattern like "7F 23 03 D5 FF C3 01 D1 ?? ?? 00 94"
    // '?' = wildcard byte (matches anything)
    // Scans from baddr within search range
    // Returns address of match or 0
}
```

### Address Decoding (arm64)
```c
uint64_t decode_adrp_add(uint64_t addr, uint64_t offset) {
    // Decode ADRP+ADD instruction pair to get actual address
    // Extracts immediate from ADRP and ADD instructions
    // Combines with page-aligned base
}
```

### Objective-C Runtime Introspection
```c
static inline id get_ivar_value(id instance, const char *name) {
    id result = nil;
    object_getInstanceVariable(instance, name, (void **) &result);
    return result;
}
```
