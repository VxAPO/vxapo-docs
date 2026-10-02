# Code Format Specification

This document defines **code formatting** for the VxAPO repositories. There is one rule:
**formatting is not a human decision.**

It records the outcome of `vxapo-format-policy-plan.md`; that plan file may be deleted at
any time, so this document is the long-term reference.

---

## 1. Single authority

Formatting is decided **only by `cargo fmt`**.

| Item | Value |
|---|---|
| Line-breaking strategy | **rustfmt defaults** (no custom width settings) |
| `style_edition` | `2021` (pinned explicitly) |
| `newline_style` | `Unix` |
| Toolchain | `channel = "1.97.1"` (pinned via `rust-toolchain.toml`) |

`style_edition` is pinned so that a future crate `edition` bump (2021 → 2024) does **not**
force another whole-repo reflow. `newline_style = "Unix"` matches `.gitattributes`
`eol=lf`.

Config file locations:

| Repository | `rustfmt.toml` / `rust-toolchain.toml` |
|---|---|
| `vxapo-driver` | repository root |
| `vxapo-cli` | repository root (shared with `protocol/`) |
| `vxapo-app` | **`src-tauri/`** (the root is not a Cargo package) |

All three are **byte-identical**; verify with a hash:

```sh
sha256sum vxapo-driver/rustfmt.toml vxapo-cli/rustfmt.toml vxapo-app/src-tauri/rustfmt.toml
```

### Why defaults instead of `use_small_heuristics = "Max"`

Measured over the whole driver crate:

| Option | hunks | added | removed | net | lines >90 cols |
|---|---|---|---|---|---|
| **Default (adopted)** | 445 | 1802 | 991 | +811 | **257** |
| `Max` | 504 | 962 | 1885 | −923 | **431** |

`Max` raises the count of hard-to-read `>90`-column lines from 287 to 431 (+68%) and
requires maintaining a custom config forever. Defaults need no local configuration at
all — any editor, CI system, or downstream tool produces the same formatting out of the box.

---

## 2. Equivalent mechanical changes made by `cargo fmt`

When reviewing a formatting commit, the following are **not** logic changes:

1. **Removal of a leading UTF-8 BOM** (U+FEFF) — driver had 17 such files.
2. **Trailing-comma normalisation** in multi-line constructs.
3. **Reordering of `use` lists and `mod` declarations** (lexicographic).
4. **Brace insertion/removal** for single-expression arms (`=> expr` ↔ `=> { expr }`).
5. **Semicolon insertion** on tail expressions (`else { return }` → `else { return; }`).

Verified: for the driver formatting commit, all 95 files are character-for-character
identical once whitespace, BOM, commas, braces, semicolons and `use`/`mod` ordering are
normalised away — **zero logic changes**.

---

## 3. Commit discipline

### 3.1 Formatting commits must be pure

**Never mix logic changes with a whole-repo reformat in one commit.**

`.git-blame-ignore-revs` relies on the fact that the commit is pure formatting. A commit
that also carries logic changes would cause `git blame` to hide the real changes too,
destroying traceability.

### 3.2 Blame protection

