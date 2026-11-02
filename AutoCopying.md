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
