# CLAUDE.md

btop configuration kept under version control **in place**: on a configured
machine this repo *is* `~/.config/btop`. There is no build, no install step, and
no test suite — editing `btop.conf` here is editing the live config.

Upstream: <https://github.com/aristocratos/btop>

This file is the decision log for the repo. See
[Keeping this file current](#keeping-this-file-current) at the bottom — it is
part of the contract, not a footnote.

---

## Layout

| Path         | Tracked | Notes                                                       |
| ------------ | ------- | ----------------------------------------------------------- |
| `btop.conf`  | yes     | The entire config. Flat `key = value`; btop rewrites it whole. |
| `README.md`  | yes     | Install notes (Homebrew / pacman).                          |
| `.gitignore` | yes     | Differs between branches — see below.                       |
| `themes/`    | no*     | Machine-local. See [Theme resolution](#theme-resolution).    |
| `.claude/`   | no      | Ignored on both branches.                                    |

\* `themes/` is ignored on `btop-omarchy` only. On `main` the ignore rule was
added and then reverted (`c68b7a0` → `5b5e0d9`), so `main` does not ignore it.

---

## Branches

One branch per platform:

| Branch         | Machines            | btop    | `color_theme`  |
| -------------- | ------------------- | ------- | -------------- |
| `main`         | macOS               | 1.4.5   | `"flat-remix"` |
| `btop-omarchy` | Omarchy (Arch + Hyprland) | 1.4.7   | `"current"`    |

The branches have diverged (neither is an ancestor of the other), but the only
**substantive** difference is `color_theme`. Everything else in the diff is
btop's own serializer noise:

- `True`/`False` on `main` vs `true`/`false` on `btop-omarchy` — btop changed
  its boolean casing between 1.4.5 and 1.4.7.
- Keys that 1.4.7 writes and 1.4.5 does not: `disable_presets`, `disable_mouse`,
  `terminal_sync`, `graph_symbol_gpu`, `show_gpu_info`, `proc_follow_detailed`,
  `keep_dead_proc_usage`, `freq_mode`, `swap_upload_download`,
  `save_config_on_exit`, `nvml_measure_pcie_speeds`, `rsmi_measure_pcie_speeds`,
  `gpu_mirror_graph`, `shown_gpus`, `custom_gpu_name0`–`5`.

Every shared setting that a human actually chose (`vim_keys`, `update_ms`,
`proc_sorting`, `show_disks`, `cpu_invert_lower`, `theme_background`, …) already
holds the same value on both branches.

> **Open question:** because the real divergence is one key, the two branches
> could collapse into `main` — set `color_theme = "current"` everywhere and let
> each machine supply its own ignored `themes/current.theme`. This has not been
> done; if it is, replace this section and record it under Decisions.

---

## Theme resolution

btop follows the Omarchy desktop theme through a two-part bridge. Only the
second part lives in this repo — **Omarchy knows nothing about btop.conf.**

### Omarchy's half

`omarchy theme set <name>` (`/usr/share/omarchy/bin/omarchy-theme-set`) stages
the theme into `~/.local/state/omarchy/current/next-theme`, then
`omarchy-theme-set-templates` renders `btop.theme` from
`/usr/share/omarchy/default/themed/btop.theme.tpl`, substituting
`{{ background }}`, `{{ foreground }}`, `{{ accent }}`, `{{ selection }}`,
`{{ muted }}` … from that theme's `colors.toml`. The staging directory is then
swapped in atomically:

```sh
rm -rf ~/.local/state/omarchy/current/theme
mv     ~/.local/state/omarchy/current/next-theme \
       ~/.local/state/omarchy/current/theme
```

So `btop.theme` is **generated, not shipped** — it does not exist in
`/usr/share/omarchy/themes/<name>/`. Last, `omarchy-restart-btop` (which is just
`pkill -SIGUSR2 btop`) tells a running btop to reload.

### Our half

`color_theme = "current"` makes btop load `~/.config/btop/themes/current.theme`,
which is a hand-made symlink:

```
themes/current.theme -> /home/anwar/.local/state/omarchy/current/theme/btop.theme
```

Omarchy does not create or maintain this symlink — it is machine-local setup.
That is why `themes/` is ignored rather than committed, and why **tracking the
symlink was tried and failed** (see Decisions).

**Why the symlink survives theme switches:** it targets a *path*, not a
particular theme's file. The swap above deletes and recreates the whole
directory on every switch, and the symlink simply re-resolves to the freshly
generated file. A symlink pointed straight at one theme's colors would break on
the first switch.

---

## Decisions

Newest first. Each entry is a choice someone made on purpose, with the reason —
not a changelog of what changed.

### Theme follows the desktop, via an ignored symlink

`color_theme = "current"` (`1f0e1c5`, 2025-12-03), `themes/` fully ignored
(`4f6e125`, 2025-12-21).

The intermediate attempt is the instructive part. The symlink was committed
(`16a1a68`, 2025-12-03), then deleted 18 days later (`eb42b55`) and the ignore
rule widened from `themes/current.theme` to the whole `themes` directory
(`4f6e125`). The committed symlink pointed at
`~/.config/omarchy/current/theme/btop.theme`; Omarchy has since relocated that
to `~/.local/state/omarchy/current/theme/btop.theme`. A tracked symlink would
today point at nothing, on every machine.

**Rule that follows:** machine-resolved paths do not belong in this repo.

`main` reached the same conclusion earlier by a different route — the ignore
rule was added (`c68b7a0`) and reverted (`5b5e0d9`).

### btop rewrites `btop.conf` on exit

`save_config_on_exit = true` (`btop-omarchy`, since `fa80a58`).

Quitting btop re-serializes the whole file. Consequences to expect:

- `git diff` shows changes nobody typed, just from running btop.
- A btop upgrade lands as a bulk rewrite — new keys at their defaults, plus
  reformatting. `fa80a58` (1.4.5 → 1.4.6) and `ad2eb74` (1.4.6 → 1.4.7) are both
  this.
- **A real setting change can hide inside one of those bulk commits.** It has
  already happened once: `theme_background` flipped `True` → `false` inside
  `fa80a58`, with no note saying whether that was deliberate. Current value on
  both branches is `false`, which keeps the terminal background transparent.

When reviewing a bulk-rewrite diff, compare non-comment lines only (command in
the last section) and call out any value that actually moved.

### Disks shown in the memory box

`show_disks = true` (`786b616`, 2025-12-09). Set true at initial setup, turned
off in `1f0e1c5`, turned back on deliberately by `786b616` — the commit exists
for this one line. It has stayed on since, on both branches.

### Interaction and layout preferences

Set at initial setup (`d77c7df`, 2025-12-03) and unchanged on both branches
since. Listed because "unchanged for a year on two branches" is itself the
signal — treat these as intentional, not as defaults nobody looked at:

| Setting             | Value      | Effect                                        |
| ------------------- | ---------- | --------------------------------------------- |
| `vim_keys`          | `true`     | `hjkl` navigation; `h`/`k` need shift.        |
| `update_ms`         | `2000`     | 2s refresh — slower than btop's default.      |
| `proc_sorting`      | `"memory"` | Process list sorted by memory, not CPU.       |
| `cpu_invert_lower`  | `true`     | Lower CPU graph mirrored.                     |
| `shown_boxes`       | `"cpu mem net proc"` | GPU box not shown by default.       |
| `rounded_corners`   | `true`     |                                               |

---

## Conventions

- **Commits:** lowercase, no trailing period, no body. Merged as PRs, so
  subjects carry a `(#N)` suffix.
- **One decision per commit.** `786b616` changing a single line is the model,
  not the exception.
- **Never commit on top of a bulk btop rewrite.** Land the rewrite alone, then
  make the real change in its own commit, so the diff stays readable.
- Version-bump commits name the version: `updated for btop v.1.4.6`.

---

## Keeping this file current

This file earns its place only if it moves with the repo. **When you change
anything tracked here, update this file in the same commit.**

Update it when:

- **A `btop.conf` value changes on purpose** → add or amend an entry under
  Decisions with the value, the date, and *why*. A setting nobody can explain is
  a setting nobody can safely change later.
- **A decision is reversed or contradicted** → do not delete the old entry.
  Amend it to say what replaced it and what went wrong. The reverted
  symlink attempt is the most useful entry in this file precisely because it
  failed.
- **btop or Omarchy upgrades and rewrites the config** → bump the version in
  Branches, then diff non-comment lines and record only values that genuinely
  moved.
- **`.gitignore`, the branch layout, or theme resolution changes** → rewrite the
  matching section rather than appending to it.

Do **not** record: btop's reformatting (`True` → `true`), new keys sitting at
their defaults, or anything `git log` already says plainly.

To diff two revisions ignoring comments and blank lines:

```sh
git show <rev>:btop.conf | grep -vE '^\s*(#|$)' > /tmp/old.keys
grep -vE '^\s*(#|$)' btop.conf > /tmp/new.keys
diff /tmp/old.keys /tmp/new.keys
```

A `Stop` hook in `.claude/settings.local.json` (untracked) prints a reminder
when `btop.conf` or `.gitignore` is modified without this file being touched.
It is a backstop for the rule above, not a replacement for it — and it cannot
fire on a machine that has no `.claude/settings.local.json`, so do not rely on
it.
