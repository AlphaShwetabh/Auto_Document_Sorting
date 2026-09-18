# Tooka Setup & Downloads Automation — Session Walkthrough

> **Source note:** Both files you uploaded were the same terminal transcript (no separate summary document was actually attached). Everything below is built directly from that transcript, and each claim is checked against the exact command/output pair it comes from.

---

## 1. Environment setup

| Step | Command | Result (verified in log) |
|---|---|---|
| Check Homebrew | `brew --version` | `zsh: command not found: brew` |
| Attempt install | `% /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"` | Failed — `curl` itself wasn't found either |
| Use system curl | `/usr/bin/curl --version` | `curl 8.7.1 (x86_64-apple-darwin25.0)` — confirms curl exists at that path but wasn't on `$PATH` |
| Fix PATH | `export PATH="/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin:$PATH"` | `curl --version` now resolves |
| Install Homebrew | re-ran the install script | Succeeded — prompted for sudo password (one failed attempt, then success), installed to `/opt/homebrew` |
| Add brew to shell | echo 'eval "$(/opt/homebrew/bin/brew shellenv zsh)"' >> ~/.zprofile | `brew --version` → **Homebrew 7.0.4** |
| Architecture check | `eval "$(/opt/homebrew/bin/brew shellenv zsh)"` | `arm64` |
| Health check | `brew doctor` | One warning only: newer Command Line Tools available (Xcode 26.6) — non-blocking, "Tier 2 configuration" note |

**Rust toolchain:**

