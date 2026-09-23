**Script**
```
cat > "$HOME/TookaAutomation/duplicate_scanner.sh" << 'SCRIPT_EOF'
#!/bin/bash
set -uo pipefail

EXCLUDE_DIRS=(".git" "node_modules" "Library/Caches" ".Trash")
LOG_FILE="$HOME/TookaAutomation/logs/duplicates.log"
WORK_DIR="$HOME/TookaAutomation/tmp/$$"
TAB=$'\t'

DRY_RUN=false
ROOT=""
for arg in "$@"; do
  case "$arg" in
    --dry-run) DRY_RUN=true ;;
    *) ROOT="$arg" ;;
  esac
done
if [[ -z "$ROOT" ]]; then
  echo "Usage: $0 [--dry-run] <MAIN_FOLDER>"
  exit 1
fi
ROOT="${ROOT/#\~/$HOME}"
if [[ ! -d "$ROOT" ]]; then
  echo "ERROR: '$ROOT' is not a directory (drive unmounted / iCloud path wrong?)"
  exit 1
fi
ROOT="$(cd "$ROOT" && pwd)"

DUP_DIR="$ROOT/Duplicated"
mkdir -p "$DUP_DIR" "$WORK_DIR"
trap 'rm -rf "$WORK_DIR"' EXIT

log() { echo "$(date '+%Y-%m-%d %H:%M:%S') $*" >> "$LOG_FILE"; }
log "===== SCAN START root=$ROOT dry_run=$DRY_RUN ====="

PRUNE_PATHS=("$DUP_DIR")
for d in "${EXCLUDE_DIRS[@]}"; do PRUNE_PATHS+=("$ROOT/$d"); done
PRUNE_ARGS=(); first=true
for p in "${PRUNE_PATHS[@]}"; do
  if $first; then PRUNE_ARGS+=(-path "$p"); first=false
  else PRUNE_ARGS+=(-o -path "$p"); fi
done

# pass 1: size index (skip 0-byte files: empty files hash-collide trivially and aren't meaningful "duplicates")
SIZES_TSV="$WORK_DIR/sizes.tsv"; : > "$SIZES_TSV"
while IFS= read -r -d '' f; do
  sz=$(stat -f%z "$f" 2>>"$LOG_FILE") || { log "SKIP stat-failed: $f"; continue; }
  [[ "$sz" -eq 0 ]] && { log "SKIP zero-byte: $f"; continue; }
  printf '%s\t%s\n' "$sz" "$f" >> "$SIZES_TSV"
done < <(find "$ROOT" \( "${PRUNE_ARGS[@]}" \) -prune -o -type f ! -name ".DS_Store" ! -name "*.icloud" -print0 2>>"$LOG_FILE")

# pass 2: sizes with 2+ files
DUP_SIZES="$WORK_DIR/dup_sizes.txt"
cut -f1 "$SIZES_TSV" | sort | uniq -d > "$DUP_SIZES"

CANDIDATES="$WORK_DIR/candidates.tsv"
awk -F'\t' 'NR==FNR{d[$1]=1;next} ($1 in d)' "$DUP_SIZES" "$SIZES_TSV" > "$CANDIDATES"

# pass 3: hash candidates
HASHED="$WORK_DIR/hashed.tsv"; : > "$HASHED"
while IFS=$'\t' read -r sz f; do
  [[ -z "$f" ]] && continue
  mt=$(stat -f%m "$f" 2>>"$LOG_FILE") || { log "SKIP stat-failed: $f"; continue; }
  h=$(shasum -a 256 "$f" 2>>"$LOG_FILE" | awk '{print $1}')
  [[ -z "$h" ]] && { log "SKIP hash-failed (unreadable / iCloud not downloaded?): $f"; continue; }
  printf '%s\t%s\t%s\n' "$h" "$mt" "$f" >> "$HASHED"
done < "$CANDIDATES"

SORTED="$WORK_DIR/sorted.tsv"
sort -t "$TAB" -k1,1 -k2,2n -k3,3 "$HASHED" > "$SORTED"

# pass 4: group + move
TOTAL=0
prev_hash=""; original=""
while IFS=$'\t' read -r h mt f; do
  [[ -z "$f" ]] && continue
  if [[ "$h" != "$prev_hash" ]]; then
    original="$f"; prev_hash="$h"
    continue
  fi
  base=$(basename "$f")
  dest="$DUP_DIR/$base"
  n=1
  while [[ -e "$dest" ]]; do
    if [[ "$base" == *.* && "$base" != .* ]]; then
      dest="$DUP_DIR/${base%.*}_$n.${base##*.}"
    else
      dest="$DUP_DIR/${base}_$n"
    fi
    n=$((n+1))
  done
  TOTAL=$((TOTAL+1))
  if $DRY_RUN; then
    echo "[DRY-RUN] Would move: $f -> $dest"
    log "DRY-RUN Original=$original Duplicate=$f SHA256=$h WouldMoveTo=$dest"
  else
    if mv "$f" "$dest"; then
      echo "Moved: $f -> $dest"
      log "DUPLICATE Original=$original Duplicate=$f SHA256=$h MovedTo=$dest"
    else
      log "ERROR move-failed: $f"
    fi
  fi
done < "$SORTED"

log "===== SCAN END total_duplicates=$TOTAL ====="
echo "Done. $TOTAL duplicate(s) $($DRY_RUN && echo 'found (dry-run, nothing moved)' || echo 'moved to Duplicated/')."
SCRIPT_EOF
chmod +x "$HOME/TookaAutomation/duplicate_scanner.sh"
echo "Script updated (tab-delimited, avoids the bash 3.2 \\x01 read bug)."
```

