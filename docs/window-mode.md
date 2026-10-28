# Atrox window mode patch

The patch changes four places in the Atrox executable so the game runs in an 800x600 window. It skips the block that switches to fullscreen and always passes 800x600 to the resolution function. It also changes where the view starts and divides the values passed to the scroll functions by 8.


## Target

| Item | Value |
|---|---|
| Game | Atrox v1.10 build 1 |
| File | UPX-unpacked `atrox.exe` (2,965,504 bytes) |
| md5 | `b4c64c06c2afd830995ef497478718be` |

The retail `atrox.exe` (md5 `01d0c56a04f61ffa40d219eb0378154c`) is packed with UPX, so its code cannot be read directly. Unpacking it with `upx -d` gives the file in the table above.

The image base is `0x400000`. Every change is inside the `.text` section, and `.text` maps virtual address `0x401000` to file offset `0x1000`. So **file offset = virtual address - 0x400000**.

| Section | Virtual address | File offset | Size in file |
|---|---|---|---|
| `.text` | `0x00401000` | `0x1000` | `0x1D4000` |
| `.rdata` | `0x005D5000` | `0x1D5000` | `0x4A000` |
| `.data` | `0x0061F000` | `0x21F000` | `0x86000` |


## Changed bytes

| Virtual address | File offset | Original bytes | New bytes | Meaning |
|---|---|---|---|---|
| `0x0057C626` | `0x17C626` | `0F 84 C0 00 00 00` | `E9 C1 00 00 00 90` | Always skip the fullscreen block |
| `0x00489521` | `0x89521` | `8B 40 28 50 E8 AD 80 F7 FF` | `EB C8 90 90 90 90 90 90 90` | Jump to the code below |
| `0x004894EB` | `0x894EB` | 13 x `CC` | `B8 01 00 00 00 50 E8 E1 80 F7 FF EB 32` | Pass resolution mode 1 (800x600) |
| `0x004DC496` | `0xDC496` | `BF 01 00 00 00` | `BF 00 00 00 00` | Starting view position |
| `0x0042D477` | `0x2D477` | `56 E8 FD 9D FD FF` | `EB 3C 90 90 90 90` | Redirect the first scroll call |
| `0x0042D481` | `0x2D481` | `53` | `90` | Move `push ebx` below |
| `0x0042D484` | `0x2D484` | `E8 8B 4F FD FF` | `EB 3A 90 90 90` | Redirect the second scroll call |
| `0x0042D4B5` | `0x2D4B5` | 11 x `90` | `C1 FE 03 56 E8 BC 9D FD FF EB BD` | Divide `esi` by 8, first call |
| `0x0042D4C0` | `0x2D4C0` | 16 x `CC` | `C1 FB 03 53 E8 4B 4F FD FF EB BE 90 90 90 90 90` | Divide `ebx` by 8, second call |

`0x004894EB`, `0x0042D4B5`, and `0x0042D4C0` are free space between functions (filled with `CC` or `90`), used here for new code.


## 1. Skipping the fullscreen block

The conditional jump at `0x0057C626` goes to `0x0057C6EC` when `al` is 0. Otherwise it falls into a fullscreen block that clears the `0x00C80000` bits (title bar and system menu) from the window style with `GetWindowLongA` / `SetWindowLongA`.

```
0057C61D  84 C0              test al, al
...
0057C626  0F 84 C0 00 00 00  je   0x0057C6EC        ; original
0057C626  E9 C1 00 00 00     jmp  0x0057C6EC        ; patched
0057C62B  90                 nop
0057C62C  8B B4 24 54 04 00 00  mov esi, [esp+0x454]
0057C633  6A F0              push -16              ; GWL_STYLE
0057C635  56                 push esi
0057C636  FF 15 A8 72 E6 00  call [GetWindowLongA]
0057C63C  25 FF FF 37 FF     and  eax, 0xFF37FFFF
0057C641  50 6A F0 56        push eax ; push -16 ; push esi
0057C645  FF 15 BC 71 E6 00  call [SetWindowLongA]
```

The `JE rel32` was replaced with a `JMP rel32` to the same target. `JMP` is one byte shorter, so the distance grows from `0xC0` to `0xC1` and one `NOP` is left at the end.


## 2. Locking the resolution mode to 1

`0x004DC840` takes a resolution mode and sets the screen size. Mode 0 uses 640x480 and mode 1 uses 800x600.

```
004DC85A  E8 26 BD F2 FF        call 0x00408585      ; returns the global object 0x00B1DE78
004DC85F  C7 40 20 20 03 00 00  mov  [eax+0x20], 800 ; mode 1: width
004DC866  E8 1A BD F2 FF        call 0x00408585
004DC86B  C7 40 24 58 02 00 00  mov  [eax+0x24], 600 ; mode 1: height
...
004DC899  E8 E7 BC F2 FF        call 0x00408585
004DC89E  C7 40 20 80 02 00 00  mov  [eax+0x20], 640 ; mode 0: width
004DC8A5  E8 DB BC F2 FF        call 0x00408585
004DC8AA  C7 40 24 E0 01 00 00  mov  [eax+0x24], 480 ; mode 0: height
```

