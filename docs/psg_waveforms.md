# Saint Dragon (Tenseiryuu) — PSG Waveform Table

## ROM Info

- **File**: `Tenseiryuu - Saint Dragon (Japan) (En).pce`
- **Size**: 393,216 bytes (384 KB, 48 banks × 8 KB)
- **CRC**: `2E278CCB`

---

## How Waveforms Work on the PC Engine

The HuC6280 PSG has 6 channels, each with a 32-sample waveform buffer. Each sample is 5 bits (values 0–31). When a channel plays, it cycles through its 32 samples to produce sound.

Waveform data is written to I/O register `$0806` (R6). Writing 32 consecutive bytes loads the entire waveform buffer for the currently selected channel. The channel must be disabled (R4 bit 7 = 0) before loading a new waveform.

---

## Waveform Table Location

| Item | Logical Address | ROM Offset | Details |
|------|----------------|------------|---------|
| Waveform loading function | `$4390`–`$43B5` | `$12390`–`$123B5` | In bank `$09` |
| `STA $0806` instruction | `$43A6` | `$123A6` | The actual PSG write |
| Waveform pointer table | `$4647` | `$12647` | 11 entries × 2 bytes (lo/hi) |
| Waveform data start | `$49B7` | `$129B7` | 11 × 32 bytes = 352 bytes |
| Waveform data end | `$4B16` | `$12B16` | Last byte of waveform 10 |

### Waveform Loading Code (at `$4390`)

The sound driver's `$F0 nn` command calls this function. The waveform index `nn` is doubled and used as an index into the pointer table at `$4647`:

```asm
; Input: A = waveform index × 2
    TAX                  ; X = doubled index
    LDA $4647,X          ; waveform pointer lo
    STA $AD              ; → ZP pointer lo
    LDA $4648,X          ; waveform pointer hi
    STA $AE              ; → ZP pointer hi
    LDX #$20             ; 32 samples to write
    CLY                  ; Y = 0
loop:
    LDA ($AD),Y          ; read sample byte
    STA $0806            ; write to PSG waveform register
    INY
    DEX
    BNE loop             ; repeat 32 times
    PLY
    RTS
```

### Pointer Table (`$4647`, ROM `$12647`)

| Entry | Bytes (lo hi) | Pointer | ROM Offset |
|-------|--------------|---------|------------|
| 0 | `B7 49` | `$49B7` | `$129B7` |
| 1 | `D7 49` | `$49D7` | `$129D7` |
| 2 | `F7 49` | `$49F7` | `$129F7` |
| 3 | `17 4A` | `$4A17` | `$12A17` |
| 4 | `37 4A` | `$4A37` | `$12A37` |
| 5 | `57 4A` | `$4A57` | `$12A57` |
| 6 | `77 4A` | `$4A77` | `$12A77` |
| 7 | `97 4A` | `$4A97` | `$12A97` |
| 8 | `B7 4A` | `$4AB7` | `$12AB7` |
| 9 | `D7 4A` | `$4AD7` | `$12AD7` |
| 10 | `F7 4A` | `$4AF7` | `$12AF7` |
| 11 | `00 00` | `$0000` | NULL — fire SFX uses noise mode, waveform irrelevant |

---

## Waveform Data

All values are 5-bit (0–31). Each waveform has 32 samples.

### Waveform 0 — Pulse (wide, asymmetric)

```
██ ████    █████                
```

```
31 31 0 31 31 31 31 0 0 0 0 31 31 31 31 31 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
```

Raw hex: `1F 1F 00 1F 1F 1F 1F 00 00 00 00 1F 1F 1F 1F 1F 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00`

### Waveform 1 — Half-triangle / ramp

```
 ▁▂▃▄▅▆███████████▆▅▄▃▂▁      
```

```
0 4 8 12 16 20 24 28 31 31 31 31 31 31 31 31 31 28 24 20 16 12 8 4 0 0 0 0 0 0 0 0
```

Raw hex: `00 04 08 0C 10 14 18 1C 1F 1F 1F 1F 1F 1F 1F 1F 1F 1C 18 14 10 0C 08 04 00 00 00 00 00 00 00 00`

### Waveform 2 — Complex / FM-like

```
▂  ▁▁▂▂▂▁▁▁▁▁▁▁▃▅██▆▆▅▅▄▆▆▆▆▆▆▆▄
```

```
8 0 2 4 6 8 10 11 7 7 7 7 7 7 7 15 23 31 29 27 25 23 21 19 24 24 24 24 24 24 24 16
```

