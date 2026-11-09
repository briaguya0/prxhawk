# Can Neversoft's LZSS compression be reproduced byte for byte?

Type: research
Status: resolved

## Question

For the full round trip, recompressing an unchanged entry must give exactly the original bytes. Can a compressor reproduce Neversoft's match choices? Check Revert's thugkit PRE codec (claims a byte-identical round trip), other existing compressors, and test against the user's own THUG2 / THUG Pro archives locally. Parameters: doakickflip notes/formats.md. Nothing game-derived committed.

Research: [research/reproduce-lzss.md](../research/reproduce-lzss.md)

## Answer

Yes. Okumura's LZSS.C (binary-tree match finder, ring pre-filled with 0x20) re-encodes every compressed entry exactly: all 1972 in THUG Pro with a fresh buffer per entry, all 9668 in THUG2 with the buffer carried over between entries (the two installs differ on that). Greedy encoders and zero-filled rings do far worse. No existing tool re-encodes byte-identically (thugkit keeps original bytes). Zero-filled decoders (doakickflip's prx.py, nspre, thugkit) mis-decode 473 entries. Details: research/reproduce-lzss.md.
