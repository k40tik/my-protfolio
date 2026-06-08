# TODO - Fix image not rendering in Hugo Congo

## Step 1
Explain why the image isn’t rendering: theme uses `resources.Get`, which requires the asset to be in Hugo’s asset pipeline (typically `assets/`). The file is currently only in `themes/congo/images/`.

## Step 2
Move/copy the file:
- create `assets/images/`
- copy `themes/congo/images/k40tik.jpeg` → `assets/images/k40tik.jpeg`

## Step 3
Ensure config points correctly (keep):
- `config/_default/languages.en.toml`: `image = "/images/k40tik.jpeg"`

## Step 4
Rebuild and verify:
- run `hugo --minify --printPathWarnings`
- confirm the author image appears / no missing asset warnings.