The function at `0x00489500` passes the `+0x28` value of the global object `0x00B26DFC` (the resolution mode saved in the options) as the mode to this function, calling `0x004DC840` through `0x0040787E`. `0x004015D7` only returns the address of that object.

```
0048951A  6A 01              push 1
0048951C  E8 B6 80 F7 FF     call 0x004015D7
00489521  8B 40 28           mov  eax, [eax+0x28]   ; mode saved in the options
00489524  50                 push eax
00489525  E8 AD 80 F7 FF     call 0x004015D7
0048952A  8B C8              mov  ecx, eax
0048952C  E8 4D E3 F7 FF     call 0x0040787E        ; 0x004DC840(mode, 1)
```

The 9 bytes at `0x00489521` became a short jump to the free space at `0x004894EB`, which puts 1 in the mode slot, makes the same call, and returns to `0x0048952A`.

```
00489521  EB C8              jmp  0x004894EB
004894EB  B8 01 00 00 00     mov  eax, 1
004894F0  50                 push eax
004894F1  E8 E1 80 F7 FF     call 0x004015D7
004894F6  EB 32              jmp  0x0048952A
```


## 3. Starting view position

`0x004DC480` is an initialization function that calls `0x004DC840`. It loads 1 into `edi` and uses that value for both arguments of the `0x004DC840` call and for the `+0x48`, `+0x4C`, and `+0x54` fields. Changing that 1 to 0 makes the view start centered.

```
004DC496  BF 01 00 00 00     mov  edi, 1            ; original
004DC496  BF 00 00 00 00     mov  edi, 0            ; patched
004DC49B  8B CE              mov  ecx, esi
004DC49D  57 57              push edi ; push edi
...
004DC4C4  E8 B5 B3 F2 FF     call 0x0040787E        ; 0x004DC840(edi, edi)
...
004DC4F1  89 7E 48           mov  [esi+0x48], edi
004DC4F4  89 7E 4C           mov  [esi+0x4C], edi
004DC4F7  89 7E 54           mov  [esi+0x54], edi
```

What each field means still needs to be checked.


## 4. Reducing the scroll amount

The code around `0x0042D477` passes the computed movement amounts `esi` and `ebx` to two functions (`0x0040727A`, `0x00402414`). Scrolling is too fast in window mode, so the patch shifts both values right by 3 bits to divide them by 8 before passing them on.

```
; original
0042D473  8B 4C 24 18        mov  ecx, [esp+0x18]
0042D477  56                 push esi
0042D478  E8 FD 9D FD FF     call 0x0040727A
0042D47D  8B 4C 24 18        mov  ecx, [esp+0x18]
0042D481  53                 push ebx
0042D482  8B F0              mov  esi, eax
0042D484  E8 8B 4F FD FF     call 0x00402414
0042D489  8B F8              mov  edi, eax
```

```
; patched
0042D477  EB 3C              jmp  0x0042D4B5
0042D479  90 90 90 90
0042D47D  8B 4C 24 18        mov  ecx, [esp+0x18]
0042D481  90                 nop
0042D482  8B F0              mov  esi, eax
0042D484  EB 3A              jmp  0x0042D4C0
0042D486  90 90 90
0042D489  8B F8              mov  edi, eax

0042D4B5  C1 FE 03           sar  esi, 3
0042D4B8  56                 push esi
0042D4B9  E8 BC 9D FD FF     call 0x0040727A
0042D4BE  EB BD              jmp  0x0042D47D
0042D4C0  C1 FB 03           sar  ebx, 3
0042D4C3  53                 push ebx
0042D4C4  E8 4B 4F FD FF     call 0x00402414
0042D4C9  EB BE              jmp  0x0042D489
```

There was no room for a `sar` at the original spot, so each call was moved into free space with a `sar` in front of it. Execution then jumps back to the original flow. A moved `CALL` sits at a new address, so its relative distance was recomputed. The original `E8 FD 9D FD FF` and the patched `E8 BC 9D FD FF` both call `0x0040727A`.

`sar` is a sign-preserving shift, so negative (opposite direction) movement is divided by 8 as well.


## Patching by hand

1. Unpack the retail `atrox.exe` with `upx -d` and check that its md5 is `b4c64c06c2afd830995ef497478718be`.
2. In a hex editor, go to each file offset in the table above and check that the original bytes match.
3. Overwrite them with the new bytes. Each replacement is the same length as the original.
