# Changelog

## 0.1.0

The first release: a driver for the INMP441 and the SPH0645LM4H, MEMS
microphones with an I2S output and no control bus.

- `Inmp441`, the type `attach` builds over a board's `I2s` handle,
  implementing devices-nv's `Microphone`: `mic_start` configures the handle
  for 32-bit slots, two channels, the board as master, Philips framing, and
  answers the start-up time; `mic_read` answers 24-bit signed samples;
  `mic_stop`, `mic_lost` and `mic_format` complete the trait. `mic_set_gain`
  keeps the trait's body, `None`, because both parts have a fixed gain.
- `MicConfig` and its constructors `config_inmp441` and `config_sph0645`:
  the rate, the slot the L/R or SELECT pin names, a constant offset to
  remove, the INMP441's standby start and a late-read shift.
- The frame arithmetic as functions over `MicConfig`: the bit clock, the
  clock ranges of both parts, the start-up time (2^18 SCK cycles from
  power-up and 2^14 from standby for the INMP441, 50 ms for the SPH0645LM4H),
  the unpacking of a slot word to a 24-bit sample, with the eight undriven
  bits shifted away and the SPH0645LM4H's six unused bits cleared.
- `read_samples`, the read over any `I2s` handle: both words of each frame
  are read and the part's slot kept, in passes of 128 frames, so a buffer of
  256 samples takes two.
- `dc_block`, an optional DC blocker whose state the caller holds.
- The suite runs over a double of the `I2s` handle and covers every line of
  `src/`.
- The device program `examples/mic_level.nv`, a peak meter. Built for the
  nRF52840-DK it is 11,172 B of flash and for the nRF54LM20-DK 12,492 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries either microphone,
  and no board package has an I2S block. The device program is given a
  stand-in handle that refuses to be configured, and prints `refused`.
- The manifest names devices-nv and embedded-hal-nv by the registry range
  `^0.3.3`, the first releases with `Microphone` and `I2s`.
- Not implemented: the sampling edge of SCK (the `I2s` trait has no field
  for it), the async form of `Microphone`, and the PDM microphones.
- The data sheets give no DC offset for either part. `dc_offset` is the
  caller's.
- `I2sConfig` has one `bit_depth`, so the package asks the handle for 32 bits
  to get the 64 SCK cycles per frame the parts require. A board whose block
  has no 32-bit slot has to refuse or map it.
