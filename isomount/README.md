<div align="center">

# Mount / Eject ISO…

**One entry, two states.** Mount a disc image read-only — or eject it if it is already mounted.

[![Strata action](https://img.shields.io/badge/Strata-custom%20action-7aa2f7?logo=linux&logoColor=black)](../README.md)
[![Runtime](https://img.shields.io/badge/runtime-command%20%C2%B7%20per%20item-9ece6a)](#action-anatomy)
[![Python](https://img.shields.io/badge/Python-3-3776AB?logo=python&logoColor=white)](#requirements)
[![udisks2](https://img.shields.io/badge/udisks2-udisksctl-bb9af7)](#requirements)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](../LICENSE)

</div>

Right-click an `.iso` or `.img` in Strata → **Actions → Mount / Eject ISO…**. The same entry does both jobs, because it first asks the system what state the image is already in and then does the opposite. Mount it, look inside, run it again, and the loop device is gone. No disk utility, no terminal, no second menu entry to remember.

The image is attached **read-only**, so browsing an ISO can never write through to the file.

## Contents

- [What it does](#what-it-does)
- [How it works](#how-it-works)
  - [The three states](#the-three-states)
  - [Where it mounts](#where-it-mounts)
  - [Why read-only](#why-read-only)
  - [Why udisks2 and not mount(8)](#why-udisks2-and-not-mount8)
  - [Notifications](#notifications)
- [Action anatomy](#action-anatomy)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)
- [Limitations](#limitations)
- [License](#license)

## What it does

- **Toggles** the image's mount state: unmounted → mounted, mounted → ejected.
- **Attaches a loop device** for the image on first use, and **releases it** on eject, so nothing is left behind in `/dev` or in the mount table.
- **Mounts read-only** and **never prompts**: every udisks2 call passes `--no-user-interaction`.
- **Reports the outcome** as a desktop notification with the mount point, or the exact reason it failed.
- **Works on a whole selection**: `per-item` mode runs it once per image, so selecting three ISOs mounts (or ejects) all three independently.

## How it works

The helper is a single Python script with no third-party imports. It shells out to three well-known tools — `losetup` to find an existing loop device, `findmnt` to find where that device is mounted, and `udisksctl` to attach, mount, unmount and detach — and it decides what to do from what those report.

```text
iso-mount IMAGE
  │
  ├─ losetup -j IMAGE          loop device already attached?
  │    └─ yes → findmnt -no TARGET DEVICE   is it mounted?
  │
  ├─ attached + mounted        → udisksctl unmount → udisksctl loop-delete   (eject)
  ├─ attached, not mounted     → udisksctl mount                             (mount)
  └─ not attached              → udisksctl loop-setup --read-only → udisksctl mount
                                 (mount failure also deletes the loop device)
```

### The three states

| State found | What the action does | Loop device | udisks2 calls |
| --- | --- | --- | --- |
| No loop device for this file | Attaches a read-only loop device, then mounts it | new `/dev/loopN` | `loop-setup -f --read-only`, `mount` |
| Loop device attached, unmounted | Mounts the existing device | unchanged | `mount` |
| Loop device attached and mounted | Unmounts, then releases the device | removed | `unmount`, `loop-delete` |

"Eject" means exactly that pair: unmount first, then `loop-delete`. If the unmount fails — a file inside the image is still open in an editor, for example — the script stops there and reports the reason, leaving the device attached rather than half-detaching it. The `loop-delete` result itself is not checked, because a device that is busy is a state the next invocation will find and handle anyway.

### Where it mounts

The mount point is not chosen by this script. udisks2 decides, and the script reports what it chose by asking `findmnt -no TARGET /dev/loopN`. On a default desktop that is `/run/media/$USER/<label>`, with the volume label (or the device name, when the image has no label) as the directory. Because the answer is read back from the mount table rather than assumed, a distribution that mounts elsewhere still gets the right notification.

### Why read-only

`udisksctl loop-setup -f IMAGE --read-only` marks the **loop device** read-only, which is the strongest guarantee available at this layer: the image file cannot be modified through the device, whether by a file manager, an editor, or a program that ignores conventional warnings. ISO 9660 images are read-only filesystems in any case, so for the common case this costs nothing.

A writable filesystem inside an image (an `.img` holding ext4, for instance) will still ask for a read-write mount. If udisks2 refuses that request against a read-only device, the action reports udisks2's own error message rather than silently forcing a read-write mount — the refusal is the feature.

### Why udisks2 and not mount(8)

`mount(8)` for a loop image needs root or a sudoers rule, and either one turns a right-click into a privilege prompt. udisks2 already runs as a privileged daemon with polkit in front of it, and desktop distributions commonly permit loop setup and mounting for the **active local session** — which is exactly what a file-manager action runs in. The action's job is therefore to be a well-behaved client: precise arguments, `--no-user-interaction` so a policy mismatch fails loudly instead of hanging on an invisible prompt, and no parsing of anything it did not ask for.

### Notifications

Outcomes are reported with `notify-send` (`-a "Mount ISO"`, `-i media-optical`, `-u critical` on failure), and only when `WAYLAND_DISPLAY` or `DISPLAY` is set. Run from a plain terminal with neither, the helper prints to stderr and stays quiet — a small thing that makes it usable in a script without a notification storm. A missing `notify-send` never fails the action; the mount has already happened by then.

## Action anatomy

The published manifest, field by field — this is the whole integration surface with Strata:

| Field | Value | Why |
| --- | --- | --- |
| `schema_version` | `1` | The only version current Strata accepts; a newer manifest is refused with an explanation. |
| `id` | `isomount` | Must equal the action's **directory** name. Cannot change later. |
| `name` | `Mount / Eject ISO…` | The menu label. The ellipsis is honest: state, not a parameter, decides what it does. |
| `description` | mount read-only / eject if mounted | Shown as the accessible description. |
| `icon` | `disc` | A bundled Lucide slug; it also marks the job's row in the Jobs dashboard. |
| `enabled` | `true` | A single action's off switch that keeps it in Settings but out of menus. |
| `menu` | `submenu` | Groups it under **Actions** instead of lengthening the menu body. |
| `[when].kinds` | `["file"]` | Folders are never inputs; the image is a file. |
| `[when].extensions` | `["iso", "img"]` | Lowercase, no dots. Strata only offers the action when **every** selected entry matches. |
| `[when].min_items` | `1` | There is no "mount the folder you are in" case. |
| `[run].runtime` | `command` | No script beside the manifest — an installed executable is enough. |
| `[run].program` | `/usr/local/bin/iso-mount` | Absolute, because a GUI session's `PATH` is not the shell's. |
| `[run].args` | `["{path}"]` | The single input of a per-item run. A token inside a larger string would be rejected, so no quoting games. |
| `[run].mode` | `per-item` | Each image is its own decision with its own loop device; three ISOs means three toggles. |
| `[run].on_error` | `continue` | One unsupported image does not cancel the rest of the selection. |
| `[run].working_directory` | `home` | Irrelevant to the script — kept explicit so no invocation depends on where Strata was started. |
| `[run].confirm` | `false` | Both directions are reversible and non-destructive. |

## Requirements

| Dependency | Needed for | Notes |
| --- | --- | --- |
| Strata with custom actions | The menu entry | Manifests here use `schema_version = 1`. |
| `python3` | The helper | Standard library only: `os`, `re`, `subprocess`, `sys`. |
| udisks2 (`udisksctl`) | Attach, mount, unmount, detach | The privileged half. Usually already installed on a desktop system. |
| util-linux (`losetup`, `findmnt`) | Detecting the current state | Part of a base Linux install. |
| libnotify (`notify-send`) | Result popups | Optional: missing `notify-send` never fails the action. |
| A polkit agent in your session | Authorization | Failures say `Not authorized` when this is the problem. |
| Kernel support for the filesystem | Mounting | `isofs` for ISOs; `ext4`, `ntfs3`, `exfat`, `udf` and friends for `.img` files. |

## Installation

```bash
install -Dm755 iso-mount /usr/local/bin/iso-mount   # sudo if /usr/local/bin is root-owned
install -Dm644 action.toml ~/.config/strata/actions/isomount/action.toml
```

Then right-click any `.iso` or `.img` → **Actions → Mount / Eject ISO…**.

The directory name is not cosmetic: Strata reads `~/.config/strata/actions/isomount/` and refuses the manifest if its `id` is anything but `isomount`. If you install the helper somewhere else, edit `program` in `action.toml` — it must be absolute or a name that the session's executable lookup can resolve.

If the entry does not appear, look at **Settings → Actions → Problems** before anything else; a manifest that fails validation is listed there with the reason.

## Usage

```bash
# Mount an image, then run the identical command to eject it
iso-mount ~/Downloads/archlinux-x86_64.iso
findmnt /dev/loop0                      # where it landed

# Images with spaces are ordinary arguments, because no shell is involved
iso-mount ~/Downloads/'My Disc Image.iso'
```

| Exit code | Meaning |
| --- | --- |
| `0` | Mounted, or ejected. |
| `1` | Error, with the reason printed to stderr and shown in an urgent notification. |
| `130` | Interrupted with `Ctrl`+`C`. |

In Strata the exit status is what marks a job succeeded or failed; the Jobs dashboard shows the captured stderr in **Details**.

## Troubleshooting

| Symptom | Cause | What to do |
| --- | --- | --- |
| Notification: **Not authorized** / `NotAuthorized` and no mount | udisks2's polkit policy refused the request — the usual reason is that the program doing the asking is not in the active local desktop session (a file manager started over SSH or from a root shell, for example) | Start Strata from your desktop launcher. The session, not the working directory, is what polkit checks. If your distribution's policy requires admin authentication even for a local session, a polkit rule for `org.freedesktop.udisks2.loop-setup` is the correct fix — this action will never prompt for a password. |
| `wrong fs type, bad option, bad superblock` | The kernel has no driver for that filesystem, or the image holds a partition table instead of a filesystem | Install the filesystem support, or mount the partition rather than the image (see Limitations). |
| `mount failed` with the loop device already attached | A previous attempt attached the device and failed before mounting | Run the action again: the second state (attached, unmounted) just mounts it. |
| Eject fails with `target is busy` | Something still has a file open inside the image | Close it — an editor, a preview, an extracted file — then run the action again. |
| Entry missing from the menu | Manifest not validated (wrong `id` vs directory name, unknown key, a selection that does not match `[when]`) | **Settings → Actions → Problems** names the reason. Then check that every selected item is a file with an `.iso`/`.img` extension. |
| Mounts fine, no notification | No notification daemon, or no `WAYLAND_DISPLAY`/`DISPLAY` in the environment | Cosmetic only; the mount has already happened. Check with `findmnt`. |

## Limitations

- **Read-only, always.** There is no write path, no "mount read-write" variant, and no burn or remaster feature. This is a viewing tool.
- **One filesystem per image.** The loop device holds the whole image, so a **partitioned** disk image (a full-disk `.img` with a partition table) cannot be mounted directly — the kernel sees a partition table where it expected a filesystem. Convert it, or mount the partition with `losetup --partscan` plus a partition offset outside this action.
- **First filesystem only, in practice.** Multi-session or mixed-content discs expose whatever the kernel's default mount offers; there is no session or track picker.
- **No bookmarks or automation.** An ejected image leaves no record; mounts disappear from the sidebar when they go.
- **State is the mount table, nothing else.** The toggle acts on any loop device that points at the same file, which is why it works after a reboot or from another shell — and also why it will not touch an image mounted by some mechanism it cannot see (a fixed `/dev/loopN` in a systemd unit, for instance).
- **No sandbox, like every Strata action.** The script runs with your permissions; review it before enabling it, and treat disc images from unknown sources the way you would treat any untrusted file.

## License

MIT — see [LICENSE](../LICENSE).
