# Saint Dragon (Tenseiryuu) — PC Engine ROM Reverse Engineering Notes

## ROM Info

- **File**: `Tenseiryuu - Saint Dragon (Japan) (En).pce`
- **Size**: 393,216 bytes (384 KB, 48 banks × 8 KB)
- **CRC**: `2E278CCB`
- **Mapper**: Standard HuCard
- **Console**: PC Engine / TurboGrafx-16

---

## Memory Map (Bank Layout)

| Bank   | ROM Offset       | Logical Address | Contents                     |
|--------|------------------|-----------------|------------------------------|
| `$00`  | `$00000–$01FFF`  | `$E000–$FFFF` (page 7) | Main code, vectors, IRQ handlers |
| `$09`  | `$12000–$13FFF`  | `$4000–$5FFF` (page 2) | **Sound driver code + data** |
| `$0A`  | `$14000–$15FFF`  | `$6000–$7FFF` (page 3) | **SFX data tables** (extended) |

The sound driver is mapped to page 2 (`$4000–$5FFF`) using bank `$09`. The SFX data tables extend into page 3 (`$6000–$7FFF`) using bank `$0A`.

At runtime, the game swaps banks in/out using:
```asm
LDA #$09
TAM #$04    ; map bank $09 to page 2 ($4000-$5FFF)
LDA #$0A
TAM #$08    ; map bank $0A to page 3 ($6000-$7FFF)
```

---

## PSG Hardware Summary (HuC6280)

6 channels total. Channels 4 and 5 (zero-indexed) have noise generators.

| I/O Address | Register | Function                                          |
|-------------|----------|---------------------------------------------------|
| `$0800`     | R0       | Channel Select (0–5)                              |
| `$0801`     | R1       | Main Amplitude (global L/R volume)                |
| `$0802`     | R2       | Frequency Low (8 bits)                            |
| `$0803`     | R3       | Frequency High (4 bits)                           |
| `$0804`     | R4       | Channel On / DDA / Volume (bit7=on, bits4-0=vol)  |
| `$0805`     | R5       | L/R Balance                                       |
| `$0806`     | R6       | Waveform Data (5-bit samples × 32)                |
| `$0807`     | R7       | **Noise Enable/Freq** (bit7=enable, bits4-0=freq) — ch4,5 only |
| `$0808`     | R8       | LFO Frequency                                     |
| `$0809`     | R9       | LFO Control                                       |

---

## Sound Driver Architecture

### Entry Points

| Logical Addr | ROM Offset | Function                                    |
|--------------|------------|---------------------------------------------|
| `$4000`      | `$12000`   | PSG init — zeros all channels, disables all |
| `$4025`      | `$12025`   | SFX queue push — enqueue SFX for playback   |
| `$404F`      | `$1204F`   | SFX queue pop — dequeue and start SFX       |
| `$4110`      | `$12110`   | Sound driver tick — called every frame via timer IRQ |

### Wrappers (Bank $00)

| Logical Addr | Purpose                                         |
|--------------|--------------------------------------------------|
| `$E5C3`     | Sound driver update wrapper — saves/restores MPRs, maps bank $09/$0A, calls `$4110` |
| `$E5DF`     | SFX play wrapper — maps banks, reads SFX ID from `$2631`, calls `$4025` |

### How the Game Triggers SFX

1. Game code stores SFX ID byte into RAM `$2631`
2. Game calls `JSR $E5DF` (or the internal path `JSR $E5E2`)
3. Wrapper at `$E5DF` saves registers, maps sound banks, reads `$2631`, calls `$4025`
4. `$4025` enqueues the SFX into a 4-entry FIFO at `$209F–$20A6`
5. On next driver tick (`$4110`), `$404F` pops the queue and starts playback

### Sound Driver Tick Flow (`$4110`)

```
$4110: STA $1403         ; ack timer IRQ
       BBS 7,$A7,$4129   ; if currently processing, skip
       LDA $37F0          
       BNE $40E5          ; if fade active, handle fade
       LDA $A7
       ORA $A8
       BEQ $412A          ; if nothing active, mute
       RMB 7,$A7          ; clear busy flag
       JSR $404F          ; pop SFX queue
       RTS
```

### Channel Update Loop (`$41A6`)

