# Arduino-Teensy 4.1

Drawings and bring-up sketches for the BurgessWorld Custom Electronics Teensy 4.1 board. The r10 schematic title block names Chris Burgess as the designer. This repository is not an Arduino library and has no build files.

The sketches below are stored sources. This page describes those sources and the text of the r10 schematic. It is not a record of a board passing the tests.

Git history is a series of "Add files via upload" commits. r12 photos arrived on 2022-06-20. There is no changelog that says what changed between r10 and r12.

## Drawings

| File | Contents |
| --- | --- |
| `Arduino-Teensy41_PLUS_ESP_r10_SCH.pdf` | One-page schematic. Title block says rev 1. |
| `Arduino-Teensy41_PLUS_ESP_r10_BRD.pdf` | Board plot. Extractable text is the Teensy 4.1, USB, JP1, and J3 labels. |
| `Arduino-Teensy41_PLUS_ESP_r10_TOP.png`, `..._BOTTOM.png` | r10 assembly photos. |
| `Arduino-Teensy41_PLUS_ESP_r10_Dimensions.bmp`, `..._Dimensions.jpg` | r10 mechanical drawing. |
| `Arduino-Teensy41_PLUS_ESP_r12_TOP.png`, `..._BOTTOM.png` | r12 photos. |
| `Arduino-Teensy41.jpg` | Board photo. |

r12 in this repo is photos only. Use the r10 schematic for nets and part names.

## r10 schematic constraints

Application notes printed on the sheet:

- TEENSY INPUTS USE 3.3V LOGIC-LEVEL
- THEY ARE NOT 5V TOLERANT
- DO NOT PUT 5V INTO ANY PIN

The power section is labeled `INPUT 5V REGULATION 3v3`. The sheet names regulators `ZLDO1117G33` and `NCP1117ST50T3G`, and a `CR1220`.

CAN: two `MCP2558FD-H_SN` transceivers. Each transceiver block on the sheet lists 1k, 2k, and 120 ohm. Net names include `CAN1H`, `CAN1L`, `CAN3H`, `CAN3L`, `CRX1_H`, and `CRX3_H`. Teensy pin labels include `D22/A8/CTX1`, `D23/A9/CRX1/MCLK1`, `D30/CRX3`, and `D31/CTX3`.

ESP-style module pins on the sheet are EN, IO0, IO2, TX, RX, and RST. Nets include `ESP-TX`, `ESP-RX`, `ESP-RST`, `ESP-PGM`, `PROG_ESP`, `ESP_EN`, and `ESP_GPIO2`. The LED sketch, not the schematic text, calls this module an ESP8266.

Serial-LED nets are `RX1L` and `TX1L`. Color words on the sheet are red, green, amber, and yellow. The sheet does not say which net has which color.

`HC-KEY` is a net name. The schematic text does not name an HC-05 or HC-06 module.

## Sketches

The three `.ino` files were added together on 2021-05-19. Open them with Teensyduino. This repo does not select a board or install libraries.

### LEDs and ESP — `Arduino_Teensy4_test_script.ino`

USB serial is 115200. Setup prints `Started` and continues without waiting for the serial monitor, so those lines are easy to miss if the monitor opens late.

Once, before `Serial1.begin(115200)`:

1. Pin 13 is an output.
2. Pin 1 (the sketch's TX1 LED): `OUTPUT`, `LOW` for 2 seconds, `HIGH`, then `INPUT`.
3. Pin 0 (the sketch's RX1 LED): the same sequence.

The file header says the serial-port LEDs are active-low, and that TX1/RX1 are checked once because the Teensy does not handle disabling and re-enabling serial ports correctly. The loop leaves Serial1 running and does not repeat that LED check.

Each loop pass:

- Pin 13 `HIGH` for 2 seconds, then `LOW`. The header calls this LED active-high.
- Pins 30 and 31 go `HIGH` together for 2 seconds, then `LOW`. Setup already set both as outputs.
- `Serial1.println("AT+GMR")`, then wait up to 5000 ms for the substring `OK`. Bytes received during the wait are copied to the USB serial monitor. A match prints `....Success`. Otherwise the sketch prints `Failed`. It then prints `Test Complete` and discards remaining Serial1 bytes.

The header says to press RESET to run the sketch again.

Pitfalls in this file:

- The header says pin D24 is the Bluetooth STATE pin and is active-high. The writes that follow the `Bluetooth STATE pin` print are `digitalWrite(30, ...)` and `digitalWrite(31, ...)`, and both of those lines are commented as D24. The sketch never writes pin 24. On the r10 schematic, D24 is labeled `D24/A10/TX6/SCL2`. D30 and D31 are labeled `D30/CRX3` and `D31/CTX3`.
- `char trash = ""` and `char rdChar = ""` are invalid C++. `g++` rejects them with `invalid conversion from 'const char*' to 'char'`.
- Success is the substring `OK` inside the 5 second window. The sketch does not decode an AT error reply.

### CAN — `TEENSY_4.1_CAN_TEST_2.ino`

The header says this is a Teensy 4.0 triple-CAN demo for the SK Pang board (October 2019), that CAN1 and CAN2 are CAN 2.0B, and that CAN3 is CAN FD. The filename says 4.1. At runtime the sketch prints `Teensy 4.0 Triple CAN test`.

The controllers it starts:

- `FlexCAN_T4<CAN1, RX_SIZE_256, TX_SIZE_16>` at 500000 bit/s, with FIFO, a FIFO interrupt, and callback `canSniff20`.
- `FlexCAN_T4FD<CAN3, RX_SIZE_256, TX_SIZE_16>` with `CLK_24MHz`, arbitration rate 500000, data rate 2000000, `propdelay` 190, `bus_length` 1, `sample` 75, `setRegions(64)`, and `setBaudRateAdvanced(config, 1, 1)`.

The sketch creates no CAN2 object.

CAN3 filter calls, in this order: `ACCEPT_ALL`, mailbox 13 accepts id `0x1`, mailbox 12 accepts `0x1` and `0x3`, mailbox 8 accepts the range `0x1` through `0x04`. Interrupts are enabled on mailboxes 8, 12, and 13. `enhanceFilter` is called for mailboxes 8 and 10. Then `distribute()` and `mailboxStatus()`.

`timer.begin(sendframe, 500000)` matches the comment "Send frame every 500ms". Each tick:

- Fills an 8-byte frame, sets the id to `0x401`, then sets the id to `0x402` and byte 1 to an incrementing counter, and passes that frame to `can1.write`. The id in the frame that is written is `0x402`.
- Fills a 64-byte CAN FD frame, id `0x4fd`, `seq` 1, byte 0 set to the same counter, and writes it on CAN3.
- Both `write` return values are ignored.

`loop` calls `can1.events()` and `FD.events()`. It also calls `FD.readMB`, so one CAN3 frame can be printed from `canSniff` and printed again from the mailbox read.

Setup sets pin 13 `HIGH`, waits 1 second, then sets pins 13, 34, and 35 `LOW`. Pins 34 and 35 stay low. The r10 schematic labels them `D34/RX8` and `D35/TX8`. The sketch does not say what that level is for.

The header requires FlexCAN_T4: <https://github.com/tonton81/FlexCAN_T4>.

### USB host and SD — `Teensy4_Mouse_and_SD_card_v1.ino`

The file says the USB host example is in the public domain. It includes `SD.h`, `SPI.h`, and `USBHost_t36.h`, and adds an SD readback.

`setup` stays in `while (!Serial)` until a serial monitor is open. With no monitor, USB host and the SD check never start.

The card is `BUILTIN_SDCARD`. When `SD.begin` fails, setup prints `initialization failed!` and returns. `myusb.begin()` is after that return, so a failed `SD.begin` skips the USB host.

`CheckSD` opens `test.txt` with `FILE_WRITE`, writes `testing 1, 2, 3.`, closes the file, opens it again, and copies it to the serial monitor. The function is declared `bool` and has no `return`. Setup calls it after a successful `SD.begin`. A mouse button value of exactly `1` calls it again. Any other button value does not.

Host objects created: two hubs, two keyboards, one joystick, `BluetoothController bluet(myusb)`, five HID parsers, one mouse, and two raw HID controllers. `rawhid2` uses usage `0xffc90004`. The pairing constructor is commented out. The live constructor's comment says it assumes the device is already paired.

The first character of a serial-monitor line:

| Character | Effect |
| --- | --- |
| `b` or `B` | Joystick axis-change mask `0x3ff`. Prints `Only notify on Basic Axis changes`. |
| `f` or `F` | Full axis mask. Prints `Only notify on Full Axis changes`. |
| anything else | Toggles all fields vs changed fields only. |

The rest of that line is discarded. When `rawhid1` evaluates true, `loop` reads the serial monitor again and sends those bytes with `rawhid1.sendPacket`. The table above has already consumed the line, so that keypress is a mode change and is not also sent as a raw HID packet.

Joystick rumble uses the trigger axes this sketch reads:

| Type | Axes | Call |
| --- | --- | --- |
| PS4 | 3 and 4 | `setRumble(ltv, rtv)` |
| PS3 | 18 and 19 | `setRumble(ltv, rtv, 50)` |
| Xbox One, Xbox 360 | 4 and 5 | `setRumble(ltv, rtv)`, and the sketch prints `Set Rumble` |

On a PS3 button change the sketch calls `setLEDs((buttons >> 12) & 0xf)`. On any other joystick type it passes button bits 0, 1, and 2 to `setLEDs` as `0x00` or `0xff`.

A keyboard on `keyboard1` whose vendor id is `0x04D9` is forced into the boot protocol. The sketch comment calls that vendor Gigabyte. The unconditional `keyboard1.forceBootProtocol()` call is commented out. Its comment says forcing boot protocol on a combined wireless keyboard and mouse can stop the mouse working.
