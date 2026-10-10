# 100on100 v1.2.0 (evaluation licence)

This is an evaluation release of the 100on100 decoder. It is licensed for evaluation only: read
`LICENSE` first. It is not for production use.

The decoder reads a 100on100 container (a compressed raw sensor frame) and writes the samples back
exactly, or refuses the file with a numbered reason.

## What is in the package

| Path | What it is |
|---|---|
| `100on100.h`, `100on100v5.h` | the C headers of the original decoder and the version 5 colour decoder (the small builds, unchanged from earlier releases) |
| `100on100k0.h`, `100on100v4opt.h`, `100on100v5fast.h` | the C headers of the new files: the generic-plane decoder and the two fast builds |
| `<platform>/100on100` (`100on100.exe` on Windows), `<platform>/lib100on100.a` | command-line tool and static library of the original decoder |
| `<platform>/100on100-v5` (`100on100-v5.exe` on Windows), `<platform>/lib100on100v5.a` | command-line tool and static library of the version 5 colour decoder |
| `<platform>/100on100-k0` (`.exe` on Windows), `<platform>/lib100on100k0.a` | command-line tool and static library of the version 5 generic-plane decoder (new) |
| `<platform>/100on100-v4opt`, `<platform>/lib100on100v4opt.a` | fast build of the original decoder (new) |
| `<platform>/100on100-v5fast`, `<platform>/lib100on100v5fast.a` | fast build of the version 5 colour decoder (new) |
| `wasm32-wasip1/*.wasm` | the command-line tools as WebAssembly (WASI) modules: the two small builds and the three new ones |
| `100on100-v5-riscv64.elf` | the version 5 decoder as a RISC-V reference image (see below) |
| `LICENSE`, `THIRD-PARTY-NOTICES` | licence terms and third-party notices |
| `SHA256SUMS`, `PROVENANCE.txt` | checksums of every file, and how the files were built |

`<platform>` is one of `linux-x86_64`, `linux-aarch64`, `macos-arm64`, `macos-x86_64` (macOS 11 or
later), `windows-arm64`, `windows-x86_64`. The original decoder files are the same files as in
version 1.0.0.

Check the files with `sha256sum -c SHA256SUMS` (on macOS: `shasum -a 256 -c SHA256SUMS`).
Every exported symbol of the original static libraries starts with `i100_`, and of the version 5
small colour libraries with `i100v5_`, the fast colour libraries with `i100v5f_`, the fast original libraries with `i100o_` and the generic-plane libraries with `i100k0_`; all of them can be linked into one program.

## Original decoder, command-line tool

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

## Original decoder, C library, in plain words

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

## Version 5 colour decoder (new in 1.1.0)

    100on100-v5 input > samples
    100on100-v5 -t PLANE TILE input > samples
    echo $?

- `input` is one version 5 colour container. The tool writes the decoded picture to standard
  output as interleaved R, G, B (and A, when the picture has it) samples: one octet per sample when
  the picture's bit depth is 8 or less, otherwise two octets, little-endian. There is no PGM
  header; the width, height and depth are in the container header and `i100v5_probe` reports them.
- With `-t PLANE TILE` the tool decodes one tile of one plane on its own and writes its samples,
  row by row, two octets each, little-endian. Planes are numbered in coding order: 0 = G, 1 = R,
  2 = B, 3 = A.
- The exit status is the decoder's numbered code: `0` on success, otherwise the refusal code.
  Nothing is written to standard output when the code is not 0. The tool's own statuses are 64
  (wrong number of arguments), 66 (input not readable), 70 (out of memory) and 74 (write error).
- WebAssembly: `wasmtime run --dir=. 100on100-v5.wasm input > samples`.