Each repository has `.git-blame-ignore-revs` listing the full hashes of pure formatting
commits. Each machine must run once (GitHub's web UI recognises the file automatically):

```sh
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

> **Measured (driver, 2026-10)**: after the formatting commit, only 27 lines across the
> whole repository still blame to it, and all are structurally **new** lines created by
> rustfmt (`}`, `},`, `);`, `assert!(`). Such lines have no predecessor and are inherently
> unaffected by the ignore file. Conversely, **pure re-indentation** is already
> re-attributed correctly by git itself. The file therefore yields no measurable benefit
> this round, but is kept because it will matter for a future rustc upgrade whose rustfmt
> genuinely relocates many existing lines.

---

## 4. Enforcement

### 4.1 Local pre-commit hook

Each repository has `.githooks/pre-commit`, which runs the format check and rejects
unformatted commits. The hook **only checks; it never rewrites** — auto-formatting would
fold unintended changes into the commit and hide the real content.

**Hooks do not propagate with a clone.** Every new clone must run once:

```sh
git config core.hooksPath .githooks
```

### 4.2 Editor

`.vscode/settings.json` is committed (`editor.formatOnSave` + rust-analyzer), so saving a
Rust file formats it automatically.

### 4.3 CI (not implemented)

All four repositories have GitHub remotes; `.github/workflows/` could run
`cargo fmt --all -- --check` and clippy. **Not created in this round** — optional hardening.

---

## 5. Per-repository differences (**the easiest thing to get wrong**)

| Repository | Format check command | Note |
|---|---|---|
| `vxapo-driver` | `cargo fmt --all -- --check` | self-contained |
| `vxapo-cli` | `cargo fmt -p vxapo-cli -p vxapo-protocol -- --check` | **must not use `--all`**, see below |
| `vxapo-app` | `cargo fmt --manifest-path src-tauri/Cargo.toml -- --check` | root is not a Cargo package |

### The `vxapo-cli` boundary trap

`vxapo-cli` depends on `../vxapo-driver` by path, so **`cargo fmt --all` also formats
driver's files**. Measured:

| Command | total hunks | of which driver files |
|---|---|---|
| `cargo fmt --all` | 589 | **445** |
| `cargo fmt -p vxapo-cli -p vxapo-protocol` | 141 | 0 |

Driver files would then appear in a cli commit, breaking the repository boundary and
complicating blame-ignore and rollback. **Hence cli always uses explicit `-p`.** Driver
files are only ever modified inside the driver repository.

---

## 6. Line endings

- `.gitattributes`: `* text=auto eol=lf`, with `*.cmd` / `*.bat` kept as CRLF.
  All four repositories have identical effective rules.
- Per git's rules this **overrides global `core.autocrlf`**, so the spurious
  "modified but content unchanged" state no longer occurs.
- `.editorconfig` covers non-Rust files (indentation, line endings, final newline).

> **Note**: `vxapo-app/.gitignore` is **GBK-encoded** (not UTF-8). Edit it at byte level;
> re-encoding it as UTF-8 would corrupt its Chinese comments.

---

## 7. Upgrading rustc (**an independent event**)

rustfmt output drifts between versions. **Never combine the upgrade with formatting**:

1. First complete the rustc / dependency upgrade and commit it (formatting may be red).
2. Then run the format command per repository. **If output changes**:
   - make it a separate `style:` commit (respecting §3.1);
   - append its full hash to that repository's `.git-blame-ignore-revs`;
   - update `channel` in that repository's `rust-toolchain.toml`.
3. If output does not change: just update `rust-toolchain.toml`.

The three Rust repositories pin the same version; update their `rust-toolchain.toml`
**together**.

---

## 8. Quality baseline (recorded before execution)

| Repository | `cargo test` | clippy |
|---|---|---|
| `vxapo-driver` | 491 passed / 0 failed / 1 ignored (release 483) | **exit 0** (the 222 → 0 cleanup must not regress) |
| `vxapo-cli` | 22 passed / 0 failed | exit 101 (**pre-existing**, not caused by formatting) |
| `vxapo-app` | passes (0 tests) | exit 101 (**pre-existing**, not caused by formatting) |

The `vxapo-cli` and `vxapo-app` clippy failures predate the format change: verified by
stashing back to the pre-fmt state, where the error sets are **identical**. Cleaning them
up is a separate task and must not be mixed into formatting.

The driver `1 ignored` is the real-machine test
`active_dependents_enumerates_active_dependents`, which needs
`cargo test -- --ignored`.

---

## 9. Targeted exemptions

Where a macro expansion or hand-aligned table genuinely becomes unreadable, use a
**targeted** `#[rustfmt::skip]` with a nearby justification. **Wholesale use is an
anti-pattern** — it recreates the inconsistency this work removed.

**No `#[rustfmt::skip]` was used anywhere in this round.**

> One exception was handled differently: in `src/object/apo/rtdump.rs`, rustfmt expanded a
> single-line `if x { unsafe { .. } } else { .. }`. The `SAFETY` comment that had sat above
> the `let` then became separated from its `unsafe` block by the new `if` line, tripping
> `clippy::undocumented_unsafe_blocks` (denied in driver's `lib.rs`). The fix moves the
> SAFETY comment directly above the `unsafe` block — **without** `#[rustfmt::skip]`. That
> fix is a separate commit, not part of the formatting commit.
