# SpaceMonger 1.4 → Rust: Modernization Plan

Status: proposal · Scope: full port, **same UI**, modern Rust security and performance practice.
Crate versions below were checked against crates.io in October 2026; the toolchain is Rust 1.98 stable.

---

## 1. What is in the repository today

| Area | Files | Lines | Notes |
|---|---|---|---|
| App shell / commands / settings | `SpaceMonger.cpp/.h` | ~620 | `CWinApp`, command handlers, registry settings |
| Scanner + tree model | `FolderTree.cpp/.h` | ~770 | Recursive `FindFirstFileA` walk, parallel `malloc` arrays, radix sort |
| Treemap view | `FolderView.cpp/.h` | ~1230 | Layout algorithm, GDI drawing, hit-testing, tips, context menu, zoom animation |
| Custom tooltip control | `TipWnd.cpp/.h` | ~990 | Raw Win32 window class driven by `WM_COMMAND` sub-messages |
| Frame + toolbar | `Mainfrm.cpp/.h` | ~340 | `CFrameWnd`, `CToolBar`, hand-written layout |
| Dialogs | `DriveDialog`, `SetupDlg`, `AboutDialog` | ~560 | Drive picker (ListView, large icons), settings, about |
| Doc/View shim | `xapp.h`, `Freedoc.cpp`, `Freeview.cpp` | ~440 | "Liberated" doc/view, GDI box helpers |
| i18n | `lang.h`, `lang_en.cpp`, `lang_fr.cpp` | ~220 | Struct of `const char*` per language (US, UK, FR), Windows-1252 encoded |
| Resources | `SpaceMonger.rc`, `resource.h`, `res/*` | ~430 | 4 dialogs, 2 toolbar bitmaps (EN/FR), 4 scan-animation frames, icon, version info |
| Build | `SpaceMonger.dsp/.dsw` (+ `.ncb`, `.opt` binaries) | – | Visual C++ 6, static MFC (`/MT`), ANSI/MBCS, x86 only |

### UI surface the port must keep (parity checklist)

- [ ] Main frame: title bar text (`path – % – size – … – SpaceMonger`), system-menu "About" entry, window placement persistence
- [ ] Toolbar: Open, Rescan | Zoom Full, In, Out | Show Free Space (toggle) | Run/Open, Delete | Settings | About, with per-language bitmaps and enable/disable rules (`UpdateButtonsForView`, `GeneralIgnoreUpdate`)
- [ ] Treemap: greedy two-way split layout (`SizeFolders`), density presets, horizontal/vertical bias, 12 color schemes (rainbow by depth, Windows colors, fixed colors), 3-D bevel boxes, folder title bars, file size and date in large boxes, free-space block with stats
- [ ] Interaction: click to select, double-click to zoom into a folder or run a file, right-click context menu, rollover highlight boxes, XOR "animated zoom" rectangle
- [ ] Name-tips and info-tips (custom `TipWnd`: delay, auto-position, icon, path, name, size, date, attributes)
- [ ] Scan dialog: path, files/folders counters, progress bar, 4-frame animation, Cancel
- [ ] Drive dialog: large-icon ListView of drives A–Z with shell display names
- [ ] Settings dialog: every control in `IDD_SETTINGS`, including the live language switch that reverts on Cancel
- [ ] About dialog
- [ ] Languages: English (US), English (UK), French
- [ ] Settings stored under `HKCU\Software\65\SpaceMonger\1.3`

---

## 2. Defects and risks found in the scan

These are bugs in the current code. The port should **not** reproduce them, and each one should get a regression test.

### Memory safety / security

