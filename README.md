# STM32 GPS Location Tracker

An embedded system that reads raw satellite data from a **NEO-6M GPS module** and extracts the current
**latitude, longitude and UTC time** on an **STM32H743** microcontroller. Firmware is written in C with the STM32 HAL.

The project was built to get hands-on experience with embedded hardware: wiring real components, receiving a continuous
UART data stream without losing characters, parsing it on the microcontroller, and debugging on limited hardware.

## Hardware

| Component | Role |
|---|---|
| STM32H743VIT6 development board | Runs the firmware, receives and parses the GPS stream |
| GY-NEO6MV2 (NEO-6M GPS + ceramic antenna) | Locks onto satellites and outputs NMEA sentences over UART at 9600 baud |
| ST-Link V2 (clone) | Flashing and debugging over SWD |

### Wiring

**NEO-6M → STM32 (USART3)**

| NEO-6M | STM32 |
|---|---|
| VCC | 3.3 V |
| GND | GND |
| TX | PD9 (USART3_RX) |
| RX | PD8 (USART3_TX) |

**ST-Link V2 → STM32 (SWD header)**

| ST-Link | STM32 |
|---|---|
| 3.3V | 3.3 V reference |
| GND | GND |
| SWDIO | SWDIO |
| SWCLK | SWCLK |
| RST | NRST |

## How it works

```
NEO-6M ──UART (9600)──> USART3 RX interrupt ──> Ring buffer ──> Wait_for("GGA") ──> Copy_upto("*") ──> get_nmea_field()
                                                (background)                                            time / lat / lon
```

1. **Interrupt-driven reception.** The GPS module sends characters continuously and does not wait for the
   microcontroller. Plain UART polling dropped characters, so reception runs in the background: `Ringbuf_init()` enables
   the RXNE interrupt and every incoming byte is written into a circular buffer by the interrupt handler, independently of
   the main loop.
2. **Filtering NMEA sentences.** The module outputs several sentence types (`$GPGLL`, `$GPGSA`, `$GPGSV`, ...). The main
   loop only needs `$GPGGA`, which carries position and fix data. `Wait_for("GGA")` scans the buffer for that header and
   `Copy_upto("*", ...)` copies the rest of the sentence (up to the checksum) into a local array.
3. **Parsing.** `get_nmea_field()` walks through the sentence, counts commas and returns the field at a given index.
   The main loop uses it to fill `time`, `latitude`, `latitude_direction`, `longitude` and `longitude_direction`.

```c
get_nmea_field(gps_ham_veri, 1, time);               // hhmmss.ss (UTC)
get_nmea_field(gps_ham_veri, 2, latitude);           // ddmm.mmmmm
get_nmea_field(gps_ham_veri, 3, latitude_direction); // N / S
get_nmea_field(gps_ham_veri, 4, longitude);          // dddmm.mmmmm
get_nmea_field(gps_ham_veri, 5, longitude_direction);// E / W
```

The parsed values are also formatted as `LAT:... LON:... TIME:...` and sent out on USART1 (see
[Limitations](#limitations-and-future-work)).

## Results

The system was tested outdoors, since the GPS module needs a clear view of the sky (satellite lock took about 4–5
minutes). The `$GPGGA` sentence was correctly isolated and split into fields, for example:

| Field | Raw value | Meaning |
|---|---|---|
| Time | `125351.00` | 12:53:51 UTC |
| Latitude | `41xx.xxxxx N` | ≈ 41.74° N |
| Longitude | `022xx.xxxxx E` | ≈ 22.20° E |

Converted from NMEA's degrees-and-minutes format to decimal degrees, the coordinates point to Štip, North Macedonia,
where the test was carried out. A small offset from the exact test location was observed, which is expected from a
low-cost GPS module.

Values were verified in STM32CubeIDE by halting at breakpoints and reading the variables in the *Expressions* view.

## Debugging notes

Most of the project time went into hardware-level debugging with a clone ST-Link rather than into the C code itself.
What finally made the setup stable:

- **Flashing errors** (`Failed to execute MI command`, `vFlashErase`): switched the debug probe to OpenOCD, physically
  wired the RST pin, set the reset mode to *Software System Reset* and lowered the SWD frequency to **480 kHz**
  (4 MHz was not stable with this probe).
- **Lost synchronization with the core:** updated the clone ST-Link's firmware (took several attempts) and added
  OpenOCD startup commands to halt and sync before running:
  ```
  monitor reset
  monitor halt
  monitor gdb_sync
  ```
- **Empty "Live Expressions" view:** the clone probe cannot read memory while the core is running. Variables are only
  readable when execution is halted, so verification was done with breakpoints and the regular *Expressions* view.
- **No serial output:** the clone ST-Link has no Virtual COM Port, so `printf`/UART output cannot be viewed through it.
  An external USB-to-TTL converter is needed for that.

## Build and flash

1. Open the project in **STM32CubeIDE** (the peripheral configuration is in `gps2.ioc`).
2. Build the `Debug` configuration.
3. Flash over SWD with an ST-Link. With a clone probe, use the OpenOCD settings from [Debugging notes](#debugging-notes).
4. Take the setup outdoors and wait for the GPS module's LED to start blinking (satellite lock), then run.

## Project structure

```
Core/Src/main.c            main loop, UART initialization, NMEA field parser (get_nmea_field)
Core/Src/UartRingbuffer.c  interrupt-driven UART ring buffer (Wait_for, Copy_upto, ...)
gps2.ioc                   STM32CubeMX peripheral configuration
Presentation/              project report with hardware photos and IDE screenshots
```

## Limitations and future work

- **Serial output not yet verified:** the formatted coordinates are sent on USART1, but no USB-to-TTL converter was
  available during the project, so this output could not be viewed on a PC terminal.
- **No fix-quality check:** fields are read even before the module has a satellite fix; the GGA fix-quality field
  could be used to skip invalid sentences.
- **No checksum validation:** the NMEA checksum after `*` is not verified.
- **Raw format:** coordinates stay in NMEA's degrees-and-minutes format; converting them to decimal degrees on the
  device would make the output directly usable on a map.

## Credits

The UART ring buffer (`UartRingbuffer.c/.h`) is based on the library by
[Controllerstech](https://controllerstech.com/). The NMEA sentence filtering in the main loop and the
`get_nmea_field()` parser were written for this project.

## Author

Arif Taha Bülbül — [LinkedIn](https://www.linkedin.com/in/arif-taha-b%C3%BClb%C3%BCl-824a451b4/)