# Test Run
```
"$HOME/TookaAutomation/duplicate_scanner.sh" --dry-run "$HOME/Downloads"
```
# Actual Run
```
"$HOME/TookaAutomation/duplicate_scanner.sh" "$HOME/Downloads"
```
# Then confirm the move worked and nothing looks wrong:
```
ls -la "$HOME/Downloads/Duplicated"
tail -20 "$HOME/TookaAutomation/logs/duplicates.log"
```
**To Confirm that automation is working** 
Setting->General->Login items & extentions -> App Background Activity

# Automation
```
cat > "$HOME/Library/LaunchAgents/com.user.tookadupes.plist" << PLIST_EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.tookadupes</string>
    <key>ProgramArguments</key>
    <array>
        <string>$HOME/TookaAutomation/duplicate_scanner.sh</string>
        <string>$HOME/Downloads</string>
    </array>
    <key>StartInterval</key>
    <integer>3600</integer>
    <key>RunAtLoad</key>
    <false/>
    <key>StandardOutPath</key>
    <string>$HOME/TookaAutomation/logs/launchd_stdout.log</string>
    <key>StandardErrorPath</key>
    <string>$HOME/TookaAutomation/logs/launchd_stderr.log</string>
</dict>
</plist>
PLIST_EOF
launchctl load "$HOME/Library/LaunchAgents/com.user.tookadupes.plist"
echo "Loaded — will run every 3600s (1 hour) against $HOME/Downloads."
```
**To Confirm that automation is working** 
Setting->General->Login items & extentions -> App Background Activity

```
launchctl list | grep tookadupes
cat "$HOME/TookaAutomation/logs/launchd_stdout.log" 2>/dev/null
tail -20 "$HOME/TookaAutomation/logs/duplicates.log"
```
# Disable it
```
launchctl unload "$HOME/Library/LaunchAgents/com.user.tookadupes.plist"
```
**To remove it entirely later**
```
launchctl unload "$HOME/Library/LaunchAgents/com.user.tookadupes.plist" 2>/dev/null
rm "$HOME/Library/LaunchAgents/com.user.tookadupes.plist"
```
