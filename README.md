# Atrox Window Mode

[한국어](README.ko.md)

A byte patch for the Atrox (Joymax) executable that makes the game run in an 800x600 window. It skips the block that switches to fullscreen, fixes the resolution at 800x600, and cuts the scroll amount to one eighth so the view is easy to handle inside a window. The changed bytes and their assembly are described in [docs/window-mode.md](docs/window-mode.md).

**Compatible version**: Atrox v1.10 build 1 (UPX-unpacked `atrox.exe`, md5 `b4c64c06c2afd830995ef497478718be`)

<p>
  <img src="docs/screenshots/screenshot-1.jpg" width="480" alt="Atrox running in a window">
</p>


## How to use

You apply the patch by hand, using the table in the docs.

1. The retail `atrox.exe` is packed with UPX, so unpack it first with `upx -d`.
2. In a hex editor, go to each file offset in the table in [docs/window-mode.md](docs/window-mode.md), check the original bytes, and overwrite them with the new bytes.
3. Run the patched executable from the Atrox folder and it opens in an 800x600 window.

Mouse scroll and keyboard scroll in the game options need to be set to the third notch for the view to move at a suitable speed. For edge scrolling, keep the mouse inside the window with a mouse capture tool such as WindowMouseCapturer.


## How it works

**The fullscreen block is skipped.** A fullscreen block strips the title bar from the window style with `GetWindowLongA` / `SetWindowLongA`, and the conditional jump in front of it decides whether that block runs. That `JE` was replaced with a `JMP` to the same target.

**The resolution mode is fixed at 1.** The resolution function uses 640x480 for mode 0 and 800x600 for mode 1. The spot that reads the mode saved in the options and passes it on now jumps to a short piece of code in free space that always passes 1. One value used by an initialization function was also changed from 1 to 0 so the view starts centered.

**The scroll values are divided by 8.** There was no room for a `sar` where the movement amounts are passed to the two scroll functions, so both calls were moved into free space between functions with `sar esi, 3` and `sar ebx, 3` placed in front of them, and execution then jumps back to the original flow.

```
0042D477  EB 3C              jmp  0x0042D4B5
...
0042D4B5  C1 FE 03           sar  esi, 3
0042D4B8  56                 push esi
0042D4B9  E8 BC 9D FD FF     call 0x0040727A
0042D4BE  EB BD              jmp  0x0042D47D
```


## Files

| Path | Contents |
|---|---|
| `docs/window-mode.md` | Target file, table of changed bytes, assembly notes |
| `docs/screenshots/` | Screenshots |


## License

zlib. See [LICENSE](LICENSE).
