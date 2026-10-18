<div align="center">

# Strata custom actions

**Every layer, one menu away.** A small, hand-built collection of [Strata](https://github.com/lgse/strata) custom actions — manifests, the helpers they run, and the reasoning behind each one.

[![License: MIT](https://img.shields.io/github/license/CarloCamacho/strata-custom-actions)](LICENSE)
[![Actions](https://img.shields.io/badge/actions-2-9ece6a)](#actions)
[![Strata](https://img.shields.io/badge/host-Strata%20custom%20actions-7aa2f7?logo=linux&logoColor=black)](https://github.com/lgse/strata/blob/main/docs/custom-actions.md)
[![Platform: Linux](https://img.shields.io/badge/platform-Linux-FCC624?logo=linux&logoColor=black)](#requirements)

</div>

Strata puts your own programs in the file and folder context menus and gets out of the way: a small TOML manifest, an ordinary executable, and a Jobs dashboard that shows what happened. That is a low enough barrier that the actions worth writing are the ones you reach for several times an hour — and those are the ones worth sharing.

This repository collects the actions I use daily. Each folder is one action, complete on its own: the exact `action.toml` Strata loads, the helper it calls, and a README covering what it is for, how it works, what it needs, and what it deliberately will not do.

## Contents

- [Actions](#actions)
  - [Mount / Eject ISO…](#mount--eject-iso)
  - [Send via LocalSend…](#send-via-localsend)
- [How a folder maps to Strata](#how-a-folder-maps-to-strata)
- [Installing an action](#installing-an-action)
- [Requirements](#requirements)
- [Conventions in this repository](#conventions-in-this-repository)
- [Using the helpers outside Strata](#using-the-helpers-outside-strata)
- [Testing an action](#testing-an-action)
- [Contributing](#contributing)
- [License](#license)

## Actions

| Action | What it does | Runtime | Helpers and services |
| --- | --- | --- | --- |
| [Mount / Eject ISO…](isomount/README.md) | Mounts an `.iso`/`.img` read-only through udisks2, or ejects it if it is already mounted. One entry, toggled by the item's current state. | `command`, per item | `iso-mount` (Python, stdlib), udisks2, util-linux, libnotify |
| [Send via LocalSend…](localsend/README.md) | Sends the whole selection to a peer on your local network, picking the device in the Noctalia LocalSend panel. | `command`, whole selection | `noctalia-localsend` (Python, stdlib), Noctalia shell, LocalSend plugin, `noctalia` CLI |

### Mount / Eject ISO…

**State over configuration.** Disc images are the one file type where the interesting operation depends on what happened last time: you mount an ISO, you look inside it, you want it gone. A single entry that reads the current state and does the opposite means you never hunt for a second action or open a disk utility to unmount. It attaches the loop device read-only, so the image itself can never be modified by the trip.

Read [the full action README](isomount/README.md) for the three states, the polkit rules that make it rootless, and why a partitioned `.img` is out of scope.

### Send via LocalSend…

**One dialog instead of two file managers.** LocalSend already moves files between Linux, macOS, Windows and Android without a server; what it does not have is a context-menu entry. This action turns a selection into a single structured request, hands it to the Noctalia LocalSend panel, and lets you pick the destination there. No shell is involved at any point, so a filename with quotes, spaces or newlines is just a filename.

Read [the full action README](localsend/README.md) for the dispatch contract, the validation limits, and the one dependency that is not in this repository.

## How a folder maps to Strata

Strata reads one directory per action from `$XDG_CONFIG_HOME/strata/actions/`, and **the directory name must equal the manifest's `id`**. That constraint is why every folder here is named after the action's id rather than after its title:

```text
$XDG_CONFIG_HOME/strata/actions/
  isomount/
    action.toml          # the manifest, validated strictly on load
  localsend/
    action.toml
```

Every folder in this repository mirrors that layout. `action.toml` is loaded exactly as published — there is no templating, no placeholder substitution, and no generated config. Anything machine-specific (where a helper is installed, which panel plugin id to talk to) is either an absolute path you may edit or a constant documented in the action's README.

Helpers live outside the action directory. A `command` action does not need a script beside its manifest: `program` plus `args` is enough, and it keeps the action directory small enough to copy around. The scripts in this repository are therefore installed to a `PATH` location and referenced from the manifest.

## Installing an action

Each action installs the same way: the manifest into Strata's actions directory, the helper somewhere executable, then a right-click.

```bash
# Mount / Eject ISO…
install -Dm644 isomount/action.toml ~/.config/strata/actions/isomount/action.toml
install -Dm755 isomount/iso-mount  /usr/local/bin/iso-mount   # sudo if needed

# Send via LocalSend…
install -Dm644 localsend/action.toml ~/.config/strata/actions/localsend/action.toml
install -Dm755 localsend/noctalia-localsend /usr/local/bin/noctalia-localsend
```

Strata validates a manifest when it loads actions. A definition that cannot load is listed under **Settings → Actions → Problems** with the reason and never silently disappears from the menus — so if an action is missing, check there first rather than assuming the copy failed.

The published manifests point at `/usr/local/bin/`. If you install a helper elsewhere, edit `program` in `action.toml` to the absolute path. A bare program name is legal and resolved through the executable lookup, but GUI sessions frequently start with a `PATH` that does not include `~/.local/bin`, so an absolute path is the safer default for anything a file manager launches.

## Requirements

- **Strata** on Linux — the custom-actions system documented in [Strata's guide](https://github.com/lgse/strata/blob/main/docs/custom-actions.md). Manifests here use `schema_version = 1`.
- **A graphical session with a polkit agent** for anything that talks to udisks2. Actions run with your permissions; there is no sandbox.
- **Per-action dependencies** are listed in each README and are not installed by Strata. A missing interpreter or program leaves the action visible but disabled, with the reason in its accessible description.

## Conventions in this repository

- **The manifest is the source of truth.** `action.toml` files are published as loaded. If a value is machine-specific, it is documented instead of hidden.
- **No shell, ever.** Helpers receive argv entries directly from Strata, and every path is absolute, so a filename can never be read as a command or as an option such as `-rf`. Nothing here re-parses a command line.
- **Helpers are ordinary CLIs.** Both scripts work by hand from a terminal, which is how they are tested; the action adds a menu entry, not a code path.
- **Failures are explained, not swallowed.** Each helper prints one line to stderr and exits non-zero, which is what Strata shows in the Jobs dashboard.
- **Read-only where it matters.** Actions that can plausibly damage data — mounting images, mostly — are written so the destructive direction is the one that does not exist.

## Using the helpers outside Strata

Neither script needs Strata or its environment:

```bash
iso-mount ~/Downloads/archlinux-x86_64.iso      # mount, then run again to eject
noctalia-localsend "$PWD"                       # hand the current folder to the panel
```

That matters for scripting and for testing: if a helper misbehaves, reproduce it from a terminal before blaming the manifest. The one difference is polkit — a helper started outside the active local desktop session is refused by udisks2 (`Not authorized`), which is a property of the session, not of the script.

## Testing an action

Enable the action, right-click a file that satisfies `[when]`, and choose it. The Jobs dashboard shows the invocation, its captured output, and its exit status; failures appear in **Details**, not in a banner. Cancel is a real signal to the whole process group, and it stops work that has not happened yet — it does not roll back changes already written to disk.

## Contributing

Issues and pull requests are welcome. Keep an action self-contained: manifest, helper, README, no local paths, no state that lives outside `$XDG_RUNTIME_DIR` or the action's own scratch directory. New actions should explain their failure modes and their scope the same way the existing READMEs do.

## License

MIT. Each helper keeps its own module docstring describing the same contract; see [LICENSE](LICENSE).
