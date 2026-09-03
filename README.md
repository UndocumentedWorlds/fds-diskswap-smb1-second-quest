# FDS disk swap — unlocking SMB1's second quest from Zelda

A TAS that performs arbitrary code execution in *The Legend of Zelda*
(Famicom Disk System), writes a single byte into the console's internal
RAM, and swaps the disk for *Super Mario Bros.* That byte makes SMB1
behave as if the game had already been beaten once.

Video: <YouTube link>

## How it works

On boot, SMB1 checks `$07FF`. If it holds `$A5`, the game treats it as a
warm boot and clears only `$0700-$07D6`. Anything above that survives.

`$07FC` (`WorldSelectEnableFlag`) lives in the surviving region, and it is
copied to `$076A` (the second quest flag) when a game starts. It also
enables the world select on the title screen.

Writing `$07FC = $01` and `$07FF = $A5` from Zelda therefore gives, on a
fresh power-on, both the second quest and the ability to jump straight to
World 8.

### Payload

21 bytes, written in 232 frames using the shift-write primitive from
[#4709S](https://tasvideos.org/4709S).

```
$06C3:  A9 01     LDA #$01
        8D FC 07  STA $07FC     ; second quest
        A9 A5     LDA #$A5
        8D FF 07  STA $07FF     ; warm boot validation
        A9 00     LDA #$00
        8D 00 20  STA $2000     ; disable NMI
        8D 02 01  STA $0102     ; clear the FDS BIOS reset flag
        4C D5 06  JMP $06D5     ; halt and wait for the disk swap
```

NMI is disabled and the CPU parks in that loop, holding RAM steady while
the disk is changed. The screen looks broken at that point; it is not a
crash.

### Why the ACE is necessary

`$A5` is an arbitrary sentinel value, chosen to avoid coincidental
matches. I checked several FDS titles (Zelda, Metroid, Kid Icarus,
Castlevania) to see whether any of them happens to leave `$07FF = $A5` on
its own, which would make the ACE unnecessary. All of them leave that
address at `$00`, even while running. The top of page 7 is simply not used
by most games.

## Timing

Power-on to touching the axe in 8-4 (RTA timing for the "Both Quests"
category).

| | Time | |
|---|---|---|
| Both Quests (TAS, NES) | 7:57.231 | periwinkle & Kriller37 |
| This TAS (FDS) | 6:18.9 | 22,773 frames |

These are different versions — the reference runs on the NES cartridge
release, this one on the FDS release, which has a BIOS boot sequence and
disk loading between levels. Neither exists on the NES version, so the
difference works against the FDS side. There is no existing "Both Quests"
TAS on the FDS version.

The gap is not better play. Reaching arbitrary code execution in Zelda
takes 3:11; completing Super Mario Bros. takes about 4:59. The second
quest flag is worth roughly five minutes of gameplay, and writing it
costs four seconds.

This run is not optimized and does not use the flagpole glitch.

## Files

| File | |
|---|---|
| `zelda_diskswap_smb1_w8.tasproj` | the movie |
| `patch/` | FDS support for the hot-swap fork |

ROMs are not distributed.

```
Zelda no Densetsu - The Hyrule Fantasy (Japan) (v1.0)
  SHA1 135AC9CBDF3983AA77D7581B26D685D02615D36A
Super Mario Brothers (Japan) [FDS]
  SHA1 383AD8E3890A95DE9595F0A6087648F51177DA13
FDS BIOS (disksys.rom)
  SHA1 57FE1BDEE955BB48D357E463CCBF129496930B62
```

## Emulator

[100thCoin's BizHawk hot-swap fork](https://github.com/100thCoin/BizHawk)
(BizHawk 2.9.1), with FDS support added. See [patch/README.md](patch/README.md).

The fork implements `HotSwap()` for NES cartridges, but it fails on `.fds`
images because the FDS BIOS is never passed to `Init()`. The patch keeps
the BIOS obtained during the normal load and passes it through.

## Unfinished

Connecting this setup to the SMB1 game end glitch
([#10297S](https://tasvideos.org/10297S)) does not work yet. Skipping the
first quest changes the RNG state, and the Buzzy Beetle needed to load the
second Bowser walks the wrong way. Ideas are welcome.

## Credits

- Zelda game end glitch — TASeditor, Masterjun, sockfolder
  ([#4709S](https://tasvideos.org/4709S))
- Both Quests TAS, used as the timing baseline — periwinkle & Kriller37
- SMB1 route reference — HappyLee
- BizHawk hot-swap fork — [100thCoin](https://github.com/100thCoin/BizHawk)
