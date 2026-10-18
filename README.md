# EndCleaner

![Version](https://img.shields.io/badge/version-1.0.0-orange)
![Endstone](https://img.shields.io/badge/endstone-0.11.11%2B-blue)
![Python](https://img.shields.io/badge/python-3.14%2B-blue)
![Minecraft](https://img.shields.io/badge/minecraft-26.51%2B-green)
![License](https://img.shields.io/badge/license-GPL--3.0-lightgrey)

A scheduled entity cleanup plugin for [Endstone](https://github.com/EndstoneMC/endstone) Minecraft Bedrock servers.
It sends warnings before each cleanup, protects valuable items with a whitelist, and provides an in-game admin menu.

## Features

- **Automatic scheduled cleanup**, with an optional fixed time for the first run. The schedule is kept after a restart.
- **Anti time-drift**: the countdown uses a monotonic clock, so changing the system time does not shift the schedule. Each cleanup is anchored to its planned time, so the schedule does not drift over time.
- **Warnings before cleanup** at configurable second marks, with an optional sound.
- **Silent mode**: skips a cleanup when there is nothing to clear, without sending any message.
- **Manual cleanup** with confirmation, and **dry-run scan** that deletes nothing.
- **Whitelist** of valuable items. The default list includes diamonds, netherite, shulker boxes, totems, elytra, and more.
- **Item protection** within a radius around players.
- **Chunked work across ticks** to reduce lag.
- **Cumulative statistics** and an **in-game admin menu**.

## Requirements

| Component | Version |
|---|---|
| Minecraft Bedrock Edition | `>= 26.51` |
| Endstone | `>= 0.11.11` (use the latest release that matches your Bedrock version) |
| Python | `>= 3.14` |

Each Endstone release targets a specific Bedrock version. For example, `0.11.11` and `0.11.12` target Bedrock `26.51`,
and `0.11.13` targets Bedrock `26.52`. Pick the Endstone release that matches the version your server is running.

## Installation

First, install Endstone `>= 0.11.11` and Python `>= 3.14` on your server. Then use one of the following methods.

### Method 1: Install with pip

```bash
pip install endstone_endcleaner-1.0.0-py3-none-any.whl
```

### Method 2: Drop into the plugins folder

Copy `endstone_endcleaner-1.0.0-py3-none-any.whl` into the server's `plugins/` folder.

### After installing

Restart the server. When the plugin loads successfully, the console prints `v1.0.0 loaded`.

Configuration data is created automatically in `plugins/endcleaner/` on the first run.

## Usage

### Commands

| Command | Description |
|---|---|
| `/endcleaner` or `/endcleaner help` | List commands |
| `/endcleaner menu` | Open the admin menu (players with OP only) |
| `/endcleaner clean` | Clean up now, with confirmation and a warning |
| `/endcleaner scan` | Count trash, **without** deleting |
| `/endcleaner stats` | Show statistics |
| `/endcleaner reload` | Reload config, whitelist, and statistics from disk |

Only the **console** and **players with OP** (`/op <name>`) can use the commands and the menu. The plugin does not use tags or custom permissions.

### Admin menu

Open it with `/endcleaner menu` (OP players only).

```
┌──────────────────────────────┐
│       EndCleaner Admin       │
├──────────────────────────────┤
│  [ Whitelist (n) ]           │  --> Add held item, add by ID, remove items
│  [ Trash types to clear ]    │  --> Switch mode, add or remove actor types
│  [ Settings ]                │  --> Auto-clearing, clear settings, messages
│  [ Statistics ]              │  --> View and reset statistics
│  [ Count trash ]             │  --> Count trash, no deletion
│  [ Clear trash now ]         │  --> Clean immediately, with confirmation
│  [ Reset ]                   │  --> Four reset options, with confirmation
└──────────────────────────────┘
```

The main menu contains:

- **Whitelist (n):** add the item you are holding, add an item by ID, or remove an item from the list.
- **Trash types to clear:** view the trash types being cleaned. Switch between list mode and items-only mode, and add or remove actor types.
- **Settings:** three groups:
  - *Auto-clearing:* turn automatic cleanup on or off, set the first clear time, and toggle silent mode when there is nothing to clear.
  - *Clear settings:* clear interval, warning marks, actors and entities processed per tick, item protection radius, and minimum trash threshold.
  - *Messages & interface:* the three messages (warning, result, nothing to clear), chat prefix, UTC offset, and warning sound on or off and its sound ID.
- **Statistics:** view statistics, or reset them to 0 (with confirmation).
- **Count trash (no deletion):** count trash without deleting it.
- **Clear trash now:** clean up immediately (with confirmation).
- **Reset:** four options, each with a confirmation step:
  - Reset the whitelist to defaults.
  - Clear the entire whitelist.
  - Reset trash types to defaults.
  - Restore all settings, including the whitelist and statistics.

## Configuration

Most settings can be changed in the **Settings** menu. You can also edit `config.json` directly while the server is stopped,
or run `/endcleaner reload` after editing it.

| Key | Description |
|---|---|
| `interval_minutes` | Time between two automatic cleanups, in minutes |
| `warning_marks` | Warning times before cleanup, in seconds, e.g. `[15, 5]` |
| `message_cleanup` | Warning message. Use `{seconds}` |
| `message_done` | Result message. Use `{items}`, `{entities}`, `{count}` |
| `message_empty` | Message shown when there is nothing to clear |
| `prefix_format` | Prefix for all plugin messages |
| `yield_batch_size` | Maximum actors checked per tick (helps prevent lag) |
| `remove_per_tick` | Maximum entities removed per tick |
| `protect_radius` | Radius (blocks, max 64) around players where items are protected. `0` = off |
| `warn_sound` / `warn_sound_id` | Enable and choose the warning sound |
| `dimensions` | Dimensions to clean, e.g. `overworld`, `nether`, `the_end` |
| `trash_types` | Actor types counted as trash |
| `clean_mode` | `default` (clean by `trash_types`) or `items` (clean dropped items only) |
| `auto_enabled` | Enable or disable automatic cleanup |
| `silent_when_empty` | When on, no warning is sent if there is nothing to clear |
| `full_cycle_when_empty` | When there is nothing to clear, still run the full warning cycle and report 0 results |
| `min_trash_to_clean` | Skip an automatic cleanup if there is less trash than this |
| `first_clear_time` | First clear time after auto is enabled, in `H:MM` format. Leave empty to use the normal interval |
| `utc_offset_minutes` | Time zone offset from UTC in minutes, e.g. `420` for UTC+7 |

Invalid types or values are reset to defaults when the config loads. The broken original is saved to
`config.corrupt.json` so you can restore it manually.

### Yield-based load distribution

The plugin does not clean the whole server in one tick. The scan is split into several steps. Each step runs within one server tick (20 ticks per second) and then yields control back to the server.

- `yield_batch_size` is the maximum number of actors checked in one tick, whether or not they are trash.
- `remove_per_tick` is the maximum number of entities removed in one tick. Removal costs more than checking, so it has its own limit.
- The next tick resumes where the previous one stopped, so actors that were already checked are not checked again.
- With the defaults (`yield_batch_size = 1000`, `remove_per_tick = 20`), the maximum removal rate is 20 entities per tick, which is 400 entities per second.

You can raise or lower these two values to suit your server.

## Data

All data is stored as JSON files in `plugins/endcleaner/`:

| File | Contents |
|---|---|
| `config.json` | Configuration |
| `config.json.bak` | Last known-good configuration, used to recover if `config.json` is corrupted |
| `config.corrupt.json` | Copy of a corrupted `config.json`, kept for manual recovery |
| `whitelist.json` | Items that are never cleaned |
| `whitelist.corrupt.json` | Copy of a corrupted whitelist, kept for manual recovery |
| `stats.json` | Cumulative statistics |
| `state.json` | Next scheduled cleanup time, kept across restarts |

## Acknowledgements

Thanks to the [Endstone](https://github.com/EndstoneMC/endstone) team for building the plugin API for Bedrock Dedicated Server.

## Author

[@dyan1619](https://github.com/dyan1619)

## License

Released under the [GNU GPL v3.0](LICENSE).