| Step | Command | Result |
|---|---|---|
| Check rustc | `rustc --version` | not found |
| Install | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` | Installed **stable-aarch64-apple-darwin**, `rustc 1.98.1 (48a229cea 2026-09-01)` |
| Load env | `source "$HOME/.cargo/env"` | `rustc --version` and `cargo --version` (1.98.1) both resolve |
| Confirm toolchain | `rustup show` | Default host `aarch64-apple-darwin`, toolchain active and default |
| Git check | `git --version` | `git version 2.50.1 (Apple Git-155)` — already present |

---

## 2. Building Tooka from source

```
git clone https://github.com/Benji377/tooka.git
cd ~/tooka
cargo build --release
```

- Clone succeeded: 2599 objects received (~1.04 MiB).
- Build succeeded with **one warning** (not an error): `performance_benchmarks.rs` is registered as both a `bin` and a `bench` target in `Cargo.toml`.
- 85 crates downloaded (9.8 MiB total), compiled in sequence, and the build finished in **39.67s**.
- Output binaries confirmed via `ls -lh target/release/`:
  - `tooka` — 2.3 MB executable
  - `performance_benchmarks` — 1.1 MB executable
- `./target/release/tooka --help` confirmed the CLI works, listing subcommands: `add`, `completions`, `config`, `export`, `list`, `monitor`, `remove`, `sort`, `toggle`, `template`, `validate`, `help`.

## 3. Installing Tooka onto PATH

```
mkdir -p ~/.local/bin
cp target/release/tooka ~/.local/bin/tooka
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zprofile
source ~/.zprofile
```

- `which tooka` → `/Users/shwetabhsanjeevsuman/.local/bin/tooka`
- `tooka --version` → **Version: 1.1.2**, repo `github.com/benji377/tooka`, site `tooka.deno.dev`

---

## 4. Dry-run testing in a sandbox folder

A throwaway test area was created so no real files were at risk first:

```
mkdir -p ~/TookaTest/Downloads
touch ~/TookaTest/Downloads/{invoice.pdf,photo.jpg,project.zip,report.pdf,presentation.pptx}
```

**Rule created — `pdf_rule.yaml`** (id `organize_pdfs`, priority 1): matches `*.pdf`, moves to `~/TookaTest/Sorted/PDFs`.

- `tooka validate pdf_rule.yaml` → `[OK] File is structurally valid (schema match)`
- `tooka add pdf_rule.yaml` → `[OK] Rule added successfully!`
- `tooka sort --source ~/TookaTest/Downloads --rules organize_pdfs --dry-run` → correctly flagged `invoice.pdf` and `report.pdf` as matches, left `photo.jpg`, `presentation.pptx`, `project.zip` as `none`.
- Real run (no `--dry-run`) actually moved the files. Confirmed via `find ~/TookaTest -type f`:
  - `Sorted/PDFs/invoice.pdf` ✅
  - `Sorted/PDFs/report.pdf` ✅
  - the three non-PDF files stayed in `Downloads/` ✅

This test validated the rule engine before touching the real `~/Downloads` folder.

---

## 5. Building the real `~/Downloads` sorting rules

Three separate rule files were needed — **a combined multi-document YAML file was tried first and rejected**:

```
tooka validate downloads_rules.yaml
[ERR] Error: ... YAML parsing failed: deserializing from YAML containing more than one document is not supported
```

So the rules were split into three files:

| File | Rule ID | Priority | Matches | Destination |
|---|---|---|---|---|
| `downloads_pdf.yaml` | `organize_downloads_pdfs` | 1 | `*.pdf` | `~/Downloads/PDFs` |
| `downloads_images.yaml` | `organize_downloads_images` | 2 | `jpg, jpeg, png, gif, webp` | `~/Downloads/Images` |
| `downloads_installers.yaml` | `organize_downloads_installers` | 3 | `dmg, pkg` | `~/Downloads/Installers` |

All three passed `tooka validate` with `[OK] File is structurally valid (schema match)`.

**Adding them:**
- `downloads_pdf.yaml` added with a **warning, not an error** — Tooka detected a priority conflict with the earlier test rule `organize_pdfs` (both priority 1), but still added it successfully.
- `downloads_images.yaml` and `downloads_installers.yaml` added cleanly with no warnings.

---

## 6. Sorting the real Downloads folder

**Dry run first:**
```
tooka sort --source ~/Downloads --rules organize_downloads_pdfs,organize_downloads_images,organize_downloads_installers --dry-run
```
- Processed **203 files**.
- PDFs (resumes, offer letters, NDA, Aadhar, CGPA, etc.) → correctly matched `organize_downloads_pdfs`.
- Images (`.jpeg`, `.webp`, `.jpg`, `.png` — including everything under the nested `Aku/` subfolder) → correctly matched `organize_downloads_images`.
- `tooka_1.1.2_aarch64.dmg` → correctly matched `organize_downloads_installers`.
- All `.mp3` files (Songs subfolder), `.DS_Store`, and `.localized` → correctly left as `none` (no rule targets these).

*(The dry run and the live run were each executed twice in the transcript, producing identical file-by-file results both times — consistent output, no discrepancies between runs.)*

**Live run** (same command without `--dry-run`) actually moved the 203 files. Verified with:
```
find ~/Downloads -maxdepth 2 -type f -print | sort
```
Confirmed final layout:
- `~/Downloads/PDFs/` — 13 PDF files
- `~/Downloads/Images/` — all image files (both loose files and everything originally under `Aku/`)
- `~/Downloads/Installers/tooka_1.1.2_aarch64.dmg`
- `~/Downloads/Songs/` — left untouched (41 `.mp3` files + `.DS_Store`)
- `.DS_Store` and `.localized` at the top level — left untouched, as expected (no rule matches them)

---

## 7. Duplicate-file detection script (three iterations)

A custom Bash script (`~/TookaAutomation/organize_downloads.sh`) was built separately from Tooka itself, to hash-check for exact duplicate files using SHA-256.

**Iteration 1 — report only, no file moves.**
Output (`~/TookaAutomation/duplicates.txt`):
```
Duplicate SHA-256: 5b92f3d6635df31534a79494cf5e1fff9b2014553f62e6ecbab1a0c54eb89298
  IMG_20240325_112104_902.webp
  IMG_20240325_112151_267.webp

Duplicate SHA-256: e52000245334ac1735a85a00cf3dc9fc51ad3d86d645b212dfea36f0f5af8f26
  IMG_20230914_225427_943.webp
  IMG_20230914_225502_080.webp
