# ZabbixExtremeIQ — AP Detail Dashboard

A Zabbix 7.x dashboard widget that surfaces a single Extreme Networks access point's full operational picture by joining live data from Zabbix, Extreme Cloud IQ (XIQ), and PacketFence (PF).

The widget targets Extreme **AP_305C** hardware running IQ Engine 10.7rx, deployed against Zabbix 7.0 / 7.2 / 7.4 with PHP 8.0.30 and PacketFence 15.

---

## What it does

Renders eight tabs for a selected AP host:

- **Overview** — CPU/Memory health rings, uptime, 9-cell live telemetry sparkline strip (Zabbix + XIQ d360), connectivity issues panel, system info, network info, recent events.
- **Wireless** — per-radio cards (wifi0 / wifi1) with band, channel, width, TX power, noise floor, channel utilization, retry rate, client count, plus an SSID broadcast table.
- **Wired** — interface stats table (state, throughput, speed, errors, LLDP neighbor) and a PoE power budget bar sourced from the upstream switch port.
- **Clients** — connected-clients table joining XIQ live RSSI/SNR/data-rate against PacketFence user / role / posture / OS, with `All / Issues / Students / Faculty / Guests` filter pills and an Auth Failures table.
- **Events / Alerts** — unified Zabbix + PF event feed, native Zabbix Problems widget for active triggers, and aggregated NAC violations.
- **Graphs / Latest Data** — broadcast-driven hand-off to native Zabbix widgets.

---

## Repository layout

```
ZabbixExtremeIQ/
├── apdetail/                          AP Detail widget (this project)
│   ├── manifest.json
│   ├── Widget.php                     extends Zabbix\Core\CWidget
│   ├── actions/WidgetView.php
│   ├── includes/
│   │   ├── WidgetForm.php
│   │   ├── XIQClient.php              XIQ Cloud API client
│   │   └── PFClient.php               PacketFence API client
│   ├── views/
│   │   ├── widget.view.php
│   │   └── widget.edit.php
│   └── assets/
│       ├── js/class.widget.js
│       └── css/widget.css
├── xiq_ap_status/                     companion XIQ status widget
├── templates/                         Zabbix templates (see below)
└── Zabbix Extreme/                    project plan, references, mockups
```

**Install path:** `/usr/share/zabbix/ui/modules/apdetail/`
**PHP namespace:** `Modules\APDetail\…` (must be globally unique across all Zabbix modules).

---

## Data sources

| Source | Role |
|---|---|
| **Zabbix API** | History / items / events / triggers via `API::Item`, `API::History`, `API::Event`. SNMPv3 polling against the `.26928` Extreme enterprise OID tree (CPU, memory, noise, channel, TX power, interfaces, PoE). |
| **Extreme Cloud IQ (XIQ)** | Live wireless telemetry — `/clients/active` (rich client roster), `/devices/{id}/interfaces/wifi`, `/devices/{id}/ssid/status`, `/d360/wireless/interfaces-graph`, `/network-policies/{id}/ssids`. Permanent bearer token, APCu cache with `/tmp/zabbix_xiq_cache/` fallback. |
| **PacketFence (PF)** | NAC enrichment — `locationlog` + `/nodes/search` two-call flow per AP IP, `radius_audit_logs` for auth failures, `/audit_log` for the unified events feed. |

### Required Zabbix templates (in `templates/`)

- **Extreme XIQ APs by API** — fleet template; polls `/devices?views=FULL` every 5 minutes and uses LLD to auto-create per-AP hosts.
- **Extreme AP via SNMPv3** — per-AP template linked via host prototype; provides CPU / memory / noise / channel / TX power / wired interfaces.
- **Extreme EXOS by SNMP w POE** — upstream switch template; the AP's PoE draw is read from the LLDP neighbor switch port.

---

## Requirements

- Zabbix **7.0 / 7.2 / 7.4** (7.4 is the strictest target).
- **PHP 8.0.30** — no `readonly` properties, no `array_is_list()`, no enums, no `never`, no intersection types, no `new` in initializers. Polyfills must live inside the calling namespace (function lookup is namespace-first then global).
- Vanilla **ES2020+** browser JS — no build step, no npm.
- Zabbix proxy with SNMPv3 reachability to the AP and outbound HTTPS to the XIQ and PF API endpoints.
- XIQ permanent API token. The widget reads SSID summaries directly from the network-policy response, so the `view_ssid` token scope is **not** required.
- PacketFence 15 API credentials.

