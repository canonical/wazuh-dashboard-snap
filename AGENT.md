# Copilot instructions for wazuh-dashboard-snap

## What this repo is

This repo does **not** contain Wazuh Dashboard's application source — it packages the
upstream `wazuh-dashboard` apt package (from `packages.wazuh.com`) into a strictly-confined
snap. All "build" logic lives in `snap/snapcraft.yaml` (parts, apps, hooks, environment) plus
a small set of bash scripts. See `FORK.md` for the upstream/fork relationship and branch
strategy — read it before touching branch history or the packaging pivot it describes.

## Build, test, run

- **Build the snap:** `snapcraft pack --debug` (produces `wazuh-dashboard_<version>_amd64.snap`).
- **Install locally:** `sudo snap install ./wazuh-dashboard_<version>_amd64.snap --dangerous --jailmode`.
- **CI build/test** (`.github/workflows/ci.yaml`, runs on every PR): builds the snap via
  `snapcore/action-build`, then spins up a real `wazuh-indexer` snap + installs the built
  `wazuh-dashboard` snap and drives it with `curl` against `localhost:5601` (login,
  auth-rejection, log presence) and `localhost:9684/metrics` (Prometheus exporter). There is
  no unit test framework — all "tests" are these end-to-end shell steps in `ci.yaml`. To
  exercise a single check, run the corresponding step's commands manually against a running
  installed snap rather than trying to isolate a step in CI.
- **No linter is configured** in this repo (no shellcheck/yamllint step in CI).
- **Live debugging** (from `CONTRIBUTOR.md`):
  ```
  sudo sysctl -w kernel.printk_ratelimit=0 ; journalctl --follow | grep wazuh-dashboard
  snappy-debug scanlog --only-snap=wazuh-dashboard
  ```

## Architecture

- **`snap/snapcraft.yaml`** is the source of truth for how the snap is assembled. Key parts:
  - `wazuh-dashboard` part: pins the exact apt package version (`wazuh-dashboard=4.11.0-1` —
    keep this in sync with the top-level `version:` field), then patches the shipped
    `opensearch_dashboards.yml` (comments out all defaults, appends Wazuh-specific opensearch
    connection settings) and rewrites hardcoded plugin constants
    (`WAZUH_DATA_PLUGIN_PLATFORM_BASE_ABSOLUTE_PATH`) via `sed` in `override-prime` to point at
    `/var/snap/wazuh-dashboard/current/config` instead of the package's baked-in path.
  - `wrapper-scripts` / `helper-scripts` parts stage `scripts/wrappers/*` and `helpers/*.sh`
    into `opt/opensearch-dashboards/` inside the snap — these are sourced at runtime by the
    hooks and daemon wrappers, not run at build time.
  - `dependencies` part stages the Prometheus exporter apt package (from a separate PPA) and
    `yq`, used for YAML manipulation in helper scripts.
  - Two daemons: `opensearch-dashboards-daemon` (main app) and `exporter-daemon` (Prometheus
    exporter), the latter declared `after: [opensearch-dashboards-daemon]`.
- **Environment variable naming is intentionally legacy**: most env vars still say
  `OPENSEARCH_DASHBOARDS_*` even though the product is `wazuh-dashboard`, because this repo
  is a fork of `canonical/opensearch-dashboards-snap` and most of the plumbing is unchanged.
  New Wazuh-specific variables (e.g. `WAZUH_CONFIG_PATH`) coexist with the legacy ones —
  don't rename existing `OPENSEARCH_DASHBOARDS_*` vars without checking every hook/script
  that reads them (`snap/hooks/install`, `scripts/wrappers/*.sh`, `helpers/*.sh`).
- **Hooks vs wrapper scripts**: `snap/hooks/install` runs once at install time (creates
  directories, sets ownership, writes initial config via `set_yaml_prop`). `snap/hooks/configure`
  runs on every `snap set` and only manages the `scheme` option (http/https, read by the
  exporter). `scripts/wrappers/start.sh` / `start-exporter.sh` are the actual daemon
  entrypoints (`command:` in `snapcraft.yaml`), executed via `setpriv` to drop to `snap_daemon`.
- **`helpers/*.sh`** are generic bash utilities for manipulating the snap's YAML config and
  file permissions (`get-conf.sh`, `set-conf.sh`, `io.sh`), all built around `yq`. Config keys
  containing `.` are addressed with a `/`-separated key path (see `set_yaml_prop`), since `.`
  is yq's own path separator.
- **Runtime config lives outside the snap's read-only squashfs**: everything under
  `SNAP_DATA_CURRENT` (`/var/snap/wazuh-dashboard/current`) is read-write and set up by the
  install hook from the read-only `${SNAP}/etc/` template.

## Branching and release conventions

- `main` is the active development branch; pushing to it triggers `release.yaml`, which
  builds via `ci.yaml` then publishes to the Snap Store on a channel derived from the
  `version:` field in `snapcraft.yaml`: `"${version%.*}/edge"` (e.g. version `4.11.0` →
  `4.11/edge`, **not** `4/edge` — this differs from typical snap track conventions, see the
  comment in `release.yaml`).
- Track/channel branches (e.g. `2/edge`) are maintained as a small, squashed set of
  Wazuh-specific commits rebased on top of upstream (`canonical/opensearch-dashboards-snap`)
  — see `FORK.md` for the exact workflow (`git rebase --onto`, `git rerere`). Don't let these
  branches accumulate an unsquashed patch history again.
- Bumping the shipped Wazuh version requires updating **both** the top-level `version:` and
  the `stage-packages`/`override-stage` apt package pin in the `wazuh-dashboard` part of
  `snap/snapcraft.yaml` — they must match.