The C library (`100on100v5.h`, `lib100on100v5.a`) has three functions:
`i100v5_probe` (header only: sizes and the workspace a decode needs), `i100v5_decode` (the whole
picture) and `i100v5_decode_tile` (one tile of one plane). As in the original library, you
allocate the workspace and the output, the library allocates nothing and keeps no global state,
two decodes with two workspaces may run at the same time, and the output is to be read only after a
return of 0. Codes 100 and above are the library's own: 100 workspace too small, 101 output too
small, 102 sizes too large for this platform's `size_t`, 103 a NULL pointer or a negative plane.
Refusal codes the header lists for the decoder itself: header stage 11, 12 to 15, 16, 50, 17, 22,
20, 21, 23, 42, 41, 45, 29, 36, 37; payload stage 19 (tile length), 31, 38 (tile checksum), 40, 44,
46, 47, 48, 49. The header gives the workspace formula in octets.

For colour pictures the format is locked and a reference decoder is built. Single-plane greyscale and infrared, signed samples, and multispectral planes follow. This release includes a decoder for colour pictures.

8,167 octets (the small build of 1.1.0 only, not the fast build): the whole version 5 colour C99 decoder library, code and data (x86-64, built -Os with function sections, linked with --gc-sections, minus a no-codec baseline)

Pooled bits per sample, version 5 colour coder: 3.5913 on 120 COCO pictures (8-bit), 3.5131 on 35 CC0 camera pictures (8-bit sRGB), 9.2552 on the same 35 pictures (16-bit linear), 2.6962 on 120 Wikimedia Commons pictures (8-bit); every file decoded exactly by the C library.

Among coders whose decoder is at most 98,304 octets, the version 5 colour coder is the smallest in size on COCO-120, the CC0 cameras and Commons-120. Pooled, its files are smaller than JPEG-LS (CharLS, whole picture, 84,335-octet decoder) by 21.0% / 6.2% / 9.9% on COCO / cameras 8-bit / cameras 16-bit and by 20.8% on Commons, and smaller than CCSDS 121 (libaec, per tile-plane, 6,435-octet decoder) by 29.8% / 19.7% / 14.6% and by 33.7% on Commons.

JPEG XL (effort 9) produces smaller files than the version 5 colour coder: by 9.5% on COCO-120, 7.9% on the CC0 cameras (8-bit sRGB), 1.6% (16-bit linear) and 20.9% on Commons-120. JPEG XL's decoder is 713,139 octets and does not meet the 98,304 cap. Commons-120 is a quality-filtered pool (selection bias). Decoder sizes are x86-64 builds, not RISC-V.

## Fast builds (new in 1.2.0)

`100on100-v4opt` and `100on100-v5fast` (with `lib100on100v4opt.a`, `lib100on100v5fast.a` and the
headers `100on100v4opt.h`, `100on100v5fast.h`) are fast builds of the original decoder and of the
version 5 colour decoder. They are used exactly like the small builds `100on100` and
`100on100-v5`: same command line, same exit status, same output. They are offered beside the small
builds, which are unchanged and remain the recommended choice where footprint matters. The
functions are named `i100o_*` and `i100v5f_*`, so a program can link a small and a fast build
together. No size and no speed figure is stated for the fast builds.

## Version 5 generic-plane decoder (new in 1.2.0)

Version 1.2.0 includes a decoder for generic planes (format version 5, kind 0): 1 to 255 separate planes of 1 to 16 bits, unsigned or signed, or the four phases of a raw sensor mosaic as one array. The format is locked.

    100on100-k0 input > samples
    100on100-k0 -t PLANE TILE input > samples
    100on100-k0 -i input
    echo $?

- `input` is one generic-plane container. The tool writes the planes one after the other, each row
  by row; for a mosaic it writes the one combined array (plane 0 at even row and even column,
  plane 1 at even row and odd column, plane 2 at odd row and even column, plane 3 at odd row and odd
  column). Each sample is one octet when the bit depth is 8 or less, otherwise two octets,
  little-endian; signed samples are two's complement. There is no PGM header.
