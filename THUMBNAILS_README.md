# Thumbnail Handoff

This repo is a static GitHub Pages dish visual catalog:

- Site URL: `https://alvlkorotkov.github.io/dish-visuals/`
- Main page files: `index.html` and `dishes.html`
- Data source: `dishes.json`
- Full-size images: `assets/dishes/<slug>/<angle>.png`
- Current image set: 27 dishes x 5 angles = 135 images

The page is slow because `dishes.html` renders every dish card immediately and each card uses the full-size PNG as the `<img src>`. Many files are several MB each. The intended improvement is:

- show a small thumbnail in the card
- keep the full-size PNG as the link target
- make the browser lazy-load thumbnails

## Current Rendering

In `dishes.html`, `renderCards()` currently builds each gallery item like this:

```js
const src = dish.images && dish.images[angle.key];
const content = src
  ? `<a class="image-link" href="${escapeHtml(src)}" target="_blank" rel="noopener"><img src="${escapeHtml(src)}" alt="${escapeHtml(dish.title)} · ${escapeHtml(angle.label)}"></a>`
  : `<div class="placeholder">Нужно сгенерировать<br>${escapeHtml(angle.label)}</div>`;
```

This means the thumbnail and full-size image are the same file.

## Recommended File Layout

Keep full-size images exactly where they are. Add a thumbnail next to each original:

```text
assets/dishes/<slug>/<angle>.png
assets/dishes/<slug>/<angle>.thumb.webp
```

Example:

```text
assets/dishes/mushroom-risotto/top-shot.png
assets/dishes/mushroom-risotto/top-shot.thumb.webp
```

Why this layout:

- It does not break existing full-size image links.
- It keeps each dish folder self-contained.
- It avoids adding a second parallel tree such as `assets/thumbs`.

## Thumbnail Specs

Good default:

```text
format: WebP
size: 480x360
quality: 72-78
crop: center crop to 4:3
filename: <angle>.thumb.webp
```

The gallery cards use `aspect-ratio: 4/3`, so thumbnails should be 4:3. `480x360` is enough for the current card grid and much smaller than the full PNGs.

If you want a slightly sharper desktop view, use `640x480`, but `480x360` is the better first pass for performance.

## Generate Thumbnails

Use `ffmpeg`, which is already available in this environment.

One file:

```bash
ffmpeg -y \
  -i assets/dishes/mushroom-risotto/top-shot.png \
  -vf "scale=480:360:force_original_aspect_ratio=increase,crop=480:360" \
  -quality 76 \
  assets/dishes/mushroom-risotto/top-shot.thumb.webp
```

Batch all referenced images from `dishes.json`:

```bash
node - <<'NODE'
const fs = require("fs");
const { spawnSync } = require("child_process");

const data = JSON.parse(fs.readFileSync("dishes.json", "utf8"));
const seen = new Set();

for (const dish of data.dishes) {
  for (const [angle, imagePath] of Object.entries(dish.images || {})) {
    if (!imagePath || seen.has(imagePath)) continue;
    seen.add(imagePath);

    const thumbPath = imagePath.replace(/\.png$/i, ".thumb.webp");
    const result = spawnSync("ffmpeg", [
      "-y",
      "-i", imagePath,
      "-vf", "scale=480:360:force_original_aspect_ratio=increase,crop=480:360",
      "-quality", "76",
      thumbPath
    ], { stdio: "inherit" });

    if (result.status !== 0) {
      process.exit(result.status);
    }
  }
}
NODE
```

Expected output count: 135 `.thumb.webp` files.

Quick check:

```bash
find assets/dishes -name '*.thumb.webp' | wc -l
```

## Update `dishes.json`

Add a sibling object named `thumbs` for each dish. Keep `images` as the full-size source of truth.

Example:

```json
"images": {
  "top-shot": "assets/dishes/mushroom-risotto/top-shot.png",
  "close-up": "assets/dishes/mushroom-risotto/close-up.png"
},
"thumbs": {
  "top-shot": "assets/dishes/mushroom-risotto/top-shot.thumb.webp",
  "close-up": "assets/dishes/mushroom-risotto/close-up.thumb.webp"
}
```

Batch update:

```bash
node - <<'NODE'
const fs = require("fs");

const path = "dishes.json";
const data = JSON.parse(fs.readFileSync(path, "utf8"));

for (const dish of data.dishes) {
  dish.thumbs = {};
  for (const [angle, imagePath] of Object.entries(dish.images || {})) {
    dish.thumbs[angle] = imagePath.replace(/\.png$/i, ".thumb.webp");
  }
}

fs.writeFileSync(path, JSON.stringify(data, null, 2) + "\n");
NODE
```

