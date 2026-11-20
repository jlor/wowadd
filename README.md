# wowadd

A small CLI for installing and updating World of Warcraft addons from CurseForge
and GitHub releases, aimed at a Linux/Wine install where the usual addon managers
are awkward.

Single bash script. Needs `curl`, `jq`, `unzip` and `tar`, nothing else.

## Why the project id is needed

CurseForge puts its slug search and project pages behind Cloudflare, which answers
403 to anything that isn't a browser. The per-project *file* endpoints are open:

| Endpoint | Works without a key |
| --- | --- |
| `api/v1/mods/<id>/files` | yes |
| `api/v1/mods/<id>/files/<fid>/download` | yes |
| `api/v1/mods/search?slug=` | no (403) |
| the project HTML page | no (403) |

So wowadd needs the numeric project id once per addon. It's on the project page
under **About Project**, and it's remembered in the manifest afterwards.

`CURSEFORGE_API_KEY` (free from console.curseforge.com) is supposed to lift that,
but CurseForge approves keys **per endpoint**: a key can return 200 on
`/v1/games`, `/v1/mods/<id>` and `/v1/mods/<id>/files` and still answer 403 on
`/v1/mods/search` with the misleading *"API Key missing or invalid"*. Slug
resolution and `wowadd search`'s CurseForge half both need that one endpoint, so
with an unapproved key you still pass `--project-id` once per addon. wowadd says
which it was rather than blaming the key as a whole.

GitHub has no such restriction, so `github:owner/repo` works with no setup — and
for addons that publish there it's the better source, since the release carries a
`release.json` stating the flavour and interface version.

## Setup

```sh
wowadd config-init "/path/to/World of Warcraft/_forever_/Interface/AddOns"
```

Writes `~/.config/wowadd/config`:

```sh
ADDONS_DIR="…/Interface/AddOns"
GAME_VERSION="1.60.1"      # CurseForge's name for the flavour
GAME_INTERFACE="16001"     # what a .toc should claim
BACKUP_DIR="$HOME/wow-backups"
# CURSEFORGE_API_KEY=""
```

Any of those can be overridden per-run with `WOWADD_ADDONS_DIR`,
`WOWADD_GAME_VERSION`, `WOWADD_GAME_INTERFACE`, `WOWADD_BACKUP_DIR`.

## Use

```sh
wowadd install platynator --project-id 1361299   # first time
wowadd install platynator                        # id remembered
wowadd install github:Questie/Questie            # GitHub release
wowadd install github:Questie/Questie@bundle/v12.6.2+v1.0.11   # a specific tag

# A CurseForge URL works too; the slug and any file id are read out of it
wowadd install https://www.curseforge.com/wow/addons/forever-loot-sparkles/files/9084031 \
  --project-id 1729017

wowadd list
wowadd update --dry-run       # what would change
wowadd update                 # all of them
wowadd update questie         # just one

wowadd search questie         # find something to install
wowadd remove questie         # delete it and forget it
```

`--stable-only` restricts to release builds; `--alpha` allows alpha. The default
accepts release and beta, because some addons only ever ship beta.

## Search

```sh
wowadd search details damage meter
wowadd search bigwigs --limit 20
wowadd search questie --all     # don't filter CurseForge hits by game version
```

Both sources are queried. GitHub needs no setup (set `GITHUB_TOKEN` to raise the
rate limit); CurseForge needs a `CURSEFORGE_API_KEY` **approved for
`/v1/mods/search`** (see above) — without one, only the GitHub half runs and
wowadd prints why. CurseForge hits are filtered to builds for `GAME_VERSION`
unless `--all` is passed, and each row prints the project id, so the result can be
pasted straight into an install:

```
  NAME                           ID        SLUG                      DOWNLOADS 1.60.1
  Platynator                     1361299   platynator                    12345 yes
```

## Remove

```sh
wowadd remove questie                 # prompts, backs up, then deletes
wowadd remove questie plater --yes    # several, no prompt
wowadd remove questie --dry-run       # what would be deleted
wowadd remove questie --keep-files    # stop managing it, leave the folders
```

Only folders the manifest records for that addon are touched, and a folder that
another managed addon also claims is left in place with a warning — two addons
shipping the same embedded library would otherwise break the one left behind.
Folders are tarred into `BACKUP_DIR` first, as on install; `--no-backup` skips
that. Without `--yes` it prompts, and refuses to delete anything when stdin isn't
a terminal.

Settings in the `WTF` directory are not touched, so re-installing an addon later
keeps its saved variables.

## What it checks before installing

- the archive has no absolute or `../` paths
- it contains no `.exe` / `.dll` / `.so` / `.dylib`
- each addon folder's `.toc` claims `GAME_INTERFACE` — a mismatch warns but does
  not block, since plenty of addons work anyway

Everything is extracted and checked in a temp directory first, so a failed
download or a bad archive can't leave a half-installed addon behind. Existing
folders are tarred into `BACKUP_DIR` before being replaced.

## State

`<AddOns>/.wowadd.json` records, per addon: the source, project id or repo, the
installed file id or tag, the version string, and which top-level folders it
created. That last one matters for addons that ship more than one — Questie
installs both `Questie` and `QuestieDB`, and both are replaced together on update.

`wowadd list` also reports folders in `AddOns` that it didn't install, so a
hand-managed addon is visible rather than silently ignored.

## Limitations

- No dependency resolution. Addons that need a library shipped separately still
  need that library installed by hand.
- No dependency tracking on removal either: a library folder another addon needs
  is only spared if that addon is also managed by wowadd.
- Addons installed by hand aren't adopted automatically; re-install through
  wowadd to bring one under management. `remove` won't touch them.
- `update` compares the CurseForge file id or the GitHub tag, not a version
  number, so re-tagged releases are seen as changes.
