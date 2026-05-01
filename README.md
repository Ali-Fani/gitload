# Git-dozdi — Commit-Driven Downloader for GitHub

A single GitHub Actions workflow (`download.yml`) that turns any repository into
a remote download/relay machine, **driven entirely by commit messages**.

Push a commit whose message contains a magic command (e.g. `download: <url>`)
and the workflow will:

- Pull the file on a fast GitHub-hosted runner using `aria2c` (multi-connection)
- Either upload it as a **GitHub Release asset** (up to 2 GB per file) or **commit it back into the repo** (auto-split as a multi-volume zip when over 95 MB)
- Skip itself entirely when no command is present — no wasted minutes

---

## ✨ Features

- **Early skip**: a tiny detection job decides whether to run anything at all. No deps installed, no time wasted on unrelated commits.
- **Two destinations**:
  - `download:` / `download-zip:` → published as a **GitHub Release**
  - `downloadc:` → **committed into the repo** under `commit-files/`
- **Smart splitting** for repo commits: files over 95 MB are wrapped in a `zip -s 95m -0` (store mode, no compression) multi-volume archive that Windows users can extract with **7-Zip / WinRAR / PeaZip** in one click.
- **Per-volume manifest** with extraction instructions for Windows, Linux and macOS.
- **Fast downloads** via `aria2c` with 4 parallel connections per server.

---

## 🚀 Quick Start — add to any repo

1. Create a new (or use an existing) GitHub repository.
2. Add the workflow file at `.github/workflows/download.yml` — copy the contents of [`download.yml`](./download.yml).
3. (Optional) Enable workflow write permissions:
   - Repo → **Settings** → **Actions** → **General** → **Workflow permissions** → select **Read and write permissions**.
   - This is required for `downloadc:` (commit) and for creating Releases.
4. Push any commit whose message contains a command — see below.

That's it. No secrets, no extra setup; the built-in `GITHUB_TOKEN` is used.

---

## 📜 Commands

All commands are **prefixes inside the commit message**. URLs follow them, separated by spaces. Multiple commands can coexist in the same commit.

| Command          | Destination                | Size limit       | Behaviour                                                                          |
| ---------------- | -------------------------- | ---------------- | ---------------------------------------------------------------------------------- |
| `download:`      | GitHub Release                | 2 GB / file      | Each URL becomes its own release asset                                          |
| `download-zip:`  | GitHub Release                | 2 GB total       | All URLs are bundled into a single `release-bundle.zip` asset                   |
| `downloadc:`     | Repo commit (`commit-files/`) | 100 MB / object  | Files ≤ 95 MB committed as-is. Larger files are split into a multi-volume zip.  |
| `telegram:`      | Repo commit (`commit-files/`) | 2 GB / file      | Downloads all media from a `t.me/…` post (albums too). Requires 3 TG secrets.  |

> **Note**: GitHub blocks individual git objects > 100 MB, which is why `downloadc:` and `telegram:` split at 95 MB. Releases have no such limit (up to 2 GB per asset).

---

## 🧪 Usage examples

### Release a single file

```bash
git commit --allow-empty -m "download: https://ash-speed.hetzner.com/100MB.bin"
git push
```

→ Creates a release `auto-YYYYMMDD-HHMMSS` containing `100MB.bin`.

### Release several files in one zip

```bash
git commit --allow-empty -m "download-zip: https://example.com/a.iso https://example.com/b.iso"
git push
```

→ Creates a release containing `release-bundle.zip` with both files inside.

### Commit a large file into the repo

```bash
git commit --allow-empty -m "downloadc: https://ash-speed.hetzner.com/1GB.bin"
git push
```

→ Workflow auto-commits to the repo:

```
commit-files/
└── 1GB/
    ├── 1GB.z01            (95 MB)
    ├── 1GB.z02            (95 MB)
    ├── …
    ├── 1GB.zip            (last volume)
    └── MANIFEST.txt
```

### Combine both in one push

```bash
git commit --allow-empty -m "download: https://example.com/public.bin
downloadc: https://example.com/private.bin"
git push
```

→ `public.bin` goes to a Release; `private.bin` is committed to the repo.

### Download a Telegram post

```bash
# Single post (public channel)
git commit --allow-empty -m "telegram: https://t.me/durov/142"
git push
```

→ All media from the post is committed to `commit-files/`.

```bash
# Album post — all items in the group are downloaded automatically
git commit --allow-empty -m "telegram: https://t.me/somechannel/15"
git push
```

```bash
# Private / restricted channel (your session account must be a member)
git commit --allow-empty -m "telegram: https://t.me/c/1234567890/678"
git push
```

```bash
# Mix with a regular download
git commit --allow-empty -m "telegram: https://t.me/durov/142
download: https://example.com/file.bin"
git push
```

→ Telegram media → `commit-files/`; `file.bin` → GitHub Release.

### Push without a command

```bash
git commit -m "fix typo"
git push
```

→ The `check-command` job runs (a few seconds), detects no command, and the workflow ends. No download, no release, no commit.

---

## 📦 Reassembling split files (for `downloadc:` / `telegram:` over 95 MB)

The workflow uses **standard zip multi-volume format** (`zip -s 95m -0`), so no custom scripts are needed.

### Windows

1. Download the entire folder (`commit-files/<name>/`) — keep **all** `.z01`, `.z02`, …, `.zip` files together.
2. Right-click `<name>.zip` → **7-Zip** → **Extract Here**.
   (Or use WinRAR / PeaZip — they all auto-detect the split.)

> ⚠️ **Windows Explorer's built-in "Extract All" cannot handle split zips.** Install [7-Zip](https://www.7-zip.org/) (free).

### Linux / macOS

