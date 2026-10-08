# CODEX TASK — MZ-SD2CMT2-Reborn: M12 fast-loader support, M2I sidecar, universal safe fallback

## Repository

`bales0/MZ-SD2CMT2-Reborn`

Work against the current default branch / current master.

---

# 0. Mandatory first phase: audit the current implementation

Before making changes, inspect the current repository and verify whether any part of this task has already been implemented since this specification was written.

At minimum, inspect:

- `src/play/loader_mode.h`
- `src/play/mzf_loader.h`
- `src/play/mzf_loader.cpp`
- `src/play/mzf_playback.cpp`
- `src/formats/file_format.h`
- `src/formats/file_format.cpp`
- `src/formats/mz_loader_profiles.h`
- `src/formats/mz_loader_profiles.cpp`
- MFI/MTI sidecar implementation and parser
- AUTO loader-selection logic
- documentation describing MZF/MZT/M12 and MFI/MTI
- any tests covering loader selection, MZT/MZQ logical records, sidecars, and loader fallback

Do **not** blindly duplicate code that already exists.

Document in the final Codex report:

1. what was already present,
2. what was missing,
3. what was changed,
4. what was intentionally left unchanged.

---

# 1. Goal

Extend the existing Sharp tape playback architecture with:

1. full manual fast-loader support for `.M12`,
2. AUTO sidecar support for `.M12` using a dedicated `.M2I` extension,
3. a universal, safe fallback to `NORMAL 1:1` whenever a requested generated/turbo/UL loader cannot be prepared safely.

Do not add unrelated features or new UI workflows.

**Important QuickDisk scope clarification:** the safe loader fallback is not
limited to logical MZQ records. It must also cover records obtained from the
physical QuickDisk / physical-MFM analysis path. The current source maps
`.MZQ`, `.QD`, and `.QDF` to `FILE_FORMAT_MZQ` and distinguishes physical
QuickDisk input internally; both paths are in scope for the fallback fix.

---

# 2. M12 must be accepted by the loader layer

The current architecture treats M12 as a Sharp tape format with the same basic single-record layout:

- 128-byte Sharp tape header,
- followed by the declared data block.

The loader subsystem should therefore be allowed to process a valid M12 record using the same loader machinery as an MZF logical record.

## Required behavior

Manual loader/profile selection for `.M12` must support the same applicable modes as `.MZF`, subject to the existing safety checks.

This includes the currently implemented loader families such as:

- MZ700 FAST3,
- IC 1:1,
- IC 1:2,
- IC 1:3,
- IC 1:4,
- TC 1:1,
- TC 1:2,
- TC 1:3,
- UL,
- UL_MZ800,
- UL_MZ700,

plus all already existing direct NORMAL profiles.

Do not weaken any existing content or memory-safety validation.

Existing rules must continue to apply, including at least:

- supported Sharp file types,
- IC requiring machine-code type where already required,
- non-zero payload length,
- 64 KiB address-space bounds,
- file-size/source bounds,
- low/high loader receiver placement,
- overlap checks,
- MZ700 loader/runtime overlap checks,
- IC workspace overlap checks,
- TC workspace overlap checks,
- MZ800/UL high-stage overlap checks,
- QADCN encodability checks,
- any other current loader-specific safety checks.

## Preferred architecture

Do not create duplicate M12-specific loader binaries.

Prefer one of these approaches:

### Minimal safe change

Adapt M12 to the existing logical-MZF loader path before calling `mzf_loader_prepare()`.

For example, conceptually:

```cpp
file_format_t loader_format = mzf_format;

if ((mzf_format == FILE_FORMAT_M12) ||
    file_format_is_record_container(mzf_format))
{
    loader_format = FILE_FORMAT_MZF;
}
```

Use the current project abstractions and naming rather than copying this literally if the code has changed.

### Better architectural option

If the audit shows that `mzf_loader_prepare()` does not actually need the source container format, refactor the loader interface so that it receives a logical Sharp record rather than a physical file-format enum.

Do this only if the refactor is small, clear, and regression-safe.

Do not broaden the task into a large architecture rewrite.

---

# 3. Add a dedicated `.M2I` sidecar for `.M12`

M12 must gain AUTO sidecar support.

The sidecar mapping must be:

| Tape file | AUTO sidecar |
|---|---|
| `GAME.MZF` | `GAME.MFI` |
| `GAME.MZT` | `GAME.MTI` |
| `GAME.M12` | `GAME.M2I` |

The reason for a dedicated extension is that files with the same basename may legitimately coexist:

```text
FLAPPY.MZF
FLAPPY.MFI
FLAPPY.M12
FLAPPY.M2I
```

## Important rules

### `.M2I` is a new extension, not a new syntax

`.M2I` must use the same single-record sidecar syntax and semantics as `.MFI`.

Do not invent a new parser or a divergent format unless strictly necessary.

Prefer reusing the existing MFI parser/structures.

### No cross-format sidecar fallback

Sidecar lookup must be exact:

```text
MZF -> MFI
M12 -> M2I
MZT -> MTI
```

The following behavior is forbidden:

- `.M12` falling back to `.MFI`,
- `.MZF` loading `.M2I`,
- `.MZT` loading `.MFI` or `.M2I`.

Example:

If both

```text
FLAPPY.MZF
FLAPPY.M12
```

exist, then:

- `FLAPPY.MZF` + AUTO may use only `FLAPPY.MFI`,
- `FLAPPY.M12` + AUTO may use only `FLAPPY.M2I`.

If the expected sidecar is missing, use the normal AUTO fallback behavior.

### Manual mode has priority

Preserve the current rule:

- a manually selected loader/profile overrides sidecar AUTO selection.

---

# 4. AUTO behavior for M12

Current M12 behavior without a dedicated sidecar effectively falls back to normal playback.

Change this so that:

```text
M12 + AUTO:
    if matching .M2I exists and is valid:
        use its requested loader/profile
    else:
        NORMAL 1:1
```

Do not infer MZ700 vs MZ800 from the `.M12` extension.

The sidecar or explicit manual setting determines the requested loader/profile.

If the sidecar requests a loader that cannot safely be prepared for the actual record, apply the universal safe fallback specified below.

---

# 5. Universal safe fallback when loader preparation fails

This is a mandatory bug fix and applies beyond M12.

The current implementation must be audited for the case where:

1. the selected mode is a generated/turbo/UL loader,
2. playback timing/leader parameters are configured for that mode,
3. `mzf_loader_prepare()` rejects the loader,
4. playback continues without fully restoring NORMAL 1:1 configuration.

A known problematic family is:

- `UL`,
- `UL_MZ800`,
- `UL_MZ700`.

Those modes may configure UL-specific shortened leaders before loader preparation succeeds.

If preparation subsequently fails and the mode is not fully reset, the device can emit a normal data block using UL-style leader timing without a valid UL receiver.

That is incorrect.

## Required fallback behavior

For every applicable Sharp source path:

- MZF,
- M12,
- MZT logical records,
- MZQ logical QuickDisk records,
- QDF/QuickDisk image records,
- physical QuickDisk records obtained through the physical/MFM-backed QD analysis path,

if a requested generated/turbo/UL loader is rejected or cannot be safely prepared:

This requirement is intentionally source-path independent. In the current code,
`.MZQ`, `.QD`, and `.QDF` all resolve to `FILE_FORMAT_MZQ`, while playback also
tracks whether the selected QuickDisk record comes from the physical source path
(e.g. `mzf_mzq_physical_source`). The safe fallback must therefore apply equally
to:

- ordinary/logical MZQ records,
- QDF-backed records,
- records extracted from a physically analysed QuickDisk/MFM image.

Do not implement the fallback only in the lightweight logical-record adapter.
It must be effective after loader preparation fails regardless of how the Sharp
record was obtained.

1. set the effective loader mode to `NORMAL 1:1`,
2. re-run the complete normal-profile configuration,
3. restore normal header leader,
4. restore normal data leader,
5. restore normal pulse timing,
6. clear any UL-specific leader state,
7. clear/disable any prepared loader state,
8. ensure status/UI/debug information reports the effective mode rather than the rejected request where applicable,
9. continue with safe standard playback rather than partially configured turbo playback.

The fallback should be generic and centralized.

Do not retain a special fallback only for MZQ if a common path can safely cover MZF/M12/MZT/MZQ.

---

# 6. Distinguish requested mode from effective mode if needed

If the current architecture stores one variable for both:

- the user/sidecar-requested mode,
- and the mode actually used for playback,

audit whether this contributes to the fallback bug.

If useful, introduce or clarify the distinction:

```text
requested_mode
effective_mode
```

This is optional if the same correctness can be achieved cleanly with the current state model.

The user-visible behavior must remain simple.

No new menu item is required.

---

# 7. MZT and all QuickDisk source-path regression requirements

MZT and the QuickDisk playback code already expose contained Sharp records to the
loader layer. QuickDisk support must be treated as more than only a logical MZQ
container.