```
$41A6: INC $AA            ; next channel slot
       INC $AB            ; next PSG channel
       LSR $A9            ; shift channel bitmask
       BCS $41B1          ; if bit set, process this channel
       BNE $41A6          ; loop if more channels
       RTS
$41B1: LDA $AB
       STA $0800          ; select PSG channel
       LDX $AA
       DEC $374B,X        ; decrement note duration counter
       BEQ $4211          ; if zero, fetch next note
       JSR $412F          ; per-frame processing (vibrato, etc.)
       ...
```

### Note Fetching (`$4211`)

When a channel's duration counter hits zero, new data is read from the stream:

```
$41CC: TXA
       LDY $3733,X       ; current stream position
       INY                ; advance (byte 0 is skipped/header)
       ASL A
       TAX
       STA $AC
       LDA $371B,X        ; stream ptr lo
       STA $AD
       LDA $371C,X        ; stream ptr hi
       STA $AE
$41E1: LDA ($AD),Y        ; read next byte from stream
       STA $3757,X        ; store as new duration
       INY
       LDA ($AD),Y        ; read note/command byte
```

### Note Value Interpretation (`$427B–$4334`)

| Value Range | Meaning                                           | Code Path |
|-------------|---------------------------------------------------|-----------|
| `$00–$47`   | Regular waveform note (pitch index)               | `$42A6` — lookup freq from table at `$4521` |
| `$48`       | REST — silence channel (treated as end-of-phrase) | `$4259` via `CPX #$48; BEQ` |
| `$49–$5F`   | Regular waveform note (higher pitches)            | `$42A6` |
| `$60–$7F`   | **NOISE note** — enables noise, freq = val & $1F  | `$4314` via `CPX #$60; BCS` |
| `$80–$FF`   | Command bytes (see below)                         | `$4222` via `BPL $427B` |

### Command Bytes in Music/SFX Streams

| Byte     | Operand | Meaning                                  |
|----------|---------|------------------------------------------|
| `$F0 nn` | 1 byte  | Set waveform index                       |
| `$F1 nn` | 1 byte  | Set transpose                            |
| `$F2 nn` | 1 byte  | Set vibrato index                        |
| `$F3 nn` | 1 byte  | Set detune                               |
| `$F6 lo hi` | 2 bytes | Subroutine call to address              |
| `$F8 lo hi` | 2 bytes | Loop/jump to address                    |
| `$F9 nn` | 1 byte  | Set loop counter                         |
| `$FA nn` | 1 byte  | Set duty/command                         |
| `$FF`    | —       | End of stream                            |
| `$C0–$EF nn` | varies | Set balance/volume commands            |
| `$80–$BF` | —      | Volume/enable (bit7=ch_on, bits4-0=vol)  |

### Noise Enable Code Paths

**`$4314–$4334`**: Noise note handler
```asm
$4314: TXA                ; X = note value (>= $60)
       LDX $AA
       AND #$1F           ; extract noise frequency (5 bits)
       ORA #$80           ; set bit 7 = noise enable
       TAX
$431F: LDY $AC
       TXA
       STA $37CF,Y        ; store in channel freq table (noise mode)
       STA $37D0,Y
$4331: STX $0807          ; WRITE TO NOISE REGISTER
       BRA $42F3
```

**`$42D1`**: Regular note clears noise
```asm
$42D1: STZ $0807          ; clear noise register (disable noise)
       STY $0802           ; set freq lo
       STX $0803           ; set freq hi
```

**`$4500`**: Per-frame noise update (from `$44D0` code path)
```asm
$44D8: TXA
       ASL A
       TAX
       LDA $37CF,X        ; load cached noise params
       BIT $37D0,X        ; test bit 7
       BMI $4500           ; if noise enabled, write it
...
$4500: STA $0807           ; write cached noise value
       BRA $44EF           ; continue channel update
```

---

## PSG Register Write Locations (ROM scan)

| Instruction      | ROM Offset | Logical | Purpose                        |
|------------------|------------|---------|--------------------------------|
| `STA $0807`      | `$012500`  | `$4500` | Per-frame noise update         |
| `STZ $0807`      | `$01200F`  | `$400F` | PSG init (zero all channels)   |
| `STZ $0807`      | `$0122D1`  | `$42D1` | Clear noise on regular note    |
| `STZ $0807`      | `$0122DC`  | `$42DC` | Clear noise (alternate path)   |
| `STZ $0807`      | `$0124E3`  | `$44E3` | Clear noise in per-frame update|
| `STX $0807`      | `$012331`  | `$4331` | **Enable noise on noise note** |

---

## SFX Pointer Table

Located at `$45B1` (ROM `$125B1`). Each entry is 2 bytes (lo/hi pointer to SFX header).

