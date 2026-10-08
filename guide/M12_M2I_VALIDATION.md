# M12 / M2I and safe loader fallback — 2026-10-08

Validated firmware version: **2.0.2**.

## Audit and changes

Already present: all NORMAL, MZ700 FAST3, IC, TC and UL families; MFI and
streaming per-record MTI parsing; logical MZQ/QDF and physical MFM QD adapters;
physical-source callbacks for generated loaders; immediate physical FIFO fill
with full duration measurement deferred to PLAY. No new QD loader was needed.

Missing: M12 adaptation to the MZF loader input, M2I AUTO lookup and browser
indication, and a source-independent recovery after loader rejection. The
complete fallback was restricted to MZQ. Single-file mode getters could also
return the rejected request rather than the effective playback profile.

Implemented:

- Adapt only M12's loader input to MZF, preserving source identity and existing
  content, file-size, address-space, receiver and workspace safety checks.
- Exact metadata mapping MZF -> MFI, M12 -> M2I, MZT -> MTI. MFI/M2I share one
  single-record reader/parser. Manual selection wins; absent/invalid metadata
  uses NORMAL 1:1. No cross-format lookup or machine inference is added.
- Apply the existing QD fallback centrally to every Sharp source. Reset the
  loader, restore the entire NORMAL 1:1 profile and clear UL leader state.
  Recalculate single-file duration under the recovered profile. Containers
  retain their existing per-record/deferred physical measurement paths.
- Expose the effective profile through the existing getters. Single Sharp
  READY screens show it, and AUTO M12 shows M2I as its metadata origin. M2I
  uses the existing browser presence indicator and is hidden from entry counts.

Unchanged: loader binaries, safety rules, direct NORMAL/MZ700 1:1 timing,
EEPROM mode values, recording, SD initialization/retries, physical QD parsing,
FIFO/source handling, UL handshake/progress behavior and keypad navigation.

## Changed files

- `src/formats/mzi_sidecar.h`, `.cpp`: exact mapping and shared single-record API.
- `src/play/mzf_playback.cpp`, `.h`: M12 adapter, generic fallback and effective mode.
- `src/play/play_engine.cpp`: M2I AUTO resolution and effective mode getter.
- `src/play/play_controller.cpp`: common sidecar presence lookup.
- `src/drivers/sdcard.cpp`: hide M2I metadata in the browser.
- `src/ui/browser.cpp`, `src/ui/play_screen.cpp`: presence, M2I prefix and READY profile.
- `README.md`, `guide/MFI_MTI_FORMAT.md`, `guide/USER_HANDBOOK.md`,
  `guide/QUICK_REFERENCE_GUIDE.md`, this report: usage, fallback and validation.
- Local `verification/quickdisk_host.py`, `verification/keypad_host.py`,
  `verification/README.md`: automated regression coverage. The repository's
  existing ignore rules exclude `verification/` and `tests/`; these changes
  and fixtures remain local unless explicitly included when sharing them.

## Automated validation

Passed on 2026-10-08:

```text
python verification/quickdisk_host.py C:\msys64\ucrt64\bin\g++.exe
python verification/keypad_host.py C:\msys64\ucrt64\bin\g++.exe
pio run -e sd2cmt2
```

The host tests compile production playback, loader, profiles, sidecar parser,
AUTO preparation and LCD/event code with memory SD and AVR substitutes. The
browser filter is compiled verbatim from the SD driver. Coverage includes:

- Exact extensions, mixed-case source names/FAT-style lookup, no cross-lookup,
  sidecar presence and hidden metadata; directories remain visible.
- M12 AUTO without M2I, valid/invalid M2I, manual priority, coexistence of
  same-basename MZF/M12, and unsafe M2I loader fallback.
- MZF/MFI regression and genuine streaming MZT/MTI per-record resolution.
- All manual M12 modes compared to equivalent MZF preparation, including
  every fast family, direct NORMAL profiles and MZ700 1:1.
- Rejection of all eleven generated profiles in MZF, M12, MZT, logical MZQ,
  legacy logical QD, QDF, physical FlashFloppy and HxC paths. Assertions compare
  header/data leader and mark counts, all contextual short/long pulse pairs,
  duplicate gap, UL state, inactive loader variant, effective mode and duration
  against explicit NORMAL 1:1 preparation of the same record.
- Existing source bounds, UL source transfer, corruption/I/O errors, FIFO
  prefill, deferred duration cancellation, record navigation and keypad repeat
  regressions; M2I/MFI LCD prefixes and effective NORMAL/UL READY labels.

Physical fallback tests modify the synthesized header after real MFM record
analysis, retaining the actual decoder/payload source. HxC uses a synthetic
descriptor around the captured FlashFloppy track, not an independent HxC
capture. These tests do not establish physical CMT timing or handshake reliability.

Build: flash **134,004 / 253,952 bytes (52.8%)**, static SRAM
**5,688 / 8,192 bytes (69.4%)**. SRAM remains unchanged; flash increases by
274 bytes against the previous 133,730-byte build. Stack headroom is not
included in static SRAM accounting. Upload image:
`.pio/build/sd2cmt2/firmware.hex`.

## Hardware checklist — not yet performed

1. On the intended Sharp target, play compatible M12 files manually with IC,
   TC, MZ700 FAST3 and each applicable UL variant; compare loading/execution
   against the same record as MZF. Also check NORMAL/MZ700 1:1.
2. Test M12 AUTO with valid, missing and malformed M2I. Put MZF/MFI and M12/M2I
   with the same basename on the card and assign different profiles. Check
   exact lookup, manual override, browser `I`, hidden metadata and M2I prefix.
3. Use records rejected by existing loader safety checks for UL, UL800, UL700,
   IC, TC and FAST3. Check the effective N11 label, normal duration and successful
   standard loading; with a logic analyzer compare both leaders, marks and
   data pulses with explicit NORMAL 1:1. Repeat on logical MZQ, QDF and actual
   physical HxC/FlashFloppy QD, especially the rejected UL cases.
4. Check physical selection already fills the FIFO, PLAY completes the deferred
   duration scan, cancellation leaves the selector usable, and compatible
   records still use the originally requested fast profile. Verify all three
   Barbarian II records remain selectable with short/held FFWD/REWIND presses.
5. Recheck MZT per-record AUTO/MTI, MOTOR/SELECT boundaries, STOP, completion,
   card removal/read errors, sustained MFM throughput and Mega2560 stack headroom.
