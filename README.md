# SPI EEPROM Simulator — 25LC256

A browser-based interactive simulator for the **Microchip 25LC256** SPI EEPROM chip. Built with vanilla HTML, CSS, and JavaScript — no dependencies, no build step, just open and run.

Designed for students, embedded engineers, and anyone who wants to understand how SPI EEPROM communication actually works at the signal level.

---

## What It Simulates

The **25LC256** is a real 32KB SPI EEPROM. This simulator models a simplified 256-byte version of it, faithfully replicating:

- The SPI instruction set (WREN, WRDI, RDSR, WRITE, READ)
- The status register with all its protection bits
- The write-protect locking mechanism (WEL must be set before any write)
- The WIP (Write In Progress) timing flag that auto-clears after a write
- Real SPI bus signal behavior across all four lines: CS, CLK, MOSI, MISO

---

## How to Run

No installation needed.

```bash
# Just open the file in any modern browser
open spi-eeprom-sim.html
```

Or drag the file into Chrome, Firefox, or Edge. That's it.

---

## Interface Overview

The simulator is split into three panels:

```
┌──────────────────┬─────────────────────────────────┐
│                  │   OSCILLOSCOPE (waveforms)       │
│  COMMAND PANEL   ├─────────────────────────────────┤
│  + STATUS REG    │   MEMORY EDITOR (hex grid)       │
│  + TX LOG        │                                  │
└──────────────────┴─────────────────────────────────┘
```

### Command Panel (left)
Where you send instructions to the simulated EEPROM. Select a command, enter hex values for address and data, then click **EXECUTE**.

### Oscilloscope (top right)
Shows the four SPI bus signals rendered as a timing diagram after every transaction. Each byte phase is labeled (OPCODE, ADDR\_HIGH, ADDR\_LOW, DATA, DUMMY). Individual bit values are annotated on MOSI and MISO lines.

### Memory Editor (bottom right)
A 16×16 hex grid showing all 256 bytes of EEPROM memory. Click any cell to edit it directly. Non-zero cells are highlighted. ASCII representation is shown on the right side of each row.

---

## SPI Instruction Set

### `WREN` — Write Enable (opcode `0x06`)
Must be called **before every WRITE**. Sets the WEL (Write Enable Latch) bit in the status register to 1. The EEPROM ignores all write commands until this is issued.

```
CPU  →  EEPROM :  0x06
EEPROM  →  CPU :  (nothing, MISO stays 0)
```

### `WRDI` — Write Disable (opcode `0x04`)
Clears WEL back to 0. The EEPROM becomes write-protected again.

```
CPU  →  EEPROM :  0x04
EEPROM  →  CPU :  (nothing)
```

### `RDSR` — Read Status Register (opcode `0x05`)
Reads the 8-bit status register. The CPU sends the opcode, then clocks out a dummy byte — during which the EEPROM drives the status byte back on MISO.

```
CPU  →  EEPROM :  0x05  0xFF (dummy)
EEPROM  →  CPU :  0x00  0xXX ← status byte appears here on MISO
```

### `WRITE` — Write Byte (opcode `0x02`)
Writes one byte to a 16-bit address. Requires WEL=1 first, or the command is silently ignored. After the write completes, WEL is automatically cleared and WIP pulses high for ~800ms (simulated write time).

```
CPU  →  EEPROM :  0x02  ADDR_H  ADDR_L  DATA
EEPROM  →  CPU :  (MISO stays flat at 0 — EEPROM only listens during write)
```

### `READ` — Read Byte (opcode `0x03`)
Reads one byte from a 16-bit address. No WEL needed. The CPU sends opcode + address, then clocks out a dummy `0xFF` byte — the EEPROM responds with the stored data on MISO during that final byte.

```
CPU  →  EEPROM :  0x03  ADDR_H  ADDR_L  0xFF (dummy)
EEPROM  →  CPU :  0x00  0x00    0x00    0xXX ← data appears here on MISO
```

---

## The Correct Write → Read Workflow

```
1. WREN              → WEL = 1, EEPROM is now writable
2. WRITE 0x0005 0xC8 → Stores 0xC8 (200) at address 5, WEL clears, WIP pulses
3. READ  0x0005      → Returns 0xC8 on MISO — you see the bump on the blue line
```

If you skip step 1 (WREN) and go straight to WRITE, the transaction log will show `⚠ WEL not set — ignored` and nothing is stored.

---

## Status Register

An 8-bit register that describes the current state of the EEPROM.

| Bit | Name | R/W | Description |
|-----|------|-----|-------------|
| 0 | **WIP** | R | Write In Progress — 1 while an internal write is happening |
| 1 | **WEL** | R | Write Enable Latch — 1 after WREN, cleared after WRITE or WRDI |
| 2 | **BP0** | R/W | Block Protect bit 0 — protects upper memory regions from writes |
| 3 | **BP1** | R/W | Block Protect bit 1 — expands protection range when combined with BP0 |
| 7 | **WPEN** | R/W | Write Protect Enable — enables hardware write protection pin |

