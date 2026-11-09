# nspre write fidelity against THUG2 / THUG Pro archives

Question: does nspre's writer (`nspre::write`) reproduce THUG2 / THUG Pro PRE/PRX
archives byte-identical? Ticket: [02](../issues/02-nspre-write-fidelity.md).

**Answer: no.** It cannot reproduce a single archive from either game. Its
layout (header, entry framing, name-length rule, data alignment) is right; the
differences are compression, the name checksum, slash rewriting, and
padding-byte content.

Sources:

- nspre at the pinned commit:
  <https://github.com/BryanRykowski/nspre/blob/34c0658503d9f1a613738d7565bbe4ed6b9782ee/nspre.hpp>
  (cited as `nspre.hpp:L`)
- doakickflip `notes/formats.md` and `tools/prx.py`. Its `replace()` copies the
  original name bytes, checksum, and compressed blobs verbatim and zero-pads
  everything, and a no-op repack reproduces the originals.
- Empirical check: a Python re-implementation of `nspre::write` as written
  (`nspre.hpp:621-720`), run on the user's 115 THUG2 and 206 THUG Pro
  archives. Each archive was parsed, compressed entries were decompressed with
  `prx.lzss_decompress`, the entries were fed to the simulated writer in their
  original order with their stored names, and the output was compared. Only
  counts are given here.

## What nspre writes (`nspre.hpp:621-720`)

| Field | nspre behaviour | Source |
|---|---|---|
| total size (u32 @0) | `tellp()` after all entries, patched in at the end | 712-714 |
| version (u16 @4) | constant 3 | 695-696 |
| magic (u16 @6) | constant 0xABCD | 697-698 |
| file count (u32 @8) | caller's count | 699 |
| entry order | caller's order, unchanged | 706-710 |
| size | length of the source file | 658 |
| compressed_size | **always 0**: no compressor exists (README: "Can't perform compression") | 659 |
| name_len | `len + (4 - len % 4)`: always 1-4 NULs, rounded up to a multiple of 4 | 650-655, 660 |
| name bytes | caller's prepath with **`/` rewritten to `\`**; case is kept | 645-648 |
| name_checksum | CRC-32 without the final XOR (`string_crc`) over the caller's prepath **as given**: not lowercased, and taken from the original string, not the slash-converted one | 593-606, 661 |
| name padding | zero bytes | 654 |
| data | raw source bytes | 679 |
| data padding | zero bytes up to a multiple of 4 | 674-677 |

These agree with formats.md / prx.py: the header layout, version and magic,
the total-size semantics (the header total equals the file length in all 321
archives), the 16-byte entry header, name_len always a multiple of 4 with at
least one NUL (all 16,214 entries follow nspre's rule exactly), data padded to
4, the CRC polynomial and seed, and caller-controlled order (the originals use
no single sort rule, so the caller must keep the original order).

## Differences

Counts are entries / archives from the user's install.

| # | Difference | THUG2 (14,199 entries, 115 archives) | THUG Pro (2,015 entries, 206 archives) |
|---|---|---|---|
| 1 | **Compression.** Originals contain LZSS-compressed entries (`compressed_size != 0`). nspre writes every entry stored, so the compressed_size, data bytes, data padding, every later offset, and the total size all differ. | 9,668 compressed / 4,531 stored; all 115 archives have at least one compressed entry | 1,972 compressed / 43 stored; all 206 archives have at least one |
| 2 | **Checksum case.** Originals hash the *lowercased* path (formats.md). nspre hashes the prepath as given and never lowercases it (`nspre.hpp:661`). An entry whose stored name has uppercase letters cannot get both its name bytes and its checksum right. | 12,587 entries with uppercase names | 955 entries in 62 archives |
| 3 | **Forward slashes.** Some THUG2 entries store their name with `/`. nspre always rewrites `/` to `\` in the stored name (`nspre.hpp:645-648`), so those name bytes cannot be reproduced. The original checksum for these entries is the CRC of the lowercased name *with `/` converted to `\`* (verified for all 3,740). nspre hashes the unconverted string instead (`nspre.hpp:661`), so it gets these checksums wrong even when the caller passes the name lowercased. | 3,740 entries | 0 |
| 4 | **Non-zero name padding.** In the originals the bytes after the name's NUL are not always zero (they look like leftover buffer contents). nspre always writes zeros. | 8,464 entries, 113 archives | 0 |
| 5 | **Non-zero data padding.** The same applies to the 1-3 alignment bytes after entry data. | 7,546 entries, 108 archives | 0 |

Checksum totals: for 12,890 THUG2 entries (111 archives), the stored checksum
is not the CRC of the stored name bytes, which is what nspre would compute when
given the stored name. Rows 2 and 3 explain the gap. THUG Pro: 955 entries (62
archives).

Result of the simulated writer: 0 of 115 THUG2 archives and 0 of 206 THUG Pro
archives are byte-identical. Compression (row 1) alone makes every archive
differ. Leaving compression aside, THUG Pro still differs on checksums in 62
archives, and THUG2 differs in nearly all archives because of rows 2-5.

## Read → write round trip through nspre's own Reader

This path does not appear in the examples, but it matters for a port:

- `Reader` stores `prepath` as the full `name_len` bytes, **including the
  trailing NUL padding** (`nspre.hpp:264-278`). If that string goes straight
  back into `write`, the length is already a multiple of 4, so 4 more NULs are
  added (`nspre.hpp:652`). name_len grows by 4 for **every** entry, and the
  checksum is computed over the NUL and padding bytes, so it is wrong for every
  entry. The caller must cut the prepath at the first NUL.
- `Reader` rejects archives with more than `NSPRE_MAX_COUNT` = 200 entries
  (`nspre.hpp:42-44, 242`): 22 THUG2 archives and 1 THUG Pro archive. The
  limit can be overridden with a define.
- `Reader::Subfile::subheader()` copies 16 bytes into an empty vector
  (`nspre.hpp:302-305`), which is undefined behaviour. Not on the write path.
- `ns-unpack` extracts each entry to `outdir / filename()`, which drops the
  directories (`unpack/main.cpp:237`), and `ns-pack` needs the caller to pass
  an explicit `source,internal\path` list (`pack/main.cpp:41-65`). The example
  tools have no built-in way to rebuild an archive.

## What a port needs to be byte-identical

To reproduce an archive, the writer must be able to carry these through
unchanged from the original entry, as prx.py's `replace()` does:
`compressed_size` and the compressed blob (or a bit-exact compressor, see
ticket 01), the stored name bytes (including `/` and case), the original
checksum (or a function that computes it from lowercase with `/`→`\`), and the
original padding bytes. nspre's layout logic can be kept as it is.