| SFX ID | Pointer | ROM Offset | Notes                        |
|--------|---------|------------|------------------------------|
| 0      | `$4B17` | `$12B17`   | Terminator only ($FF)        |
| 1      | `$4B18` | `$12B18`   | Stage 1 music                |
| 2      | `$5093` | `$13093`   | Stage 2 music                |
| 3      | `$56F3` | `$136F3`   | Stage 3 music (has noise)    |
| 4      | `$593A` | `$1393A`   | Stage 4 music (has noise)    |
| 5      | `$5B85` | `$13B85`   | Stage 5 music (has noise)    |
| 6      | `$5EB2` | `$13EB2`   | Stage 6 music (has noise)    |
| 7      | `$609F` | `$1409F`   | Music/SFX (has noise)        |
| 8      | `$617F` | `$1417F`   | Music/SFX (has noise)        |
| 9      | `$62AE` | `$142AE`   | Music/SFX (has noise)        |
| 10     | `$4B18` | `$12B18`   | = SFX 1                      |
| 11     | `$6505` | `$14505`   | SFX (has noise)              |
| **12** | `$653B` | `$1453B`   | **Fire SFX** (button press)  |
| **13** | `$6552` | `$14552`   | **Fire SFX variant**         |
| 14     | `$6582` | `$14582`   | SFX (has noise)              |
| 15     | `$65A3` | `$145A3`   | SFX (has noise)              |
| 16     | `$65D0` | `$145D0`   | SFX (has noise)              |
| 17     | `$65FB` | `$145FB`   | SFX (has noise)              |
| 18     | `$6624` | `$14624`   | SFX (has noise)              |
| 19     | `$6634` | `$14634`   | SFX (has noise)              |
| **20** | `$653B` | `$1453B`   | **Fire SFX** (= SFX 12)     |
| 21     | `$66C1` | `$146C1`   | SFX (has noise)              |
| 22     | `$66F1` | `$146F1`   | SFX (has noise)              |
| **23** | `$653B` | `$1453B`   | **Fire SFX** (= SFX 12)     |

### Channel Slot Bitmask Table (`$4505`)

```
$4505: 01 02 04 08 10 20 01 02 04 08 10 20
       ─── music ch 0-5 ───  ── SFX ch 6-11 ──
```

Indices 0–5 = music channel bitmasks, indices 6–11 = SFX channel bitmasks.
Both map to PSG channels 0–5 respectively (modulo 6).

---

## Fire SFX Analysis

### SFX IDs Triggered by Fire Button

Observed during dynamic analysis (breakpoint on `$E5F6`):

| SFX ID | Dec | Points To | Exclusive to fire? | Notes |
|--------|-----|-----------|--------------------|-------|
| `$0C`  | 12  | `$653B`   | **Yes** — direct `LDA #$0C` in fire handler | |
| `$0D`  | 13  | `$6552`   | **Yes** — loaded from weapon table `$89DF` | variant |
| `$0E`  | 14  | `$6582`   | **Yes** — loaded from weapon table `$89DF` | variant |
| `$11`  | 17  | `$65FB`   | **Yes** — loaded from weapon table `$89DF` | variant |
| `$17`  | 23  | `$653B`   | **Yes** — loaded from weapon table `$89DF` | = SFX 12 |
| `$14`  | 20  | `$653B`   | **No — shared with explosions/enemy hits** | = SFX 12 |

SFX `$14` (20) is triggered by both player fire AND enemy explosions. Patching it silences both.
SFX `$0C` and the table-driven IDs (`$0D`, `$0E`, `$11`, `$17`) are exclusive to the player fire handler.

### Fire SFX Data Structure (SFX 12/20/23 at `$653B`)

```
Header: 0A 41 65 0B 49 65 FF
         │  └─┬──┘  │  └─┬──┘  └─ end marker
         │   ptr    │   ptr
         │ $6541    │ $6549
         slot 10    slot 11
         (ch4)      (ch5)
```

- **Slot 10** → PSG channel 4 (noise-capable), data at `$6541`
- **Slot 11** → PSG channel 5 (noise-capable), data at `$6549`

### Channel 4 Stream (`$6542`, ROM `$014542`)

```
F0 0B    ; set waveform #11
E4       ; volume = 4, channel on
90       ; (duration/note-related)
64       ; ** NOISE note, freq=4 **
91       ; (duration/note-related)
61       ; ** NOISE note, freq=1 **
FF       ; end of stream
```

### Channel 5 Stream (`$654A`, ROM `$01454A`)

