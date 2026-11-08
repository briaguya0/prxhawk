# Can Neversoft's LZSS compression be reproduced byte for byte?

Type: research
Status: open

## Question

For the full round trip, recompressing an unchanged entry must give exactly the original bytes. Can a compressor reproduce Neversoft's match choices? Check Revert's thugkit PRE codec (claims a byte-identical round trip), other existing compressors, and test against the user's own THUG2 / THUG Pro archives locally. Parameters: doakickflip notes/formats.md. Nothing game-derived committed.
