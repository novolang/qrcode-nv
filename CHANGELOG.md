# Changelog

All notable changes to qrcode-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `qrmatrix` — the load-bearing interface. A module is `QrFree`,
  `QrLight` or `QrDark`, not a boolean, because placing a symbol runs
  in stages and every stage after the first has to know which modules
  the earlier ones already own. The quiet zone is NOT in the matrix:
  ISO/IEC 18004 § 6.3 requires four light modules around the symbol
  and says nothing else about them, so they belong to the drawing and
  a caller compositing a symbol is never fighting an offset it did
  not ask for. The eight masks, the four penalty rules scored
  separately, and the trial that picks one are all public, because the
  penalty score is the part of the standard most often implemented
  wrongly.
- `qrmode` — the four modes with their costs, the four levels, and
  `plan`, which is a shortest path over the three text modes at a
  named version rather than a greedy scan. The version is an argument
  because the character count field widens at version 10 and again at
  27, so the cheapest segmentation is not the same at every size.
  ECI is a segment, not a flag, because § 7.4.2 makes it a prefix that
  stays in force to the end of the symbol.
- `qrversion` — the tables: module count, alignment centres, codeword
  counts, the two-group block plan of Table 9, the remainder bits of
  § 7.7, and the BCH-coded format and version information.
  `smallest_for` searches rather than computes, for the same reason
  `plan` takes a version.
- `qrcode` — the encoder, with each stage separately public:
  `data_bit_stream`, `block_ec`, `error_correction_blocks`,
  `interleave`. Annex I publishes those intermediate values, so a
  suite can name the stage that disagreed instead of asserting on a
  finished picture.
- `qrrender` — text, SVG and a raster mask. The SVG is ONE PATH with
  horizontal runs merged, not one `<rect>` per module, and
  `to_svg_node` answers a node so a symbol can go inside a larger
  drawing. `to_raster` answers a MASK — one byte per pixel, 1 dark and
  0 light — and `to_grey8` and `to_rgba8` paint it, so the two colours
  live in the caller's palette rather than in every pixel.
- `qrfault` — eleven refusals, each carrying the number that would
  have made it succeed. `is_retryable` separates a pin a caller can
  raise from the end of the format.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  qrcode-nv.<module>.<fn>`.
- **Decoding and Micro QR are not declared at all**, rather than
  declared and stubbed. Both are named in the README as later
  releases.
- **Kanji mode takes Shift JIS bytes and this package does not
  transcode.** There is no Shift JIS row on the Orbit grid; when one
  lands, a convenience that takes text is a minor release.