```
F0 0B    ; set waveform #11
EB       ; volume = 11, channel on
90       ; (duration/note-related)
62       ; ** NOISE note, freq=2 **
91       ; (duration/note-related)
66       ; ** NOISE note, freq=6 **
FF       ; end of stream
```

---

## How the Game Triggers Fire SFX

The game writes the SFX ID to `$2631` then calls `JSR $E5DF` (or its internal entry `$E5E2`).

### SFX Call Pattern

```asm
LDA #$xx          ; SFX ID (direct immediate)
STA $2631         ; store to SFX trigger RAM
JSR $E5E2         ; call SFX play wrapper
```

Or the dynamic (table-driven) variant:
```asm
LDA $89DF,Y       ; load SFX ID from weapon table (Y = weapon type index)
STA $2631
JSR $E5E2
```

### Weapon SFX Table at `$89DF` (ROM `$089DF`)

The fire handler in bank `$04` uses this table indexed by weapon type:
```
$089DF: 0D 0D 0D 0E 0E 0E 17 17 17 11 11 11 ...
```
All entries are fire-weapon SFX IDs. The dynamic call that reads this table is at ROM `$089BC`.

### Key Addresses

- `$2631` (WRAM `$0631`): SFX ID staging register
- `$E5DF`: SFX play routine (external entry — `JMP $E5E2`)
- `$E5E2`: SFX play routine (internal entry — saves regs, maps banks, calls `$4025`)
- `$E5F6`: `LDA $2631` inside SFX wrapper — good breakpoint to catch all SFX triggers

### All `STA $2631` + `JSR $E5E2` Call Sites in ROM

| ROM Offset | Bank | Logical | SFX ID | Fire? | Notes |
|------------|------|---------|--------|-------|-------|
| `$0014D`   | `$00` | `$E14D` | dynamic | — | Init sequence |
| `$00D74`   | `$00` | `$ED74` | `$00` | no | |
| `$00DA6`   | `$00` | `$EDA6` | `$07` | no | |
| `$00F21`   | `$00` | `$EF21` | `$16` | no | |
| `$01777`   | `$00` | `$F777` | dynamic | no | Stage music trigger (table at `$F78F`) |
| `$01EAD`   | `$00` | `$FEAD` | `$0A` | no | |
| `$040D2`   | `$02` | `$40D2` | `$14` | **shared** | SFX $14 also used by explosions |
| `$042C1`   | `$02` | `$42C1` | dynamic | no | Table at `$7000` — all `$FF` (unused) |
| `$04B10`   | `$02` | `$4B10` | dynamic | no | ZP indirect `($14),Y` |
| `$04C8F`   | `$02` | `$4C8F` | dynamic | no | Table at `$7006` — all `$FF` (unused) |
| `$04CA0`   | `$02` | `$4CA0` | dynamic | no | Table at `$7006` — all `$FF` (unused) |
| `$06DC6`   | `$03` | `$6DC6` | dynamic | no | Table at `$BD86` — boss/stage related |
| `$06E44`   | `$03` | `$6E44` | dynamic | no | Table at `$BD86` |
| `$06EBD`   | `$03` | `$6EBD` | dynamic | no | Table at `$BD86` |
| `$087E0`   | `$04` | `$87E0` | `$14` | **shared** | SFX $14 also used by explosions |
| `$089B9`   | `$04` | `$89B9` | dynamic | **YES** | Reads from table at `$89DF` — all fire IDs → **PATCH** `$089BC` |
| `$089D7`   | `$04` | `$89D7` | `$0C` | **YES** | Direct fire SFX `$0C` → **PATCH** `$089DA` |
| `$11030`   | `$08` | `$11030` | `$14` | **shared** | SFX $14 also used by explosions |
| `$11069`   | `$08` | `$11069` | `$14` | **shared** | SFX $14 also used by explosions |
| `$110A2`   | `$08` | `$110A2` | `$14` | **shared** | SFX $14 also used by explosions |

---

## Patching Strategy

### Goal

Disable the fire-button noise SFX without affecting music percussion or explosion sounds.

### Key Finding: SFX `$14` is Shared

SFX ID `$14` (20) is triggered by **both** player fire bullets and enemy explosions/hits. Patching all `STA $2631` sites that store `$14` silences explosions too.

The fire-**exclusive** SFX IDs are:
- `$0C` — loaded directly: `LDA #$0C` at ROM `$089D5`
- `$0D`, `$0E`, `$11`, `$17` — loaded dynamically from weapon table at `$89DF` (ROM `$089DF`)

