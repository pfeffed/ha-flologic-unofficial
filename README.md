# FloLogic (Unofficial) for Home Assistant

Home Assistant integration for [FloLogic](https://flologic.com/) leak-detection
water shutoff valves, built on
[`pyflologic-unofficial`](https://github.com/pfeffed/pyflologic-unofficial).

FloLogic now publishes an official Web API, and
[`ha-flologic`](https://github.com/pfeffed/ha-flologic) is the integration
built on it. This one reaches much more of the device — every setting the app
exposes, battery and signal diagnostics, the notification log, push updates
within a second — because it speaks the mobile app's own protocol,
reverse-engineered from its traffic. It is not affiliated with FloLogic, and a
server-side change could break it.

The two use different domains (`flologic_unofficial` here, `flologic` there),
so they can be installed side by side.

### Upgrading from 0.4.x

Until 0.5.0 this integration used the `flologic` domain, which now belongs to
the official-API integration. Home Assistant cannot move a config entry
between domains, so:

1. Note any automations or dashboards that use FloLogic entities.
2. Remove the FloLogic integration entry under **Settings → Devices &
   services**. Doing this first is what lets the entity IDs come back
   unchanged.
3. In HACS, add `https://github.com/pfeffed/ha-flologic-unofficial` as a new
   custom repository and install it. The old `pfeffed/ha-flologic` entry now
   holds the official-API integration: update it if you want that too, or
   remove it. Restart Home Assistant.
4. Add **FloLogic (Unofficial)** and sign in again. Entity IDs follow the valve
   names, so they normally come back unchanged; check for a `_2` suffix if an
   old entity was left behind.

Do one house at a time, while someone is there: leak protection through Home
Assistant is down between steps 2 and 4.

## One entry per account, every valve

A FloLogic account often holds several valves — and a G-Connect gateway or leak
sensors in the same device list. **One config entry covers the whole account**
and creates a device per controllable valve. Gateways, sensors and repeaters are
recognised and skipped.

## Entities, per valve

| Entity | Notes |
| --- | --- |
| `valve` | Open or closed — **whether** the water is off |
| `sensor` — Shutoff reason | **Why** it is off: `Not shut off`, `Shut off manually`, `Leak detected`, `Flow ran too long`, `Freeze protection`… |
| `sensor` — Problem | What else is wrong while the valve stays open: `Valve failure`, `Communication error`, `Battery needs changing`… |
| `select` — Mode | `home` / `away` / `bypass` / `shutoff` / `disabled` |
| `sensor` — Status | One-line headline, with the decoded mode bits as attributes |
| `sensor` — Current flow | oz/min, matching the app's own unit. See the caveat below |
| `sensor` — Temperature | |
| `sensor` — Shutoff countdown | See the caveat below |
| `binary_sensor` — Water flow detected | Yes/No. See the caveat below |
| `binary_sensor` — Connectivity | Stays available while the valve is offline |
| `event` — Notification | FloLogic's own log: shutoffs, mode changes, notices |

**Shutoff reason** is the one to automate on. A valve reports `SHUTOFF`
whether you closed it or it closed itself, so the bit alone cannot tell a
leak from a deliberate shutoff — the reason sensor can, and reports
`Shut off manually` only when nothing caused it.

Read-only diagnostics (signal strength, elapsed flow, active flow limit) are
present but disabled by default.

### Settings you can change

| Entity | Notes |
| --- | --- |
| `number` — Home / Away flow limit | Minutes. Sub-minute values are normal: 0.5 is "30 seconds" |
| `number` — Bypass duration, Flow sensitivity, Pre-alert notice | |
| `switch` + `number` — Auto Away, Delay Away, Winter mode | A switch and its value, as in the app |
| `switch` + `number` — Low temperature alert / shutoff | Shown in your units; FloLogic stores Fahrenheit |

FloLogic disables these settings by *negating* the stored value rather than
clearing it, so switching one off keeps the value it would use when switched
back on — and changing a value does not switch anything on. That is why each
is a pair rather than a single entity.

**These are safety thresholds.** An automation that widens a flow limit or
turns off a freeze shutoff has disabled part of your leak protection, silently.
They are exposed because the app exposes them; treat them accordingly.

## Two caveats worth reading before you automate

These come from live testing, not from guesswork.

**Short flows are invisible.** FloLogic's cloud reports flow with tens of
seconds of latency. In three consecutive real auto-shutoffs on a 30-second Away
limit, `flowState` never left "no flow" — the valve shut itself off with no
observable flow beforehand. So **Water flowing** may never turn on, and
**Shutoff countdown** may stay unknown, during an actual leak event. Both are
useful for long draws and absent for brief ones. Do not build "is there a leak
right now" on them; use **Water off** and **Status**, which are driven by the
mode bitfield and are reliable.

**A closed valve does not mean dry taps.** Downstream pipes drain for tens of
seconds after the valve closes — longer on a hot line, where the water heater
sits below the valve and keeps delivering its tank. The `valve` entity reports
the valve, not the plumbing.

## Install

1. **HACS → ⋮ → Custom repositories**, add
   `https://github.com/pfeffed/ha-flologic-unofficial` with category
   **Integration**.
2. Find **FloLogic (Unofficial)** in HACS, install it, and **restart Home
   Assistant**.
3. **Settings → Devices & services → Add integration → FloLogic (Unofficial)**,
   and sign in with your FloLogic account.

Every valve on the account is added, so one entry per account is enough even
if your valves are at different addresses.

`pyflologic-unofficial` is not on PyPI, so the integration ships its own copy
under `vendor/` and declares no requirements: nothing is downloaded at setup,
and a boot without internet access does not cost you leak protection.

Your password is stored in Home Assistant's config entry and sent only to
FloLogic. If it stops working, the integration prompts you to re-authenticate
rather than failing silently.

## Options

**Polling interval** — a backstop only. Updates normally arrive by push within
about a second, so the default is deliberately long. Polling harder risks rate
limiting and gains nothing.

## Development

```bash
uv sync
uv run pytest
uv run ruff check .
```

Tests run against a mocked client; no account needed. Protocol-level work
belongs in `pyflologic-unofficial`, which is vendored here by
`scripts/vendor.py`.

## License

MIT
