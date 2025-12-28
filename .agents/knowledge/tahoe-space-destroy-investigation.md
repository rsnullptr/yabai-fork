# macOS 26 Tahoe - Space Destroy Investigation

## Issue Summary
- **GitHub Issue**: https://github.com/asmvik/yabai/issues/2730
- **Status**: UNRESOLVED - The supposed fix (commit e33e94c) is already applied but doesn't work on latest Tahoe
- **Symptom**: `yabai -m space --destroy` fails silently on macOS 26 Tahoe while `--create` works

## Investigation Date
2025-12-28

## Reference Links
- **GitHub Issue**: https://github.com/asmvik/yabai/issues/2730
- **Attempted Fix Commit**: https://github.com/asmvik/yabai/commit/e33e94c0b25f807891e0e70101122afaec796e02
- **Upstream Repo**: https://github.com/koekeishiya/yabai
- **Fork Repo**: https://github.com/asmvik/yabai

## Commit History Context
```
df5d671 (HEAD -> fix/destroy-space) knowledge based and space destroy
996e26d (origin/master) rename
e33e94c #2693 fix space destroy pattern for macOS 26.1 arm64   <-- This fix is already applied
9868ae3 v7.1.16
3861367 #2680 replace (autoreleasepool handling) CFRunLoopRunInMode...
```

## Technical Background

### How Space Operations Work
1. Yabai injects a scripting addition (OSAX) payload into Dock.app
2. The payload locates internal Dock functions by scanning for byte patterns
3. Pattern scanning starts at version-specific offsets
4. Functions are called via inline assembly with specific calling conventions

### Pattern Scanning Flow
```
arm64_payload.m: get_remove_space_pattern()  → Returns byte pattern
arm64_payload.m: get_remove_space_offset()   → Returns scan start offset
payload.m: do_space_destroy()                → Finds function, calls it
```

### Key Files
- `src/osax/payload.m` - Contains `do_space_destroy()` implementation (lines 517-594)
- `src/osax/arm64_payload.m` - Contains pattern definitions
- `src/osax/x64_payload.m` - x86_64 patterns (not relevant for M-series)

## Binary Analysis Results

### Tahoe Dock Binary Location
Extracted to: `/tmp/tahoe_dock/Dock_arm64e`
Source: `/Users/rs/Downloads/Dock.zip`

### Sequoia Dock Binary Location
Extracted to: `/tmp/Dock_arm64e`
Source: `/System/Volumes/Preboot/Cryptexes/App/System/Applications/Dock.app/Contents/MacOS/Dock`

### Pattern Scan Offsets (from get_remove_space_offset)
```c
if (os_version.majorVersion == 26) {
    return 0x1E0000;  // Tahoe starts scan here
} else if (os_version.majorVersion == 15) {
    return 0x100000;  // Sequoia
}
```

### Remove Space Pattern Analysis

#### Current Pattern (v26 - from e33e94c fix):
```c
return "7F 23 03 D5 FF ?? ?? D1 FC ?? ?? A9 FA ?? ?? A9 F8 ?? ?? A9 F6 ?? ?? A9 F4 ?? ?? A9 FD ?? ?? A9 FD ?? ?? 91 ?? 03 03 AA F5 03 02 AA F4 03 01 AA";
```

