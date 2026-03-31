# STM32F401 SD Card Read/Write
### SPI1 + FatFS + UART Debug  |  STM32 IDE

---

## Wiring

```
SD Card Module       STM32F401 (Black Pill / Nucleo)
─────────────        ──────────────────────────────
VCC          ──────► 3.3V
GND          ──────► GND
CS           ──────► PA4
MOSI         ──────► PA7   (SPI1_MOSI)
MISO         ──────► PA6   (SPI1_MISO)
SCK          ──────► PA5   (SPI1_SCK)

USB-UART Adapter     STM32F401
────────────────     ─────────
RX           ──────► PA9   (USART1_TX)
GND          ──────► GND
```

> **Note:** Use a proper 3.3V SD card module (with level shifter if your
> SD card module is 5V).

---

## Files to Add to Your STM32 IDE Project

```
YourProject/
├── Core/
│   ├── Src/
│   │   ├── main.c          ← replace generated main.c
│   │   ├── sd_spi.c
│   │   ├── uart_debug.c
│   │   ├── file_io.c
│   │   └── diskio.c        ← FatFS glue
│   └── Inc/
│       ├── sd_spi.h
│       ├── uart_debug.h
│       ├── file_io.h
│       └── ffconf.h        ← FatFS config
└── Middlewares/
    └── Third_Party/
        └── FatFS/          ← Download from elm-chan.org/fsw/ff
            ├── ff.c
            ├── ff.h
            └── diskio.h
```

---

## Setup Steps

### 1. Download FatFS
- Go to: https://elm-chan.org/fsw/ff/00index_e.html
- Download the latest source zip
- Copy `ff.c`, `ff.h`, `diskio.h` into your project's FatFS folder
- Add the folder to your include paths in STM32 IDE:
  Project → Properties → C/C++ Build → Settings → Include Paths

### 2. Add source files to build
In STM32 IDE, right-click each `.c` file → Resource Configurations →
make sure "Exclude from build" is NOT checked.

### 3. If using CubeMX
- Enable SPI1 in Full-Duplex Master mode
- Enable USART1 in Async mode (115200, 8N1)
- Set PA4 as GPIO_Output
- Do NOT enable FatFS in CubeMX middleware (we handle it manually)
- Replace the generated `main.c` with the provided one, or copy
  the `SD_Demo()` call and peripheral inits into your generated main

### 4. Linker / compiler flags
Add to your project's C symbols if not already present:
```
USE_HAL_DRIVER
STM32F401xE   (or STM32F401xC depending on your chip)
```

### 5. SD Card format
Format the SD card as **FAT32** on your PC before use.
Cards > 32GB may need a third-party formatter (e.g. SD Card Formatter
from sdcard.org) to force FAT32.

---

## Expected UART Output (115200 baud, open with PuTTY / CoolTerm)

```
[INFO ] main.c:42  ========================================
[INFO ] main.c:43    STM32F401 SD Card Demo
[INFO ] main.c:44    SPI1  |  USART1 @ 115200
[INFO ] main.c:45  ========================================
[INFO ] diskio.c:18  diskio: disk_initialize
[INFO ] sd_spi.c:95  SD: Starting initialisation...
[INFO ] sd_spi.c:104 SD: CMD0 OK – card idle
[INFO ] sd_spi.c:138 SD: Init OK – card type: SDv2 [SDHC/SDXC]
[INFO ] file_io.c:28 FS_Mount OK – filesystem mounted
[INFO ] file_io.c:96 FS_ListDir: root directory contents:
[INFO ] file_io.c:97   (empty)
[INFO ] main.c:71  --- TXT demo ---
[INFO ] file_io.c:44 FS_WriteText: 'README.TXT' (79 bytes)
[INFO ] file_io.c:57 FS_WriteText: wrote 79 bytes OK
[INFO ] file_io.c:72 FS_ReadText: opening 'README.TXT'
[INFO ] file_io.c:83 FS_ReadText: read 79 bytes from 'README.TXT'
[INFO ] main.c:78  README.TXT contents (79 bytes):
Hello from STM32F401!
This file was written via SPI SD card + FatFS.
UART debug output is on PA9 @ 115200 baud.
...
[INFO ] main.c:67  Demo complete.
```

---

## Speeding Up SPI After Init

SD cards initialise at ≤400 kHz but can run much faster after that.
After `SD_Init()` returns, add this in `main.c`:

```c
if (SD_Init()) {
    // Speed up SPI to ~10.5 MHz (APB2=42MHz / 4)
    hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_4;
    HAL_SPI_Init(&hspi1);
}
```

---

## Changing the Log Level

In `uart_debug.h`, change:
```c
#define DEBUG_LEVEL   LOG_INFO   // only INFO and ERROR
#define DEBUG_LEVEL   LOG_DEBUG  // everything
#define DEBUG_LEVEL   LOG_NONE   // silent
```

---

## Logging Sensor Data Periodically

```c
char row[64];
snprintf(row, sizeof(row), "%lu,%d,%d\r\n",
         HAL_GetTick(), temperature, rpm);
FS_AppendText("LOG.CSV", row);
```

Call this in your main loop or timer callback every N milliseconds.