In the current code, `.MZQ`, `.QD`, and `.QDF` resolve to `FILE_FORMAT_MZQ`, and
the playback implementation additionally distinguishes a physical QuickDisk
source path (for example through `mzf_mzq_physical_source` and the physical/MFM
analysis/cache path).

Do not break any of these paths.

Verify that:

- MZT AUTO/MTI behavior remains unchanged,
- MZT per-record loader selection remains unchanged,
- logical MZQ record playback remains unchanged,
- QDF-backed QuickDisk playback remains unchanged,
- physical QuickDisk / physical-MFM-backed record playback remains unchanged,
- the universal loader-rejection fallback works for both logical and physical
  QuickDisk records,
- current MZQ/QD fallback behavior is preserved functionally but preferably
  absorbed into the new generic fallback,
- MZF/MFI behavior remains unchanged except for the corrected generic fallback.

The fallback must not depend on `mzf_mzq_physical_source == false`. A record
originating from the physical QuickDisk analysis path must receive the same
NORMAL 1:1 recovery if its requested loader is rejected.

---

# 8. Sidecar implementation requirements

Reuse the existing MFI/MTI infrastructure as much as possible.

Preferred design:

- MFI and M2I share the same single-record parser and data model,
- MTI remains the multi-record format,
- only sidecar extension resolution differs by source type.

Avoid copy/paste parsers.

Conceptually, there should be one mapping layer similar to:

```cpp
switch (format) {
    case FILE_FORMAT_MZF: return ".MFI";
    case FILE_FORMAT_M12: return ".M2I";
    case FILE_FORMAT_MZT: return ".MTI";
    default:              return NULL;
}
```

Use the project's actual naming and structure.

---

# 9. Documentation

Update the relevant documentation.

It must clearly state:

## Sidecar mapping

```text
.MZF -> .MFI
.M12 -> .M2I
.MZT -> .MTI
```

## M2I semantics

- `.M2I` is the M12 AUTO sidecar.
- It uses the same single-record syntax/semantics as `.MFI`.
- It is intentionally a distinct extension to allow `.MZF` and `.M12` with the same basename to coexist.
- There is no fallback between MFI and M2I.

## M12 loader support

Document that M12 can use the same supported manual fast-loader profiles as a compatible MZF logical record, subject to the normal file-type, memory-overlap, and loader-specific safety checks.

## Fallback

Document that an incompatible or unsafe requested loader automatically falls back to `NORMAL 1:1`.

---

# 10. Tests

Add or update automated tests where the project has a suitable test framework.

At minimum cover the following logic.

## A. Sidecar filename resolution

Verify:

```text
GAME.MZF -> GAME.MFI
GAME.MZT -> GAME.MTI
GAME.M12 -> GAME.M2I
```

Verify case handling according to the existing filesystem policy.

Verify no cross-format fallback.

Examples:

- `GAME.M12` must not consume `GAME.MFI`,
- `GAME.MZF` must not consume `GAME.M2I`.

---

## B. M12 AUTO without sidecar

Input:

```text
GAME.M12
```

with loader selection `AUTO` and no `GAME.M2I`.

Expected:

```text
effective mode = NORMAL 1:1
```

---

## C. M12 AUTO with valid M2I

Input:

```text
GAME.M12
GAME.M2I
```

M2I requests a supported loader.

Expected:

- M2I is parsed with the same semantics as MFI,
- requested loader is used if it passes existing safety checks.

---

## D. M12 manual loaders

Exercise the loader preparation path for representative modes:

- MZ700 FAST3,
- IC,
- TC,
- UL generic,
- UL_MZ800,
- UL_MZ700.

The test does not need to emulate the target computer, but must prove that `.M12` is no longer rejected purely because its `file_format_t` is `FILE_FORMAT_M12`.

Existing loader-specific content restrictions must still be enforced.

---

## E. Rejected UL loader fallback

This is critical.

Create/identify a record that causes each relevant UL mode to be rejected, for example due to:

- unsupported file type,
- receiver/workspace overlap,
- invalid memory range,
- another existing safety condition.

Verify after rejection:

```text
effective mode == NORMAL 1:1
```

and verify that the active leader/timing configuration is exactly the normal 1:1 profile.

Specifically prove that no UL shortened leader remains active.

Test:

- `UL`,
- `UL_MZ800`,
- `UL_MZ700`.

---

## F. Rejected non-UL generated loader fallback

Verify at least one failed:

- IC,
- TC,
- MZ700 FAST3,