Raw hex: `08 00 02 04 06 08 0A 0B 07 07 07 07 07 07 07 0F 17 1F 1D 1B 19 17 15 13 18 18 18 18 18 18 18 10`

### Waveform 3 — Sine (smooth)

```
▃▄▅▆▆▆▆█████▆▅▅▄▃▃▂▁▁       ▁▂▂▃
```

```
15 18 21 24 25 26 27 28 29 30 29 28 25 23 21 18 15 12 9 7 5 3 1 0 0 0 1 3 5 9 9 12
```

Raw hex: `0F 12 15 18 19 1A 1B 1C 1D 1E 1D 1C 19 17 15 12 0F 0C 09 07 05 03 01 00 00 00 01 03 05 09 09 0C`

### Waveform 4 — Sine variant (offset phase)

```
▄██▅▄▃▂▁    ▁▁▂▃▄▅▅▆▆███▆▆▅▄▄▂  
```

```
16 31 31 22 16 12 8 5 3 2 2 3 5 7 11 14 17 20 23 25 27 28 28 28 27 25 22 19 16 9 0 0
```

Raw hex: `10 1F 1F 16 10 0C 08 05 03 02 02 03 05 07 0B 0E 11 14 17 19 1B 1C 1C 1C 1B 19 16 13 10 09 00 00`

### Waveform 5 — Square wave (50% duty, rounded edges)

```
▆▆████████████▆▆▁▁            ▁▁
```

```
25 27 29 31 31 31 31 31 31 31 31 31 31 29 27 25 6 4 2 0 0 0 0 0 0 0 0 0 0 2 4 6
```

Raw hex: `19 1B 1D 1F 1F 1F 1F 1F 1F 1F 1F 1F 1F 1D 1B 19 06 04 02 00 00 00 00 00 00 00 00 00 00 02 04 06`

### Waveform 6 — Staircase / quantized

```
▅▅▅▅████▄▄▄▄▆▆▆▆▁▁▁▁▃▃▃▃    ▂▂▂▂
```

```
23 23 23 23 31 31 31 31 19 19 19 19 27 27 27 27 4 4 4 4 12 12 12 12 0 0 0 0 8 8 8 8
```

Raw hex: `17 17 17 17 1F 1F 1F 1F 13 13 13 13 1B 1B 1B 1B 04 04 04 04 0C 0C 0C 0C 00 00 00 00 08 08 08 08`

### Waveform 7 — Sine (near-pure)

```
▅▆████████████▆▅▂▁            ▁▂
```

```
23 27 29 30 30 31 31 31 31 31 31 30 30 29 27 23 8 4 2 1 1 0 0 0 0 0 0 1 2 3 4 9
```

Raw hex: `17 1B 1D 1E 1E 1F 1F 1F 1F 1F 1F 1E 1E 1D 1B 17 08 04 02 01 01 00 00 00 00 00 00 01 02 03 04 09`

### Waveform 8 — Pulse (narrow variant)

```
██▄████▄▄▄▄█████                
```

```
31 31 16 31 31 31 31 16 16 16 16 31 31 31 31 31 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
```

Raw hex: `1F 1F 10 1F 1F 1F 1F 10 10 10 10 1F 1F 1F 1F 1F 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00`

### Waveform 9 — Triangle with harmonics

```
 ▁▂▃▄▅▆████▆██████▆▅▄▃▂▁  ▁▂▁   
```

```
0 4 8 12 16 20 24 28 31 31 28 24 28 31 31 31 31 28 24 20 16 12 8 4 0 0 4 8 4 0 0 0
```

Raw hex: `00 04 08 0C 10 14 18 1C 1F 1F 1C 18 1C 1F 1F 1F 1F 1C 18 14 10 0C 08 04 00 00 04 08 04 00 00 00`

### Waveform 10 — Complex / noisy harmonic

```
 ▄███████████▃   ▂▄▆███▄        
```

```
0 16 31 30 29 30 28 30 29 30 28 29 30 15 1 0 0 8 16 24 31 31 31 18 1 2 3 2 3 1 0 0
```

Raw hex: `00 10 1F 1E 1D 1E 1C 1E 1D 1E 1C 1D 1E 0F 01 00 00 08 10 18 1F 1F 1F 12 01 02 03 02 03 01 00 00`

---

## Usage in SFX Streams

Waveforms are selected in music/SFX data streams using the `$F0 nn` command byte:

```
F0 00   ; use waveform 0 (pulse)
F0 03   ; use waveform 3 (sine)
F0 07   ; use waveform 7 (pure sine)
F0 0B   ; use waveform 11 (NULL — noise SFX, waveform not loaded)
```

