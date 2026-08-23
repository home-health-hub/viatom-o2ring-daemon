# viatom-o2ring-daemon

![viatom-o2ring-daemon: pulse oximeter ring data over Bluetooth to a local home server and database](docs/images/viatom-o2ring-daemon-banner.png)

![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-3776AB?logo=python&logoColor=white) ![Bash](https://img.shields.io/badge/shell-Bash-4EAA25?logo=gnu-bash&logoColor=white) ![Bluetooth LE](https://img.shields.io/badge/Bluetooth-LE-0082FC?logo=bluetooth&logoColor=white)

[![License: GPL-3.0](https://img.shields.io/badge/license-GPL--3.0-blue)](https://github.com/home-health-hub/viatom-o2ring-daemon/blob/main/LICENSE) [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/home-health-hub/viatom-o2ring-daemon#contributing) [![Discussions](https://img.shields.io/badge/discussions-welcome-blue)](https://github.com/home-health-hub/viatom-o2ring-daemon/discussions)

A standalone Linux daemon that connects to a Viatom/Wellue O2Ring (and
related ring-family) pulse oximeter over Bluetooth Low Energy (BLE), logs
live SpO2/pulse readings to a local SQLite database, and downloads stored
overnight/session recordings from the ring's onboard memory. No cloud
account, no companion app required.

It's a thin wrapper around the
[`viatom-o2ring-ble`](https://github.com/home-health-hub/viatom-o2ring-ble)
library, packaged to run unattended as a `systemd` service on something like
a Raspberry Pi sitting near the ring.

**Disclaimer: This is an unofficial, community-developed project. It is not
affiliated with, officially maintained by, or in any way officially
connected with Viatom Technology Co., Ltd. or Wellue. Nothing here is
medical advice; the SpO2 category labels and alerting are informational
only. Talk to a doctor about your oxygen saturation readings.**

**Work in progress -- not yet verified against real hardware.** This daemon
is built on `viatom-o2ring-ble`. Its legacy-protocol support (the default,
`monitor.protocol = legacy`) is protocol-correct-on-paper only as of this
writing (cross-checked against five independent sources but not yet
tested against an actual O2Ring-family device). Its O2Ring-S support
(`monitor.protocol = oxyii`) is ported from a source that *has* verified
its own implementation against real hardware -- a meaningfully stronger
starting point -- but the port itself, as wired into this daemon, hasn't
been independently re-verified here either. See `viatom-o2ring-ble`'s
`CLAUDE.md` and README for details on both.

## Supported devices

Two device families, selected via `monitor.protocol` in the config file
(see [Device protocol](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Device-Protocol)):

- `protocol = legacy` (the default): O2Ring, KidsO2, RingO2, and O2 Max --
  whatever `viatom-o2ring-ble`'s `O2RingClient` supports.
- `protocol = oxyii`: the O2Ring-S (T8520), which speaks a completely
  different BLE protocol -- via `viatom-o2ring-ble`'s `OxyIIClient`.

## Device protocol

```ini
[monitor]
protocol = legacy   # or oxyii
```

Everything downstream of live-reading capture -- storage, reports,
alerting, the HTTP API, pruning -- is protocol-agnostic; only live
streaming and stored-session sync actually branch on `monitor.protocol`. A
daemon instance targets exactly one device (`monitor.address`), so this is
a single fixed choice per config file, not auto-detected. See
[Device Protocol](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Device-Protocol)
on the wiki for the full internals of how each protocol's live readings and
stored sessions get adapted into the same database tables.

## Features

- Scans for the device on first run, then pins its BLE address into the
  config file so future restarts connect directly instead of re-scanning
- Streams live readings (SpO2, pulse, battery, perfusion index, worn/
  calibrating state) to a local SQLite database
- Downloads stored `.vld` session files (e.g. overnight sleep recordings)
  from the ring's onboard memory that aren't already in the database, either
  automatically after each streaming session or on its own schedule
- Runs as a `systemd` service with automatic restart on failure
- Optional PDF/CSV reports: an SpO2-category time-distribution pie chart,
  SpO2/pulse trend charts, a reading table shaded by SpO2 category (choice
  of full, compact, or weekly/monthly rollup layouts), and a summary table
  of downloaded sessions
- Optional Apprise-based alerting on stale data, a low-SpO2 reading, or low
  battery
- Optional read-only HTTP API and MQTT publishing
- One optional `[profile]` section for the ring's wearer (report
  personalization, alert overrides) -- a ring has exactly one wearer at a
  time, so unlike a shared blood-pressure cuff or scale, there's no
  multi-person "who was this?" tagging to solve
- Supports two device families/protocols (`monitor.protocol = legacy` or
  `oxyii`) -- see [Device protocol](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Device-Protocol)

## Installation

Requires Python 3.11+.

### Quick install

```bash
git clone https://github.com/home-health-hub/viatom-o2ring-daemon.git
cd viatom-o2ring-daemon
sudo ./install.sh
```

This creates a venv at `/opt/viatom-o2ring-daemon`, installs the package
from the checkout, seeds `/etc/viatom-o2ring-daemon/config.ini` (if it
doesn't already exist), creates a `viatom-o2ring-daemon` system user, and
installs and enables the systemd service. It also installs (but does not
enable) the
[stored-session file sync](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Stored-Session-Files),
[scheduled report generation](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Scheduled-Reports),
and [alerting](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Alerting)
timer units, and the [HTTP API](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/HTTP-API)
service. It's safe to re-run: it skips steps that are already done. Edit
the config and `sudo systemctl restart viatom-o2ring-daemon` afterward.

`config.ini` can hold real secrets (API tokens, `apprise_urls` with embedded
credentials), so `install.sh` sets it to mode `600`, owned by the
`viatom-o2ring-daemon` user, every time it runs (including on re-runs, in
case it was ever loosened). Running the CLI tools by hand afterward needs
`sudo -u viatom-o2ring-daemon`, e.g.:

```bash
sudo -u viatom-o2ring-daemon viatom-o2ring-report --config /etc/viatom-o2ring-daemon/config.ini
```

### Advanced installation and configuration

For a manual (non-`install.sh`) install, or to enable the optional
stored-session sync, scheduled reports, alerting, HTTP API, or per-wearer
profile, see the wiki:
[Installation](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Installation),
[Stored Session Files](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Stored-Session-Files),
[Scheduled Reports](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Scheduled-Reports),
[Alerting](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Alerting),
[HTTP API](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/HTTP-API),
and [Profile](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Profile).

## Manual usage

### On-demand capture instead of a long-running service

```bash
viatom-o2ring-daemon --config /etc/viatom-o2ring-daemon/config.ini --once --once-timeout 60
```

Connects, waits up to `--once-timeout` seconds for a single reading, records
it, and exits. Exit code is `1` if no reading arrived in time. For when
you'd rather not run the daemon continuously.

## Database schema

Three tables: `live_readings` (one row per streamed reading), `sessions`
(one row per downloaded `.vld` file), and `session_records` (one row per
sample within a session). See
[Database Schema](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Database-Schema)
on the wiki for the full column reference.

## Reports

```bash
viatom-o2ring-report --config /etc/viatom-o2ring-daemon/config.ini
```

Generates a PDF with SpO2/pulse trend charts, a category-shaded reading
table, and a downloaded-sessions summary -- handy to bring to a doctor's
appointment. Supports date-range filtering, CSV export, and exporting one
session's raw per-sample records. See
[Generating Reports](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Generating-Reports)
on the wiki for the full CLI reference and rendered sample output.

## Pruning old data

```bash
viatom-o2ring-prune --config /etc/viatom-o2ring-daemon/config.ini --older-than 365 --yes
```

Deletes `live_readings` and `sessions` rows (with cascaded
`session_records`) older than the given number of days. See
[Pruning](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Pruning)
on the wiki for the dry-run flag and details.

## MQTT

```ini
[mqtt]
enabled = yes
host = mqtt.example.com
```

Publishes each live reading as JSON to MQTT. See
[MQTT](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/MQTT)
on the wiki for the topic format and payload details.

## Troubleshooting

Common issues: the device not being discovered, no Bluetooth scanner
available, wrong readings on the legacy protocol, and O2Ring-S sessions not
appearing until their trailer flushes. See
[Troubleshooting](https://github.com/home-health-hub/viatom-o2ring-daemon/wiki/Troubleshooting)
on the wiki for fixes, or run `--check-config` for a section-by-section
config report.

## Acknowledgments

- Built on
  [`viatom-o2ring-ble`](https://github.com/home-health-hub/viatom-o2ring-ble),
  which cross-checks five independent community/official sources for its
  legacy-protocol decoding, and separately ports its O2Ring-S (OxyII)
  support from
  [nglessner/o2ring-s-protocol](https://github.com/nglessner/o2ring-s-protocol)
  -- see that repo's README and `CLAUDE.md`.
- Project layout modeled on
  [`etekcity-scale-daemon`](https://github.com/home-health-hub/etekcity-scale-daemon)
  and [`etekcity-bp-daemon`](https://github.com/home-health-hub/etekcity-bp-daemon).
- Code review, implementation, and documentation assisted by
  [Claude](https://www.anthropic.com/claude).

## Contributing

Contributions are welcome!

- **Bug reports**: [Open an issue](https://github.com/home-health-hub/viatom-o2ring-daemon/issues).
- **Everything else** (questions, feature requests, ideas, general discussion): [Use Discussions](https://github.com/home-health-hub/viatom-o2ring-daemon/discussions).
- Pull requests are welcome for bug fixes or discussed features.

## License

This project is licensed under the **GNU General Public License v3.0**.

See [LICENSE](LICENSE) for more information.
