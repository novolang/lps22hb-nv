# lps22hb-nv

A driver for ST's LPS22HB and LPS22HH, two absolute pressure sensors with
an I2C and an SPI interface.  The package drives both with one register
map and a switch on WHO_AM_I (0xB1 for the LPS22HB, 0xB3 for the LPS22HH).
It implements the `Barometer`, `Thermometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv).

The LPS22HB facts are from ST's "LPS22HB MEMS nano pressure sensor:
260-1260 hPa absolute digital output barometer" data sheet, DocID027083
Rev 5, November 2016 (the document ST now numbers DS10829).  The one
statement taken from an application note, that the LPS22HB's low-pass
filter is reset by reading LPFP_RES, is the data sheet's own (section 9.23)
and AN4833's, which was seen as a search excerpt only.  The LPS22HH facts are from ST's register header `lps22hh_reg.h`
(the `lps22hh-pid` repository of STMicroelectronics) and from Zephyr's
`lps22hh` driver.  **The LPS22HH data sheet (DS12754 in the brief of this
package) was not read:** ST's server did not answer from the machine the
package was written on, and the copies found elsewhere could not be
opened.  Every LPS22HH figure that the header and the driver do not give
is marked "not read" at its site in `src/lps22hb.nv`.  AN4672, the
hardware guidelines the brief of this package names, was not opened.

## Install

```sh
novo pkg add lps22hb-nv
```

## Example

A device program reads the part over I2C.  The breakout's SDA and SCL go
to the board's I2C pins, its CS pin to the supply, which selects I2C, and
its SDO/SA0 pin to ground or nothing, which puts the part at 0x5C.  The
program waits for the part to start, takes the register map from
WHO_AM_I, brings the part up with the part powered down, and four times a
second starts one conversion, waits for it to finish and prints one line
over RTT.  At sea level the pressure is about 101 325 000 millipascals.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use lps22hb
use lps22hb.{ Lps22 }

// The part's address with the SDO/SA0 pin low or open.
const ADDR = 0x5C

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw]
    var part = Lps22.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    hal.timer.delay_ms(10)
    if not part.begin()
        hal.uart.write("lps22: no answer\n")
        return
    loop
        match part.sample_once(4000)
            Some(s) =>
                hal.uart.write("lps22")
                field(" mC=", s.milli_c)
                field(" mPa=", s.milli_pa)
                hal.uart.write("\n")
            None =>
                hal.uart.write("lps22: no result\n")
        hal.timer.delay_ms(250)
```

The program is `tests/device_read.nv`.  Build it for a board with:

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

| Board | Flash | Of which the program | RAM at start |
|---|---|---|---|
| nRF52840 DK | 18,024 bytes | 7,428 bytes | 3,252 bytes |
| nRF54LM20 DK | 19,088 bytes | 7,666 bytes | 3,236 bytes |
| FRDM-MCXN947 | 20,032 bytes | 9,174 bytes | 2,748 bytes |

The Nordic DKs drive I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA P0.26
and SCL P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04) and SPI
with a SPIM, on the pins `docs/hal/README.md` names under "The Nordic DKs'
buses", so the images carry the driver's reading path.  The program uses
the library's integer arithmetic only; the driver's own code allocates
nothing, every buffer is a fixed-size `Vec` in the caller's frame.

**Hardware: NOT VERIFIED.**  No board carries an LPS22HB or an LPS22HH
yet.  The program builds, runs under QEMU to its refusal (the emulated
board has no I2C device), and the driver's functions run over a double of
the part that follows the register maps above.  A reply from a part over a
board's bus has not been taken.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The conversions are pure.  The driver performs no input or
output of its own and allocates nothing of its own.  The bus functions of
devices-nv build one short frame per transaction inside a `system` block
and release it when the call returns.  The module checks at every tier,
`embedded` included; the two exceptions are the host-scheduler async impls
at the end of the module, which carry `@tier(app)` on their methods, and
`drdy_arm`, which takes the effect of the pin-edge source it is handed.

## What it is

A barometer measures air pressure.  Air pressure at sea level is about
101 325 pascals (Pa), and falls by about 12 Pa for each metre of height
near the ground.  The package gives no altitude: that needs the pressure at
sea level where the board is, which changes with the weather.

Both parts measure 260 to 1260 hPa.  The pressure is a 24-bit two's
complement count with 4096 counts a hectopascal, so one count is 24.4 mPa.
The temperature is a 16-bit two's complement count with 100 counts a degree
Celsius.  The trait answers whole pascals, rounded to the nearest; the
sample answers millipascals and millidegrees.

The two parts keep most registers in the same places.  The ones that
differ, which the driver switches on:

