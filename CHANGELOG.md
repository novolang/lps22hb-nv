# Changelog

## 0.1.0

The first release: a driver for ST's LPS22HB and LPS22HH absolute pressure
sensors on an I2C or an SPI bus.

- `Lps22` (I2C) and `Lps22Spi` (SPI), the types a board's device table
  builds, implementing devices-nv's `Barometer`, `Thermometer` and
  `Identifiable`, with their own methods for the sample in millipascals and
  millidegrees, the one-shot conversion, the output data rate, the low-pass
  filter, the FIFO, the pressure threshold and its interrupt, and register
  access.  `begin` takes the register map from WHO_AM_I (0xB1 an LPS22HB,
  0xB3 an LPS22HH); `with_family` names it.
- The register access as functions over `Lps22Port`, a trait that a type
  holding a bus implements, and the framing of each bus as functions over
  `I2cBus` and `SpiBus`.  An SPI read carries no dummy byte.
- The conversions: the 24-bit pressure at 4096 counts a hectopascal and the
  16-bit temperature at 100 counts a degree.
- The one-shot conversion with its ready flag (ONE_SHOT clears itself),
  after reading the two held high bytes so that block data update does not
  leave the next result unloaded.
- The FIFO modes of both parts under names of this package, the two parts'
  different codes, registers and depths, and a drain into two lists.
- The reference pressure (AUTOZERO, AUTORIFP, the signed reference
  register), the one-point-calibration offset, the pressure threshold and
  its interrupt, the interrupt pad's level and stage, and INT_SOURCE.
- Two async forms: `drdy_arm` and `drdy_disarm`, which arm the INT_DRDY pin
  on any `AsyncEdge` source, and `host_of`, which gives the host scheduler's
  `BarometerAsync` and `ThermometerAsync`.
- The suite runs over a double of the part on both buses, as both parts,
  and covers every line of `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries an LPS22HB or an
  LPS22HH.
- The LPS22HH data sheet was not read.  Its facts come from ST's register
  header and Zephyr's driver; the one-shot time, the FIFO depth and the
  rates that allow low-noise mode are marked "not read" in the source.
- The LPS22HB data sheet is DocID027083 Rev 5 (November 2016).  The brief
  names DS10829 for it; the revision that was fetched is the one above.
- Neither data sheet gives the time of a one-shot conversion.  The 14000
  and 6000 microseconds `conversion_us` answers are bounds.
- The data sheet's note asks for IF_ADD_INC = 0 and single-byte reads on
  I2C when BDU is set.  The driver reads bursts with both set, as ST's C
  driver and Zephyr do.
- The three-wire SPI mode, and the LPS22HH's I3C and pull-up bits, are not
  implemented.
- No board's GPIO implements `AsyncEdge`, so `drdy_arm` has no source on a
  board yet.
