# Rhooti Smart Traffic Light Controller

**CS 240 | Micro Architecture & Assembly Language | Cuesta College | Dr. Dube**  
`8086 Assembly` `EMU8086` `Virtual Device` `Collaborative Project`

---

## Overview

An 8086 assembly program that simulates a fully operational 4-way smart traffic light controller using the EMU8086 Traffic Light virtual device. The controller manages a complete intersection across all four directions (North, South, East, West) and responds in real time to pedestrian crossing requests and emergency vehicle overrides.

---

## How It Works

The controller runs a continuous main loop cycling through six phases:

```
NS Green -> NS Yellow -> All Red -> EW Green -> EW Yellow -> All Red -> repeat
```

After every phase, the program performs a non-blocking keyboard check. If a flag is set, the program responds immediately without waiting through the next phase.

### Modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| Normal Cycle | Automatic | Loops Green, Yellow, Red with timed delays |
| Pedestrian | `P` or `p` | Brief All Red flash, then extended EW Green for safe crossing |
| Emergency | `E` or `e` | Forces All Red immediately, holds for extended duration |

### BH Flag System

```
BH = 0   normal cycle
BH = 1   pedestrian waiting
BH = 2   emergency override
```

---

## Port & Bit Map

The program communicates with the traffic light hardware via **Port 4** using the `OUT` instruction. The device uses a 12-bit layout across four sides of the intersection:

| Side | Red | Yellow | Green |
|------|-----|--------|-------|
| East | Bit 0 | Bit 1 | Bit 2 |
| North | Bit 3 | Bit 4 | Bit 5 |
| West | Bit 6 | Bit 7 | Bit 8 |
| South | Bit 9 | Bit 10 | Bit 11 |

### Hex States

```
NS Green  + EW Red    = 030Ch
NS Yellow + EW Red    = 028Ah
All Red               = 0249h
EW Green  + NS Red    = 0861h   (also used for Pedestrian)
EW Yellow + NS Red    = 0451h
```

---

## Non-Blocking Input

Keyboard input uses DOS service `06h` with `DL=FFh` to check for a waiting key without halting execution. The timer never freezes.

```asm
MOV AH, 06h     ; direct console I/O, non-blocking
MOV DL, 0FFh    ; check mode: returns key if waiting, sets ZF if not
INT 21h         ; ZF set = no key, AL = key if pressed
```

---

## Procedures

| Procedure | Description |
|-----------|-------------|
| `MAIN` | Main loop, phase sequencing, flag checks |
| `CHECK_INPUT` | Non-blocking keyboard read, sets BH flag |
| `NS_GREEN` | North-South green, East-West red |
| `NS_YELLOW` | North-South yellow, East-West red |
| `ALL_RED` | All directions red, safe transition state |
| `EW_GREEN` | East-West green, North-South red |
| `EW_YELLOW` | East-West yellow, North-South red |
| `PEDESTRIAN_MODE` | Extended crossing with All Red flash |
| `EMERGENCY_MODE` | Immediate All Red, extended hold |

---

## Running the Project

1. Open **EMU8086**
2. Load `Micro Traffic Light Procedure Project.asm`
3. Open the **Traffic Light** virtual device under peripherals
4. Run the program
5. Press `P` or `p` for pedestrian crossing
6. Press `E` or `e` for emergency override

---

## License

BSD 3-Clause License. See `LICENSE` for details.



