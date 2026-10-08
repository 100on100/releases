# 100on100 v1.0.0 (evaluation licence)

This is an evaluation release of the 100on100 decoder. It is licensed for evaluation only: read
`LICENSE` first. It is not for production use.

The decoder reads a 100on100 container (a compressed raw sensor frame) and writes the samples back
exactly, or refuses the file with a numbered reason.

## What is in the package

| Path | What it is |
|---|---|
| `100on100.h` | the C header (the library interface) |
| `linux-x86_64/`, `linux-aarch64/`, `macos-arm64/` | `100on100` (command-line tool) and `lib100on100.a` (static library) |
| `wasm32-wasip1/100on100.wasm` | the command-line tool as a WebAssembly (WASI) module |
| `LICENSE`, `THIRD-PARTY-NOTICES` | licence terms and notices for the WebAssembly system library |
| `SHA256SUMS`, `PROVENANCE.txt` | checksums of every file, and how the files were built |

Check the files with `sha256sum -c SHA256SUMS` (on macOS: `shasum -a 256 -c SHA256SUMS`).
Every exported symbol of the static library starts with `i100_`.

## Command-line tool

    100on100 input > output.pgm
    echo $?

- `input` is one container file. The tool writes a 16-bit PGM (`P5`, maximum value 65535) to
  standard output.
- The exit status is the decoder's own numbered code: `0` on success, otherwise the refusal code
  (see below). Nothing is written to standard output when the code is not 0. The tool's own
  statuses are 64 (wrong number of arguments), 66 (input not readable), 70 (out of memory) and
  74 (write error).
- WebAssembly: `wasmtime run --dir=. 100on100.wasm input > output.pgm` (any WASI runtime that can
  preopen a directory works).

## C library, in plain words

Include `100on100.h` and link `lib100on100.a`. There are four functions.

- `i100_probe(buf, len, &info)` reads the header only. It fills an `i100_info` with the version,
  the width and height, the bit depth, the tile grid, the size of the output, and the number of
  octets of workspace a decode needs. It returns 0, or a header-stage refusal code.
- `i100_decode(buf, len, ws, ws_len, out, out_count)` decodes the whole container into `out`,
  using your workspace `ws`. You allocate the workspace and the output; the library allocates
  nothing and keeps no global state.
- `i100_decode_tile(buf, len, plane, tile, ws, ws_len, out, out_count, &tw, &th)` decodes one tile
  of one plane on its own and returns that tile's size in `tw` and `th`.
- `i100_build()` returns the build's version string.

Rules stated in the header:

- Two decodes on two threads with two workspaces are safe at the same time; do not share one
  workspace between concurrent calls.
- `i100_decode` and `i100_decode_tile` zero their own workspace. After a nonzero return the output
  is unspecified; read it only after a 0.
- Codes 100 and above are the library's own: 100 workspace too small, 101 output too small,
  102 frame too large for this platform's `size_t`, 103 a NULL pointer or an index out of range.
  A too-small buffer is reported before anything is written.
- Output samples: the header states that a sample is the PGM's two octets,
  `((v / 256) & 255) << 8 | ((v % 256) & 255)`, computed with C99's truncating signed `/` and `%`,
  and that this is NOT `(uint16_t)v`. A sample can decode, with status 0, to a negative value, for
  which the reference decoder writes `0x00FF` for -1 where a cast gives `0xFFFF`. The `out`
  array holds samples already in this form. Do not cast decoded values yourself. The header does
  not name a separate conversion function, and this package provides none.

## Refusal codes stated in the header

| Code | Meaning (as the header states it) |
|---|---|
| 9, 19 | stream length |
| 21, 31 | value-list index |
| 25, 35 | plane checksum (CRC) |
| 38 | tile checksum (CRC) |
| 39 | a plane above 3, or a tile out of range (tile decode) |
| 40 | a sample below 0 or at or above 2^depth (version 4) |
| 44 | an invalid predictor letter |
| 45 | impossible geometry |
| 100 to 103 | the library's own codes (above) |

The header says the decoder's codes stop at 45. Other numbers from 1 to 45 are further refusals of
damaged or unsupported input and are returned as plain numbers; this release does not document
them individually. The descriptions of 44 and 45 above are not in the header.

## Status

Evaluation release. Source code is not provided. See `RELEASE-NOTES.md` for what is and is not
included, and `LICENSE` for the terms.
