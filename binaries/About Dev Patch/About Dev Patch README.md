# Post Bar Widget Patch for About Page

## Binary SHA256: `39c0681ff3680d83cec223934c64be0c0b14d28df18cc5e07b6f2b9a13f07e05`

## Overview

This patch re-enables the **`about_dev_iv_post_bar`** widget on the About page, allowing it to open a QR code dialog just like the Facebook, WeChat and Microblog widgets.

Previously, the Post Bar click handler was a dead redirect — it contained a single `j 0x529F80` instruction that jumped to the Facebook handler, making it display Facebook's QR code (index 0) instead of its own. This patch replaces that jump with a standalone handler that calls the shared dialog function `FUN_0052bd20` with **index 3**, pointing to a new `img_path_3` slot in `url_qrcode.dlg`.

## How It Works

All four About page social widgets share the same dialog mechanism: the click handler loads the widget's data pointer, then calls `FUN_0052bd20` with an index in `$a2`. That function stores the index in a global variable, then opens `url_qrcode.dlg` as a popup. The dialog's imageview reads the corresponding `img_path_N` entry to determine which QR code image to display.

## Patch

### Post Bar click handler (`0x529FC0`)

The Post Bar handler occupies a 32-byte slot at `0x529FC0`–`0x529FDC` (8 instruction words). The original content was a single jump to the Facebook handler followed by 7 NOPs — a code cave left by the developers when they disabled the widget.

The new handler follows the same structure as the Facebook and WeChat handlers, minus the null-checks on `$a0` and `$a1` (omitted to fit the 8-instruction slot; safe because the UI framework guarantees non-null values for active widget callbacks):

```mips
# Post Bar click handler — opens QR dialog with index 3
lw     $a1, 0x50($a1)           # a1 = widget data pointer
addiu  $sp, $sp, -32            # allocate stack frame
sw     $ra, 28($sp)             # save return address
jal    0x0052BD20               # FUN_0052bd20 — show QR dialog
addiu  $a2, $zero, 3            # delay slot: a2 = image index 3
lw     $ra, 28($sp)             # restore return address
jr     $ra                      # return to caller
addiu  $sp, $sp, 32             # delay slot: restore stack
```

### Handler comparison

All four social widget handlers call the same function. The only difference is the image index passed in `$a2`:

| Widget | Handler VA | `$a2` value | `img_path` slot |
|---|---|---|---|
| `about_dev_iv_facebook` | `0x529F80` | 0 | `img_path_0` |
| `about_dev_iv_wechat` | `0x529F40` | 1 | `img_path_1` |
| `about_dev_iv_microblog` | `0x529EA0` | 2 | `img_path_2` |
| `about_dev_iv_post_bar` | `0x529FC0` | **3** | **`img_path_3`** |

### Index validation in `FUN_0052bd20`

The dialog callback at `0x52BEA4` performs `sltiu $v1, $v0, 4`, confirming that indices 0–3 are valid. Index 3 was already supported by the firmware but never used.

## Execution Flow

```
User taps Post Bar icon on About page
  → Post Bar click handler (0x529FC0)
    ├─ lw $a1, 0x50($a1)              — load widget data pointer
    └─ jal FUN_0052bd20 (a2 = 3)      — show QR dialog with index 3
         ├─ stores index 3 to global
         ├─ opens url_qrcode.dlg
         ├─ dialog reads img_path_3
         └─ displays about_dev/postbar_qrcode.png
```

## Key Firmware Functions Referenced

| Address | Description |
|---|---|
| `0x52BD20` | `FUN_0052bd20` — stores image index and opens `url_qrcode.dlg` as a popup dialog |
| `0x494160` | Dialog creation function called by `FUN_0052bd20` |

## Binary Patch Summary (Post Bar Widget)

Total modified bytes: **32**

| File offset | VA | Original bytes | Patched bytes | Description |
|---|---|---|---|---|
| `0x129FC0` | `0x00529FC0` | `E0 A7 14 08` | `50 00 A5 8C` | `lw $a1, 0x50($a1)` |
| `0x129FC4` | `0x00529FC4` | `00 00 00 00` | `E0 FF BD 27` | `addiu $sp, $sp, -32` |
| `0x129FC8` | `0x00529FC8` | `00 00 00 00` | `1C 00 BF AF` | `sw $ra, 0x1c($sp)` |
| `0x129FCC` | `0x00529FCC` | `00 00 00 00` | `48 AF 14 0C` | `jal 0x52BD20` |
| `0x129FD0` | `0x00529FD0` | `00 00 00 00` | `03 00 06 24` | `addiu $a2, $zero, 3` |
| `0x129FD4` | `0x00529FD4` | `00 00 00 00` | `1C 00 BF 8F` | `lw $ra, 0x1c($sp)` |
| `0x129FD8` | `0x00529FD8` | `00 00 00 00` | `08 00 E0 03` | `jr $ra` |
| `0x129FDC` | `0x00529FDC` | `00 00 00 00` | `20 00 BD 27` | `addiu $sp, $sp, 32` |

# Other patches included in the binary

This binary also contains the Sorting, DB Manager, Playlist and Done Dialog Patches. For details see: [Sorting Patch README.md](https://github.com/hiby-modding/hiby-mods/blob/main/binaries/Sorting%20Patch/Sorting%20Patch%20README.md), [DB Manager Patch README.md](https://github.com/hiby-modding/hiby-mods/blob/main/binaries/DB%20Manager%20Patch/DB%20Manager%20Patch%20README.md), [Playlist Patch README.md](https://github.com/hiby-modding/hiby-mods/blob/main/binaries/Playlist%20Patch/Playlist%20Patch%20README.md) and [DB Manager Dialog Patch README.md](https://github.com/hiby-modding/hiby-mods/blob/main/binaries/DB%20Manager%20Dialog%20Patch/DB%20Manager%20Dialog%20Patch%20README.md)
