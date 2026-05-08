# App Sync Patch — Enable ADB automatically over USB for HiBy R3 Pro II
# Also increase binary size for future patches

**Binary:** `hiby_player` ELF32 MIPS32r2 Little Endian (Ingenic X1600, HiByOS)
**Binary SHA256:** `1b4fda0683d9bec2c3c67e369649b3afdd0ea1269d70614b0589aad1d0f79d29`
**Original binary size:** 5,598,244 bytes
**Patched binary size:** 10,485,760 bytes (expanded to 10 MiB for future patches)

---

## Overview

This patch adds an **App Sync** toggle to the device's system settings. When enabled and the device is connected via USB, ADB starts automatically instead of mass storage. A custom view page (`hiby_app_sync.view`) is displayed during sync, with a stop button to disable ADB and return to normal operation.

---

## User-Facing Behavior

**Toggle ON:** creates the marker file `/usr/data/adb_usb_mode`. ADB does not start until USB is connected.

**Toggle OFF:** deletes the marker file. If ADB is currently running, it is stopped immediately and the App Sync page is closed.

**USB connect with toggle ON:** mass storage initializes normally, then `FUN_00480f20(1)` is called (the firmware's own ADB activation function, same as the 10-tap About page toggle), which stops mass storage and starts ADB. The `hiby_app_sync.view` page is shown instead of the standard USB page.

**USB connect with toggle OFF:** normal mass storage.

**USB disconnect with toggle ON:** the App Sync page stays visible (teardown is suppressed). This prevents any action done on the device, e.g. playing music, while the device is in ADB mode to prevent unwanted behaviors.

**Stop button on App Sync page:** removes the marker file, clears the runtime flag, calls `FUN_00480f20(0)` to stop ADB and restore mass storage, then closes the page.

**Reboot:** at boot, the early init reads the marker file and sets the runtime flag for the toggle display. ADB is **not** started at boot — it only starts when USB is physically connected.

---

## Patch Architecture

The patch consists of two groups: the **settings toggle** (adds the App Sync entry to the Settings page with a working on/off switch) and the **USB integration** (handles ADB activation on USB connect, the custom view page, and the stop button).

### Code Caves Used

| Region | Address Range | Size | Contents |
|--------|--------------|------|----------|
| Sorting cave extension | `0x41c120` – `0x41c420` | 768 B | View trampoline, ADB gate, marker helper, teardown gate, stop callback, control table |
| `.rodata` zero region | `0x7f2d4c` – `0x7f2f78` | 556 B | Click handler, NL trampolines, early init, toggle wrapper, strings |

### Runtime Variables

| Address | Segment | Size | Purpose |
|---------|---------|------|---------|
| `0x9B6438` | BSS | 1 byte | Runtime flag: `1` = App Sync enabled, `0` = disabled |

### Marker File

`/usr/data/adb_usb_mode` — presence indicates App Sync is enabled. Created/deleted by the click handler. Checked at boot (early init) and by the ADB auto-start gate on USB connect.

---

## Part 1: Settings Toggle

### Dispatch Table

A new entry is added at slot 36 (address `0x937434`), with dispatch ID `0x24` and name `app_sync`.

| Address | Size | Description |
|---------|------|-------------|
| `0x937434` | 68 B | Dispatch entry: `{0x24, "app_sync\0", padding}` |

### Loop Limits

Two loop limit constants are incremented by 1 to include the new entry.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x4e0b38` | `addiu $s3, $zero, 0x24` | `addiu $s3, $zero, 0x25` | Dispatch lookup loop: 36 → 37 |
| `0x4e6bec` | `addiu $s3, $zero, 0x18` | `addiu $s3, $zero, 0x19` | Name list loop: 24 → 25 |

### Name List Trampolines

Two hook points add `"app_sync"` to the settings name list. Each replaces two `lui` instructions with a jump to a trampoline that stores the string pointer, then re-executes the replaced instructions and returns.

| Address | Original | Patched | Target |
|---------|----------|---------|--------|
| `0x4e6cf8` + `0x4e6cfc` | `lui $s0, 0x88` + `lui $v0, 0x82` | `j 0x7f2e20` + `nop` | Non-RS2 trampoline |
| `0x4e6b48` + `0x4e6b4c` | `lui $s0, 0x88` + `lui $v0, 0x82` | `j 0x7f2e40` + `nop` | RS2 trampoline |

**Non-RS2 trampoline** at `0x7f2e20` (28 B): stores `"app_sync"` pointer at `$sp+0x4c`, restores `$s0` and `$v0`, jumps to `0x4e6d00`.

```
0x007f2e20:  3c02007f  lui $v0, 0x007f
0x007f2e24:  24422f00  addiu $v0, $v0, 0x2f00          # $v0 = 0x7f2f00 ("app_sync")
0x007f2e28:  afa2004c  sw $v0, 0x4c($sp)
0x007f2e2c:  3c100088  lui $s0, 0x0088                  # replaced instruction
0x007f2e30:  3c020082  lui $v0, 0x0082                  # replaced instruction
0x007f2e34:  08139b40  j 0x004e6d00
0x007f2e38:  00000000  nop
```

**RS2 trampoline** at `0x7f2e40` (28 B): same logic, stores at `$sp+0x44`, jumps to `0x4e6b50`.

```
0x007f2e40:  3c02007f  lui $v0, 0x007f
0x007f2e44:  24422f00  addiu $v0, $v0, 0x2f00          # $v0 = 0x7f2f00 ("app_sync")
0x007f2e48:  afa20044  sw $v0, 0x44($sp)
0x007f2e4c:  3c100088  lui $s0, 0x0088                  # replaced instruction
0x007f2e50:  3c020082  lui $v0, 0x0082                  # replaced instruction
0x007f2e54:  08139ad4  j 0x004e6b50
0x007f2e58:  00000000  nop
```

> Both RS2 and non-RS2 functions were patched as I was not sure if the binary uses them together to populate the settings page.

### Click Handler

Replaces the dispatch call at `0x4e14c0`.

| Address | Original | Patched |
|---------|----------|---------|
| `0x4e14c0` | `jal 0x4e0ae0` | `jal 0x7f2d4c` |

**Click handler** at `0x7f2d4c` (188 B):

1. Calls `FUN_004e0ae0` to get the dispatch ID for the clicked item
2. If dispatch ID ≠ `0x24`: returns the ID unchanged (other items handled normally)
3. If dispatch ID = `0x24` (App Sync):
   - Toggles the BSS flag at `0x9B6438` (XOR with 1)
   - If turning **ON**: `system("touch /usr/data/adb_usb_mode")`
   - If turning **OFF**:
     - `system("rm -f /usr/data/adb_usb_mode")`
     - `FUN_00480f20(0)` — stops ADB immediately
     - `FUN_0048a380(1)` — closes the App Sync page if open
     - Re-clears BSS flag at `0x9B6438` to 0 (safety measure)
   - Calls `FUN_004af060($s2, "vg_listview_sys_set")` to refresh the list display
   - Returns `-1` (signals the dispatcher that the click was handled)

```
0x007f2d4c:  27bdffe0  addiu $sp, $sp, -32
0x007f2d50:  afbf001c  sw $ra, 28($sp)
0x007f2d54:  afb00018  sw $s0, 24($sp)
0x007f2d58:  afb10014  sw $s1, 20($sp)
0x007f2d5c:  afb20010  sw $s2, 16($sp)
0x007f2d60:  0c1382b8  jal 0x004e0ae0                   # get dispatch ID
0x007f2d64:  00000000  nop
0x007f2d68:  00408021  addu $s0, $v0, $zero             # $s0 = dispatch ID
0x007f2d6c:  24110024  addiu $s1, $zero, 0x24
0x007f2d70:  1611001e  bne $s0, $s1, 0x7f2dec           # if ID != 0x24, goto epilogue_normal
0x007f2d74:  00000000  nop
# --- App Sync item clicked ---
0x007f2d78:  3c08009b  lui $t0, 0x009b
0x007f2d7c:  25086438  addiu $t0, $t0, 0x6438           # $t0 = 0x9b6438 (BSS flag)
0x007f2d80:  91090000  lbu $t1, 0($t0)
0x007f2d84:  39290001  xori $t1, $t1, 0x0001            # toggle flag
0x007f2d88:  a1090000  sb $t1, 0($t0)
0x007f2d8c:  11200006  beq $t1, $zero, 0x7f2da8         # if turning OFF, goto rm_and_stop
0x007f2d90:  00000000  nop
# --- Turning ON: touch marker ---
0x007f2d94:  3c04007f  lui $a0, 0x007f
0x007f2d98:  0c239c24  jal 0x008e7090                   # system()
0x007f2d9c:  24842f24  addiu $a0, $a0, 0x2f24           # "touch /usr/data/adb_usb_mode"
0x007f2da0:  1000000b  b 0x7f2dd0                       # goto refresh
0x007f2da4:  00000000  nop
# --- rm_and_stop: Turning OFF ---
0x007f2da8:  3c04007f  lui $a0, 0x007f
0x007f2dac:  0c239c24  jal 0x008e7090                   # system()
0x007f2db0:  24842f44  addiu $a0, $a0, 0x2f44           # "rm -f /usr/data/adb_usb_mode"
0x007f2db4:  0c1203c8  jal 0x00480f20                   # FUN_00480f20(0) — stop ADB
0x007f2db8:  00002021  addu $a0, $zero, $zero            # $a0 = 0
0x007f2dbc:  0c1228e0  jal 0x0048a380                   # FUN_0048a380(1) — close page
0x007f2dc0:  24040001  addiu $a0, $zero, 1               # $a0 = 1
0x007f2dc4:  3c08009b  lui $t0, 0x009b
0x007f2dc8:  25086438  addiu $t0, $t0, 0x6438
0x007f2dcc:  a1000000  sb $zero, 0($t0)                  # re-clear BSS flag
# --- refresh ---
0x007f2dd0:  3c05007f  lui $a1, 0x007f
0x007f2dd4:  24a52f64  addiu $a1, $a1, 0x2f64           # "vg_listview_sys_set"
0x007f2dd8:  0c12bc18  jal 0x004af060                   # FUN_004af060 — refresh list
0x007f2ddc:  02402021  addu $a0, $s2, $zero             # $a0 = $s2 (thread ptr)
0x007f2de0:  2402ffff  addiu $v0, $zero, -1              # return -1
0x007f2de4:  10000001  b 0x7f2dec                        # goto epilogue
0x007f2de8:  00000000  nop
# --- epilogue_normal ---
0x007f2dec:  02001021  addu $v0, $s0, $zero             # return original dispatch ID
# --- epilogue ---
0x007f2df0:  8fbf001c  lw $ra, 28($sp)
0x007f2df4:  8fb00018  lw $s0, 24($sp)
0x007f2df8:  8fb10014  lw $s1, 20($sp)
0x007f2dfc:  8fb20010  lw $s2, 16($sp)
0x007f2e00:  03e00008  jr $ra
0x007f2e04:  27bd0020  addiu $sp, $sp, 32
```

### Toggle Display Wrapper

Replaces both calls to `FUN_004e0be0` in the display callback, providing toggle state for App Sync without modifying the toggle state table or any struct fields.

| Address | Original | Patched |
|---------|----------|---------|
| `0x4e0eb4` | `jal 0x4e0be0` | `jal 0x7f2ea0` |
| `0x4e0ee4` | `jal 0x4e0be0` | `jal 0x7f2ea0` |

**Toggle wrapper** at `0x7f2ea0` (88 B):

1. Calls original `FUN_004e0be0` with the same argument
2. If result ≥ 0: returns it (existing toggle item)
3. If result = -1 (not a known toggle): calls `FUN_004e0ae0` to get dispatch ID
4. If dispatch ID = `0x24`: returns BSS flag value (0 or 1)
5. Otherwise: returns -1

```
0x007f2ea0:  27bdffe8  addiu $sp, $sp, -24
0x007f2ea4:  afbf0014  sw $ra, 20($sp)
0x007f2ea8:  afb00010  sw $s0, 16($sp)
0x007f2eac:  afa4000c  sw $a0, 12($sp)               # save original arg
0x007f2eb0:  0c1382f8  jal 0x004e0be0                 # call original
0x007f2eb4:  00000000  nop
0x007f2eb8:  2408ffff  addiu $t0, $zero, -1
0x007f2ebc:  1448000a  bne $v0, $t0, 0x7f2ee8         # if result != -1, return it
0x007f2ec0:  00000000  nop
0x007f2ec4:  8fa4000c  lw $a0, 12($sp)                # restore arg
0x007f2ec8:  0c1382b8  jal 0x004e0ae0                 # get dispatch ID
0x007f2ecc:  00000000  nop
0x007f2ed0:  24080024  addiu $t0, $zero, 0x24
0x007f2ed4:  14480004  bne $v0, $t0, 0x7f2ee8         # if ID != 0x24, return -1
0x007f2ed8:  2402ffff  addiu $v0, $zero, -1            # default return -1 (delay slot)
0x007f2edc:  3c08009b  lui $t0, 0x009b
0x007f2ee0:  25086438  addiu $t0, $t0, 0x6438
0x007f2ee4:  91020000  lbu $v0, 0($t0)                # return BSS flag
# --- epilogue ---
0x007f2ee8:  8fbf0014  lw $ra, 20($sp)
0x007f2eec:  8fb00010  lw $s0, 16($sp)
0x007f2ef0:  03e00008  jr $ra
0x007f2ef4:  27bd0018  addiu $sp, $sp, 24
```

### Early Init

Hooks the first function call in `FUN_0048b8e0` (app initialization).

| Address | Original | Patched |
|---------|----------|---------|
| `0x48b8f4` | `jal 0x47b380` | `jal 0x7f2e60` |

**Early init** at `0x7f2e60` (60 B):

1. Calls `access("/usr/data/adb_usb_mode", 0)` to check if marker file exists
2. Sets BSS flag at `0x9B6438` accordingly (`1` if file exists, `0` otherwise)
3. Calls original `FUN_0047b380`
4. Does **not** call `adbon` — ADB activation happens only on USB connect

```
0x007f2e60:  27bdffe8  addiu $sp, $sp, -24
0x007f2e64:  afbf0014  sw $ra, 20($sp)
0x007f2e68:  3c04007f  lui $a0, 0x007f
0x007f2e6c:  24842f0c  addiu $a0, $a0, 0x2f0c         # "/usr/data/adb_usb_mode"
0x007f2e70:  0c239890  jal 0x008e6240                  # access()
0x007f2e74:  00002821  addu $a1, $zero, $zero          # F_OK = 0
0x007f2e78:  2c490001  sltiu $t1, $v0, 1               # $t1 = (access==0) ? 1 : 0
0x007f2e7c:  3c08009b  lui $t0, 0x009b
0x007f2e80:  25086438  addiu $t0, $t0, 0x6438
0x007f2e84:  a1090000  sb $t1, 0($t0)                  # store BSS flag
0x007f2e88:  0c11ece0  jal 0x0047b380                  # call original init
0x007f2e8c:  00000000  nop
0x007f2e90:  8fbf0014  lw $ra, 20($sp)
0x007f2e94:  03e00008  jr $ra
0x007f2e98:  27bd0018  addiu $sp, $sp, 24
```

### String Data

| Address | Content |
|---------|---------|
| `0x7f2f00` | `app_sync` |
| `0x7f2f0c` | `/usr/data/adb_usb_mode` |
| `0x7f2f24` | `touch /usr/data/adb_usb_mode` |
| `0x7f2f44` | `rm -f /usr/data/adb_usb_mode` |
| `0x7f2f64` | `vg_listview_sys_set` |

---

## Part 2: USB Integration

### ADB Auto-Start on USB Connect

When the USB page constructor (`FUN_0052bfa0`) finishes building the page, control jumps directly to the ADB auto-start gate.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52c32c` | `move $v0, $s1` | `j 0x41c180` | Jump to ADB auto-start gate |

**ADB auto-start gate** at `0x41c180` (80 B):

1. Checks BSS flag (`0x9B6438`): if 0, skips to return
2. Calls `access("/usr/data/adb_usb_mode", 0)`: if file doesn't exist, skips
3. Both checks pass: calls `FUN_00480f20(1)` to activate ADB (same function as 10-tap About toggle)
4. Restores `$v0 = $s1` (replaced instruction) and jumps to `0x52c334` (epilogue continuation)

```
0x0041c180:  27bdfff0  addiu $sp, $sp, -16
0x0041c184:  afbf000c  sw $ra, 12($sp)
0x0041c188:  3c08009b  lui $t0, 0x009b
0x0041c18c:  25086438  addiu $t0, $t0, 0x6438          # $t0 = BSS flag addr
0x0041c190:  91090000  lbu $t1, 0($t0)
0x0041c194:  11200009  beq $t1, $zero, 0x41c1bc         # if flag==0, skip
0x0041c198:  00000000  nop
0x0041c19c:  3c04007f  lui $a0, 0x007f
0x0041c1a0:  24842f0c  addiu $a0, $a0, 0x2f0c          # "/usr/data/adb_usb_mode"
0x0041c1a4:  0c239890  jal 0x008e6240                   # access()
0x0041c1a8:  00002821  addu $a1, $zero, $zero           # F_OK = 0
0x0041c1ac:  14400003  bne $v0, $zero, 0x41c1bc         # if file doesn't exist, skip
0x0041c1b0:  00000000  nop
0x0041c1b4:  0c1203c8  jal 0x00480f20                   # FUN_00480f20(1) — start ADB
0x0041c1b8:  24040001  addiu $a0, $zero, 1
# --- return path ---
0x0041c1bc:  8fbf000c  lw $ra, 12($sp)
0x0041c1c0:  27bd0010  addiu $sp, $sp, 16
0x0041c1c4:  02201021  addu $v0, $s1, $zero            # replaced: move $v0, $s1
0x0041c1c8:  0814b0cd  j 0x0052c334                     # continue epilogue
0x0041c1cc:  00000000  nop
```

### Custom View File (`hiby_app_sync.view`)

When App Sync is active, the USB page loads `hiby_app_sync.view` instead of `hiby_usb.view`.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52c4f0` + `0x52c4f4` | `lui $a1, 0x94` + `addiu $a1, -0x24b0` | `j 0x41c120` + `nop` | Redirect to view selector |

**View selector trampoline** at `0x41c120` (44 B):

1. Checks BSS flag: if clear, loads `"hiby_usb.view"` (original at `0x93db50`)
2. If set, loads `"hiby_app_sync.view"` (at `0x41c160`)
3. Jumps to `0x52c4f8` to continue with `strcat`

```
0x0041c120:  3c08009b  lui $t0, 0x009b
0x0041c124:  25086438  addiu $t0, $t0, 0x6438
0x0041c128:  91090000  lbu $t1, 0($t0)                  # read BSS flag
0x0041c12c:  15200004  bne $t1, $zero, 0x41c140         # if set, goto sync_view
0x0041c130:  00000000  nop
# --- default: USB view ---
0x0041c134:  3c050094  lui $a1, 0x0094
0x0041c138:  0814b13e  j 0x0052c4f8
0x0041c13c:  24a5db50  addiu $a1, $a1, -0x24b0          # $a1 = 0x93db50 ("hiby_usb.view")
# --- sync_view: App Sync view ---
0x0041c140:  3c050042  lui $a1, 0x0042
0x0041c144:  0814b13e  j 0x0052c4f8
0x0041c148:  24a5c160  addiu $a1, $a1, -0x3ea0          # $a1 = 0x41c160 ("hiby_app_sync.view")
```

**String** at `0x41c160`: `hiby_app_sync.view`

### Path-Builder Stabilization

Prevents the custom filename from causing a path validation failure inside `FUN_0052c4a0`.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52c520` | `beq ..., 0x52c548` (conditional) | `b 0x52c548` | Force unconditional branch to path copy |
| `0x52c524` | `addu $s0, $v0, $zero` | `addu $s0, $zero, $zero` | Force success return value (`$s0 = 0`) |

### Marker-File Helper

A reusable helper that checks the marker file. Called from the teardown gate and by any function that previously called `FUN_00480f80`.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x480f80` + `0x480f84` | `lui $v0, 0x97` + `jr $ra` | `j 0x41c220` + `nop` | Redirect `FUN_00480f80` |

**Marker-file helper** at `0x41c220` (40 B):

1. Calls `access("/usr/data/adb_usb_mode", 0)`
2. Returns `1` if file exists (ADB active), `0` if not (via `sltiu` — inverts access return)

```
0x0041c220:  27bdfff0  addiu $sp, $sp, -16
0x0041c224:  afbf000c  sw $ra, 12($sp)
0x0041c228:  3c04007f  lui $a0, 0x007f
0x0041c22c:  24842f0c  addiu $a0, $a0, 0x2f0c          # "/usr/data/adb_usb_mode"
0x0041c230:  0c239890  jal 0x008e6240                   # access()
0x0041c234:  00002821  addu $a1, $zero, $zero           # F_OK = 0
0x0041c238:  2c420001  sltiu $v0, $v0, 1                # $v0 = (access==0) ? 1 : 0
0x0041c23c:  8fbf000c  lw $ra, 12($sp)
0x0041c240:  03e00008  jr $ra
0x0041c244:  27bd0010  addiu $sp, $sp, 16
```

This replaces `FUN_00480f80` which originally read `DAT_00969d04` (cached ADB state). The helper reads the marker file directly, ensuring the USB mode decision always reflects the actual file state.

**Compatibility:** `FUN_00529860` (About page 10-tap ADB path) calls `FUN_00480f80` at `0x52990c`. This call is preserved and now goes through the marker-file helper, which is compatible because the About page flow also creates/removes the marker file.

### Disconnect Teardown Gate

Keeps the App Sync page visible when USB is disconnected (only in App Sync mode).

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52bda0` + `0x52bda4` | `addiu $sp, $sp, -0x78` + `addiu $a2, $zero, 0x40` | `j 0x41c260` + `nop` | Redirect teardown entry |

**Teardown gate** at `0x41c260` (80 B):

1. Checks BSS flag: if 0, falls through to original teardown
2. Calls `access()` on marker file: if file doesn't exist, falls through
3. Both checks pass (App Sync active + file exists): returns 0 early (skips teardown, page stays)
4. Otherwise: executes replaced instructions and jumps to `0x52bda8` (original teardown code)

```
0x0041c260:  3c08009b  lui $t0, 0x009b
0x0041c264:  25086438  addiu $t0, $t0, 0x6438
0x0041c268:  91090000  lbu $t1, 0($t0)                  # read BSS flag
0x0041c26c:  1120000d  beq $t1, $zero, 0x41c2a4         # if 0, goto original teardown
0x0041c270:  00000000  nop
# --- check marker file ---
0x0041c274:  27bdfff0  addiu $sp, $sp, -16
0x0041c278:  afbf000c  sw $ra, 12($sp)
0x0041c27c:  3c04007f  lui $a0, 0x007f
0x0041c280:  24842f0c  addiu $a0, $a0, 0x2f0c          # "/usr/data/adb_usb_mode"
0x0041c284:  0c239890  jal 0x008e6240                   # access()
0x0041c288:  00002821  addu $a1, $zero, $zero
0x0041c28c:  8fbf000c  lw $ra, 12($sp)
0x0041c290:  27bd0010  addiu $sp, $sp, 16
0x0041c294:  14400003  bne $v0, $zero, 0x41c2a4         # if file doesn't exist, goto original
0x0041c298:  00000000  nop
# --- suppress teardown ---
0x0041c29c:  03e00008  jr $ra                            # return early
0x0041c2a0:  00001021  addu $v0, $zero, $zero           # $v0 = 0
# --- original teardown ---
0x0041c2a4:  27bdff88  addiu $sp, $sp, -0x78            # replaced instruction 1
0x0041c2a8:  0814af6a  j 0x0052bda8                      # continue original
0x0041c2ac:  24060040  addiu $a2, $zero, 0x40           # replaced instruction 2 (delay slot)
```

### Stop Button

Adds a clickable stop control to the App Sync page view.

#### Control Table Expansion

`FUN_0052c5a0` was modified to return a pointer to a new control table with more entries.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52c5a8` | `lui $v0, 0x94` | `lui $v0, 0x42` | Table pointer high |
| `0x52c5ac` | `addiu $v0, -0x2340` (→ `0x93dcb4`) | `addiu $v0, -0x3d00` (→ `0x41c300`) | Table pointer low |
| `0x52c5b8` | `addiu $v0, $zero, 1` | `addiu $v0, $zero, 3` | Entry count: 1 → 3 |

**Control table** at `0x41c300` (2 active entries, each 96 bytes with inline name string + callback pointer at offset `+0x48`):

| Entry | Name (inline) | Callback |
|-------|--------------|----------|
| 0 | `usb_iv_back` | `0x52c480` (original back handler) |
| 1 | `app_sync_stop` | `0x41c2c0` (new stop callback) |

**Stop callback** at `0x41c2c0` (64 B):

1. `system("rm -f /usr/data/adb_usb_mode")` — removes marker file
2. Clears BSS flag at `0x9B6438` to 0
3. `FUN_00480f20(0)` — stops ADB, restores mass storage
4. `FUN_0048a380(1)` — closes the App Sync page
5. Returns 0

```
0x0041c2c0:  27bdffe8  addiu $sp, $sp, -24
0x0041c2c4:  afbf0014  sw $ra, 20($sp)
0x0041c2c8:  3c04007f  lui $a0, 0x007f
0x0041c2cc:  0c239c24  jal 0x008e7090                   # system()
0x0041c2d0:  24842f44  addiu $a0, $a0, 0x2f44           # "rm -f /usr/data/adb_usb_mode"
0x0041c2d4:  3c08009b  lui $t0, 0x009b
0x0041c2d8:  25086438  addiu $t0, $t0, 0x6438
0x0041c2dc:  a1000000  sb $zero, 0($t0)                  # clear BSS flag
0x0041c2e0:  0c1203c8  jal 0x00480f20                   # FUN_00480f20(0) — stop ADB
0x0041c2e4:  00002021  addu $a0, $zero, $zero           # $a0 = 0
0x0041c2e8:  0c1228e0  jal 0x0048a380                   # FUN_0048a380(1) — close page
0x0041c2ec:  24040001  addiu $a0, $zero, 1
0x0041c2f0:  8fbf0014  lw $ra, 20($sp)
0x0041c2f4:  00001021  addu $v0, $zero, $zero           # return 0
0x0041c2f8:  03e00008  jr $ra
0x0041c2fc:  27bd0018  addiu $sp, $sp, 24
```

---

## Required Configuration

### `set_functions.json`

Add `app_sync` to the `sys_set` section:

```json
{"app_sync":1}
```

### `sys_set.ini` (UTF-16LE with BOM)

Add the setting and sync page button label:

```
<app_sync>App Sync</app_sync>
<sync_stop>Stop Sync</sync_stop>
```

### View Files

Place `hiby_app_sync.view` in both theme directories:

- `/usr/share/resource/layout/theme1/hiby_app_sync.view`
- `/usr/share/resource/layout/theme2/hiby_app_sync.view`

The view must contain a clickable element named `app_sync_stop` for the stop button. Text for the stop button is read from the view's own text/ini binding (not hardcoded in the binary).

---

## Key Firmware Functions Referenced

| Address | Description |
|---------|-------------|
| `0x480f20` | ADB toggle: `param=1` calls `/usr/bin/adbon`, `param=0` calls `/usr/bin/adboff` |
| `0x480f80` | ADB state query (patched to check marker file instead of cached state) |
| `0x48a380` | Close current page (used by stop button and click handler OFF path) |
| `0x4af060` | Refresh a listview by name |
| `0x4e0ae0` | Get dispatch ID for a settings list item |
| `0x4e0be0` | Get toggle state for a settings list item |
| `0x8e6160` | `strcat()` |
| `0x8e6240` | `access()` |
| `0x8e7090` | `system()` |

---

## What's NOT Modified

- The original 10-tap ADB toggle on the About page (`FUN_00529860` / `FUN_00480f20`)
- `FUN_0047d760` (USB gadget handler) — completely untouched
- `FUN_004e0be0` (toggle state table reader) — body untouched, only callers redirected
- The toggle state table at `0x819590` — untouched
- The main event loop call at `0x48bbd0` — untouched
- All pre-existing patches: Sorting, DB Manager, Playlist, DB Manager Dialog

---

# Binary Size Expansion (10 MiB)

Increased binary file size to allow more room for future patching work, while preserving current behavior.

Target size used:
- `10 MiB = 10,485,760 bytes`

## Method Used

1. Read current file size.
2. Confirm ELF is valid and inspect `PT_LOAD` segments.
3. Ensure all existing loadable file data ends before EOF.
4. Append `0x00` bytes at the end of the file until it reaches 10 MiB.

No bytes were inserted in the middle of the file, and no offsets were shifted.

## Why This Is Safe

`PT_LOAD` segment file data already ended before EOF, so adding trailing bytes does not move:

- instruction addresses
- string/data addresses
- existing patch offsets

It only adds unused trailing space.

## Actual Values From This Expansion

- Original size: `5,598,244 bytes`
- New size: `10,485,760 bytes`
- Last loadable segment file end: `0x556560` (`5,596,512 bytes`)
- Trailing padding after last loadable segment: `4,889,248 bytes`

## Verification Performed

1. Confirmed final size is exactly `10,485,760` bytes.
2. Confirmed tail bytes are zero-padded.
3. Confirmed ELF is still parsed correctly.

## Notes

- This expansion used **10 MiB** (binary units), not decimal 10,000,000 bytes.
- The added space is not yet usable, it needs another patch.
- The device ram or firmware size are not affected by the increased size as the added bytes are all zeros.
---

## Other Patches in This Binary

This binary also contains the Sorting, DB Manager, Playlist, DB Manager Dialog and About Dev patches. For details see their respective README files.