No Composer, no npm — Zabbix modules load inside the Zabbix PHP process and use `curl` / `curl_multi` for outbound HTTP.

---

## Installation

1. Copy `apdetail/` into the Zabbix UI module directory:
   ```
   /usr/share/zabbix/ui/modules/apdetail/
   ```
2. In Zabbix → **Administration → General → Modules**, scan the directory and enable **AP Detail**.
3. Import both Extreme templates from `templates/` into Zabbix.
4. Add the widget to a dashboard. Pair it with a **Host Navigator** widget so it can receive the broadcast `_hostid`.
5. Configure the widget's host field, and provide XIQ + PF credentials via the macros referenced by the templates.

After any change to `manifest.json`, **delete and re-add every existing `apdetail` widget instance on every dashboard** — Zabbix only assigns the broadcast `reference` at widget-creation time.

---

## Broadcast contract

| Wire string | Direction | Meaning |
|---|---|---|
| `_hostid` | IN | Host selected via the widget form, or pushed by an upstream Host Navigator. |
| `_timeperiod` | IN | Dashboard time-range changes. |
| `_itemids` | OUT | Emitted when the user clicks a sparkline; drives a native Graph (classic) widget. |

### sessionStorage keys

- `ap.activeTab` — `overview` / `wireless` / `wired` / `clients` / `events` / `alerts` / `graphs` / `latest`.
- `ap.timeRange` — last received `_timeperiod`, for tab persistence.
- `ap.clientFilter` — Clients tab pill (`all` / `issues` / `students` / `faculty` / `guests`).

---

## Project status

| Milestone | Status |
|---|---|
| **M0 — Foundation** (scaffold, API references, templates, validation) | Closed |
| **M1 — Client libraries + template deployment** (`XIQClient.php`, `PFClient.php`, smoke test 17/20 against pilot AP) | Closed |
| **M2 — Overview tab** | In progress |
| **M3 — Wireless + Wired tabs** | In progress (parallel with M2) |
| **M4 — Clients tab** | Queued |
| **M5 — Events / Alerts / Sidecar / E2E** | Queued |

Pilot host: `BHS-56-Hallway` · `172.16.97.59` · Zabbix host id `10847` · XIQ device id `70849781384129`.

### Known carry-over fixes

- **G31** — `XIQClient::getClients()` capped at `limit=500`; XIQ rejects with HTTP 400. Lower to 100 and paginate via `total_count`.
- **G33** — `XIQClient::getPolicySsids()` per-SSID fan-out hits 403; read SSID summaries directly from the network-policy response instead.
- **G32** — `XIQClient::getLocation()` must be POST (deferred to the M5 sidecar).

---

## Development conventions

- **PHP:** `declare(strict_types=1);`, 4-space indent, type hints everywhere PHP 8.0 allows, `match()` over `switch`, named arguments on multi-param API calls.
- **JS:** vanilla ES2020+, `const` / `let`, arrow functions. Lifecycle overrides (`onInitialize`, `onActivate`, `onDeactivate`, `setContents`) MUST call `super.<method>()` first or `enterWidgetEditing` later throws.
- **CSS:** CSS custom properties for colors, BEM-ish class naming (`.ap-detail__panel`, `.ap-conn-issues__item`).
- **External calls:** PHP `curl` only. Use `curl_multi_init()` for parallel fetches.
- **Item key strings:** verbatim from `Zabbix Extreme/AP_Template_DataSource_Reference.html` §6 — do not paraphrase.
- **Commit messages:** `[<phase>] <component>: <change> (Gxx)` — e.g. `[M2] Live Telemetry: XIQ d360 channel-util sparkline (G8, G9)`.

The full execution plan, gotcha catalog (G1–G33), and per-panel acceptance criteria live in `Zabbix Extreme/` alongside the M0 reference HTML.

---

## License

See `LICENSE`.
