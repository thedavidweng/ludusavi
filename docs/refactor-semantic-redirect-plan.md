# Refactor: Semantic Paths → Dynamic Wine Redirect

## Context

PR #614 introduced `PathFormat::Semantic` to solve cross-platform Windows/Wine save portability.
The maintainer (mtkennerly) prefers a less invasive approach: keep absolute paths in the backup
format, store source environment metadata alongside, and generate redirects dynamically at restore
time. This preserves backward compatibility and limits the blast radius.

This plan restructures the branch to match that direction.

## Design Principles

1. **Focus on redirect logic first.** Add only the `scan.redirectWine` config switch requested by
   mtkennerly. GUI/CLI controls come later; the first stage is about making `game_file_target`
   generate Wine redirects and having the backup scan populate minimal `semantics` metadata.
2. **Best effort.** No new error types. If a redirect can't be determined, fall through to existing
   behavior.
3. **Minimal schema.** MVP stores only `kind: wine` per prefix. Windows special folders use
   heuristics (path string matching) for now. Additional `kind` variants and per-directory metadata
   (`user`, `drives`) are incremental additions after the core logic works.
4. **No UI/CLI changes in this stage.** mtkennerly explicitly said: "I'd like to keep this first
   stage focused on the redirect logic rather than the config/UI side." That means the required
   `scan.redirectWine` config field is included, while the GUI toggle and CLI flag are deferred.

---

## Phase 1: Strip

Remove the PathFormat/PathContext/MappingPathKey machinery and all GUI/CLI coupling.

### 1.1 Remove types from `src/scan/layout.rs`

- [ ] Delete `PathFormat` enum (line 243)
- [ ] Delete `PathContext` struct (line 265) and its `validate()` impl
- [ ] Delete `MappingPathKey` enum (line 288) and all its methods
- [ ] Delete `CONTEXT_PREFIX` constant (line 284)
- [ ] Delete `is_default_path_contexts` helper (line 362)
- [ ] Delete `semantic_restore_fallback_target` helper (line 368)
- [ ] Remove `path_format` field from `FullBackup` (line 393)
- [ ] Remove `path_contexts` field from `FullBackup` (line 403)
- [ ] Remove `is_default_path_format` helper (line 406)
- [ ] Revert `plan_backup_kind` logic that references `path_format` / `path_contexts` / `has_semantic_keys` (line 1330-1345)
- [ ] Revert `plan_full_backup` logic that populates `path_format` / `path_contexts`

### 1.2 Strip `src/scan.rs` semantic scan logic

- [ ] Remove `semantic_paths_enabled` parameter from `scan_game_for_backup` (line 534)
- [ ] Remove Wine prefix collection block (lines 545-571)
- [ ] Remove `known_folders` caching block (lines 573-579)
- [ ] Remove all `semantic_key` assignment logic (lines 775, 884 area)
- [ ] Remove `path_contexts` construction (line 1011)
- [ ] Remove `mapping_context_id` assignment (line 1010)
- [ ] Remove `wine_prefix_find_match` function (line 1175)
- [ ] Revert `ScannedFile` field additions in `src/scan/saves.rs`: `origin`, `semantic_key`, `mapping_context_id`, `restore_error`
- [ ] Revert `ScanInfo` field additions in `src/scan/preview.rs`: `will_start_new_semantic_full_backup`, `cached_semantic_conflicts`, `path_contexts`, and related methods

### 1.3 Remove semantic submodules

- [ ] Delete `src/semantic/conflict.rs` (conflict detection — not needed without semantic keys)
- [ ] Delete `src/semantic/preview.rs` (backup preview analysis — references PathFormat)
- [ ] Delete `src/semantic/signals.rs` (backup signal comparison — references SemanticPath in backup)
- [ ] Delete `src/semantic/restore_prompt.rs` (GUI/CLI decision seam — replaced by simpler logic)
- [ ] Delete `src/semantic/materialize.rs` (semantic→physical resolution — replaced by redirect generation)
- [ ] Delete `benches/semantic_scan.rs`
- [ ] Delete `tests/semantic_properties.rs`

### 1.4 Remove config and error variants

- [ ] Remove `BackupConfig.semantic_paths` field
- [ ] Remove `RestoreConfig.preferred_wine_prefixes` field
- [ ] Remove `RestoreConfig.wine_prefix` field
- [ ] Remove `RestoreConfig.drive_mappings` field
- [ ] Remove `GameWinePrefixPreference` struct
- [ ] Remove `Config` event variants: `CustomGameWinePrefix`, `PreferredWinePrefixPath`, `PreferredWinePrefixRemove`
- [ ] Remove `Error::WinePrefixConflict`, `Error::WinePrefixAmbiguity`, `Error::WineUserAmbiguity` from `src/prelude.rs`

