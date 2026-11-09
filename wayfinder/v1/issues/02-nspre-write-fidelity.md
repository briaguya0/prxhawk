# Does nspre write THUG2 and THUG Pro archives byte-identical (header fields, entry order, padding)?

Type: research
Status: resolved

## Question

nspre (BryanRykowski/nspre at 34c0658503d9f1a613738d7565bbe4ed6b9782ee) is the code we port. Does its writer reproduce THUG2 / THUG Pro archives exactly: header fields, entry order, alignment / padding, checksums? List any differences.

Research: [research/nspre-write-fidelity.md](../research/nspre-write-fidelity.md)

## Answer

No: 0 of 115 THUG2 and 0 of 206 THUG Pro archives match. Header, entry headers, data padding, CRC and order are right. Wrong: never compresses; hashes names without lowercasing; hashes before converting / to \\; zero padding where some THUG2 originals have non-zero bytes. Reader keeps name padding (breaks a read-write round trip), refuses archives over 200 entries, zero-fills the LZSS ring. Port its structure, not its behaviour. Details: research/nspre-write-fidelity.md.