also falls back to true `NORMAL 1:1`.

---

## G. Existing formats and QuickDisk source paths

Regression tests for:

- MZF + MFI,
- MZT + MTI,
- logical `.MZQ` QuickDisk record,
- `.QDF` QuickDisk record path,
- physical `.QD` / physical-MFM-backed QuickDisk record path,
- manual MZF loader selection,
- NORMAL profiles.

QuickDisk fallback must be tested separately for logical and physical source
paths. At minimum:

1. Select a logical MZQ record, request a generated/turbo/UL loader that is
   rejected by an existing safety check, and verify true `NORMAL 1:1`.
2. Select a record through the physical QuickDisk/MFM analysis path
   (`mzf_mzq_physical_source == true`, or the current equivalent after audit),
   request a loader that is rejected, and verify the exact same true
   `NORMAL 1:1` fallback.
3. Cover `.QDF` according to the current parser/source-path distinction found
   during the mandatory audit; if it uses a distinct path, test that path
   explicitly.

For the physical-QD test, verify not only the mode enum but also that normal
leader/timing state is restored and no UL-specific leader configuration
survives.

---

# 11. Hardware-test plan

Codex cannot certify real-hardware behavior unless real hardware is actually available.

Do not claim hardware verification.

Provide a concise manual test plan for the real device.

Suggested cases:

### M12

1. `M12 + manual IC`
2. `M12 + manual TC`
3. `M12 + manual UL`
4. `M12 + manual MZ700 FAST3` where applicable
5. `M12 + AUTO + valid M2I`
6. `M12 + AUTO + missing M2I`

### coexistence

Place together:

```text
FLAPPY.MZF
FLAPPY.MFI
FLAPPY.M12
FLAPPY.M2I
```

Confirm each tape file loads only its own sidecar.

### fallback

Create/use a record known to reject UL loader preparation and verify on
serial/log/UI that playback falls back to standard 1:1 and still loads normally.

Repeat this with a record selected from a physical QuickDisk image / physical
MFM-backed QD path, not only from logical MZQ/QDF data. Verify that the physical
QuickDisk source receives exactly the same safe fallback.

---

# 12. Non-goals

Do **not** add any of the following in this task:

- new UI screens,
- loader compatibility preview screens,
- archive/recovery workflows,
- automatic RAW->MZF/MZT extraction,
- new RECORD functionality,
- timing scope/diagnostic UI,
- multipart M12 invention,
- automatic machine detection from `.M12`,
- new loader binaries unless the audit proves an existing binary is technically incompatible,
- changes to current loader algorithms unrelated to M12 acceptance or fallback correctness,
- cross-format MFI/M2I fallback,
- a new `.M12I` extension.

The agreed M12 sidecar extension is:

```text
.M2I
```

---

# 13. Acceptance criteria

The task is complete only when all of the following are true:

- `.M12` is accepted by the existing compatible fast-loader layer.
- Manual M12 fast-loader selection works through the same safety checks as MZF.
- `.M12 + AUTO` looks only for `.M2I`.
- `.M2I` uses the existing MFI single-record syntax/semantics.
- `.MZF` still uses only `.MFI`.
- `.MZT` still uses only `.MTI`.
- `FLAPPY.MZF/.MFI` and `FLAPPY.M12/.M2I` can coexist without ambiguity.
- Missing `.M2I` results in safe `NORMAL 1:1`.
- Invalid/unsafe loader preparation results in true `NORMAL 1:1`.
- UL/UL_MZ800/UL_MZ700 cannot leave UL-specific shortened leader timing active after loader rejection.
- MZF/MZT behavior is not regressed.
- Logical MZQ/QDF QuickDisk behavior is not regressed.
- Physical QuickDisk / physical-MFM-backed QD behavior is not regressed.
- Loader rejection from a physical QuickDisk record produces the same true
  `NORMAL 1:1` fallback as MZF/M12/MZT/logical MZQ.
- Documentation is updated.
- Automated tests are added/updated where feasible.
- The final report explicitly separates automated verification from real-hardware verification.

---

# 14. Final Codex report

At the end, provide:

1. audited current-state summary,
2. list of modified files,
3. concise explanation of the M12 loader integration,
4. concise explanation of M2I resolution,
5. concise explanation of the universal fallback fix,
6. test/build results,
7. any remaining risks,
8. explicit real-hardware test checklist.

Do not expand the scope beyond this specification unless a necessary correctness issue is discovered. If such an issue is found, document it first and make only the minimal required fix.
