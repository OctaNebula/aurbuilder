# aurbuilder

Personal Arch Linux package repository. AUR packages listed in
[`packages.txt`](packages.txt) are built by GitHub Actions on a 6-hour cron
(plus a weekly cron for VCS packages), signed, and published as a rolling
release tagged [`repo`](../../releases/tag/repo). My machines pull prebuilt
binaries instead of compiling locally.

## Client setup

Add to `/etc/pacman.conf` (above `[core]` so it has priority):

```ini
[aurbuilder]
SigLevel = Required
Server = https://github.com/OctaNebula/aurbuilder/releases/download/repo
```

Trust the signing key:

```sh
curl -fsSL https://github.com/OctaNebula/aurbuilder/releases/download/repo/aurbuilder.pub.gpg \
  | sudo pacman-key --add -
sudo pacman-key --lsign-key <YOUR_KEY_FINGERPRINT>
sudo pacman -Sy
```

Replace `<YOUR_KEY_FINGERPRINT>` with the fingerprint of the GPG key whose
private half is stored in this repo's `GPG_PRIVATE_KEY` secret. You can read
it from any signed package on the release with
`gpg --list-packets aurbuilder.pub.gpg | grep -i fingerprint`.

## Maintenance

- **Add a package**: append the AUR name to `packages.txt`, commit, push.
  The next scheduled run picks it up.
- **Remove a package**: delete the line. (The old `.pkg.tar.zst` will linger
  in the release until manually pruned.)
- **Force a VCS rebuild now**: `gh workflow run build.yml -R OctaNebula/aurbuilder -f rebuild_vcs=true`
- **Force a full run now**: `gh workflow run build.yml -R OctaNebula/aurbuilder`

## How it works

- `archlinux:base-devel` container, non-root `builder` user.
- `aurutils` (`aur sync`) handles AUR-to-AUR dependency resolution and the
  "only build if newer" check by diffing AUR versions against the local repo
  db.
- Each run downloads the previous release's assets first, builds whatever is
  out of date, signs it, runs `repo-add`, then re-uploads everything with
  `gh release upload --clobber`.
- VCS packages are skipped on the 6-hourly runs; the Sunday 04:00 UTC cron
  rebuilds them with `aur sync --rebuild`.
- Per-package failures don't fail the workflow; results land in the run's
  step summary.

## Costs

Public repo, GitHub Actions free tier. A handful of small packages on a
6-hour cron stays well inside the included minutes. Heavy packages (browsers,
electron apps, anything LLVM-based) chew minutes fast — keep that in mind
before adding `firefox-developer-edition` or similar.
