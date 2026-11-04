```
SRC="$HOME/Library/Mobile Documents/com~apple~CloudDocs"
DST="/Volumes/YourSSD/iCloud-Backup"
mkdir -p "$DST"
for d in "$SRC"/*/; do
  name=$(basename "$d")
  echo "== $name"
  rsync -a "$d" "$DST/$name/" || { echo "COPY FAILED: $name"; continue; }
  a=$(find "$d" -type f | wc -l)
  b=$(find "$DST/$name" -type f | wc -l)
  if [ "$a" -eq "$b" ]; then
    brctl evict "$d" && echo "OK, space freed: $name"
  else
    echo "COUNT MISMATCH ($a vs $b), left downloaded: $name"
  fi
done
```
```
SRC="$HOME/Library/Mobile Documents/com~apple~CloudDocs"
export DST="/Volumes/Extreme SSD/iCloud-Backup/com~apple~CloudDocs"
mkdir -p "$DST"
caffeinate -i -w $$ &
cd "$SRC" || exit 1
CP='rsync -aR "$@" "$DST/" && for f in "$@"; do brctl evict "$f"; done'
for d in */; do
  name="${d%/}"; start=$SECONDS
  echo "== $name"
  find "./$name" -type f ! -name .DS_Store -print0 | xargs -0 -n 25 -P 16 sh -c "$CP" _ || echo "SOME FILES FAILED in $name (rerun to retry)"
  t=$((SECONDS-start))
  printf "DONE %s | on SSD: %s | time: %02d:%02d:%02d\n" "$name" "$(du -sh "$DST/$name" | cut -f1)" $((t/3600)) $((t%3600/60)) $((t%60))
done
echo "== loose top-level files"
find . -maxdepth 1 -type f ! -name .DS_Store -print0 | xargs -0 -n 25 -P 16 sh -c "$CP" _
echo "===== ALL FINISHED ====="
```
