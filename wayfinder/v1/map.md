# Map: prxhawk

## Destination

A buildable spec for prxhawk reading and writing THUG2 and THUG Pro `.prx`
archives, byte-identical on repack, usable from both the browser and Node.
Building happens outside the map.

## Notes

- Domain: Neversoft PRE / PRX archives and their LZSS. Background:
  doakickflip's `notes/qb-lib.md` ("PRX / LZSS") and `notes/formats.md`.
- Consumers: doakickflip's app (install / uninstall) and nsqb's playground.
- Tracker: local markdown. Tickets in [issues/](issues/), research write-ups
  in `research/`. Research stays on the docs branch; nothing pushed without
  asking.
- Nothing derived from THUG2 / THUG Pro is committed: no archives, no
  extracted entries. Describe findings in our own words.
- One question at a time when grilling. Grilling tickets: use the grilling
  and domain-modeling skills.
- Scope, decided while charting (2026-11-08):
  - A TypeScript port of BryanRykowski/nspre (MIT, pinned to
    `34c0658503d9f1a613738d7565bbe4ed6b9782ee`), keeping its notice.
  - Licence MIT, as long as nspre is the only code we port from; reconsider
    if other repos get referenced.
  - THUG2 and THUG Pro archives only.
  - The bar is the full round trip: `.prx` → `.qb` → `.ns` → `.qb` → `.prx`,
    both "after"s matching both "before"s. Stored (uncompressed) entries
    are the fallback if exact recompression isn't possible.
  - Browser and Node: bytes in and out, no file access of its own.
  - Library only.
  - Real archives are round-trip checked locally only; CI runs archives we
    build from our own content. How the local check runs is decided in
    nsqb's `wayfinder/v1` ticket 09; follow that.

## Decisions so far

## Not yet specified

- prxhawk's public API.
- How entries are addressed when their names aren't known (THUG Pro's
  checksums without names).

## Out of scope

- Other games' archive variants (PRE3, ...).
- A CLI (`ns-pack` / `ns-unpack` style).
- A playground: nsqb's playground opens `.prx` files.