| | LPS22HB | LPS22HH |
|---|---|---|
| WHO_AM_I (0x0F) | 0xB1 | 0xB3 |
| Output data rates (CTRL_REG1 ODR) | 1, 10, 25, 50, 75 Hz | those, and 100 and 200 Hz |
| FIFO | 32 samples | 128 samples (not read from the data sheet) |
| FIFO_CTRL | 0x14: mode in bits 7:5, watermark in 4:0; FIFO_EN is CTRL_REG2 bit 6 | 0x13: mode in bits 2:0; the watermark is FIFO_WTM, 0x14 |
| FIFO mode codes | bypass 0, FIFO 1, stream 2, stream-to-FIFO 3, bypass-to-stream 4, dynamic-stream 6, bypass-to-FIFO 7 | bypass 0, FIFO 1, stream 2, dynamic-stream 3, bypass-to-FIFO 5, bypass-to-stream 6, stream-to-FIFO 7 |
| FIFO data | the output registers, 0x28 to 0x2C | FIFO_DATA_OUT, 0x78 to 0x7C |
| FIFO count and flags | FIFO_STATUS 0x26 | FIFO_STATUS1 0x25 (the count), FIFO_STATUS2 0x26 (the flags) |
| INT_SOURCE | 0x25 | 0x24 |
| Reference pressure | 3 bytes at 0x15 | 2 bytes at 0x15 |
| Low-current mode | RES_CONF.LC_EN (0x1A), power-down only | CTRL_REG2.LOW_NOISE_EN, the opposite sense |
| Interrupt pad level and stage | CTRL_REG3 bits 7 and 6 | CTRL_REG2 bits 6 and 5 |

The bit places that both parts share are listed at the head of
`src/lps22hb.nv`.  STATUS is one of them: P_DA is bit 0, T_DA bit 1, P_OR
bit 4 and T_OR bit 5 in both.  Zephyr's `lps22hb.h` writes them the other
way round; Zephyr never reads them.

Each part converts in one of two ways.

| Mode | How | What it does |
|---|---|---|
| Continuous | ODR is 1 to 5 (to 7 on the LPS22HH) | converts at the rate, and the output registers hold the latest result |
| One-shot | ODR is 0 (power-down), then ONE_SHOT is written | converts once; ONE_SHOT clears itself when the result is in the output registers |

Neither data sheet gives the time of a one-shot conversion.  The driver
answers 14000 microseconds for the LPS22HB, one period of 75 Hz rounded up,
and 6000 for the LPS22HH, Zephyr's bound from a part it measured under 3 ms,
and reads ONE_SHOT for the end.  Both figures are bounds and not data sheet
values.

The two buses use the same pins:

| | I2C | SPI |
|---|---|---|
| Address | 0x5C (SA0 low) or 0x5D (SA0 high) | chip select low |
| Several registers in one read | the register address advances after each byte | the address advances after each byte |
| Read | register, repeated start, data | control byte with bit 7 set, then data, with no dummy byte |
| Write | register, then the data | control byte with bit 7 clear, then one byte |
| Clock | 400 kHz | 4 MHz, mode 3 (the data sheet allows 10 MHz) |

## What the package contains

| Module | What is in it |
|---|---|
| `lps22hb` | The types `Lps22` (I2C), `Lps22Spi` (SPI) and `Lps22Host`, `Lps22Sample`, the trait `Lps22Port`, the bus framing, the register access, the conversions, the settings, the one-shot conversion, the interrupt line, the reference pressure, the FIFO and the bring-up |

## How to choose an entry point

