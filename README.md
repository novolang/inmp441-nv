# inmp441-nv

A driver for two MEMS microphones with an I2S output and no control bus:
TDK InvenSense's INMP441, specified in the data sheet "INMP441:
Omnidirectional Microphone with Bottom Port and I2S Digital Output",
document DS-INMP441-00, Revision 1.1, 05/21/2014, and Knowles' SPH0645LM4H,
specified in "SPH0645LM4H-B Rev B Datasheet I2S Output Digital Microphone".
The driver implements the `Microphone` trait of
[devices-nv](https://novo-lang.org/packages/devices-nv) over the `I2s`
handle of [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv)
that a board gives it, and delivers 24-bit signed samples.

**Hardware verification: NOT VERIFIED.** No board on the bench carries
either microphone, and no board package has an I2S block yet. The driver
has run only against a double of the I2S handle, in the host suite, and
the device program builds for the nRF52840-DK and the nRF54LM20-DK against
a stand-in handle that refuses to be configured. A sample from a part on a
board has not been taken.

## Install

```sh
novo pkg add inmp441-nv
```

The device program is `examples/mic_level.nv`.  Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/mic_level.nv` reads an INMP441 at 16 kHz and prints the peak of
every 256 samples over the board's UART:

```
peak 18432 of 8388608
```

```novo norun:needs-pkg
use embedded_hal.{ I2s, I2sConfig }
use dev_audio.{ Microphone }
use inmp441

// The board's I2S block goes here. This stand-in refuses to be configured,
// as the Nordic board packages' absent hardware does.
@value
struct NoI2s
    id: Int

impl I2s for NoI2s
    fn configure(self, c: I2sConfig) -> Bool [hw]
        false
    fn sample_rate(self) -> Int [hw]
        0
    fn enable(self) -> Bool [hw]
        false
    fn disable(self) [hw]
        pass
    fn transmit(self, var buf: Vec[i32; 256]) -> Int [hw]
        -1
    fn receive(self, var buf: Vec[i32; 256], n: Int) -> Int [hw]
        -1
    fn xruns(self) -> Int [hw]
        0

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // The part's L/R pin is low, so it sends in the left slot.
    let mic = inmp441.attach(NoI2s { id: 0 }, inmp441.config_inmp441(16000, false))
    let settle_us = mic.mic_start()
    if settle_us < 0
        hal.uart.write("refused\n")
    else
        hal.timer.delay_ms(settle_us / 1000 + 1)
    var buf: Vec[i32; 256] = Vec.new()
    while true
        let n = mic.mic_read(var buf, 256)
        if n > 0
            hal.uart.write("peak\n")
        hal.timer.delay_ms(100)
```

The full program, which finds the peak and writes it in decimal, is
`examples/mic_level.nv`. A board package that depends on this package
builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs
the image and checks that the program prints `refused`:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 11,172 B | 934 B | 2,624 B |
| `nrf54lm20-dk` | 12,492 B | 1,254 B | 2,624 B |

The numbers are from the build's memory report. Flash also holds the
runtime, the compiler's support library and the program's tables. The
arena is the runtime's heap and is reserved whether or not the program
uses it.

**The I2S block is a stand-in.** embedded-hal-nv has a trait for an I2S
interface and no board handle for one, and neither Nordic board package
drives its I2S block. The program is written against the `I2s` trait and
is given `NoI2s`, so on both images `mic_start` answers -1 and the program
prints `refused`. The one argument of `attach` is what changes when a board
package gives a handle. The wiring the program assumes is: SCK and WS from
the board, SD to the board's data input with a 100 kohm pull-down if the
breakout has none, L/R to ground, CHIPEN and VDD to 3.3 V.

### The effects

Every function of the driver that reaches the I2S handle declares `[hw]`,
and nothing else. The driver performs no input or output of its own.
`attach` allocates one cell of the arena for the microphone, because a
struct with a generic field cannot be a `@value`; the samples are unpacked
into the caller's `Vec[i32; 256]`, and one read keeps a second buffer of
the same size in its frame. The module checks at every tier, `embedded`
included.

## What it is

A MEMS microphone with a digital output has a sensor, an
analogue-to-digital converter and a filter in the package, and sends the
result as I2S words. There are no registers: the part is a slave on the
bit clock (SCK) and the word clock (WS) the board drives, and the board
reads its data pin (SD). What a program needs from a driver is therefore
the shape of the frame and the arithmetic from words to samples.

| | INMP441 | SPH0645LM4H |
|---|---|---|
| Word | 24 bits, two's complement, MSB first | the same, 18 bits used, six unused bits zero |
| Frame | 64 SCK cycles per stereo frame, 32 per word | 64 BCLK per WS |
| SCK | 0.5 to 3.2 MHz | 2.048 to 4.096 MHz |
| Word clock | 7.8 to 50 kHz | 32 to 64 kHz |
| Format | I2S, the MSB one SCK cycle after the WS edge | the same |
| Channel pin | L/R: low is the left slot, high the right | SELECT: low is the left slot, high the right |
| SD between words | high impedance, from the clock after the LSB | high impedance in the other microphone's slot |
| Valid after | 2^18 SCK cycles from power-up, 2^14 from standby | 50 ms at most from power-up |
| Gain | fixed, sensitivity -26 dBFS at 94 dB SPL | fixed, -26 dBFS |

The 24-bit word sits in the top of a 32-bit slot. The INMP441 stops
driving SD after the last bit of the word, so the last eight bits of the
slot are whatever the line held, a pull-down's zeros or noise. The driver
shifts them away: a slot word of 0x123456A5 is the sample 0x123456.

The part sends in one slot of the frame only. The driver sets the handle to
two channels, reads both, and keeps the words of the slot the L/R or
SELECT pin names, so the board's I2S block sees the WS timing it is
configured for and the other slot, which another microphone may fill, is
discarded.

The INMP441 outputs zero for the first 2^18 SCK cycles after power is
applied. At 48 kHz that is 85.3 ms and at 16 kHz it is 256 ms, because the
clock follows the rate. The driver answers that time from `mic_start`. From
standby, which is the part enabled with its clock stopped, it is 2^14
cycles, 5.3 ms at 48 kHz, and a configuration says so with
`standby_wake`. The part has a high-pass filter of its own at 3.7 Hz
(-3 dB at 48 kHz), so its output has little DC; the data sheet gives no
offset.

## What the package contains

| Module | What is in it |
|---|---|
| `inmp441` | `MicConfig`, `MicPart`, the two configuration functions, the frame arithmetic, `read_samples`, the DC blocker and the type `Inmp441` that implements `Microphone` |

## How to choose an entry point

A program on a board builds a `MicConfig` with `config_inmp441(rate, right)`
or `config_sph0645(rate, right)`, passes it with the board's `I2s` handle
to `attach`, and uses the `Microphone` trait on the result.

| Function | What it does |
|---|---|
| `config_inmp441(rate, right)`, `config_sph0645(rate, right)` | the configuration of a part at `rate` hertz whose channel pin is high when `right` is true |
| `attach(i2s, cfg)` | the microphone on the handle; nothing is sent |
| `mic_start()` | refuses a clock outside the part's range, configures and enables the handle, answers the microseconds until the output is valid, or -1 |
| `mic_read(buf, n)` | replaces `buf` with `n` samples of 24 bits, 0 to 256, or answers -1 |
| `mic_stop()` | disables the handle |
| `mic_lost()` | the handle's count of lost buffers |
| `mic_format()` | the format of the samples: 24 bits, one channel, Philips |
| `config()` | the configuration the microphone was built with |
| `sck_hz(cfg)`, `rate_ok(cfg)`, `startup_us(cfg)` | the clock the configuration implies, whether the part takes it, and the start-up time |
| `i2s_config_of(cfg)`, `format_of(cfg)` | the settings the handle gets and the format the samples have |
| `unpack(cfg, word)`, `slot_of(cfg)` | one slot word as a sample, and the slot the part sends in |
| `read_samples(i2s, cfg, buf, n)` | the read of `mic_read` over any handle |
| `dc_new()`, `dc_block(state, buf, n)` | the optional DC blocker |

`MicConfig` has six fields: `part`, `sample_rate`, `right`, `dc_offset`,
`standby_wake` and `late_bits`. `dc_offset` is subtracted from every sample
after the shift, in counts of the 24-bit sample. `late_bits` is the number
of bits a receiver reads the word late, 0 for a receiver that samples SD on
the right edge; it shifts the word that many bits further and is for a
receiver whose shift has been measured.

`mic_read` is stateless: the `Microphone` trait gives a method a plain
`self`, so the driver keeps nothing between calls. "Not started" is the
handle's refusal of a `receive` before `enable`, and the answer is -1.
A DC blocker that needs a state between buffers is the function
`dc_block`, which takes the state and answers the new one:

```novo norun:needs-pkg
use inmp441

fn smooth(var buf: Vec[i32; 256], st: inmp441.DcState) -> inmp441.DcState
    inmp441.dc_block(st, var buf, 256)
```

The estimate follows each sample by 1/4096 of the distance, a time
constant of 4096 samples. It keeps 16 fractional bits and rounds, so a
constant offset of either sign settles to 0 and not to 1.

## The SPH0645LM4H

The SPH0645LM4H is the same package with `config_sph0645`. Its word is 18
bits in the top of the 24, so `unpack` also clears the six low bits of the
sample, and its clock range is 2.048 to 4.096 MHz, which is 32 to 64 kHz at
64 cycles per word clock. The start-up time answered is its 50 ms.

The data sheet's Figure 2 gives the data a delay of 65.92 ns from the
rising clock edge and a hold of 7 ns, and asks that WS change on the
falling edge. A receiver that needs more hold time than that, or that
samples late, can read the word shifted by a bit. The edge a receiver
samples on is a property of the board's I2S block, and the `I2s` trait has
no field for it, so this package does not select an edge. `late_bits` undoes
a shift the caller has measured, and the offset of a unit is `dc_offset`.

## The rules a user needs

1. Give the handle 32-bit slots. The INMP441 requires 64 SCK cycles per
   frame. A block whose widest sample is 24 bits maps `bit_depth` 32 to a
   32-bit slot or refuses it; the package then answers -1 from `mic_start`.
2. Wait the time `mic_start` answers before the first sample is sound. The
   samples before it are zero or settling.
3. Read the buffers in time. A read that the handle could not fill is -1
   and the buffer is empty; the buffers the block dropped are `mic_lost`.
4. A frame of two words is read for every sample, so 256 samples are two
   reads of 128 frames, and a read that is slow between them can lose one.
5. The microphone is mono. Two microphones on one data line, one with
   L/R low and one high, are two `MicConfig` values over two reads of the
   same stereo stream, which this package does not interleave.
6. `late_bits` and `dc_offset` are for a unit the caller has measured.

## What is not included

- The sampling edge of SCK, which the board's I2S block owns.
- The synchronisation of two microphones enabled at different times, which
  the data sheet puts at 0.35 ms.
- The PDM microphones, which `PdmIn` of embedded-hal-nv reads.
- A gain: both parts have none, and `mic_set_gain` answers `None`.
- The async form of `Microphone`.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `Microphone` trait this driver implements and the `AudioFormat` it
  answers.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2s` trait and its settings.
- [es8311-nv](https://novo-lang.org/packages/es8311-nv) and
  [wm8960-nv](https://novo-lang.org/packages/wm8960-nv) drive audio codecs
  with the same traits.

## Tests

`novo test` runs the module over a double of the `I2s` handle that records
how it was configured and answers scripted slot words: the settings, the
clock ranges at their edges for both parts, the start-up times, the unpacking
of hand-derived words including the extremes and the SPH0645LM4H's cleared
bits, the choice of slot, the two passes of a 256-sample read, every
refusal, the trait over the double, and the DC blocker. The suite covers
every line of `src/`.

**Hardware: NOT VERIFIED.** See the example section.

The sources read for this driver, for their fixes and not ported, are the
two data sheets above, Embassy's `embassy-nrf` I2S driver, Zephyr's
`i2s_nrfx` driver and the `esp-hal` I2S master. Embassy's sample width stops
at 24 bits on the nRF52 and the bit clock follows the master-clock ratio;
Zephyr offers 32 bits only where the chip's register has it; `esp-hal`
names the on-wire slot width and the in-memory sample width apart
(`Data32Channel24` is a 32-bit slot holding a 24-bit sample). `I2sConfig`
has one `bit_depth`, which is why the package asks for 32 and shifts, and
why rule 1 exists. None of the three has a driver for either microphone.

## Licence

Apache-2.0.