Only **two JSR call sites** need to be NOP'd:

| ROM Offset | Bytes (original) | Patch | Description |
|------------|------------------|-------|-------------|
| `$089BC`   | `20 E2 E5` (JSR $E5E2) | `EA EA EA` | Dynamic weapon table fire SFX |
| `$089DA`   | `20 E2 E5` (JSR $E5E2) | `EA EA EA` | Direct SFX `$0C` fire call |

### Current Patch: v4

ROM: `Tenseiryuu - Saint Dragon (Japan) (En) [No Fire SFX v4].pce`

```
Patch 1 — ROM $089BC:
  Before: B9 DF 89 8D 31 26 [20 E2 E5]   LDA $89DF,Y; STA $2631; JSR $E5E2
  After:  B9 DF 89 8D 31 26 [EA EA EA]   (JSR replaced with NOP NOP NOP)

Patch 2 — ROM $089DA:
  Before: A9 0C 8D 31 26 [20 E2 E5]      LDA #$0C; STA $2631; JSR $E5E2
  After:  A9 0C 8D 31 26 [EA EA EA]      (JSR replaced with NOP NOP NOP)
```

### Python Patch Script

```python
import os

ORIG_PATH = r'E:\ProjectsGe\Coding\PCEngine\RE\SaintDragon\Tenseiryuu - Saint Dragon (Japan) (En).pce'
OUT_PATH  = r'E:\ProjectsGe\Coding\PCEngine\RE\SaintDragon\Tenseiryuu - Saint Dragon (Japan) (En) [No Fire SFX v4].pce'

patched = bytearray(open(ORIG_PATH, 'rb').read())

# Only patch the two fire-exclusive JSR call sites.
# SFX $14 is intentionally left intact — it is shared with explosions.
v4_patches = {
    0x089BC: "dynamic $89DF fire table (SFX $0D/$0E/$11/$17)",
    0x089DA: "direct SFX $0C fire call",
}

for jsr_off, desc in v4_patches.items():
    assert patched[jsr_off:jsr_off+3] == bytes([0x20, 0xE2, 0xE5]), f"Unexpected bytes at ${jsr_off:05X}"
    patched[jsr_off:jsr_off+3] = bytes([0xEA, 0xEA, 0xEA])
    print(f"  Patched ROM ${jsr_off:05X}: JSR $E5E2 → NOP NOP NOP  ({desc})")

with open(OUT_PATH, 'wb') as f:
    f.write(patched)
print(f"Written: {OUT_PATH}  ({os.path.getsize(OUT_PATH):,} bytes)")
```

### Approaches Tried and Abandoned

| Approach | Result | Reason Abandoned |
|----------|--------|------------------|
| Patch SFX data (noise notes → REST) | Silences fire + explosions | SFX 12/20/23 at `$653B` shared with explosions |
| NOP all `LDA #$14` + JSR sites (v2/v3) | Silences fire + explosions | SFX `$14` is also the explosion SFX |
| NOP only fire-exclusive sites (v4) | **Fire silent, explosions intact** ✓ | Current solution |

---

## RAM Map (Sound Driver)

| Address       | Size | Purpose                                          |
|---------------|------|--------------------------------------------------|
| `$009E`       | 1    | SFX queue count                                  |
| `$009F–$00A2` | 4    | SFX queue entries (SFX IDs)                      |
| `$00A3–$00A6` | 4    | SFX queue extra data                             |
| `$00A7`       | 1    | Music channel active bitmask (bit7=busy flag)    |
| `$00A8`       | 1    | SFX channel active bitmask                       |
| `$00A9`       | 1    | Current processing bitmask                       |
| `$00AA`       | 1    | Current channel slot index                       |
| `$00AB`       | 1    | Current PSG channel number                       |
| `$00AC`       | 1    | Temp: doubled slot index                         |
| `$00AD–$00AE` | 2    | Temp: data stream pointer                        |
| `$2631`       | 1    | SFX ID staging register (game → sound driver)    |
| `$3703–$3704` | 12×2 | Per-slot stream pointers (lo/hi)                 |
| `$371B–$371C` | 12×2 | Per-slot secondary stream pointers               |
| `$3733`       | 12   | Per-slot stream position (Y index)               |
| `$373F`       | 12   | Per-slot note duration preset                    |
| `$374B`       | 12   | Per-slot note duration counter (counts down)     |
| `$3757`       | 12   | Per-slot current duration value                  |
| `$379F`       | 12   | Per-slot transpose                               |
| `$37AB`       | 12   | Per-slot detune                                  |
| `$37B7`       | 12   | Per-slot vibrato step counter                    |
| `$37C3`       | 12   | Per-slot L/R balance                             |
| `$37CF–$37D0` | 12×2 | Per-slot frequency/noise cache                   |
| `$37E1`       | 12   | Per-slot volume/enable                           |
| `$37E7`       | 12   | Per-slot vibrato state                           |
| `$37ED`       | 1    | Music channel mute mask                          |
| `$37EE`       | 1    | Music channel allocation flags                   |
| `$37EF`       | 1    | SFX channel allocation flags                     |
| `$37F0`       | 1    | Fade counter                                     |
| `$37F1`       | 1    | Fade direction/speed                             |

