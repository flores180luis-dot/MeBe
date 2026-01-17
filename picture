#!/usr/bin/env bash
# scripts/generate_and_package_images.sh
# Generates cleaned PNG/WebP images from assets/images/quickstart.svg and packages them into a zip.
# Requires: one of rsvg-convert or inkscape, ImageMagick (convert/magick), and cwebp.
# Optional optimizers: oxipng, optipng, pngcrush
# Usage: ./scripts/generate_and_package_images.sh

set -euo pipefail
SVG="assets/images/quickstart.svg"
OUTDIR="assets/images"
TMPDIR="$(mktemp -d)"
ZIPNAME="MJCI-quickstart-images.zip"

mkdir -p "$OUTDIR"
cd "$(dirname "$0")/.." || exit 1

if [ ! -f "$SVG" ]; then
  echo "ERROR: $SVG not found. Create assets/images/quickstart.svg first."
  exit 2
fi

echo "Temporary dir: $TMPDIR"

# Helper: render SVG to PNG (square large canvas) using best available tool
render_png() {
  local outpng="$1"
  local width="$2"
  local height="$3"
  echo "Rendering $outpng (${width}x${height})..."
  if command -v rsvg-convert >/dev/null 2>&1; then
    rsvg-convert -w "$width" -h "$height" "$SVG" -o "$TMPDIR/$(basename "$outpng")"
  elif command -v inkscape >/dev/null 2>&1; then
    inkscape "$SVG" --export-type=png --export-filename="$TMPDIR/$(basename "$outpng")" --export-width="$width" >/dev/null 2>&1
  else
    echo "Install 'rsvg-convert' (librsvg) or 'inkscape' to render SVG -> PNG."
    exit 3
  fi
}

# Helper: center-crop with ImageMagick
center_crop() {
  local src="$1" dst="$2" w="$3" h="$4"
  echo "Center-crop $src -> $dst (${w}x${h})"
  if command -v magick >/dev/null 2>&1; then
    magick convert "$src" -gravity center -resize "${w}x${h}^" -extent "${w}x${h}" -strip -interlace Plane -quality 92 "$dst"
  else
    convert "$src" -gravity center -resize "${w}x${h}^" -extent "${w}x${h}" -strip -interlace Plane -quality 92 "$dst"
  fi
}

# Generate hero (1200x630) - render larger then crop for quality
TMP_HERO="$TMPDIR/quickstart_large.png"
HERO_PNG="$OUTDIR/quickstart-hero.png"
render_png "$TMP_HERO" 1800 1800
center_crop "$TMP_HERO" "$HERO_PNG" 1200 630

# Generate thumbnail (640x640)
TMP_THUMB="$TMPDIR/quickstart_thumb_large.png"
THUMB_PNG="$OUTDIR/quickstart-thumb.png"
render_png "$TMP_THUMB" 1024 1024
center_crop "$TMP_THUMB" "$THUMB_PNG" 640 640

# Generate full-width scaled master PNG (optional)
MASTER_PNG="$OUTDIR/quickstart-full.png"
render_png "$MASTER_PNG" 1400 1400
# optionally trim transparent margin if your SVG has any whitespace:
if command -v magick >/dev/null 2>&1; then
  magick convert "$MASTER_PNG" -trim +repage "$MASTER_PNG"
else
  convert "$MASTER_PNG" -trim +repage "$MASTER_PNG"
fi

# WebP conversion (lossy high quality)
WEBP_HERO="$OUTDIR/quickstart-hero.webp"
if command -v cwebp >/dev/null 2>&1; then
  echo "Converting hero to WebP..."
  cwebp -q 90 "$HERO_PNG" -o "$WEBP_HERO"
else
  echo "Warning: cwebp not found, skipping WebP generation. To install: 'sudo apt install webp' or use your package manager."
fi

# PNG optimization (try oxipng > optipng > pngcrush)
optimize_png() {
  local file="$1"
  if command -v oxipng >/dev/null 2>&1; then
    echo "Optimizing $file with oxipng..."
    oxipng -o 4 --strip all "$file"
  elif command -v optipng >/dev/null 2>&1; then
    echo "Optimizing $file with optipng..."
    optipng -o3 "$file"
  elif command -v pngcrush >/dev/null 2>&1; then
    echo "Optimizing $file with pngcrush..."
    pngcrush -ow -brute "$file" || true
  else
    echo "No PNG optimizer found (oxipng/optipng/pngcrush). Skipping lossless optimization."
  fi
}

optimize_png "$HERO_PNG"
optimize_png "$THUMB_PNG"
optimize_png "$MASTER_PNG"

# Package into zip
echo "Creating zip: $ZIPNAME"
# build list of files to include
FILES_TO_ZIP=("$HERO_PNG" "$THUMB_PNG" "$MASTER_PNG")
[ -f "$WEBP_HERO" ] && FILES_TO_ZIP+=("$WEBP_HERO")

zip -j "$ZIPNAME" "${FILES_TO_ZIP[@]}"

echo "Files created:"
ls -lh "${FILES_TO_ZIP[@]}" || true

# move zip into OUTDIR
mv "$ZIPNAME" "$OUTDIR/"

# cleanup
rm -rf "$TMPDIR"
echo "Done. Zip is at $OUTDIR/$ZIPNAME"