This is ARM64 function prologue:
- `7F 23 03 D5` = hint instruction (function start marker)
- `FF ?? ?? D1` = sub sp, sp, #imm (stack allocation)
- Multiple `F* ?? ?? A9` = stp x*, x*, [sp, #imm] (save callee-saved registers)
- `FD ?? ?? 91` = add fp, sp, #imm (frame pointer setup)
- `?? 03 03 AA` = mov xN, x3 (save 4th arg) - WILDCARDED since e33e94c
- `F5 03 02 AA` = mov x21, x2 (save 3rd arg)
- `F4 03 01 AA` = mov x20, x1 (save 2nd arg)

#### Register Difference Found
- **Sequoia**: Uses `F6 03 03 AA` = `mov x22, x3`
- **Tahoe**: Uses `F3 03 03 AA` = `mov x19, x3`

This difference IS handled by the current pattern (`??` wildcard).

## Disassembly Findings

### Tahoe Dock.app at 0x1f8100 (matching area)
```
001f8100: 7f 23 03 d5  hint instruction
001f8104: f8 5f bc a9  stp x24, x23, [sp, #-0x40]!
001f8108: f6 57 01 a9  stp x22, x21, [sp, #0x10]
001f810c: f4 4f 02 a9  stp x20, x19, [sp, #0x20]
001f8110: fd 7b 03 a9  stp fp, lr, [sp, #0x30]
001f8114: fd c3 00 91  add fp, sp, #0x30
001f8118: f3 03 03 aa  mov x19, x3     <-- Uses x19, not x22
001f811c: f5 03 02 aa  mov x21, x2
001f8120: f4 03 01 aa  mov x20, x1
...
```

### Search Results in Tahoe Dock
Using pattern fragment `f?03 03aa f503 02aa f403 01aa`:
- Match at 0x119100: `f603 03aa` (old style with x22)
- Match at 0x1f8120: `f303 03aa` (new style with x19) - IN SCAN RANGE
- Match at 0x24d2b0: `f303 03aa`

The pattern at 0x1f8120 is within the scan range starting at 0x1E0000.

## Potential Root Causes (Since Fix Already Applied)

### 1. Pattern Match but Wrong Function
The pattern might be matching the wrong function entirely. The disassembly at 0x1f8100 shows a different function prologue structure than expected.

### 2. Function Signature Changed
The `removeSpace:` function may have changed parameters or calling convention in the latest Tahoe beta.

### 3. Offset Issue
The scan offset (0x1E0000) might need adjustment for newer Tahoe builds.

### 4. Additional Pattern Bytes Needed
The pattern might need more specificity to avoid matching similar prologues.

### 5. Runtime Environment Changes
The Dock's internal state management may have changed, causing the function to fail even when found.

## Code Flow for Space Destroy

### Client Side (space_manager.c)
```c
enum space_op_error space_manager_destroy_space(uint64_t sid)
{
    // Validates: not in MC, space exists, is user space, not last space
    bool success = scripting_addition_destroy_space(sid);
    if (!success) return SPACE_OP_ERROR_SCRIPTING_ADDITION;
    // ...
}
```

### Scripting Addition (sa.m)
```c
bool scripting_addition_destroy_space(uint64_t sid)
{
    // Packs message with SA_OPCODE_SPACE_DESTROY
    // Sends via Unix socket to payload in Dock
}
```

### Payload (payload.m, lines 517-594)
```c
static void do_space_destroy(char *message)
{
    uint64_t sid;
    unpack(sid);
    
    // Get display space info from Dock's internal structures
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

### Assembly Calling Convention (arm64_payload.m)
```c
static void asm__call_remove_space(id source_space, id space, void *fn)
{
    // Uses standard ARM64 calling convention
    // x0 = source_space
    // x1 = space  
    // x2 = third param (varies by version)
    // x3 = fourth param (varies by version)
}
```

## Next Investigation Steps

### 1. Verify Pattern is Actually Matching
Add debug logging to `payload.m` to print:
- The address found by pattern scan
- Whether `remove_space_fp` is non-NULL after initialization

### 2. Disassemble More of Tahoe's removeSpace
Look at the full function body to verify it's the correct function:
```bash
# Disassemble starting from match address
objdump -d --start-address=0x1f8100 --stop-address=0x1f8200 /tmp/tahoe_dock/Dock_arm64e
```

### 3. Compare with add_space Pattern
`add_space` works on Tahoe - compare its pattern matching and function signature to understand the differences.

### 4. Check Dock Instance Variables
The payload accesses Dock internal state via:
- `_currentDisplaySpace`
- `_currentSpace`
- `_monitoredSpaces`

These may have changed in Tahoe.

### 5. Test with Debug Build
Build yabai with additional logging in payload.m to trace the exact failure point.

## Environment Notes
- Test machine: macOS 15.7.4 (Sequoia) - cannot reproduce issue
- Tahoe test required on separate machine or VM
- Tahoe Dock binary available at `/tmp/tahoe_dock/Dock_arm64e`

## Related Code References
- `src/osax/arm64_payload.m:191-205` - get_remove_space_pattern()
- `src/osax/arm64_payload.m:85-94` - get_remove_space_offset()
- `src/osax/payload.m:517-594` - do_space_destroy()
- `src/osax/payload.m:230-280` - asm__call_remove_space()
- `src/space_manager.c` - space_manager_destroy_space()

## User Report Details
From GitHub issue #2730:
- User on macOS 26 (Tahoe beta)
- `space --create` works correctly
- `space --destroy` fails silently (no error, no action)
- Other yabai functionality works

---

## CRITICAL FINDING (2025-12-28)

**The pattern does NOT match the Tahoe binary!**

The current pattern expects this prologue structure:
```
7F 23 03 D5  - hint
FF ?? ?? D1  - sub sp, sp, #imm (explicit stack allocation)
FC ?? ?? A9  - stp x28, x27, [sp, #imm]
FA ?? ?? A9  - stp x26, x25, [sp, #imm]
F8 ?? ?? A9  - stp x24, x23, [sp, #imm]
F6 ?? ?? A9  - stp x22, x21, [sp, #imm]
F4 ?? ?? A9  - stp x20, x19, [sp, #imm]
FD ?? ?? A9  - stp fp, lr, [sp, #imm]
FD ?? ?? 91  - add fp, sp, #imm
```

But the Tahoe function at 0x1f8108 uses:
```
7f23 03d5  - hint
f85f bca9  - stp x24, x23, [sp, #-0x40]!  <-- PRE-INDEXED STORE (combines sub+stp)
f657 01a9  - stp x22, x21, [sp, #0x10]
f44f 02a9  - stp x20, x19, [sp, #0x20]
fd7b 03a9  - stp fp, lr, [sp, #0x30]
fdc3 0091  - add fp, sp, #0x30
f303 03aa  - mov x19, x3
f503 02aa  - mov x21, x2
f403 01aa  - mov x20, x1
```

**Key Difference**: Tahoe's compiler uses **pre-indexed store pair** (`stp rX, rY, [sp, #-imm]!`)
instead of a separate `sub sp, sp, #imm` instruction. This is an optimization where the
stack allocation and first register save are combined into a single instruction.

### Instruction Encoding Difference:
- Old pattern: `FF ?? ?? D1` = `sub sp, sp, #imm`
- Tahoe actual: `F8 5F BC A9` = `stp x24, x23, [sp, #-0x40]!` (pre-indexed)

The `A9` suffix indicates a store pair instruction, while `D1` indicates a subtract.
The pattern is fundamentally incompatible.

### Proposed Fix
Need to create a new pattern for Tahoe that matches:
```c
// Tahoe pattern with pre-indexed stp
return "7F 23 03 D5 F8 5F ?? A9 F6 57 ?? A9 F4 4F ?? A9 FD 7B ?? A9 FD ?? ?? 91 ?? 03 03 AA F5 03 02 AA F4 03 01 AA";
```

Or use a more flexible pattern that only matches the end sequence after frame setup.

## Reference Materials

### Binary Locations Used in Analysis
- **Tahoe Dock Binary**: `/tmp/tahoe_dock/Dock_arm64e` (extracted from `/Users/rs/Downloads/Dock.zip`)
- **Sequoia Dock Binary**: `/tmp/Dock_arm64e` (extracted from system Dock.app)
- **System Dock Path**: `/System/Volumes/Preboot/Cryptexes/App/System/Applications/Dock.app/Contents/MacOS/Dock`

### ARM64 Instruction Reference
- ARM64 Store Pair (STP): https://developer.arm.com/documentation/dui0802/latest/A64-Data-Transfer-Instructions/STP
- ARM64 Addressing Modes: https://developer.arm.com/documentation/dui0801/latest/A64-Data-Transfer-Instructions/Load-Store-addressing-modes
- Pre-indexed addressing: `[base, #offset]!` - updates base register after access

### Useful Commands for Binary Analysis
```bash
# Extract arm64e slice from universal binary
lipo -thin arm64e /path/to/Dock -output /tmp/Dock_arm64e

# Search for byte patterns
xxd /tmp/Dock_arm64e | grep -E "pattern"

# Disassemble specific address range
objdump -d --start-address=0xADDR --stop-address=0xADDR+0x100 /tmp/Dock_arm64e

# Search for instruction sequences (example: mov xN, x3 followed by mov x21, x2)
xxd /tmp/Dock_arm64e | grep -E "03aa f503 02aa"
```

### Pattern Scan Code Locations
- Pattern definitions: `src/osax/arm64_payload.m` (lines 175-225)
- Scan offsets: `src/osax/arm64_payload.m` (lines 85-101)
- Pattern scanner: `src/osax/common_payload.m` - `find_pattern()` function
- Space destroy implementation: `src/osax/payload.m` (lines 517-594)