The fire SFX (IDs 12, 13, 20, 23) uses `F0 0B` (waveform 11) which has a NULL pointer. Since these SFX immediately enable noise mode on channels 4/5, the waveform buffer contents are irrelevant — noise generation bypasses the waveform.

---

## Python Extraction Script

The following standalone Python script extracts all waveforms from the ROM:

```python
#!/usr/bin/env python3
"""
Extract PSG waveform data from Saint Dragon (Tenseiryuu) PC Engine ROM.

ROM: Tenseiryuu - Saint Dragon (Japan) (En).pce
CRC: 2E278CCB
Size: 393,216 bytes (384 KB, 48 banks)

The sound driver lives in bank $09 (ROM $12000-$13FFF, mapped to $4000-$5FFF).
Waveform pointer table is at logical $4647 (ROM $12647).
Each pointer is 2 bytes (lo/hi) pointing to 32 bytes of 5-bit sample data.
The waveform loading function at $4390 writes 32 samples to PSG register $0806.
"""

import sys

ROM_PATH = r'Tenseiryuu - Saint Dragon (Japan) (En).pce'

def extract_waveforms(rom_path):
    data = open(rom_path, 'rb').read()
    assert len(data) == 393216, f"Unexpected ROM size: {len(data)}"

    # Waveform pointer table: logical $4647, ROM $12647
    # Bank $09 maps to $4000, so ROM offset = logical - $4000 + $12000
    POINTER_TABLE_ROM = 0x12647
    BANK_09_BASE = 0x12000  # ROM offset for logical $4000
    BANK_0A_BASE = 0x14000  # ROM offset for logical $6000

    waveforms = []
    for i in range(32):  # scan up to 32 potential entries
        offset = POINTER_TABLE_ROM + i * 2
        lo = data[offset]
        hi = data[offset + 1]
        ptr = (hi << 8) | lo

        # Valid pointers are in $4000-$7FFF (banks $09/$0A mapped range)
        if ptr < 0x4000 or ptr > 0x7FFF:
            break

        # Convert logical address to ROM offset
        if ptr < 0x6000:
            rom_ptr = ptr - 0x4000 + BANK_09_BASE
        else:
            rom_ptr = ptr - 0x6000 + BANK_0A_BASE

        # Read 32 samples, mask to 5 bits
        samples = [data[rom_ptr + s] & 0x1F for s in range(32)]
        raw_bytes = [data[rom_ptr + s] for s in range(32)]
        waveforms.append({
            'index': i,
            'pointer': ptr,
            'rom_offset': rom_ptr,
            'samples': samples,
            'raw_bytes': raw_bytes,
        })

    return waveforms


def print_waveforms(waveforms):
    print(f"Found {len(waveforms)} waveforms\n")

    for wf in waveforms:
        idx = wf['index']
        samples = wf['samples']
        sample_str = ' '.join(str(s) for s in samples)
        hex_str = ' '.join(f'{b:02X}' for b in wf['raw_bytes'])

        # Mini ASCII visualization
        mini_viz = ""
        for s in samples:
            level = s * 8 // 32
            mini_viz += " ▁▂▃▄▅▆█"[level]

        print(f"Waveform {idx:2d} (${wf['pointer']:04X}, ROM ${wf['rom_offset']:05X}):")
        print(f"  Visual: {mini_viz}")
        print(f"  Values: {sample_str}")
        print(f"  Hex:    {hex_str}")
        print()


if __name__ == '__main__':
    if len(sys.argv) > 1:
        ROM_PATH = sys.argv[1]
    waveforms = extract_waveforms(ROM_PATH)
    print_waveforms(waveforms)
```

### How to run

```bash
python extract_waveforms.py "Tenseiryuu - Saint Dragon (Japan) (En).pce"
```

---

## Key Addresses Reference

| What | Logical | ROM Offset | Bank |
|------|---------|------------|------|
| Sound driver start | `$4000` | `$12000` | `$09` |
| Waveform load function | `$4390` | `$12390` | `$09` |
| `STA $0806` (waveform write) | `$43A6` | `$123A6` | `$09` |
| Waveform pointer table | `$4647` | `$12647` | `$09` |
| Waveform 0 data | `$49B7` | `$129B7` | `$09` |
| Waveform 10 data (last) | `$4AF7` | `$12AF7` | `$09` |
| Frequency lookup table | `$4521` | `$12521` | `$09` |
| SFX pointer table | `$45B1` | `$125B1` | `$09` |