A program on a board uses `Lps22` for a part on I2C and `Lps22Spi` for a
part on SPI.  Each is built by `attach` with the board's bus, its GPIO and
the wiring, as a board's device table builds it.  Both implement
`Barometer`, `Thermometer` and `Identifiable`.  `begin` takes a `var`
receiver: it sets the register map from WHO_AM_I.  A program that knows its
chip and does not call `begin` names the map with `with_family(FAM_HB)` or
`with_family(FAM_HH)`.  Every function that needs the map refuses
(`None`, `false`, `-1`) while it is `FAM_NONE`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin()` | makes the bus ready, reads WHO_AM_I, takes the family from it and brings the part up, see below |
| `with_family(fam)`, `family_in_use()` | names the register map; answers it |
| `baro_start()`, `temp_start()` | starts a one-shot conversion and answers the time to wait in microseconds; 0 when the part runs at a rate; -1 when the part did not answer |
| `baro_read()`, `temp_read()` | the latest pressure in pascals, and the temperature in millidegrees |
| `sample()` | the latest result as `Lps22Sample`: millipascals and millidegrees |
| `sample_once(polls)` | one conversion on request, waited for by polling ONE_SHOT at most `polls` times |
| `ready()` | both result flags of STATUS |
| `device_id()` | `Identifiable`: WHO_AM_I |
| `set_rate(code)`, `set_filter(mode)` | the output data rate code, 0 to 5 (to 7), and the low-pass filter off, ODR/9 or ODR/20 |
| `set_fifo(mode, wtm)` | the FIFO mode and watermark |
| `baro_fifo_count()`, `baro_fifo_read(dst)` | `Barometer`: the samples in the FIFO, and up to 32 of them, oldest first, as pascals in a `Vec[Pascals; 32]` |
| `baro_set_reference(p)` | `Barometer`: the pressure into REF_P, on the LPS22HB; `false` on the LPS22HH |
| `baro_set_threshold(p, on_high, on_low)` | `Barometer`: THS_P to the nearest step of 6.25 Pa and the interrupt on PHE and PLE, not latched |
| `interrupt_source()` | INT_SOURCE; reading it clears a latched event |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |
| `irq()` | the level of the INT_DRDY line, -1 for a wiring with none |

`begin` makes the bus ready, checks WHO_AM_I, resets the part with SWRESET
and waits for the bit to clear, turns the register address increment and
block data update on, powers the part down (ODR 0, which is one-shot mode),
and puts the data-ready signal on the INT_DRDY pad, active high and
push-pull.  It answers `false` for a part that answers neither WHO_AM_I
value, for a register that does not hold what was written, and for a reset
that does not end.

A program that holds its own bus, or a test with a double of the part,
implements `Lps22Port` over its bus and calls the module's functions, which
take the port: `lps22hb.sample_read(port)`, `lps22hb.start(port)`.  The
framing of each bus is `i2c_read_into`, `i2c_write`, `spi_read_into` and
`spi_write`, generic over `I2cBus` and `SpiBus`.  The conversions
(`pressure_mpa`, `pressure_pa`, `temp_milli_c`) need no bus.

### Block data update and the one-shot conversion

`begin` sets BDU, so a result is held in the output registers until its
high byte is read: the pressure until PRESS_OUT_H (0x2A), the temperature
until TEMP_OUT_H (0x2C).  The pressure is read as one burst of three bytes
that ends at 0x2A, and the temperature as one burst of two that ends at
0x2C.  A one-shot conversion started while the last result is still held
leaves the registers as they were, so `oneshot_start` reads both high
bytes first.  A program that reads only the pressure therefore does not
stall the temperature.

The data sheet's note to CTRL_REG1 asks for IF_ADD_INC = 0 and single-byte
reads on I2C when BDU is set.  ST's own C driver and Zephyr read bursts
with both set, and so does this driver.  That is the one place where the
code and the text of the data sheet differ, and a reply from a part has
not been taken to settle it.

### The FIFO

The FIFO is an option: `begin` leaves it off.  `set_fifo(mode, wtm)` takes
a mode of this package (`FIFO_BYPASS`, `FIFO_FIFO`, `FIFO_STREAM`,
`FIFO_DYNAMIC`, `FIFO_STREAM_TO_FIFO`, `FIFO_BYPASS_TO_STREAM`,
`FIFO_BYPASS_TO_FIFO`) and writes the code of the part's own table.
`baro_fifo_read` takes up to 32 samples, oldest first, into a
`Vec[Pascals; 32]` and answers how many it took; `baro_fifo_count` answers
how many are waiting.  A program that wants the temperatures as well, or the
samples in millipascals, calls `fifo_take(part, press, temp)`, which takes a
port and so takes `Lps22` and `Lps22Spi` too.  A sample is five bytes read in
one burst.  The
data sheet asks the host to discard the first sample after the mode
changes; the driver does not, and says so at `fifo_set`.

### The interrupt line

`drdy_irq` puts the data-ready signal on the INT_DRDY pad, and `begin`
does.  The pad shows what INT_S selects: the data signals, a pressure
above the threshold, below it, or either (`irq_source_set`,
`threshold_irq`).  `baro_set_threshold` sets the threshold and the two
events without a latch; `threshold_irq(part, high, low, true)` latches, and a
latched threshold event stays in INT_SOURCE until `int_source` reads it.  `irq_pad_set` sets the active level and the output
stage, in the register each part keeps them in.

## Async forms

A conversion completes in the background, so there are two async forms.

On a device, `drdy_arm(edge, w, f)` arms the INT_DRDY pin on a pin-edge
source (`AsyncEdge` of embedded-hal-nv) for the rising edge, and resolves
the Future `f` that the caller awaits with `core.rt.future_await`; the
caller then calls `sample_read`.  `drdy_disarm` takes the arm back.  No
board's GPIO implements `AsyncEdge` today, so the source is a program's own
interrupt glue; the suite runs the function over a double.

On the host scheduler, `host_of(part)` answers an `Lps22Host`, which
implements `BarometerAsync` and `ThermometerAsync`: `baro_measure` and
`temp_measure` start a conversion, park the task for the time
`conversion_us` gives, then until ONE_SHOT clears, and answer the result.
The struct is not a `@value` struct, since an `async fn` takes none, and
its impls are `@tier(app)`, since the embedded tier refuses an `async fn`.

## The rules a user needs

1. `begin` has to run before a reading, or `with_family` has to name the
   map.  The output registers read 0 before the first conversion, and a
   pressure count of 0 answers `None`.
2. The time `baro_start` answers is a bound.  `sample_once` and
   `oneshot_done` read the end from ONE_SHOT.
3. A one-shot conversion needs the part powered down.  A part running at a
   rate answers 0 from `baro_start`, and nothing is written.
4. The output pressure is a difference when AUTOZERO or AUTORIFP is on, and
   `baro_read` is then not an absolute pressure.  `autozero` and
   `autorefp` switch the modes on and off.  The offset in RPDS, set by
   `offset_set`, is subtracted from the output.
5. The reference pressure register is a signed count (24 bits on the
   LPS22HB, 16 on the LPS22HH); the lps22hb crate read it unsigned.
6. A setting is a masked update of its register, so one setting does not
   undo another.  The lps22hb crate overwrote the upper bits of
   INTERRUPT_CFG when it set the lower ones.
7. CTRL_REG2 holds three bits that clear themselves, BOOT, SWRESET and
   ONE_SHOT.  An update of another bit of CTRL_REG2 writes them as 0, so
   it never restarts a reset, a reboot or a conversion that is in flight.
8. With the low-pass filter on, the LPS22HB's filter is reset by reading
   LPFP_RES after the rate is set; `odr_set` does it.  The LPS22HH has no
   such register.
9. The LPS22HB low-current mode (`low_noise_set(false)`) is changed in
   power-down only; `low_noise_set` refuses otherwise.  ST's header offers
   the LPS22HH low-noise mode at the rates up to 75 Hz only (not read from
   the data sheet), and the driver refuses it above.
10. `device_id` answers `None` for a bus that no part drives: an SPI bus
    reads 0xFF, or 0x00 when held low.

## What is not included

- The SPI three-wire mode (SIM) and the LPS22HH's I3C and pull-up bits
  (IF_CTRL); `set_reg` reaches them.
- Any check against the LPS22HH data sheet: its one-shot time, its FIFO
  depth and the rates that allow low-noise mode are the header's and
  Zephyr's, not the data sheet's.  NOT IMPLEMENTED as read from that
  document.
- An altitude, and a sea-level pressure.
- A reference pressure on the LPS22HH: its REF_P is 16 bits, and the scale
  is not in the sources read, so `baro_set_reference` answers `false`.
- The FIFO full flag as an interrupt with its own latch; `fifo_irq` puts
  the three FIFO signals on the pad.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the traits
  this driver implements, the wiring record and the register helpers it
  uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `SpiBus`, `GpioOut` and `AsyncEdge` traits and the board
  handles.
- bmp390-nv and dps310-nv drive Bosch's BMP390 and Infineon's DPS310, two
  barometers with the same traits.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus and on an SPI bus, as an LPS22HB and as an LPS22HH, with the
self-clearing bits of CTRL_REG2, the block data update that holds a result
until its high byte is read, the flags that reading clears, the FIFO of
each part and the interrupt source of each part.  It checks the framing of
every entry point, the conversions, the settings and their refusals, the
one-shot conversion and its bound, the interrupt line, the FIFO in both
maps, the bring-up and its refusals, the awaitable line over a pin-edge
double, the host-scheduler form and the two types over the host's board
handles.  The suite covers every line of `src/`.

The conversions are checked against the data sheet's scale.  The data
sheet gives no numeric example, so the vectors are derived: 1013.25 hPa is
4150272 counts, a count is 3125/128 mPa, 2500 counts of temperature are
25 degrees.  `tests/coverage.sh` merges the suite's coverage over `src/`.

Two mutations were checked and the suite turned red: dropping the read of
the held pressure in `oneshot_start`, and giving the LPS22HH the LPS22HB's
INT_SOURCE address.  One was not killed, swapping the values of the P_DA
and T_DA masks, because the only function that uses them tests both bits
together; the mutant is equivalent.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are the
LPS22HB data sheet above, ST's `lps22hb_reg.h` and `lps22hh_reg.h`,
Zephyr's `drivers/sensor/st/lps22hb` and `lps22hh`, and the `lps22hb` crate
(nebelgrau77/lps22hb-rs, 0.1.0).  The `lps22hh-rs` crate was not opened.
Embassy and esp-hal hold no driver for the part.  Each fix kept is named in
`src/lps22hb.nv` where it is kept.

## Licence

Apache-2.0.