- With `-t PLANE TILE` the tool decodes one tile of one plane on its own and writes its samples,
  row by row, two octets each, little-endian.
- With `-i` the tool prints one line with the header's values (width, height, depth, number of
  planes, tile code, mosaic and signed flags, tile grid, octets per sample, output size and
  workspace size) and writes no samples.
- The exit status is the decoder's numbered code: `0` on success, otherwise the refusal code.
  Nothing is written to standard output when the code is not 0. The tool's own statuses are 64,
  66, 70 and 74, as for the other tools.
- WebAssembly: `wasmtime run --dir=. 100on100-k0.wasm input > samples`.

The C library (`100on100k0.h`, `lib100on100k0.a`) has three functions: `i100k0_probe` (header
only), `i100k0_decode` (the whole file) and `i100k0_decode_tile` (one tile of one plane, returned as
32-bit integers, signed values as signed). As for the other libraries, you allocate the workspace
and the output, the library allocates nothing and keeps no global state, and the output is to be
read only after a return of 0. Codes 100 and above are the library's own: 100 workspace too small,
101 output too small, 102 sizes too large for this platform, 103 a NULL pointer or a negative
plane. The header lists the decoder's refusal codes, the workspace formula and the stack use, and
states one limit: for hostile files, a tile decoded alone can differ from the whole-file decoder in
its status when chain tiles hold extreme values.

20,083 octets: the whole generic-plane (kind 0) version 5 C99 decoder program with its command line, code and data (x86-64, gcc 15.2.0, built -Os with function sections, linked with --gc-sections, minus a no-codec baseline). Working memory is separate and not included: decoding a four-plane file at 256 x 256 tiles needs about 2.2 MB (2,196,776 octets for a 512 x 512 mosaic).

On 67 raw camera sensors, as written files of the generic-plane coder (format version 5), pooled size is 0.36% smaller than JPEG XL at effort 7 (36 of the 67 cameras smaller). JPEG XL effort 7 only: effort 9 was not measured on these files. JPEG XL file sizes are as recorded earlier, with 5 of the 67 re-encoded and equal. Pooled means summed over all 67 cameras; 31 of the 67 cameras are larger than JPEG XL at effort 7, three of them by 27.2%, 8.9% and 6.9%.

For files of the generic-plane coder (format version 5), 0 of 9,600 damaged files produced undetected corruption (95% upper bound about 0.03%), tested with CRC-unaware random damage on one-tile 512 x 512 crops. Same test, same crops and seeds: JPEG XL 0.3%, JPEG-LS 9.3%, CCSDS 121 47.4%, JPEG 2000 75.0% undetected. 100on100 enforces header and per-tile checksums; CCSDS 121 and JPEG 2000 omit stream-level integrity checks by design.

This is one coding mode on one kind of data. JPEG XL at effort 9 produces smaller files than our version 5 colour coder (our files are larger by 10.5% on COCO-120, 8.6% on CC0 camera pictures in 8-bit sRGB, 1.6% in 16-bit linear, 26.4% on Wikimedia Commons-120). On hyperspectral data the format is behind CCSDS 123 (about 12 to 13%, stateless); on multispectral it is level (pooled). We do not publish speed figures.

## RISC-V reference image

`100on100-v5-riscv64.elf` is the version 5 colour decoder as a RISC-V image, for audit.

35,344 octets: the whole RISC-V decoder image, code and data (stack and heap reserved, not stored)

It is a statically linked 64-bit RISC-V executable (ELF, entry address 0x80000000, no operating
system) and is not a Linux program. It runs only with the reference runner, which is not published.
No source code is provided. This release does not document how the image is fed its input or
returns its output.

On non-conforming files (a list plane with a bad index and an out-of-range value), the image reports 40 where the C decoders report 31.

## Status

Evaluation release. Source code is not provided. See `RELEASE-NOTES.md` for what is and is not
included, and `LICENSE` for the terms.
