# rename
```bash
#!/usr/bin/env bash
# Rename newname to newname_ttf everywhere: package names, crate names in code,
# the paths the derive macro and code generator emit, the #[oldname(...)]
# attribute, and file names. The result is used exactly like the old fork:
#   use newname_ttf::...;   #[derive(DdsType)]   #[newname_ttf(key)]
#
# Run from the workspace root of a dust-dds checkout. Safe to run again: after
# an upstream merge it renames only what the merge brought back.
#
#   oldname        -> newname_ttf
#   oldname_derive -> newname_ttf_derive
#   oldname_gen    -> newname_ttf_gen
#   (and every other oldname_* name, e.g. oldname_interoperability)
 
set -euo pipefail
 
OLD=oldname
NEW=newname
 
[[ -f Cargo.toml && -d dds && -d dds_derive ]] || {
  echo "Run this from the dust-dds workspace root" >&2; exit 1; }
 
# 1. File contents (source, manifests, IDL, docs incl. README doctests).
mapfile -t files < <(grep -rlI --exclude-dir=.git --exclude-dir=target \
  --include='*.rs' --include='*.toml' --include='*.idl' --include='*.md' \
  --include='*.h' --include='*.c' --include='*.py' --include='*.pyi' \
  "$OLD" . || true)
for f in "${files[@]}"; do
  sed -i "s/$OLD/$NEW/g" "$f"
done
echo "updated ${#files[@]} files"
 
# 2. File and directory names (deepest first, so parents are renamed last).
renamed=0
while IFS= read -r -d '' path; do
  dir="$(dirname "$path")"; base="$(basename "$path")"
  git mv -k "$path" "$dir/${base//$OLD/$NEW}" 2>/dev/null \
    || mv "$path" "$dir/${base//$OLD/$NEW}"
  renamed=$((renamed + 1))
done < <(find . -depth \( -path ./.git -o -path ./target \) -prune -o \
           -name "*$OLD*" -print0)
echo "renamed $renamed files"
 
# 3. Nothing should be left.
left="$(grep -rlI --exclude-dir=.git --exclude-dir=target "$OLD" . || true)"
if [[ -n "$left" ]]; then
  echo "note: '$OLD' still appears in (check these by hand):"
  echo "$left"
fi
```