Then verify every referenced thumbnail exists:

```bash
node - <<'NODE'
const fs = require("fs");
const data = JSON.parse(fs.readFileSync("dishes.json", "utf8"));
const missing = [];

for (const dish of data.dishes) {
  for (const [angle, thumbPath] of Object.entries(dish.thumbs || {})) {
    if (!fs.existsSync(thumbPath)) {
      missing.push(`${dish.slug}:${angle}:${thumbPath}`);
    }
  }
}

console.log(`missing thumbnails: ${missing.length}`);
if (missing.length) {
  console.log(missing.join("\n"));
  process.exit(1);
}
NODE
```

## Update `dishes.html`

Change only the gallery rendering inside `renderCards()`.

Desired logic:

- `src` remains the full-size PNG.
- `thumb` comes from `dish.thumbs[angle.key]`.
- fallback to `src` if a thumbnail is missing, so the site never breaks.
- add `loading="lazy"` and `decoding="async"`.

Replace the current image content expression with this pattern:

```js
const src = dish.images && dish.images[angle.key];
const thumb = dish.thumbs && dish.thumbs[angle.key] ? dish.thumbs[angle.key] : src;
const content = src
  ? `<a class="image-link" href="${escapeHtml(src)}" target="_blank" rel="noopener"><img src="${escapeHtml(thumb)}" loading="lazy" decoding="async" alt="${escapeHtml(dish.title)} · ${escapeHtml(angle.label)}"></a>`
  : `<div class="placeholder">Нужно сгенерировать<br>${escapeHtml(angle.label)}</div>`;
```

Important: the `<a href>` must stay pointed at `src`, not `thumb`, so clicking opens the full-size image.

Also copy the same `dishes.html` change into `index.html`, because this repo currently has both files and they are expected to match.

## Optional HTML Improvement

Add fixed image dimensions to reduce layout work:

```html
<img
  src="..."
  loading="lazy"
  decoding="async"
  width="480"
  height="360"
  alt="..."
>
```

This is optional because CSS already sets `aspect-ratio: 4/3`, but dimensions are still nice for browser scheduling.

## Verify Locally

Because the site fetches `dishes.json`, use a local static server:

```bash
cd /home/korot/projects/dish-visuals-public
python3 -m http.server 8080
```

Open:

```text
http://localhost:8080/dishes.html
```

Check in browser DevTools Network:

- card images should request `.thumb.webp`
- clicking an image should open the original `.png`
- `generation-queue.json` should remain empty
- status should remain 27/27 complete

## Verify With CLI

Check JSON validity:

```bash
node -e 'JSON.parse(require("fs").readFileSync("dishes.json", "utf8")); console.log("json ok")'
```

Check thumbnail count:

```bash
find assets/dishes -name '*.thumb.webp' | wc -l
```

Check no missing thumbnail references:

```bash
node - <<'NODE'
const fs = require("fs");
const data = JSON.parse(fs.readFileSync("dishes.json", "utf8"));
const missing = [];

for (const dish of data.dishes) {
  for (const [angle, thumbPath] of Object.entries(dish.thumbs || {})) {
    if (!fs.existsSync(thumbPath)) missing.push(`${dish.slug}:${angle}:${thumbPath}`);
  }
}

if (missing.length) {
  console.error(missing.join("\n"));
  process.exit(1);
}

console.log("thumbnail references ok");
NODE
```

## Commit And Push

Expected changed files:

```text
dishes.json
dishes.html
index.html
assets/dishes/*/*.thumb.webp
```

Suggested commit message:

```text
Add WebP thumbnails for dish gallery
```

Commands:

```bash
git status --short
git add dishes.json dishes.html index.html assets/dishes
git commit -m "Add WebP thumbnails for dish gallery"
git push origin main
```

After push, check GitHub Pages:

```bash
gh api repos/alvlkorotkov/dish-visuals/pages/builds/latest
```

Then verify the published data:

```bash
curl -L "https://alvlkorotkov.github.io/dish-visuals/dishes.json?ts=$(git rev-parse --short HEAD)" \
  | node -e 'let s="";process.stdin.on("data",d=>s+=d);process.stdin.on("end",()=>{const data=JSON.parse(s); console.log(data.dishes.length, "dishes"); console.log("thumbs on first dish:", Object.keys(data.dishes[0].thumbs || {}).length);})'
```

## Expected Result

The visual behavior should be unchanged:

- same 27 dish cards
- same 5 angles per dish
- same full-size image opens on click

The loading behavior should improve:

- initial page loads small `.thumb.webp` files
- full `.png` files load only after click
- below-the-fold thumbnails are deferred by `loading="lazy"`

