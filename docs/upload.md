# Upload to GitHub

## 1. Create the repo

Open github.com/new. Name it, set visibility to Private, and leave README,
gitignore and license unchecked. The folder already has all of that.

## 2. Push

```bash
cd solara-reverse
git remote add origin https://github.com/USERNAME/solara-reverse.git
git push -u origin main
```

If it asks for a password, use a personal access token instead of the account
password. Settings, developer settings, tokens, classic, scope repo.

Or let the CLI do both steps at once:

```bash
gh auth login
gh repo create solara-reverse --private --source=. --push
```

## 3. Check

Open the repo in a browser. README should render on the front page with docs,
scripts and artifacts listed.

Drag and drop also works on the empty repo page, but it loses history and the
browser uploader is slow with the zip bundles.

## Before pushing

Start private. The files in artifacts are decompiler output, which is derived
from copyrighted binaries and can attract a takedown. The prose in docs is your
own analysis and is fine.

If you want it public, drop artifacts first and ship docs plus scripts only. The
analysis stands on its own without the pseudocode dumps.

Never commit the binaries. The gitignore blocks the obvious extensions, but check
before running add with the all flag.

Size check: 1.2 MB total, well under the 50 MB warning and 100 MB block limits.

## Later updates

```bash
git add -A
git commit -m "update"
git push
```