| # | Location | Problem | Impact |
|---|---|---|---|
| S1 | `FolderTree.cpp` `LoadFolder` | Builds child paths in a fixed `char path[MAX_PATH+270]` and appends names with no bounds check | **Stack buffer overflow** on deep trees (> ~530 chars) |
| S2 | `SpaceMonger.cpp` `OnFileDelete` | `strcpy` of the full path into `char source[MAX_PATH+8]` | **Stack buffer overflow** when deleting a deeply nested item |
| S3 | `OnFileDelete` | `SHFileOperation` with `FOF_NOCONFIRMATION`. `FOF_ALLOWUNDO` silently becomes a permanent delete on volumes without a Recycle Bin (network, many removable drives) | **Permanent data loss with no prompt** |
| S4 | `LoadFolder` | 17 KB reparse buffer on the stack in every recursion frame | Stack exhaustion on deep trees |
| S5 | `LoadFolder` | `CreateFile` result is not checked before `DeviceIoControl` | Undefined behaviour on access-denied reparse points |
| S6 | Whole app | ANSI (`_MBCS`) APIs everywhere | Names outside the active code page become `?`, so Run, Delete and Properties hit the wrong path or fail |
| S7 | `cpp_strdup` | Lowercases names that are entirely upper-case, then reuses the display name to build paths for Delete/Run | Wrong target in case-sensitive directories (supported since Windows 10) |
| S8 | `MinimalDrawDisplayFolder` | `sprintf("%u", ui64)` | Format/vararg mismatch (undefined behaviour) |
| S9 | `OnCreate`, `BuildFolderLayout`, `SizeFolders` | `delete` on `malloc`/`new[]` memory | Undefined behaviour |
| S10 | `OnDoubleClick` / Run | `ShellExecute` on any file on double-click | Expected behaviour, but it should use the wide API and an explicit working directory |

### Resource leaks

| # | Location | Problem |
|---|---|---|
| L1 | `SetupInfoTip` | `SHGetFileInfo(SHGFI_ICON)` runs on every hover change and **no `DestroyIcon` exists anywhere in the codebase**. GDI handles leak until the 10,000-handle quota is reached and drawing fails |
| L2 | `CDriveInfo::LoadDriveInfo` | Same icon leak for each drive, every time the dialog opens |
| L3 | `MinimalDrawDisplayFolder` | Creates and destroys three brushes per box per paint. Not a leak, but a large avoidable cost |

### Correctness

| # | Location | Problem |
|---|---|---|
| C1 | `LoadFolderInitial` | The power-of-two check computes `2^(bit_length)`, so `aligned` is **always false**. The fallback path then **rounds sizes down** to a cluster (the comment says "round up"), so files smaller than a cluster count as 0 bytes and never appear in the map. The `aligned` branch would also be wrong (`& mask` instead of `& ~mask`) |
| C2 | `EightBitCountingSort` | Sets `parentindex = i` (the source index) instead of `dest`. It only works because the last radix pass is a stable no-op for sizes < 2^56. `countarray[256]` is read without being initialised |
| C3 | Scanner | Hard links are double-counted. Compressed and sparse files show logical size, not allocated size (the README admits this). Only mount points and symlinks are skipped; cloud placeholder files (OneDrive) and other reparse types are not handled |
| C4 | `LoadFolder` | `GetTickCount()` wrap-around (49.7 days) breaks the `tick > last + 200` throttle |
| C5 | `UpdateTitleBar` | Dereferences `mainfrm` before the null check |
| C6 | Context menu | The "Delete" disabled flag is inverted (`!disable_delete && cur == NULL`). MFC's `CCmdUI` hides this, but `OnFileDelete` never checks `disable_delete` itself |
| C7 | `memset(&fileinfo, sizeof(SHFILEINFO), 0)` | Arguments swapped, so the struct is never zeroed |
| C8 | Scan runs on the UI thread and pumps messages from inside the recursion (`PeekMessage`) | The UI can re-enter while the tree is half built |
| C9 | Free space is a fake file named `"<<<<<<<<<<<<<<<<<<<<"`, and names starting with `* < > ? \|` are treated as special | Sentinel strings in place of a type |

