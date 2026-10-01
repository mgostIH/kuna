# Compact native switch: selector/label domain mismatch

This is an original, asset-free x86-64 ELF fixture: 85 instruction bytes,
a four-byte selector map, and four relative jump-table entries. Python standard
library only; no compiler or game files are required.

```sh
python3 repro.py --kuna /absolute/path/to/kuna --sleighpath /absolute/path/to/specs
# Optional: actually execute the authored fixture on x86-64 Linux.
python3 repro.py --build-only --native-check
```

All declarations are accepted in reliable strict mode. The entry uses an explicit
MSABI signature: RCX is `struct ModeStore *store`, EDX is unsigned `mode`.

The function guards `index = mode - 5` to 0..3, reads `map[index]`, then branches
through `target[map[index]]`:

| Index | Map byte | Native destination |
| --- | --- | --- |
| 0 | 0 | return300 |
| 1 | 3 | return270 |
| 2 | 1 | return240 |
| 3 | 2 | retry with `store->mode` |

The retry refuses stored mode8, subtracts5, bounds the new index, and returns to
the same lookup. Out-of-range values return270. Thus mode7 must return240,
independently of the stored mode. Mode6 must return270, independently of the
stored mode. Mode8 with stored5 returns300; with stored7 returns240.

Affected Kuna emits `switch(*(char *)(mode_index + MAP_ADDRESS))`, but its
`case 1` returns270 and `case 2` returns240. Those labels belong to the original
index, not to the mapped selector: mode7 reads map byte1 and returns270 in the
printed C. This is a semantic error, not merely an awkward local name.

The optional native check exercises 289 input pairs against the reference below:

```c
int reference(unsigned mode, unsigned stored) {
    if (mode == 8) mode = stored;
    return mode == 5 ? 300 : mode == 7 ? 240 : 270;
}
```

The ELF includes a ten-byte SysV-to-MSABI bridge used only by that check.
`repro.py --build-only` writes the ELF and .kuna file without running Kuna.