### 1.5 Strip GUI/CLI coupling

- [ ] Remove all semantic imports from `src/gui/app.rs` (~70 references)
- [ ] Remove `pending_prefix_selections` field from GUI state
- [ ] Remove `Modal::WinePrefixSelection` and related message handlers
- [ ] Remove all semantic imports from `src/cli.rs` (lines 32-43)
- [ ] Remove CLI `--wine-prefix` flag and related logic (lines 641-784, 923-930)
- [ ] Remove semantic report fields from `src/report.rs`
- [ ] Remove semantic translation keys from all `lang/*.ftl` files
- [ ] Remove semantic documentation from `docs/help/`
- [ ] Remove `docs/cross-platform-sync-plan.md`

### 1.6 Verify

- [ ] `cargo build` succeeds
- [ ] `cargo test` passes (existing tests should pass since we're reverting to pre-PR behavior)
- [ ] `cargo clippy` clean

---

## Phase 2: Add Config and Schema

Minimal config/schema addition. Add `scan.redirectWine`, but no GUI toggle and no CLI flag.

### 2.1 New scan config option

In `src/resource/config.rs`, add to `Scan`:

```rust
/// Generate best-effort Windows/Wine redirects during scans.
#[serde(default)]
pub redirect_wine: bool,
```

- [ ] Add `redirect_wine: bool` to `Scan`
- [ ] Ensure `Default` keeps it disabled
- [ ] Thread the value into backup and restore scan calls
- [ ] Do not add GUI or CLI controls yet

### 2.2 `semantics` metadata on `FullBackup`

In `src/scan/layout.rs`, add to `FullBackup`:

```rust
/// Source environment metadata for generating cross-platform redirects.
/// Only populated when the backup contains files from Wine prefixes.
#[serde(skip_serializing_if = "is_default_semantics")]
pub semantics: BackupSemantics,
```

New types:

```rust
#[derive(Clone, Debug, Default, Eq, PartialEq, serde::Serialize, serde::Deserialize)]
#[serde(default, rename_all = "camelCase")]
pub struct BackupSemantics {
    pub directories: BTreeMap<String, SemanticDirKind>,
}

#[derive(Clone, Debug, Eq, PartialEq, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub enum SemanticDirKind {
    /// A Wine/Proton prefix. Redirect logic uses heuristics to find
    /// the wine user and drive mappings at restore time.
    Wine,
}
```

- [ ] Add types to `src/scan/layout.rs`
- [ ] Add `is_default_semantics` helper
- [ ] Ensure `serde(default)` on `BackupSemantics` so old backups without the field deserialize fine

### 2.3 Verify

- [ ] `cargo build` succeeds
- [ ] `cargo test` passes
- [ ] Existing backup YAML files still deserialize (serde default)

---

## Phase 3: Implement Redirect Logic

All changes scoped to `game_file_target` and its two call sites (`scan_game_for_backup`,
`scan_for_restoration`).

### 3.1 Fix `game_file_target` early return

Current function returns `None` immediately when `redirects.is_empty()` (line 82). This prevents
Wine redirect logic from running when the user has no configured redirects. Change the function so
user redirects and generated Wine redirects are evaluated independently:

```rust
pub fn game_file_target(
    original: &StrictPath,
    redirects: &[RedirectConfig],
    reverse_redirects_on_restore: bool,
    scan_kind: ScanKind,
    redirect_wine: bool,
    semantics: Option<&BackupSemantics>,
    wine_redirect: Option<&WineRedirectContext>,
) -> Option<StrictPath> {
    let mut redirected = original.clone();

    // Apply user-configured redirects (existing logic, but no early return on empty).
    if !redirects.is_empty() {
        let redirects_iter: &mut dyn Iterator<Item = &RedirectConfig> =
            if scan_kind.is_restore() && reverse_redirects_on_restore {
                &mut redirects.iter().rev()
            } else {
                &mut redirects.iter()
            };
        for redirect in redirects_iter {
            // ... existing redirect logic ...
            redirected = redirected.replace(source, target);
        }
    }

    // If user-configured redirects already changed the path, done.
    if original != &redirected {
        return Some(redirected);
    }

    // Wine redirect: best effort.
    // On backup: no redirect needed (store absolute path). Semantics populated elsewhere.
    // On restore: generate redirect from stored path to current system path.
    if redirect_wine && scan_kind.is_restore() {
        return generate_restore_redirect(&redirected, semantics?, wine_redirect?);
    }

    None
}
```

- [ ] Remove the `redirects.is_empty()` early return
- [ ] Add `redirect_wine: bool` parameter
- [ ] Add `semantics: Option<&BackupSemantics>` parameter
- [ ] Add `wine_redirect: Option<&WineRedirectContext>` parameter
- [ ] Update all call sites (scan.rs line 772, layout.rs lines 943, 1084)
- [ ] Pass `None` for semantics/context at call sites that don't have backup context yet

`WineRedirectContext` should be an internal scan-layer struct, not config:

```rust
pub struct WineRedirectContext {
    /// First valid `wine_prefix` from the matching custom game, per mtkennerly's MVP heuristic.
    pub preferred_prefix: Option<ValidatedPrefix>,
    /// Current Windows known folders, only populated on Windows.
    pub known_folders: Option<KnownFolders>,
}
```

### 3.2 Implement `generate_restore_redirect`

New function in `src/scan.rs` (or a new `src/scan/wine_redirect.rs`). This is where the dynamic
redirect is generated; restore scan code should only provide context and consume the returned
`ScannedFile.redirected` value.

```rust
fn generate_restore_redirect(
    stored_path: &StrictPath,
    semantics: &BackupSemantics,
    context: &WineRedirectContext,
) -> Option<StrictPath> {
    // Linux/Wine backup -> Windows restore:
    // 1. Match stored_path under a semantics.directories entry with kind=Wine.
    // 2. Convert the Wine physical path to an internal SemanticPath.
    // 3. Convert that SemanticPath to the current Windows known-folder path.

    // Windows backup -> Linux/Wine restore:
    // 1. Use current Windows-folder heuristics to derive the internal SemanticPath.
    // 2. Convert that SemanticPath into context.preferred_prefix.

    // If any step is ambiguous or unavailable, return None.
}
```

- [ ] Implement Linux/Wine backup → Windows restore
- [ ] Implement Windows backup → Linux/Wine restore using the first valid custom-game `wine_prefix`
- [ ] Move or reimplement only the minimal SemanticPath → physical-path helpers needed for redirects;
  do not keep the old semantic materialization layer or expose it to GUI/CLI
- [ ] Return `None` for missing prefix, missing known folders, multiple Wine users, UNC/complex drive paths, or unrecognized paths
- [ ] Keep all Wine/Windows translation decisions inside this function or its scan-layer helpers

### 3.3 Populate `semantics` during backup scan

In `scan_game_for_backup`, when `config.scan.redirectWine` is enabled:

- [ ] Re-add Wine prefix detection logic (simplified from Phase 1 strip — just `validate_prefix`
  calls, no semantic key assignment)
- [ ] For each scanned file, check if it falls under a detected Wine prefix
- [ ] Record the prefix path in `BackupSemantics.directories` with `kind: Wine`
- [ ] Set `semantics` on the `FullBackup` when creating a new backup in `plan_full_backup`

The prefix detection reuses `src/semantic/prefix.rs` (which we kept). The check is a simple
`starts_with` — no semantic key derivation, no `convert.rs` calls during backup scan.

### 3.4 Restore scan: use `semantics` for prefix-aware redirect

In `scan_for_restoration` (layout.rs):

- [ ] Read `semantics` from the backup being restored
- [ ] Build `WineRedirectContext` from the current game and host
- [ ] Use the first valid `wine_prefix` entry from the matching custom game as the current prefix
  on Linux/Wine — mtkennerly's MVP heuristic
- [ ] Pass `redirect_wine`, `semantics`, and `WineRedirectContext` into `game_file_target`
- [ ] Let `game_file_target` return the translated redirect target
- [ ] If no current prefix found, fall through to existing restore behavior (best effort)

### 3.5 Incremental backup check

In the backup scan path, when comparing current files against the last backup:

- [ ] Read `semantics` from the last `FullBackup`
- [ ] For each stored file, call `game_file_target` with `ScanKind::Restore`, last-backup
  `semantics`, and current `WineRedirectContext` to resolve the current live path
- [ ] Compare the live file hash against the stored hash
- [ ] If the prefix can't be resolved, treat the file as changed (safe default)

### 3.6 Verify

- [ ] `cargo build` succeeds
- [ ] `cargo test` passes
- [ ] New tests for:
  - `scan.redirectWine=false` preserves existing behavior
  - `scan.redirectWine=true` enables generated Wine redirects without user-configured redirects
  - Backup with Wine prefix stores correct `semantics.directories`
  - Restore from Windows backup to Linux/Wine generates correct redirect
  - Restore from Linux/Wine backup to Windows generates correct redirect
  - Incremental check with redirect correctly detects unchanged files
  - Old backup without `semantics` restores without error (serde default)
  - No prefix found → falls through to existing behavior (no crash, no new error)

---

## Phase 4: Documentation

### 4.1 Update docs (commented out until release)

- [ ] Update `docs/help/redirects.md` — add Wine redirect section, **commented out** until
  release (non-technical users shouldn't see unreleased feature docs)
- [ ] Update `docs/help/transfer-between-operating-systems.md` — add cross-restore section,
  **commented out** until release

No changes to `docs/help/configuration-file.md` or `docs/schema/` in this stage; the
`scan.redirectWine` config field exists, but human-facing release docs stay deferred/commented.

### 4.2 Verify

- [ ] `cargo build` succeeds
- [ ] `cargo test` passes
- [ ] Manual test: backup on Windows, restore on Linux with Wine prefix
- [ ] Manual test: backup on Linux/Wine, restore on Windows

---

## What We Keep from Current PR

| File | What | Why |
|------|------|-----|
| `src/semantic/convert.rs` | `KnownFolders`, `windows_physical_to_semantic`, `wine_physical_to_semantic`, helpers | Internal translation logic for redirect generation |
| `src/semantic/prefix.rs` | `ValidatedPrefix`, `validate_prefix`, `detect_wine_user`, `scan_dosdevices` | Prefix detection for backup scan |
| `src/semantic.rs` | `SemanticBase`, `SemanticPath` | Internal types used by convert.rs |
| `tests/wine-prefix/drive_c/windows/system.reg` | Test fixture | Needed for prefix detection tests |

## What We Remove from Current PR

~55 of the 70 changed files:

| Category | Files | Reason |
|----------|-------|--------|
| Storage format | `PathFormat`, `PathContext`, `MappingPathKey` | Backup keeps absolute paths |
| Conflict detection | `conflict.rs` | Not needed without semantic keys in storage |
| Preview analysis | `preview.rs`, `signals.rs` | References PathFormat |
| Restore prompt | `restore_prompt.rs` | Replaced by simpler best-effort logic |
| Materialization | `materialize.rs` | Replaced by redirect in game_file_target |
| GUI modal | `Modal::WinePrefixSelection`, `pending_prefix_selections` | Deferred to later stage |
| CLI flags | `--wine-prefix`, `--semantic-paths` | Deferred to later stage |
| Error types | `WinePrefixConflict`, `WinePrefixAmbiguity`, `WineUserAmbiguity` | Best effort = no new errors |
| Config fields | `semantic_paths`, `preferred_wine_prefixes`, `wine_prefix`, `drive_mappings` | Replaced by minimal `scan.redirectWine`; richer config deferred |
| Benchmarks | `benches/semantic_scan.rs` | No semantic scan to benchmark |
| Property tests | `tests/semantic_properties.rs` | No semantic storage to test |
| All lang/*.ftl additions | 27 files | Deferred to later stage |

## Deferred (not in this PR)

These are mentioned in mtkennerly's comments but explicitly deferred:

| Feature | When | Notes |
|---------|------|-------|
| GUI toggle for `scan.redirectWine` | After config option | Simple checkbox |
| CLI `--redirect-wine` flag | After config option | |
| `preferred_wine_prefixes` config | After multi-prefix tested | Use first `wine_prefix` from custom game for now |
| `SemanticDirKind` expansion (Win*) | After Wine↔Windows works | Heuristics sufficient for MVP |
| Per-directory `user`/`drives` metadata | After multi-user tested | Current: detect at restore time |
| GUI prefix selection modal | After config option | Only needed when ambiguity can't be auto-resolved |

## Dependency Order

```
Phase 1 (Strip)
  ├── 1.1 layout.rs types
  ├── 1.2 scan.rs logic
  ├── 1.3 semantic submodules
  ├── 1.4 config + errors
  ├── 1.5 GUI/CLI
  └── 1.6 verify

Phase 2 (Config + Schema)
  ├── 2.1 scan.redirectWine config
  ├── 2.2 BackupSemantics type
  └── 2.3 verify

Phase 3 (Redirect Logic)
  ├── 3.1 fix game_file_target early return
  ├── 3.2 implement generate_restore_redirect
  ├── 3.3 populate semantics on backup scan
  ├── 3.4 restore scan redirect
  ├── 3.5 incremental check
  └── 3.6 verify

Phase 4 (Docs)
  ├── 4.1 commented-out docs
  └── 4.2 verify
```

Phases 1→2→3→4 are sequential. Within each phase, tasks can be parallelized.