```bash
zip -s 0 <name>.zip --out joined.zip
unzip joined.zip
```

The `MANIFEST.txt` next to the parts repeats these instructions.

---

## � Telegram support

### Why MTProto and not the Bot API?

| | Bot API | MTProto (Pyrogram) |
|---|---|---|
| Access public channels by URL | ❌ | ✅ |
| Access private channels | ❌ (bot must be admin) | ✅ (if account is a member) |
| Max file size | 20 MB download | 2 GB |
| Setup complexity | Token only | api_id + api_hash + session |

### One-time setup — 3 GitHub secrets

| Secret | Where to get it |
|---|---|
| `TG_API_ID` | [my.telegram.org/apps](https://my.telegram.org/apps) → `api_id` |
| `TG_API_HASH` | same page → `api_hash` |
| `TG_SESSION_STRING` | generated with `scripts/gen-session.py` (see below) |

Add them at: **Repo → Settings → Secrets and variables → Actions → New repository secret**.

### Generate the session string

The script is a [PEP 723](https://peps.python.org/pep-0723/) inline-script — **`uv run` installs its own deps automatically into an isolated environment, zero setup required**:

```bash
# Recommended — uv handles pyrogram + tgcrypto automatically
uv run scripts/gen-session.py

# Pass credentials inline to skip prompts
uv run scripts/gen-session.py --api-id 123456 --api-hash abc123def

# Via env vars
TG_API_ID=123456 TG_API_HASH=abc123def uv run scripts/gen-session.py
```

No `uv`? Fall back to plain Python (deps must be installed manually):

```bash
pip install pyrogram tgcrypto
python scripts/gen-session.py
```

The script walks you through the Telegram login (phone number → code → optional 2FA), then prints the session string framed for easy copy-paste. No `.session` file is written to disk.

> ⚠️ **Security**: the session string is equivalent to your Telegram password. Only ever paste it into GitHub Actions Secrets — never commit it to a repository.

### Caveats

- **Albums** (media groups): all items are fetched in one run automatically.
- **File size**: up to 2 GB per file (MTProto limit). Files > 95 MB are split into a multi-volume zip automatically before committing.
- **FloodWait**: Pyrogram handles Telegram rate-limit errors by sleeping and retrying automatically.
- **Private channels**: works as long as the account whose session you supplied is a member of the channel.
- **Text-only posts**: skipped with a warning — only posts with media are downloaded.

---

## �� Reproduction steps — full setup from scratch

```bash
# 1. Create the repo locally and push
mkdir my-downloader && cd my-downloader
git init
git remote add origin git@github.com:<you>/my-downloader.git

# 2. Add the workflow
mkdir -p .github/workflows
curl -fsSL https://raw.githubusercontent.com/<you>/git-dozdi/main/download.yml \
     -o .github/workflows/download.yml
# (or copy the file from this repo)

# 3. Initial commit
echo "# my-downloader" > README.md
git add .
git commit -m "ci: add download workflow"
git push -u origin main

# 4. Enable write permissions in
#    Settings → Actions → General → Workflow permissions → Read and write

# 5. Trigger your first download
git commit --allow-empty -m "download: https://ash-speed.hetzner.com/100MB.bin"
git push
```

Open the **Actions** tab to watch the run, then check the **Releases** tab for the asset.

---

## 🗂 Output layout in the repo

```
your-repo/
├── .github/workflows/download.yml
├── scripts/
│   └── gen-session.py               ← run once locally to generate TG_SESSION_STRING
├── commit-files/                    ← created by `downloadc:` and `telegram:`
│   ├── small-file.bin               (≤ 95 MB, kept as-is)
│   └── big-file/                    (> 95 MB, auto-split multi-volume zip)
│       ├── big-file.z01
│       ├── big-file.z02
│       ├── big-file.zip
│       └── MANIFEST.txt
└── (release assets live in GitHub Releases, not in the repo)
```

---

## ⚙️ How it works (high level)

```
push ──▶ check-command job  ──(no command)──▶ workflow ends
              │
              └──(command found)──▶ release-upload job
                                      │
                                      ├─ install aria2 + zip + python3
                                      ├─ (if telegram:) install pyrogram + download TG media → /tmp/commit-dl/
                                      ├─ download via aria2c
                                      ├─ (zip-mode) bundle release files
                                      ├─ split commit files via `zip -s 95m -0`
                                      ├─ git add + commit + push (if downloadc: or telegram:)
                                      └─ create GitHub Release (if download:)
```

Two-job structure means **commits without a command cost only a few seconds of runner time**.

---

## ❓ FAQ

**Why not just `wget`/`curl`?**
`aria2c` opens 4 parallel connections per server, which is significantly faster on most CDNs.

**Why not store split files with the original `.partXXXofYYY` naming?**
Multi-volume zip is a universal, well-known format with built-in CRC32 integrity checks. Every major archiver handles it natively, with no custom reassembly script required.

**Can I tweak the chunk size / worker count?**
Yes — edit `download.yml`:
- `--split=4 --max-connection-per-server=4` for aria2c parallelism
- `LIMIT=$((95 * 1024 * 1024))` and `zip -s 95m` for split size (keep them in sync)

**Does the workflow need any secrets?**
For `download:` / `download-zip:` / `downloadc:`: no — the built-in `GITHUB_TOKEN` is sufficient.
For `telegram:`: yes — `TG_API_ID`, `TG_API_HASH`, and `TG_SESSION_STRING` must be set as repo secrets.

**Will it run on every push?**
The `check-command` job runs on every push, but it's fast and free (Linux runner minutes are unmetered for public repos). The heavy `release-upload` job only runs when a command is detected.

---

## 📄 License

MIT — do whatever you want.
