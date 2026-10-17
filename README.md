# MH3U Damage Multiplier

A Cemu graphic pack for **Monster Hunter 3 Ultimate** that multiplies the damage monsters take. It's meant for playing multiplayer-scaled quests solo, without the hunt taking four times as long.

It scales monster HP loss, flinches and part damage together, so a 2.5× hunt plays like one with more hunters. Damage to your hunter is not changed.

## Requirements

- Monster Hunter 3 Ultimate, **US**, title `00050000-10118300`, **update v32**
- Cemu 2.x, tested on Cemu 2.6

The patch only applies to that exact game build: the game code checksum must be `0x9d2f15e5`. On any other version, Cemu skips it and the game runs normally. EUR and MH3G aren't supported yet.

## Install

1. Copy the `MH3U_DamageMultiplier` folder into Cemu's `graphicPacks` folder:
   - Normal install: `%APPDATA%\Cemu\graphicPacks\`. Create the `graphicPacks` folder if it doesn't exist.
   - Portable install: `graphicPacks\` next to `Cemu.exe`
2. **Restart Cemu.** It only reads graphic packs when it starts.
3. Open **Options → Graphic packs → Monster Hunter 3 Ultimate → Mods → Damage Multiplier**, tick it and choose a preset.
4. Start the game. To change the preset later, restart the game.

## Presets

| Preset | Multiplier |
|---|---|
| 2.5x (default) | ×2.5 |
| 1x (off) | ×1 |
| 1.5x | ×1.5 |
| 2x | ×2 |
| 3x | ×3 |

Each hit is rounded to the nearest whole number, with halves rounding up. So at 2.5×, a 1-damage hit becomes 3 and a 5-damage hit becomes 13. For normal-sized hits, the rounding isn't noticeable.

To confirm it's active, look in `%APPDATA%\Cemu\log.txt` for:

```
Applying patch group 'MH3U_DamageMultiplier_US_v32'
```

## What is and isn't scaled

| | Scaled |
|---|---|
| Monster HP loss from hits (weapons, Palico, bombs, etc.) | Yes |
| Flinch counter | Yes |
| Part damage (breaks, tail cut) | Yes |
| Damage to your hunter | No |
| A separate HP drain, likely poison or other status damage | No |

Large and small monsters both go through the patched step: every monster type uses the same shared update code.

Tested at 2.5×:

| Monster | Type | Result |
|---|---|---|
| Barroth | Large | Each hit came out as predicted (4 → 10, 5 → 13, 7 → 18, 8 → 20) |
| Royal Ludroth | Large | 10 of 10 hits scaled |
| Ludroth, Kelbi | Small | 5 of 5 hits scaled |

How to check a hit without a 1× run: at 2.5×, every hit ends in 0, 3, 5 or 8 (an even hit × 2.5 ends in 0 or 5, and an odd hit × 2.5 rounds up to a number ending in 3 or 8). A killing hit only takes the HP that was left, so it doesn't count.

Damage to the hunter was unchanged. Whether flinches and part breaks come sooner hasn't been measured yet.

## How it works

Addresses are Wii U addresses for US v32.

- Each monster object has a pointer at `+0x1BE0` to a stats block. In that block:
  - `+0x8B8` is HP, a 32-bit int.
  - `+0x8BC` is max HP.
- A hit does not change HP directly. It adds its damage to a per-part 16-bit table at `stats+0x8EC + 2*part`, which has 8 parts.
- Once per frame, the monster update loop at `0x025cb540` goes through the 8 parts. It loads each part's queued damage once, at `0x025cb558` (`lha r19, 0x8ec(r12)`), and that value then:
  - is added to a total. At `0x025cbbdc` the total goes to `AddHP(monster, -total)` (`0x025c6f74`), which clamps HP to [0, max].
  - is subtracted from the flinch counter at `stats+0x9DC`. When the counter reaches zero, it reloads from `+0x9DE` and sets the flinch flag at `+0x9E0`.
  - is used for the part's own data at `stats+0x978 + 8*part`.
- A separate amount at `stats+0xAB4` also goes through `AddHP`. It looks like status damage, and the patch leaves it alone.

The patch replaces the load at `0x025cb558` with a jump to a code cave. The cave computes `damage * mult / 10`, rounds it, caps it at `0x7FFF` (the table is 16-bit), and jumps back. Because the scaling happens before the value is split up, HP, flinch and part damage all scale together.

### How it was found

The monster's HP address came from [MH3U-MH3G-HP-Overlay-For-WiiU-Emulator](https://github.com/Alexander-Lancellott/MH3U-MH3G-HP-Overlay-For-WiiU-Emulator). A memory write breakpoint on HP in Cemu's PPC debugger then stopped at `0x025c6efc` inside `AddHP`, and the call chain led back to the per-part loop.

Memory breakpoints only trigger with the game profile's CPU mode set to **Single-core interpreter**. With the default recompiler, they never trigger.

### Porting to other versions

On another build, the addresses and the `moduleMatches` checksum will be different. Look for the loop that sums `lha rX, 0x8ec(rY)` over 8 parts and passes the negated total to the HP-add function, then apply the same hook there.

## License

[MIT](LICENSE)
