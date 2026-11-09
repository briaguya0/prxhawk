# Can Neversoft's LZSS be reproduced byte for byte?

Ticket: [01-reproduce-lzss](../issues/01-reproduce-lzss.md). Researched 2026-11-08.

## Answer

Yes. Every LZSS-compressed entry in the user's THUG2 and THUG Pro archives
re-encodes byte-identically with Haruhiko Okumura's classic `LZSS.C`
(1989) encoder (binary-tree match finder), run with the parameters already in
doakickflip `notes/formats.md` and a ring buffer pre-filled with spaces
(`0x20`). One extra detail decides the last few bytes of small entries: whether
the encoder's buffer is wiped between entries (THUG Pro) or carried over from the
previous entry (THUG2).

| Install  | Compressed entries | Okumura, fresh buffer | Okumura, buffer carried over | Best per install |
|----------|-------------------:|----------------------:|-----------------------------:|-----------------:|
| THUG Pro | 1972 (206 archives) | **1972** | 1969 (3 archives miss) | 1972 / 1972 |
| THUG2    | 9668 (115 archives) | 9617 (1 archive misses) | **9668** | 9668 / 9668 |

The simpler encoders do much worse. Neither existing compressor (Revert's) nor a
greedy one matches the original choices:

| Install  | greedy, longest match, nearest wins tie | greedy, longest, farthest wins tie | Okumura, ring fill `0x00` | Okumura, no initial space-string insertions |
|----------|----:|----:|----:|----:|
| THUG Pro | 79  | 227  | 12 | 1800 |
| THUG2    | 1331 | 2173 | 9 | 9224 |

## The algorithm that matches

Okumura's `LZSS.C` `Encode()`, unchanged apart from I/O:

- `N = 4096`, `F = 18`, `THRESHOLD = 2`. Flag byte LSB first, `1` = literal.
  A match is written as `pos & 0xFF`, then `(pos >> 4 & 0xF0) | (len - 3)`,
  where `pos` is the **absolute ring index** of the match source, not a
  distance.
- `text_buf` holds `N + F - 1` bytes. `text_buf[0 .. N-F)` is filled with `0x20`
  and `r = N - F` (4078). Up to `F` input bytes are read into
  `text_buf[r ..]`.
- Before the first token, the `F` strings starting at `r-1, r-2, …, r-F` (all
  spaces) are inserted into the tree, then `r` itself. That means the encoder
  can emit matches into the space-filled region before it has been written.
- One binary search tree per leading byte (`rson[N+1+c]` roots). `InsertNode(r)`
  walks the tree, comparing up to `F` bytes. It goes right on `cmp >= 0` and
  left otherwise, and keeps the **first** node on the path that gives a
  strictly longer match. On a full-length (`F`) match it replaces the old node
  with the new one, so later full matches point at the more recent copy.
  `DeleteNode(s)` removes the string that falls out of the window before each
  new byte is read.
- After each token it reads one byte per consumed position, mirroring
  `text_buf[s + N]` for `s < F - 1`. Near end of input it keeps inserting
  without reading, and the lookahead shrinks (`len--`). `match_length` is
  clamped to `len`.
- The final partial group is flushed if it holds any tokens.

These details come from the ports listed below. Okumura's original could not be
fetched during this research (the author's site timed out). Every step above
was confirmed by the experiment, not only by reading.

### The end-of-buffer detail (fresh vs carried-over buffer)

When fewer than `F` bytes remain, `InsertNode` still compares `F` bytes, so it
reads lookahead cells past the end of input. Only the space region
`[0, N-F)` and the bytes actually read get rewritten per entry, so those cells
hold whatever the buffer held before. They change only which of several
equal-length candidates the tree walk finds first:

- All 51 THUG2 misses with a fresh buffer were in a single archive. All were
  entries smaller than `N`, and all diverged within the last `F` bytes. In
  every case only the match position differed: same length, same token
  structure, same compressed size, and the original source was farther back.
- Keeping one `text_buf` across entries in archive order fixes all of them.
  The buffer starts zeroed (a C static) and is never cleared, apart from the
  per-entry space fill and the bytes the new entry writes. With that, all 115
  THUG2 archives reproduce. Feeding the stored (uncompressed) entries through
  the same buffer or skipping them made no difference.
- THUG Pro's archives were evidently built with a fresh buffer per entry.
  Carrying it over breaks 3 entries in 3 archives.

So a repacker either needs both modes, picked per archive (try fresh, and if
the archive's untouched entries don't match, try carried over), or it keeps
original bytes for unchanged entries (see below).

## Ring fill matters for *decoding* too

In THUG Pro, 140 compressed entries contain a match that reads ring cells
never yet written. THUG2 has 333 such entries. They decode to different bytes
with a `0x00` fill than with a `0x20` fill. The `0x20`-filled decode is the one
the encoder reproduces. Decoders that zero the ring are therefore wrong for
those entries:

- doakickflip `tools/prx.py:14` (`ring = bytearray(N)`, at `77dfc48`).
- nspre's reader: `ring_buffer(4096, 0)` (`nspre.hpp:327`). prxhawk is a port of
  nspre, so the port must change this to `0x20`.
- Revert `tools/prx/lzss.py:14-15` (`init=0`) and thugkit `prx/lzss.go:21`
  (`make([]byte, lzN)`).

These fill with `0x20`: neversoft-multitool `LzssDecoder.cs:25-26`, nodeqbc
`LZSS.js:102-103`, QueenBee `Lzss.cs:46-47`.

## What the existing tools do

- **Revert / thugkit**: the byte-identical round trip it claims is at the
  **archive** level. Revert's docs say `Build(Parse(x)) == x`
  (revert `docs-site/codecs.md:26`). It works because `Parse` keeps each entry's
  original blob and `Build` writes it back verbatim (thugkit `prx/prx.go:125,
  153`). The compressor is a greedy hash chain (newest-first, capped at 256
  candidates, longest wins), and its own comment says it "need not
  byte-match" (thugkit `prx/lzss.go:8-9, 60-141`). It is only checked for
  compress → decompress round trip (`prx/lzss_test.go`). It is used when an
  entry is replaced (`ReplaceCompressed`, `prx/prx.go:211-219`). The Python
  original's header still says no compressor exists
  (revert `tools/prx/lzss.py:10-11`).
- **nspre**: no compressor. `write` always emits stored entries with
  compressed size 0 (`nspre.hpp:658-659`). It reads compressed entries only.
- **nodeqbc `LZSS.js`**: a JS port of QueenBee's `Lzss.cs`, which is in turn
  Okumura's `LZSS.C`. It has the space fill (`LZSS.js:242-243`), the initial
  `F` insertions (`:269-275`), and the tree with the same tie rules
  (`:404-475`). It is the same algorithm as the matching encoder above, but it
  creates a fresh `text_buf` per handler, so it covers the THUG Pro mode only.
  It has no licence (doakickflip `notes/qb-lib.md:132`).
- **neversoft-multitool**: decoder only (`Core/BinaryIO/LzssDecoder.cs`). It
  has no LZSS encoder.

## Implications for prxhawk

- An unchanged entry can always be made byte-identical without an encoder, by
  keeping the original compressed bytes, as thugkit does. A faithful encoder is
  only needed if prxhawk recompresses content it decompressed. Example: a
  `.qb` that came back identical through `.ns`.
- If it does recompress, Okumura's `LZSS.C` is the algorithm, with a per-archive
  fresh or carried-over buffer. Licence: the map allows MIT "as long as nspre is
  the only code we port from". nodeqbc has no licence, and QueenBee was not
  checked. Writing it from this description, or from Okumura's original (which
  I recall carries a "use, distribute, and modify freely" notice; not verified
  here), would need that map note revisited.
- The nspre port's decoder must fill the ring with `0x20`.

## Experiment

The script is at
`/tmp/claude-1000/-var-home-briaguya-code-doakickflip/c228a98a-3094-45e8-a5fd-d64f593368b4/scratchpad/exp/`
(`lzss_repro.py`, `probe.py`, `probe2.py`, `probe3.py`). These are scratch files,
not committed. It decodes each compressed entry, re-encodes it with each
candidate, and compares the output with the stored bytes. It covers all 321
`.prx` files in the user's THUG Pro and THUG2 installs. Each encoder was first
checked to decode back to its input on random data. Only counts were recorded.

## Sources

- Revert, github.com/violetvandal/revert @ `422c1b9b0e438792db0b57b40b07f33b5fc985b1`
- thugkit (Revert submodule), github.com/violetvandal/thugkit @ `09f4228d2bdd199a1594b8e1f039b1596313d438`
- nspre, github.com/BryanRykowski/nspre @ `34c0658503d9f1a613738d7565bbe4ed6b9782ee`
- nodeqbc, gitgud.io/fretworks/nodeqbc @ `85500d5faa0d880ce721d884bf991453332fd371`
- Queen-Bee, github.com/Nanook/Queen-Bee @ `f30ddc034dc6168e37e59ae828952876949b6d65` (`QueenBeeParser/Qb/Lzss.cs`)
- neversoft-multitool, github.com/slfx77/neversoft-multitool @ `3c9028e0178deb9060190362d1c47039a32846e2` (`src/NeversoftMultitool/Core/BinaryIO/LzssDecoder.cs`)
- Okumura, `LZSS.C` (1989). Known here through the ports above. Not fetched.