### Obsolescence

- 256-color palette management (`CPalette`, `RealizePalette`), `LoadLibrary("KERNEL32")` for `GetDiskFreeSpaceExA`, `_WIN32_WINNT < 0x500` shims
- No DPI awareness. The 9 px "Small Fonts" raster font is unreadable at 150–200 % scaling
- `MAX_PATH` limits everywhere, no long-path support
- x86-only VC6 project; IDE binaries (`SpaceMonger.ncb`, `.opt`) are committed
- Source files are Windows-1252 encoded

---

## 3. Target architecture

### UI technology decision

To keep the UI the same, the port needs real Win32 windows, dialogs drawn from the original `.rc` templates, and GDI rendering. Options:

| Option | Same UI? | Maintenance | Verdict |
|---|---|---|---|
| **`windows` 0.62 (Microsoft windows-rs) + original `.rc` via `embed-resource` 3.0** | Yes, pixel-identical dialogs because the same templates are used | Microsoft-maintained, tracks Win32 metadata | **Recommended** |
| `winsafe` 0.0.29 | Yes (safe Win32 wrappers, supports `.rc` dialogs) | Active but pre-1.0, one maintainer | Acceptable alternative if we'd rather not write wrappers |
| `native-windows-gui` 1.0.13 | Mostly | **Unmaintained since 2022** | Avoid |
| egui / Slint / iced / Tauri | No, the look changes | Good | Out of scope; possible later because the core crate stays platform-independent |

### Workspace layout

```
spacemonger/
├─ Cargo.toml                # [workspace], shared lints, profiles
├─ rust-toolchain.toml       # channel = "1.98", components = [clippy, rustfmt, miri(nightly in CI)]
├─ deny.toml                 # cargo-deny policy
├─ crates/
│  ├─ sm-core/               # #![forbid(unsafe_code)], no Windows deps, runs on Linux CI and under Miri
│  │  ├─ tree.rs             # arena tree model
│  │  ├─ layout.rs           # port of SizeFolders/BuildFolderLayout (pure function)
│  │  ├─ format.rs           # sizes, percentages, dates, digit grouping
│  │  ├─ i18n.rs             # Lang tables (UTF-8), selection by code
│  │  ├─ settings.rs         # Settings struct + validation/clamping (from CCurrentSettings::Load)
│  │  └─ palette.rs          # BoxColors / FixedColors, color-scheme selection
│  ├─ sm-scan/               # Scanner: trait + Windows backend + portable std::fs backend
│  └─ spacemonger/           # bin: Win32 shell (frame, toolbar, view, TipWnd, dialogs)
│     ├─ build.rs            # embed-resource: SpaceMonger.rc + app.manifest
│     ├─ res/                # original .rc, bitmaps, icon (unchanged), + app.manifest
│     └─ src/win/            # the only module allowed to use `unsafe`; thin RAII wrappers
├─ fuzz/                     # cargo-fuzz targets
└─ benches/                  # divan or criterion benches
```

### Core data model (replaces `CFolder`'s five parallel `malloc` arrays)

```rust
pub struct Tree {
    folders: Vec<Folder>,       // arena; FolderId = u32 index
    names: String,              // one contiguous name buffer → far fewer allocations for millions of files
    root: FolderId,
    pub volume: VolumeInfo,     // total / free / cluster size
}
pub struct Folder {
    parent: Option<(FolderId, u32)>,   // fixed up *after* sorting (fixes C2)
    entries: Vec<Entry>,               // sorted by size, descending
    size_files: u64, size_subdirs: u64,
}
pub struct Entry {
    name: NameRef,              // (offset, len) into Tree::names, original case kept (fixes S7)
    allocated: u64,             // on-disk size used for layout (fixes C1/C3)
    logical: u64,               // shown in info-tips
    modified: FileTime,
    attrs: Attributes,          // bitflags 2.x
    kind: EntryKind,            // File | Folder(FolderId) | FreeSpace (fixes C9)
}
```