WIP and WEL are controlled by commands. BP0, BP1, and WPEN can be toggled directly in the UI by clicking them.

---

## Waveform Signals

| Signal | Color | Description |
|--------|-------|-------------|
| **~CS** | 🔴 Red | Chip Select — active LOW. Falls at start of transaction, rises at end |
| **CLK** | 🟢 Green | Clock — each rising edge shifts one bit |
| **MOSI** | 🟡 Amber | Master Out Slave In — data from CPU to EEPROM |
| **MISO** | 🔵 Blue | Master In Slave Out — data from EEPROM to CPU |

The waveform operates in **SPI Mode 0,0** (CPOL=0, CPHA=0): clock idles low, data sampled on rising edge.

### What to look for during READ

MISO stays flat (0) while the opcode and address are being sent. It only becomes active during the **last byte** (the dummy clock phase), where you'll see the bit pattern of the stored value. For `0xC8`:

```
MISO:  _ _ _ _ _ _ _ _  |  _ _ _ _ _ _ _ _  |  _ _ _ _ _ _ _ _  |  ‾ ‾ _ _ ‾ _ _ _
       ← OPCODE (idle) →    ← ADDR_H (idle) →    ← ADDR_L (idle) →    ← DATA = 0xC8 →
                                                                          1 1 0 0 1 0 0 0
```

---

## Memory Editor

A full 256-byte hex grid. Each row shows 16 bytes starting from the address on the left.

- **Click any cell** to edit it directly (bypasses SPI — direct memory access for convenience)
- **FILL 0x00** — clears all memory to zero
- **FILL RANDOM** — loads random bytes into all 256 addresses
- **Search** — enter a hex value to highlight all matching cells across the grid
- **ASCII column** — printable characters shown on the right; dots (·) for non-printable bytes

### Pre-loaded Demo Data

On startup, two values are pre-loaded so you can test immediately:

| Address | Value | Notes |
|---------|-------|-------|
| `0x0005` | `0xC8` | The worked example — try READ on this immediately |
| `0x0010–0x0014` | `48 65 6C 6C 6F` | Spells "Hello" in ASCII |

---

## Transaction Log

Every command is logged with a timestamp, color-coded by type:

- 🟢 **Green** — control commands (WREN, WRDI, BOOT)
- 🟡 **Amber** — write operations
- 🔵 **Blue** — read operations

The log scrolls automatically and keeps the last 100 transactions.

---

## Technical Notes

**Why does WRITE need WREN every time?**
The 25LC256 auto-clears WEL after every successful write as a safety mechanism — you can't accidentally corrupt memory with a stray write command. You must explicitly re-enable writing each time.

**Why does MISO show 0xFF on MOSI during READ?**
SPI is full-duplex — both lines are always active simultaneously. To receive data the CPU must generate clock pulses, so it sends `0xFF` as a throwaway dummy byte. The EEPROM ignores this value and simultaneously drives the real data back on MISO.

**Why does WIP pulse after WRITE?**
Real EEPROMs take 1–5ms to complete an internal write to flash memory. The WIP bit stays high during this time. The simulator models this with an 800ms timeout so you can see it change in the status register.

**Why is MISO flat during WRITE?**
WRITE is a one-way operation. The EEPROM is busy receiving and storing data — it has nothing to say back. MISO stays at 0 (tri-state / idle) for the entire transaction.

**SPI Mode 0,0 vs other modes**
This simulator uses Mode 0,0 (the most common). The 25LC256 also supports Mode 1,1. The difference is only in when the clock idles and which edge samples data — the byte values transmitted are identical either way.

---

## File Structure

```
spi-eeprom-sim.html   — single self-contained file, no external dependencies
README.md             — this file
```

All fonts are loaded from Google Fonts (Share Tech Mono + Orbitron). If offline, the browser falls back to a system monospace font and the layout remains fully functional.

---

## Browser Compatibility

| Browser | Status |
|---------|--------|
| Chrome 90+ | ✅ Full support |
| Firefox 88+ | ✅ Full support |
| Edge 90+ | ✅ Full support |
| Safari 15+ | ✅ Full support |
| IE 11 | ❌ Not supported |

Requires Canvas 2D API and ES6+ (arrow functions, template literals, `Uint8Array`).

---

## Possible Extensions

Ideas for taking this further:

- **Page write mode** — real 25LC256 supports writing up to 64 bytes in one transaction
- **SPI Mode 1,1 toggle** — switch between CPOL/CPHA modes and see the waveform shift
- **Sequential read** — hold CS low and keep clocking to read consecutive addresses automatically
- **Memory import/export** — save and load `.bin` files to/from real hardware dumps
- **Write protect simulation** — tie BP0/BP1 to actual address range restrictions
- **Timing annotations** — show clock frequency, bit period, and transaction duration on the waveform

---

## References

- [Microchip 25LC256 Datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/25AA256-25LC256-256-Kbit-SPI-Bus-Serial-EEPROM-20001203N.pdf)
- [SPI Bus — Wikipedia](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface)
- [EEPROM — Wikipedia](https://en.wikipedia.org/wiki/EEPROM)