---

## ROM Data Tables

| Logical Addr | ROM Offset | Size | Purpose                          |
|--------------|------------|------|----------------------------------|
| `$4505`      | `$12505`   | 12   | Channel slot → bitmask table     |
| `$4511`      | `$12511`   | 16   | Balance preset table             |
| `$4521`      | `$12521`   | 144  | Frequency lookup table (72 entries × 2 bytes) |
| `$45B1`      | `$125B1`   | 48   | SFX pointer table (24 entries × 2 bytes) |
| `$45E1`      | `$125E1`   | ?    | Duration preset table            |
| `$465D`      | `$1265D`   | ?    | Vibrato delta table              |

---

## Stage → Music SFX Mapping

Table at `$F78F` (ROM `$0178F`):

| Stage Index | SFX ID | Music Track |
|-------------|--------|-------------|
| 0           | 1      | Stage 1     |
| 1           | 2      | Stage 2     |
| 2           | 3      | Stage 3     |
| 3           | 4      | Stage 4     |
| 4           | 5      | Stage 5     |
| 5           | 6      | Stage 6     |

---

## GearGrafx MCP Debugging Workflow

### Loading and Running

```
mcp_geargrafx_load_media("E:/ProjectsGe/Coding/PCEngine/RE/SaintDragon/Tenseiryuu - Saint Dragon (Japan) (En).pce")
mcp_geargrafx_debug_continue()
```

### Catching Fire SFX Triggers

```
# Break when SFX play routine reads the SFX ID
mcp_geargrafx_set_breakpoint("E5F6")   ; LDA $2631 inside SFX wrapper
mcp_geargrafx_debug_continue()
# Press fire button, breakpoint hits
# Read $2631 to see which SFX ID: mcp_geargrafx_read_memory(area=0, offset="0631", size=1)
# Check call stack to find the caller
```

### Catching Noise Register Writes

```
# Write breakpoint on PSG noise register
mcp_geargrafx_set_breakpoint_range(start="0807", end="0807", write=true, execute=false, read=false)
# NOTE: Write breakpoints stop with PC at the NEXT instruction after the write
```

### Memory Areas

| ID | Name     | Size     |
|----|----------|----------|
| 0  | WRAM     | 8,192    |
| 1  | ZP       | 256      |
| 3  | ROM      | 393,216  |
| 4  | VRAM     | 32,768   |
| 6  | SAT      | 256      |
| 8  | PALETTES | 512      |
| 10 | BRAM     | 2,048    |

### Key Notes

- Disassembly is execution-based: only code that has run appears in `get_disassembly`. Must play the game to populate code paths.
- Fast-forward (`mcp_geargrafx_toggle_fast_forward`) is useful to advance through title screens.
- Use `mcp_geargrafx_controller_button(button="run")` to press Start at title screen.
- The call stack is very deep (250+ entries) due to the main loop being recursive `JSR $E260` calls.
- Bank context matters: sound driver disassembly needs `bank="09"` parameter.

---

## Files

| File | Description |
|------|-------------|
| `Tenseiryuu - Saint Dragon (Japan) (En).pce` | Original unmodified ROM |
| `Tenseiryuu - Saint Dragon (Japan) (En) [No Fire SFX v4].pce` | **Current patch** — fire SFX silenced, explosions/music intact |
| `Tenseiryuu - Saint Dragon (Japan) (En) [No Fire SFX v2].pce` | Superseded — patched SFX `$14` sites, silenced explosions too |
| `Tenseiryuu - Saint Dragon (Japan) (En) [No Fire SFX v3].pce` | Superseded — v2 + dynamic table patch, still silenced explosions |
| `waveforms.md` | All 11 PSG waveforms extracted with sample values and ASCII art |
| `instructions.md` | This file |
