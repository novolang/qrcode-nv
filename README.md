# qrcode-nv

A QR Code is a two-dimensional barcode: a square grid of light and dark
squares, called modules, that a camera reads from any of four
orientations. It is specified in
[ISO/IEC 18004](https://www.iso.org/standard/62021.html), and the
reference implementations this package is measured against are the Rust
crate [`qrcode`](https://docs.rs/qrcode) and the Python package
[`qrcode`](https://pypi.org/project/qrcode/). This package encodes data
into a QR Code symbol and draws it.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a QR Code is

A **symbol** is a square grid of **modules**. A module is one cell, and
it is either dark or light. The **version** is the size: version 1 is
21 modules on a side, version 40 is 177, and version `v` is `4v + 17`.
Around the symbol there must be a **quiet zone** of four light modules
on every side, or the reader cannot find the edges.

Three corners carry a **finder pattern**, a seven-by-seven target that
lets a reader locate and orient the symbol. Larger versions also carry
**alignment patterns**, smaller targets that let a reader correct for a
curved or tilted surface. Two **timing patterns**, alternating rows of
modules between the finders, give the reader the module pitch. These,
together with the areas that hold the format and version information,
are the **function patterns**: they are fixed by the standard, and data
is written into everything else.

Data goes in as a sequence of **segments**. Each segment names a
**mode** — a character set — and the mode decides what a character
costs.

| Mode | Character set | Bits per character |
| --- | --- | --- |
| Numeric | `0`–`9` | 10 bits per 3 digits |
| Alphanumeric | 45 characters: digits, upper-case letters, space, `$%*+-./:` | 11 bits per 2 characters |
| Byte | any byte | 8 |
| Kanji | Shift JIS double bytes in two ranges | 13 |

One symbol may hold several segments in different modes, so a URL whose
tail is a long number is smaller when the letters go in alphanumeric
mode and the digits in numeric mode. Choosing where those switches fall
is what section 7.4 calls the encoding, and Annex J gives the
optimisation.

The **error-correction level** decides how much of the symbol is given
over to recovering from damage. There are four, and a higher one means
a larger symbol for the same data.

| Level | Codewords recoverable |
| --- | --- |
| L | about 7% |
| M | about 15% |
| Q | about 25% |
| H | about 30% |

The recovery itself is Reed–Solomon coding over GF(256). The data is
cut into **blocks**, each block gets its own error-correction
codewords, and the blocks are then **interleaved** — the first codeword
of every block, then the second, and so on — so that damage to one
region of the picture is spread thinly across every block instead of
destroying one of them.

Finally the symbol is **masked**. Section 7.8 defines eight mask
patterns, each a formula over a module's row and column, and one of
them is XORed into the data modules. The encoder tries all eight,
scores each with four penalty rules, and keeps the lowest-scoring one.
The rules penalise long runs of one colour, solid two-by-two blocks,
anything that looks like a finder pattern in the middle of the data,
and a large imbalance between light and dark. Without masking, a
payload of mostly zeros produces a symbol a reader cannot lock on to.

## Install

```
novo pkg add qrcode-nv
```

## Example

```novo
use std.list
use qrmode
use qrcode
use qrrender

fn main() [io]
    // Encode at error-correction level Q. The version is the smallest
    // that holds the data, and the mask is the one with the lowest
    // penalty score.
    match qrcode.encode("HTTPS://EXAMPLE.COM/3141592", QrEcQuartile)
        Err(e) => println("cannot encode: ${e.message()}")
        Ok(symbol) =>
            // Draw it for a terminal: two module rows per line of
            // text, with the four-module quiet zone included.
            match qrrender.to_text(symbol, qrrender.default_style())
                Err(e)    => println("cannot draw: ${e.message()}")
                Ok(lines) => list.each(lines, line => println(line))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: qrcode-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `qrfault` | Every reason a symbol could not be produced, with the numbers a caller needs to fix it. |
| `qrmode` | The four data modes, the four error-correction levels, a segment as a value, and the planner that chooses the modes for a string. |
| `qrversion` | The forty sizes: how many modules, how many codewords, where the alignment patterns sit, and how the codewords are cut into blocks. |
| `qrmatrix` | The symbol as a grid of modules, the eight masks, and the penalty score that picks one. |
| `qrcode` | The encoder, and each of its stages separately: the bit stream, the error-correction blocks, the interleaving. |
| `qrrender` | Drawing a symbol as text, as an SVG document, and as a one-byte-per-pixel mask. |

## How to choose an entry point

**`qrcode.encode` takes a string and a level.** It plans the segments,
picks the smallest version and the best mask, and answers a matrix. Use
it unless something below applies.

**`qrcode.encode_with` takes options.** Use it to pin the version, the
mask or the error-correction level — for instance to reproduce another
system's symbols byte for byte, or to fill a label of a fixed size.

**`qrcode.encode_segments` takes a segment list you built.** Use it
when you know your data's shape and do not want the planner to search,
and for kanji mode, which the planner never chooses on its own.

**`qrrender.to_svg` answers a document; `qrrender.to_svg_node` answers
one node.** Use the node when the symbol goes inside a larger drawing,
and the document when it is the whole file.

**`qrrender.to_raster` answers a mask, not an image.** One byte per
pixel, `1` dark and `0` light. `to_grey8` and `to_rgba8` paint it with
the style's two colours when an encoder wants pixels.

## The rules a user needs

1. **The quiet zone is not optional.** Four light modules on every
   side, and a symbol drawn without them does not scan (section 6.3).
   The matrix does not carry them; every renderer adds them, and every
   renderer refuses a request for fewer than four.

2. **Lower case costs three times as much as upper case.**
   Alphanumeric mode's 45 characters do not include lower-case letters
   (section 7.4.4, Table 5), so `https://example.com` goes in byte mode
   at 8 bits a character and `HTTPS://EXAMPLE.COM` goes in
   alphanumeric mode at 5.5. URL schemes and hosts are
   case-insensitive, so upper-casing them is free.

3. **Bytes are ISO-8859-1 unless an ECI segment says otherwise.**
   Section 7.4.4 fixes that default. Text outside it — anything UTF-8
   encodes in more than one byte — needs
   `qrcode.with_eci(options, 26)`, or a reader is entitled to show
   mojibake.

4. **A pinned version can refuse data a free one would accept.**
   `QrVersionTooSmall` means a larger version would have worked;
   `QrDataTooLong` means version 40 does not.
   `qrfault.is_retryable` tells the two apart, which is what keeps an
   automatic retry from looping.

5. **The error-correction level is a trade, not a quality setting.**
   Level H on the same data is a larger symbol, and a larger symbol
   printed at the same physical size has smaller modules, which is
   harder to read. Level H is for a symbol that will be damaged, not
   for a symbol that must be read at a distance.

6. **The mask is chosen, not configured.** Section 7.8.3 defines the
   penalty score and the encoder keeps the lowest. Pinning a mask is
   for reproducing another encoder's output, and a pinned mask can
   produce a symbol that scans poorly.

7. **Kanji mode takes Shift JIS bytes.** Section 7.4.6 covers two
   ranges of double bytes and nothing else. This package does not
   transcode into Shift JIS; a caller holding UTF-8 either converts it
   or uses byte mode with ECI 26.

## What is not included

- **Decoding.** Reading a symbol from an image is a binariser, a
  finder-pattern search, a perspective transform and Reed–Solomon
  erasure decoding. It shares no code with encoding. It is planned for
  a later release.
- **Micro QR Code.** A different symbol with its own versions, format
  information and mask set. A later release.
- **rMQR and Model 1.** Neither is in ISO/IEC 18004's Model 2, which is
  what this package encodes.
- **Structured append across several symbols.** The refusal
  `QrSequenceOutOfRange` is declared for it, and the encoder that
  splits a payload across up to sixteen symbols is not.
- **Transcoding into Shift JIS.** Kanji mode takes Shift JIS bytes. The
  conversion table belongs to a text-encoding package, not here.
- **Logo overlays and rounded modules.** Both change how reliably a
  symbol scans, and neither is in the standard. `qrrender.to_svg_node`
  answers the geometry so a caller can compose its own drawing.
- **PNG or GIF output.** `qrrender.to_raster` answers the pixels;
  png-nv and gif-nv write the files.

## Related packages

**svg-nv** is the SVG document tree this package renders into. It owns
paths, styles and serialisation; this package builds one path.

**color-nv** owns `Srgba8`, the colour type the render style is
written in, which is the same type svg-nv's styles are painted with.

**png-nv**, **gif-nv** and **image-nv** write image files. This package
answers a pixel mask and they encode it.

**barcode-nv** would be the one-dimensional formats — Code 128, EAN,
UPC. They share no encoding machinery with QR Codes.

## Test vectors

The suite is written against ISO/IEC 18004 itself.

- **Annex I** works one symbol — the numeric string `01234567` at
  version 1, level M — from the bit stream through the
  error-correction codewords to the final module pattern. Every stage
  of `qrcode` is asserted against it separately, so a failure names the
  stage.
- **Table 3** fixes the character count field widths at each of the
  three version bands. **Table 5** is the alphanumeric character set.
  **Table 9** is the block plan for all 160 version-and-level
  combinations. **Table 10** is the eight mask formulas. **Table E.1**
  is the alignment pattern coordinates. Each is asserted directly.
- **Section 7.8.3.1** gives the four penalty rules, and each is scored
  separately so a suite can say which rule disagreed.
- The capacity table of Annex D is asserted at the corners: version 1
  and version 40, at each of the four levels, in each of the four
  modes.

Today every one of those assertions reaches a `not implemented` panic.

## Implementation status

| Area | Status |
| --- | --- |
| Segment modes, planning, ECI | declared, not implemented |
| Version and capacity tables | declared, not implemented |
| Reed–Solomon blocks and interleaving | declared, not implemented |
| Module placement, masking, penalty score | declared, not implemented |
| Text, SVG and raster rendering | declared, not implemented |
| Decoding | not declared — a later release |
| Micro QR | not declared — a later release |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