- Full paths are rebuilt by walking `parent` and joining as UTF-16 `\\?\` paths, so there is no fixed buffer (fixes S1, S2, S6).
- Sorting uses `sort_unstable_by_key(|e| Reverse(e.allocated))`, then a pass that rewrites child back-references. The 8-pass radix sort only stays if a benchmark shows it wins.
- The display list (`CDisplayFolder` linked list of `new`'d nodes, `si16` coordinates) becomes a `Vec<DisplayItem>` with `i32` rectangles that is reused between layouts.

### Scanner (`sm-scan`)

- Runs on a **worker thread** (fixes C8). Progress is reported through `AtomicU64` counters plus the current path in a `Mutex<String>`. The UI polls them on a 200 ms `WM_TIMER`, which keeps the scan dialog's current cadence. Cancellation uses an `AtomicBool`.
- Windows backend: open each directory with `CreateFileW(FILE_FLAG_BACKUP_SEMANTICS)` and batch-enumerate with `GetFileInformationByHandleEx(FileIdExtdDirectoryInfo)` into a 64 KB buffer. One call returns:
  - `AllocationSize` → correct size for compressed, sparse and small files (fixes C1, C3)
  - `ReparsePointTag` → skips junctions, mount points, symlinks and cloud placeholders with no extra `CreateFile`/`DeviceIoControl` per entry (fixes S4, S5)
  - `FileId` → optional hard-link de-duplication
- Parallelism: work-stealing over directories (`rayon` 1.12 scope or a `crossbeam` deque). Each worker builds subtrees that are merged into the arena at the end. Benchmark against a single-threaded run, because HDDs can get slower with parallel enumeration. Make the thread count adaptive.
- Use the volume's real cluster size from `GetDiskFreeSpaceExW` / `GetVolumeInformationByHandleW`. No `LoadLibrary`.
- Portable backend (`std::fs` + `walkdir` 2.5) is for Linux CI, tests and Miri.
- **Stretch goal:** an NTFS MFT fast path (raw `$MFT` read, administrator only), which is the approach WizTree uses. It parses untrusted on-disk structures, so it needs a dedicated fuzz target.

### Win32 shell (`spacemonger` bin)

- **Resources:** keep `SpaceMonger.rc`, `resource.h` and `res/*` exactly as they are and compile them with `embed-resource` 3.0. Dialogs, toolbar bitmaps, animation frames, icon and version info are then identical to today's. Regenerate the `.rc` as UTF-8 with `#pragma code_page(65001)`. Generate a Rust `resource_ids.rs` from `resource.h` at build time so the IDs can't drift.
- **Frame/toolbar:** a plain `WNDCLASSEXW` frame with a `TOOLBARCLASSNAMEW` toolbar loaded from the `TOOLBAR` + bitmap resources, which replaces `CToolBar::LoadToolBar`. Port the enable/disable rules into one `fn command_state(&AppState, CommandId) -> (enabled, checked)`. That one function drives the toolbar and the context menu, so C6 can't happen again.
- **View:** keep GDI so the look stays pixel-identical. Paint into a memory DC (double buffering, no flicker). Cache brushes for the 26 + 30 palette colors (fixes L3). Drop the palette code. Do the XOR zoom animation with `SetROP2(R2_NOT)` as today, but drive it from a timer, not `Sleep`.
- **TipWnd:** port it as a Rust window class. Its state lives in a `Box<TipState>` behind `GWLP_USERDATA` and is freed on `WM_NCDESTROY`. Replace the `WM_COMMAND` sub-message protocol with direct method calls. The current icon is owned through an `OwnedIcon` RAII type that calls `DestroyIcon` (fixes L1, L2).
- **Shell actions:**
  - Run/Open uses `ShellExecuteExW`.
  - Properties uses `SEE_MASK_INVOKEIDLIST`.
  - Delete uses `IFileOperation` with `FOFX_RECYCLEONDELETE`. Show a confirmation if the item can't go to the Recycle Bin (fixes S3).
  - Enforce `disable_delete` inside the command handler.
- **Settings:** read and write the same registry key with `windows-registry` 0.100. Port every clamp from `CCurrentSettings::Load` into `sm-core::settings` and fuzz it. Keep the key name so existing users' settings carry over.
- **Unsafe policy:** only `src/win/` may contain `unsafe`. Every block gets a `// SAFETY:` comment (enforced by lint). Handles (`HDC`, `HBRUSH`, `HFONT`, `HICON`, `HANDLE`) are wrapped in RAII newtypes. Window procedures never unwind; see the panic strategy below.

### Modernizations that keep the UI recognisable

These change behaviour slightly. Each needs a decision (see §8):

- **Per-Monitor-V2 DPI awareness:** scale the hard-coded pixel metrics (12 px title band, 3 px bevels, density minimums). Swap the "Small Fonts" raster font for a scaled Segoe UI or Tahoma 7–8 pt at 96 DPI so it looks the same at 100 % and stays readable at 200 %.
- **Common Controls v6 (visual styles):** without it, buttons look exactly like Windows 2000. With it, they look native to Windows 11.
- **Long-path aware manifest:** `longPathAware` plus `\\?\` prefixes.
- **Folder targets:** let the user scan any folder or UNC path, not just drive letters. This could be an extra button in the drive dialog; the current layout stays the same.

---

## 4. Toolchain and project configuration

- **Rust 1.98 stable, edition 2024.** Pin it in `rust-toolchain.toml` and set `rust-version = "1.98"` in the workspace.
- **Targets:** `x86_64-pc-windows-msvc` (primary) and `aarch64-pc-windows-msvc` (Windows on ARM). The original was x86-only.
- **Static CRT** (`-C target-feature=+crt-static`) so the app stays a single `.exe` with no installer, as the README promises.

### Dependency shortlist (latest as of 2026-10)

| Purpose | Crate | Version |
|---|---|---|
| Win32 API | `windows` | 0.62.2 |
| Resources | `embed-resource` | 3.0.12 |
| Registry | `windows-registry` | 0.100.0 |
| Parallelism | `rayon` | 1.12.0 |
| Flags | `bitflags` | 2.13.2 |
| Errors | `thiserror` (libs) / `anyhow` (bin) | 2.0.21 / 1.0.104 |
| Logging | `tracing`, `tracing-subscriber` | 0.1.44 / 0.3.23 |
| Dates (info-tips) | `jiff` | 0.2.38 |
| Allocator (if benchmarks justify it) | `mimalloc` | 0.1.52 |
| Portable walk (tests) | `walkdir` | 2.5.0 |
| Dev: snapshots / properties / benches | `insta` / `proptest` / `divan` or `criterion` | 1.49 / 1.12 / 0.1.21 or 0.8.2 |

Keep the dependency count deliberately small. Every crate added is supply-chain surface (see §5).

### Workspace `Cargo.toml` (excerpt)

```toml
[workspace]
resolver = "3"
members = ["crates/*"]

[workspace.package]
edition = "2024"
rust-version = "1.98"
license = "MIT"

[workspace.lints.rust]
unsafe_op_in_unsafe_fn = "deny"
missing_debug_implementations = "warn"
unused_qualifications = "warn"

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
undocumented_unsafe_blocks = "deny"
multiple_unsafe_ops_per_block = "deny"
missing_safety_doc = "deny"
indexing_slicing = "warn"          # nudges toward checked access in parsing code
cast_possible_truncation = "warn"  # the C++ had many silent u64→u32/i16 casts
unwrap_used = "warn"

[profile.release]
lto = "fat"
codegen-units = 1
opt-level = 3
panic = "abort"
overflow-checks = true             # cheap, catches size-math bugs like C1
debug = "line-tables-only"         # PDBs for crash symbolication
strip = "none"                     # keep PDB separate; ship stripped exe

[profile.bench]
inherits = "release"
debug = true
```

`sm-core/src/lib.rs` starts with `#![forbid(unsafe_code)]`.

### Binary hardening (`.cargo/config.toml`)

```toml
[target.'cfg(all(windows, target_env = "msvc"))']
rustflags = [
  "-C", "target-feature=+crt-static",
  "-C", "control-flow-guard",            # CFG
  "-C", "link-arg=/CETCOMPAT",           # CET shadow-stack compatible (x64)
  "-C", "link-arg=/DEPENDENTLOADFLAG:0x800",  # LOAD_LIBRARY_SEARCH_SYSTEM32: blocks DLL planting
]
```

ASLR (`/DYNAMICBASE`, `/HIGHENTROPYVA`) and DEP (`/NXCOMPAT`) are already the defaults for Rust MSVC targets. Verify them in CI with `dumpbin /headers` or a BinSkim scan.

### Application manifest (embedded via the `.rc`)

`asInvoker`, `supportedOS` Windows 10/11, `dpiAwareness = PerMonitorV2`, `longPathAware = true`, `activeCodePage = UTF-8`, and optionally Common Controls v6 (decision in §8).

### Panic strategy

Release builds use `panic = "abort"`. Since Rust 1.81, a panic that escapes an `extern "system"` window procedure aborts the process instead of unwinding into user32. Install a panic hook that writes a `tracing` record and minidump-style context to `%LOCALAPPDATA%\SpaceMonger\crash.log`, then shows a `MessageBoxW`. The scanner thread reports errors through its result type, not panics.

---

## 5. Security tooling

| Tool | Version | Use |
|---|---|---|
| `cargo-deny` | 0.20.2 | CI gate: RustSec advisories, license allow-list (MIT/Apache-2.0/BSD/Unicode/Zlib), ban duplicate major versions, restrict sources to crates.io |
| `cargo-audit` | 0.22.2 | Advisory scan of `Cargo.lock`, run on a daily schedule as well as on PRs |
| `cargo-vet` | 0.10.2 | Supply-chain audits. Import Mozilla/Google/Bytecode Alliance audit sets. Small dependency tree, so this is practical |
| `cargo-auditable` | 0.7.7 | Embeds the dependency list in the release `.exe` so scanners can audit the shipped binary |
| `cargo-geiger` | 0.13.0 | Reports `unsafe` usage across the graph and tracks the trend |
| Miri | nightly | Run the `sm-core` test suite under Miri on Linux |
| `cargo-fuzz` | 0.13.2 | Fuzz targets: layout invariants, settings validation/clamping, size/date formatting, MFT parser (if built) |
| `proptest` | 1.12 | Layout properties: children inside parents, no overlaps between siblings, area roughly proportional to size, deterministic output |
| Clippy + rustfmt | toolchain | `-D warnings` in CI |
| CodeQL (Rust) | GitHub | SAST on PRs |
| Dependabot or Renovate | – | Weekly grouped updates for crates and GitHub Actions |
| OpenSSF Scorecard | – | Track repository hygiene |

**CI hardening:** pin GitHub Actions to commit SHAs, default `permissions: contents: read`, no `pull_request_target`, cache keyed on `Cargo.lock`.

**Release integrity:**
- Authenticode-sign the `.exe` (Azure Trusted Signing or an OV/EV certificate)
- Produce a CycloneDX SBOM (`cargo cyclonedx`)
- Add SLSA build provenance with `actions/attest-build-provenance`
- Publish SHA-256 checksums

---

## 6. Performance tooling and targets

- **Benchmarks** (`divan` or `criterion`):
  - layout of synthetic trees (10 k / 1 M entries)
  - sort and finalize
  - tree build from a synthetic enumeration stream
  - path reconstruction
- **End-to-end:** use `hyperfine` with a hidden `--scan <path> --exit` CLI flag to time cold and warm scans of a fixed corpus. Compare the original 1.4.0 binary with the port on the same machine and record the numbers in the repo.
- **Profiling on Windows:**
  - `samply` (ETW-based, opens in the Firefox Profiler)
  - Windows Performance Recorder/Analyzer for I/O and thread contention during parallel scans
  - Superluminal, optional
- **Memory:** `dhat` (heap profiling in tests) and the peak working set in the `hyperfine` runs. Target: below the original's per-entry memory (5 arrays + a `malloc`'d name ≈ 40 B + name + allocator overhead) by using the contiguous name buffer and a 32-byte `Entry`.
- **Binary size:** `cargo bloat`. Target well under 2 MB, because single-exe distribution is a feature.
- **Rendering:** cached brushes, double buffering, and hit-testing over a flat `Vec` instead of a pointer-chased list. Repaint only dirty items on rollover, as today.
- **Initial targets** (to be confirmed by the baseline in phase 0):
  - full NTFS scan of 1 M files on NVMe at least 3× faster than 1.4.0
  - UI stays responsive during the scan (no message pumping in the scanner)
  - zero GDI handle growth over a 10-minute hover soak test

---

## 7. Testing strategy

1. **Layout golden tests:** extract `SizeFolders`/`BuildFolderLayout` from `FolderView.cpp` into a small test-only C harness compiled with the `cc` crate, then run differential tests: same tree in, same rectangles out. This proves the port draws the same picture before any pixels exist. Snapshot the results with `insta`.
2. **Unit tests:** formatting (`GetSizeString`, `PrintFileSize` with locale digit separators, `PrintDate` month names), settings clamping (the full matrix from `CCurrentSettings::Load`), i18n completeness (every language fills every key, enforced at compile time by a struct).
3. **Scanner integration tests** on a `windows-latest` CI runner. Build fixture trees containing:
   - junctions, directory symlinks and file symlinks
   - hard links
   - sparse and NTFS-compressed files
   - paths longer than 260 characters
   - non-ASCII and emoji names
   - case-sensitive directories (`fsutil file setCaseSensitiveInfo`)
   - access-denied folders

   Assert sizes against `GetCompressedFileSizeW`. Every S/C defect in §2 gets a named regression test.
4. **UI smoke tests:** drive the real window with the `uiautomation` crate (open a fixture folder, zoom in/out, open settings, switch language, cancel) and capture screenshots for side-by-side comparison with the original at 100 % DPI.
5. **Soak test:** GDI and USER handle counts (`GetGuiResources`) before and after 10 k simulated hovers.

### CI matrix (GitHub Actions)

| Job | Runner | Steps |
|---|---|---|
| lint | ubuntu | `fmt --check`, `clippy -D warnings` (core + scan portable), `cargo deny check`, `cargo machete` |
| core | ubuntu | `nextest` (0.9.x), Miri on `sm-core`, `proptest` |
| windows | windows-latest (x64) + windows-11-arm | build, `clippy` on all crates, `nextest` including scanner fixtures, UI smoke, binary hardening check |
| fuzz | ubuntu, nightly schedule | 10 minutes per target, corpus cached |
| release | windows, tag push | `cargo auditable build --release`, sign, SBOM, provenance, checksums |

---

## 8. Decisions needed from the maintainer

1. **UI framework:** `windows` crate with the original `.rc` (recommended) vs `winsafe`.
2. **Control styling:** classic Windows 2000 controls (exact match) vs Common Controls v6 (same layout, modern native look).
3. **Size semantics:** allocated size on disk (correct, recommended) vs logical size. Both are stored either way; the question is which one drives the treemap. The original's round-down behaviour (C1) should not be kept.
4. **Hard links:** count once (accurate, costs a hash set of FileIds) vs count at every link (the original's behaviour).
5. **Delete safety:** keep "no confirmation when it goes to the Recycle Bin", but always confirm a permanent delete (recommended).
6. **Settings storage:** same registry key (recommended for continuity) vs a TOML file in `%APPDATA%` for portability.
7. **Scope extras:** folder/UNC targets, MFT fast path, additional languages, dark mode. All are optional and come after parity.

---

## 9. Phased roadmap

| Phase | Deliverable | Exit criteria | Rough effort* |
|---|---|---|---|
| **0. Baseline** | Build the original with VS 2022 + MFC (x86) for reference. Capture screenshots and scan timings. Remove `.ncb`/`.opt`. Add `.gitignore`, `.editorconfig`. Re-encode sources to UTF-8 for reference | Reference binary, screenshots and timings committed under `docs/baseline/` | 2–3 days |
| **1. `sm-core`** | Tree model, layout port, formatting, i18n, settings validation, palette | Differential layout tests match the C harness; Miri clean; fuzz targets running | 1–1.5 weeks |
| **2. `sm-scan`** | Windows and portable backends, worker thread, cancel and progress, parallel walk | All §2 scanner defects covered by tests; benchmark beats baseline | 1–1.5 weeks |
| **3. Shell + view** | Frame, toolbar, treemap painting, selection, zoom and animation, resources via `embed-resource` | Screenshot parity at 100 % DPI on fixture trees | 1.5–2 weeks |
| **4. Dialogs + tips + actions** | Drive, scan, settings, about dialogs; TipWnd port; context menu; Run, Delete, Properties; registry persistence; language switching | Full parity checklist (§1) ticked; handle soak test passes | 1–1.5 weeks |
| **5. Hardening + release** | Manifest (DPI, long paths), signing, SBOM, provenance, CI matrix, x64 + ARM64 artifacts | Release pipeline green; BinSkim clean; README updated | 3–5 days |
| **6. Optional** | MFT fast path, folder/UNC targets, more languages, dark mode | Per feature | – |

\*One experienced Rust + Win32 engineer, roughly 6–8 weeks to parity. The original's ~5.6 k lines should become about 5–7 k lines of Rust plus tests.

Retire the C++ sources to a `legacy/` directory (or a tag) once phase 4 reaches parity, and keep them until the first Rust release ships.

---

## 10. C++ → Rust mapping

| C++ | Rust |
|---|---|
| `CSpaceMonger` (`theApp` global) | `App` struct owned by the message loop, reached via window user-data, no `static mut` |
| `CFolderTree` / `CFolder` | `sm_core::Tree` / `Folder` / `Entry` (arena) |
| `CFolderDialog` (scan dialog + UI-thread pumping) | `ScanDialog` (modeless) + `sm_scan::Scanner` on a worker thread |
| `CFolderView` + `CDisplayFolder` list | `TreemapView` + `Vec<DisplayItem>`; layout in `sm_core::layout` |
| `CTipWnd` / `TipWnd.cpp` | `win::tip::TipWindow` (Rust window class, RAII state) |
| `CMainFrame` / `CMainToolBar` | `win::frame::MainFrame`, `win::toolbar::Toolbar` |
| `CDriveDialog`, `CSettingsDialog`, `CAboutDialog` | `DialogBoxParamW` over the same `.rc` templates, one `DLGPROC` per dialog |
| `CCurrentSettings` + registry | `sm_core::Settings` (pure validation) + `win::registry` persistence |
| `Lang` struct of `const char*` | `sm_core::i18n::Lang` with `&'static str` fields (UTF-8, compile-time complete) |
| `xapp.h` box-drawing helpers | `win::gdi` drawing helpers on a `Dc` RAII wrapper |
| `CFreeDoc` / `CFreeView` | Not needed: one document, one view |
| `e.h` typedefs (`ui64`, `#define bool char`, …) | Native Rust types |
