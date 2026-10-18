<div align="center">

# Send via LocalSend…

**Right-click, pick a device, done.** Send the selected files or folders to any LocalSend peer on your network, without opening a second file manager or dragging anything anywhere.

[![Strata action](https://img.shields.io/badge/Strata-custom%20action-7aa2f7?logo=linux&logoColor=black)](../README.md)
[![Runtime](https://img.shields.io/badge/runtime-command%20%C2%B7%20whole%20selection-9ece6a)](#action-anatomy)
[![Python](https://img.shields.io/badge/Python-3-3776AB?logo=python&logoColor=white)](#requirements)
[![Noctalia](https://img.shields.io/badge/host-Noctalia%20shell-bb9af7)](https://github.com/CarloCamacho/community-plugins)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](../LICENSE)

</div>

LocalSend already moves files between Linux, macOS, Windows and Android with no server and no account; what it does not ship is a context-menu entry. This action is that entry: it takes the whole selection, validates it, and hands it to the Noctalia LocalSend panel as **one** structured request. You pick the destination in the panel, the panel does the transfer.

Two properties are worth stating up front, because they are the whole reason the file-manager half is a separate, small program:

- **No shell is involved, at any hop.** Strata passes paths as argv entries, the helper passes them as a JSON array, and the plugin receives them as data. A filename containing a quote, a space, a newline or `$(rm -rf ~)` is just a filename.
- **Nothing is sent at dispatch time.** The request opens a picker. Until you choose a device and confirm, no bytes leave the machine, and cancelling costs nothing.

## Contents

- [What it does](#what-it-does)
- [How it works](#how-it-works)
  - [The chain](#the-chain)
  - [The request contract](#the-request-contract)
  - [Validation, and why it lives here](#validation-and-why-it-lives-here)
  - [One request, not N](#one-request-not-n)
- [Action anatomy](#action-anatomy)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Limits and expectations](#limits-and-expectations)
- [Troubleshooting](#troubleshooting)
- [Status](#status)
- [License](#license)

## What it does

- Adds **Send via LocalSend…** to the **Actions** submenu for any file or folder selection.
- **Sends the whole selection as one request**, so a ten-file send produces one panel, one device choice, one transfer.
- **Validates before dispatching**: every path must be absolute, resolvable, a regular file or a directory, and readable.
- **Brings the panel forward** after dispatching, so the picker does not open behind the file manager.
- **Fails with one line on stderr and a non-zero exit**, which Strata shows in the Jobs dashboard.

## How it works

### The chain

```text
Strata: right-click → Actions → Send via LocalSend…
  │
  └─ noctalia-localsend {paths}                        ← this folder: validate + dispatch
       │
       ├─ noctalia msg plugin carlocamacho/localsend:service \
       │     all localsend_send '{"v":1,"requestId":…,"paths":[…]}'
       │        └─ plugin service: opens the send panel with that request
       │
       └─ noctalia msg panel-open carlocamacho/localsend:send
                 └─ panel: device picker → transfer (LocalSend v2.1)
```

The helper in this folder is deliberately the **dumb half**: it knows about paths, limits and one IPC call. It knows nothing about discovery, certificates, PINs or the wire protocol — that is the plugin's engine, which is where it belongs, because a device picker that does not block the file manager belongs in a panel and not in a subprocess.

### The request contract

The helper builds this payload and passes it as the single argument of one `noctalia msg` call:

| Field | Value | Meaning |
| --- | --- | --- |
| `v` | `1` | Payload version, so the plugin can reject a future shape instead of misreading it. |
| `requestId` | a UUID4 | Identifies this dispatch. The panel can de-duplicate or ignore a repeated id. |
| `paths` | array of absolute paths | The validated selection, in the order Strata supplied it. |
| `origin` | `"file_manager"` | The caller's identity, so the panel can word itself for a context-menu send rather than a drag-and-drop. |

It is `json.dumps`-encoded and passed as a single argv entry — no quoting, no `eval`, no environment variables, and nothing written to disk. Note that `noctalia msg` delivers the payload to the plugin as a **string**, so the plugin decodes it before reading fields; that is the plugin's contract to honour, and the reason the payload carries its own version.

### Validation, and why it lives here

The helper rejects the request before IPC if any path:

| Check | Limit | Refusal |
| --- | --- | --- |
| Absolute | must satisfy `os.path.abspath(p) == p` | `path is not absolute: …` |
| Length | ≤ 4096 characters | `path too long` |
| Existence | must exist | `not found: …` |
| Type | regular file or directory only | `not a file or folder: …` |
| Readability | `os.access(…, R_OK)` | `not readable: …` |
| Count | ≤ 32 paths | `too many paths (max 32)` |

Doing this in the helper rather than in the panel means a doomed request fails in the Jobs dashboard, with the offending path named, instead of opening a picker that cannot succeed. It is also the cheapest place to notice a bug in the caller: Strata always passes absolute paths, so `path is not absolute` means something is calling the helper the wrong way.

### One request, not N

Strata's `whole-selection` mode invokes a `command` action **once** with `{paths}` expanded to every selected path as separate argv entries — so multi-select is naturally a single request and needs no coalescing. This is worth knowing if you have used the same idea in a file manager that substitutes `%f` per item: there, each click spawns its own process and the helpers have to merge concurrent invocations with a queue directory and a lock, or you get one dialog per file. Here that entire class of bug does not exist.

## Action anatomy

| Field | Value | Why |
| --- | --- | --- |
| `schema_version` | `1` | The only version current Strata accepts. |
| `id` | `localsend` | Must equal the action's directory name. |
| `name` | `Send via LocalSend…` | The ellipsis is honest: a picker follows. |
| `description` | send selection to a LAN device | The accessible description. |
| `icon` | `send` | A bundled Lucide slug, also used on the job row. |
| `enabled` | `true` | Keeps it out of menus without deleting it. |
| `menu` | `submenu` | Under **Actions**, next to the other custom actions. |
| `[when].min_items` | `1` | No `kinds` or `extensions` filter: LocalSend accepts files and folders alike, so any non-empty selection matches. |
| `[run].runtime` | `command` | The bridge is an installed program; there is no script in the action directory. |
| `[run].program` | `/usr/local/bin/noctalia-localsend` | Absolute, because a GUI session's `PATH` often omits `~/.local/bin`. |
| `[run].args` | `["{paths}"]` | Whole-selection token: every path as its own argv entry. `{path}` would be rejected here, since it is a per-item token. |
| `[run].mode` | `whole-selection` | One request, one panel, one transfer for the entire selection. |
| `[run].working_directory` | `home` | The helper resolves absolute paths; the working directory must not matter, and is set explicitly to prove it. |
| `[run].confirm` | `false` | Dispatching opens a picker; there is nothing to confirm before that. |

## Requirements

| Dependency | Needed for | Notes |
| --- | --- | --- |
| Strata with custom actions | The menu entry | `schema_version = 1`. |
| `python3` | The bridge | Standard library only: `json`, `os`, `subprocess`, `sys`, `uuid`. |
| The `noctalia` CLI on `PATH` | IPC to the panel | Ships with the Noctalia shell. |
| The Noctalia **LocalSend plugin** (`carlocamacho/localsend`) | The panel, the picker and the transfer engine | Provides the `:service` entry the request is dispatched to and the `:send` panel that is opened. |
| A LocalSend peer on the network | Something to send to | Same discovery rules as the official app; UDP/TCP **53317**. |

Without Noctalia, this half cannot do anything on its own — it is a bridge by design. If you already run a LocalSend command-line sender, point `program` at it instead and keep `args = ["{paths}"]`; the manifest is the only thing that has to change.

## Installation

```bash
install -Dm755 noctalia-localsend /usr/local/bin/noctalia-localsend   # sudo if needed
install -Dm644 action.toml ~/.config/strata/actions/localsend/action.toml
```

Then enable the LocalSend plugin in Noctalia (it must be present in the shell's plugin directory and enabled, since a disabled plugin has no `:service` to receive the request), and right-click a file → **Actions → Send via LocalSend…**.

If the helper is installed somewhere else, edit `program` in `action.toml`; if the plugin's ids differ in your build, edit `PLUGIN_ENTRY` and `PANEL_ID` at the top of the helper — they are constants precisely so this is a one-line change.

## Usage

```bash
# From a terminal: dispatch the current directory to the panel
noctalia-localsend "$PWD"

# Several paths, including ones with awkward names
noctalia-localsend ~/Pictures/2026 ~/Downloads/report.pdf
```

| Exit code | Meaning |
| --- | --- |
| `0` | Request dispatched (panel open is best-effort and does not affect the status). |
| `1` | Validation or IPC error, with the reason on stderr. |

A `0` means the **request** was delivered, not that anything was transferred — the transfer, its progress and its result belong to the panel and the plugin's engine.

## Limits and expectations

- **32 paths per request**, 4096 characters per path. Beyond that, the helper refuses with a clear message rather than truncating.
- **Native local paths only.** Strata excludes Trash, GVfs and other non-native locations from custom actions, so remote shares and mounted cloud trees are not inputs here.
- **Folders are recursive.** The engine expands a folder into `folder/sub/file` relative names, which is what the official app does, so the receiver recreates the tree rather than flattening it.
- **The picker is not optional** in this flow. There is no "always send to my laptop" shortcut; that would need an option the Strata manifest cannot express (no parameter form, no per-action settings).
- **Trust is LocalSend's.** Device discovery, encryption and any PIN prompt follow the protocol and the peers' settings; nothing here changes or bypasses them. The first send to a new device may need accepting on the receiver.
- **No state, no cleanup.** The helper writes nothing to disk and leaves no queue behind; a cancelled panel leaves nothing to tidy up.

## Troubleshooting

| Symptom | Cause | What to do |
| --- | --- | --- |
| The panel does not open, exit code `0` | The request was dispatched but `panel-open` failed — it is best-effort, and the two calls are independent | Open the panel manually (`noctalia msg panel-open carlocamacho/localsend:send`) or fix the panel id. The request is already in the queue. |
| `dispatch failed: …` | The `noctalia` CLI is missing, the plugin id is wrong, or the plugin's service is not running | `noctalia msg plugin carlocamacho/localsend:service all localsend_send '{"v":1,"requestId":"test","paths":["/tmp"],"origin":"file_manager"}'` reproduces it in one line. |
| `cannot run 'noctalia'` | The CLI is not on the `PATH` of the process Strata launched | Install it, or use an absolute `program` path in `action.toml`. |
| `path is not absolute: …` | A caller passed a relative path | Strata never does; if you are testing by hand, use `"$PWD/file"` rather than `./file`. |
| `too many paths (max 32)` | Selection too large | Send in smaller batches, or raise `MAX_PATHS` deliberately and check the panel copes. |
| Entry missing from the menu | Manifest not validated, or the selection was empty | **Settings → Actions → Problems** names the reason; the id must equal the directory name. |
| Panel opens but finds no devices | Discovery, not this action | Check that both devices are on the same subnet and that **53317** is reachable; that is the engine's and the network's problem, not the bridge's. |

## Status

The panel plugin and its transfer engine (`carlocamacho/localsend`) are distributed separately from this file-manager half and do not yet have a public repository of their own; the plugin id and payload shape documented above are what it implements. Everything in **this** folder is complete and versioned independently: a manifest, one 90-line bridge, and the contract it speaks.

## License

MIT — see [LICENSE](../LICENSE).