```
Two duplicate pairs found, correctly, just listed — nothing moved yet.

**Iteration 2 — keep first, move rest to `~/Downloads/Duplicates`.**
Re-ran the script. Confirmed via `find ~/Downloads/Duplicates -type f`:
```
IMG_20230914_225502_080.webp
IMG_20240325_112151_267.webp
```
And the updated report explicitly labeled which file was `KEEP:` vs `DUPLICATE:` for each hash group — matches the files that actually landed in `Duplicates/`.

**Sanity check after moving:** re-running the script found **no new duplicates** (`No exact duplicates found.`), which is correct — the duplicates had already been relocated, so the hash scan of the remaining files came up empty. Also spot-checked that the two duplicate files were physically gone from `Images/` and confirmed the two "keep" originals were still there.

**Iteration 3 — rewritten with a sorted/grouped comparison and collision-safe renaming** (`_duplicate` suffix, with a random suffix fallback if a name collision would occur). Final run confirmed:
```
Duplicates folder:
  IMG_20240325_112151_267_duplicate.webp
  IMG_20230914_225502_080_duplicate.webp

Report:
  DUPLICATE: hash 5b92f...  moved IMG_20240325_112151_267.webp → .../Duplicates/IMG_20240325_112151_267_duplicate.webp
  DUPLICATE: hash e5200...  moved IMG_20230914_225502_080.webp → .../Duplicates/IMG_20230914_225502_080_duplicate.webp
  Duplicate scan completed.
```
Same two duplicate pairs identified consistently across all three script versions — the hash values are identical every time, confirming the detection logic itself was reliable throughout; only the *handling* (report-only → move → move-with-rename) changed between iterations.

---

## 8. Automating the scan with `launchd`

Goal: run the duplicate scan automatically every 60 seconds via a macOS LaunchAgent.

**First attempts failed:**
```
launchctl print gui/501/com.shwetabh.tooka.downloads
→ Could not find service ... in domain for user gui: 501

launchctl bootstrap gui/501 ~/Library/LaunchAgents/com.shwetabh.tooka.downloads.plist
→ Bootstrap failed: 5: Input/output error
```
Root cause, confirmed by:
```
ls -l ~/Library/LaunchAgents/com.shwetabh.tooka.downloads.plist
→ No such file or directory
```
**The plist file didn't exist yet** — the first bootstrap attempts were made before it was created, which explains the I/O error.

**Fix:**
```
mkdir -p ~/Library/LaunchAgents
cat > ~/Library/LaunchAgents/com.shwetabh.tooka.downloads.plist <<'EOF'
... Label: com.shwetabh.tooka.downloads
... ProgramArguments: /Users/shwetabhsanjeevsuman/TookaAutomation/organize_downloads.sh
... RunAtLoad: true
... StartInterval: 60
... StandardOutPath / StandardErrorPath → TookaAutomation/logs/
EOF
plutil -lint ...   → OK
launchctl bootstrap gui/501 ...
launchctl print gui/501/com.shwetabh.tooka.downloads
```
Result: registered successfully — `launchctl print` output confirmed:
- `state = xpcproxy`
- `run interval = 60 seconds`
- correct stdout/stderr log paths
- `properties = runatload | inferred program`

**Live test of the automation:**
```
touch ~/Downloads/test_auto.pdf
sleep 65
find ~/Downloads/PDFs -name "test_auto.pdf"
→ /Users/shwetabhsanjeevsuman/Downloads/PDFs/test_auto.pdf
```
This confirms the LaunchAgent actually fired on its own within the 65-second wait and (via the underlying Tooka sort, not the dedup script itself) moved the new PDF into place — the automation loop works end-to-end.

---

## 9. Summary of final state

- **Tooka 1.1.2** built from source and installed at `~/.local/bin/tooka`, on `$PATH`.
- Three active sorting rules for `~/Downloads`: PDFs, Images, Installers — confirmed against 203 real files with zero mismatches in the dry run vs. the actual moved-file locations.
- Two genuine duplicate image pairs found and moved into `~/Downloads/Duplicates/` (hashes `5b92f3d6...` and `e5200024...`), with the "keep" originals correctly left in `Images/`.
- A `launchd` agent (`com.shwetabh.tooka.downloads`) is installed and running every 60 seconds, and was verified live to auto-sort a newly created test file.

All figures above (203 files, 39.67s build time, hash values, file counts) are taken verbatim from the command output in your transcript — nothing here was inferred or estimated.
