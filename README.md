# ZabbixExtremeIQ — Extreme AP dashboard widgets for Zabbix

Two Zabbix 7.x dashboard widgets for Extreme Networks access points:

- **`apdetail/` — AP Detail.** Single-AP deep-dive that joins live data from Zabbix, Extreme Cloud IQ (XIQ), and PacketFence (PF) into eight tabs (Overview, Wireless, Wired, Clients, Events, Alerts, Graphs, Latest Data).
- **`xiq_ap_status/` — Extreme XIQ AP Status.** Fleet-wide table of XIQ-discovered APs with per-row operational actions (reboot, manage/unmanage, refresh, allow-listed `show` commands).

Both widgets tested with Extreme **AP_305C** hardware running IQ Engine 10.7rx, deployed against Zabbix 7.4 with PHP 8.0.30 and PacketFence 15. They share the same Extreme XIQ APs by API template.

---

## AP Detail widget (`apdetail/`)

Renders eight tabs for a selected AP host:

- **Overview** — CPU/Memory health rings, uptime, 9-cell live telemetry sparkline strip (Zabbix + XIQ d360), connectivity issues panel, system info, network info, recent events.
- **Wireless** — per-radio cards (wifi0 / wifi1) with band, channel, width, TX power, noise floor, channel utilization, retry rate, client count, plus an SSID broadcast table.
- **Wired** — interface stats table (state, throughput, speed, errors, LLDP neighbor) and a PoE power budget bar sourced from the upstream switch port.
- **Clients** — connected-clients table joining XIQ live RSSI/SNR/data-rate against PacketFence user / role / posture / OS, with `All / Issues / Students / Faculty / Guests` filter pills and an Auth Failures table.
- **Events / Alerts** — unified Zabbix + PF event feed, native Zabbix Problems widget for active triggers, and aggregated NAC violations.
- **Graphs / Latest Data** — broadcast-driven hand-off to native Zabbix widgets.

---

## XIQ AP Status widget (`xiq_ap_status/`)

Fleet-level companion widget. Renders a sortable table of every AP discovered by the **Extreme XIQ APs by API** template on a chosen Zabbix host (defaults to 1500 rows), with a per-row kebab menu for operational actions.

### Table

Reads template-created items (`xiq.ap.connected[*]`, `xiq.ap.clients[*]`, etc.) joined on the `ap_serial` / `ap_mac` / `ap_id` tags. Re-import the template if those tags are missing.

### Per-row actions (all OFF by default)

| Action | Required XIQ token scope |
|---|---|
| Reboot AP | `device:reboot` |
| Set Managed | `device:manage` |
| Set Unmanaged | `device:unmanage` |
| Run `show` command (allow-listed) | `device:cli` |
| Refresh now / LRO polling | `lro:r` |

The CLI dropdown enforces a hard-coded `show ` prefix plus a configurable allow-list. Mutating commands are blocked even if they reach the allow-list.

### Action token security model (Path C — file-based)

The XIQ action token never enters the database, the browser, or the Zabbix API. It lives only on the Zabbix frontend filesystem:

```
/etc/zabbix/secrets/xiq_action_token        owner root:apache, mode 0640
/etc/zabbix/secrets/                        mode 0750
```

The widget config field **Action token file path** accepts only files under `/etc/zabbix/secrets/`; `realpath()` is used before the prefix check so symlink escape is also blocked. To rotate, write the new token atomically (`mv` from a sibling file) — no widget edit, no dashboard reload, no restart.

### Permission model

| Capability | Required Zabbix user type |
|---|---|
| See the widget | Any user with read access to the XIQ host |
| See action menu items | Zabbix Admin or Super Admin |
| Fire actions | Zabbix Admin or Super Admin (also enforced server-side in `WidgetAction.php`) |
| Read the action token file | Apache process only (filesystem perms) |

Every action — success or failure — writes to Zabbix's audit log (userid, source IP, op, device IDs, LRO id or error, plus a 4 KB excerpt of CLI output).

### "Open in XIQ" / Extreme Platform ONE

Default admin URL is `https://extremeplatformone.com` (Extreme migrated the customer-facing UI from `extremecloudiq.com` to EP1 in late 2025). EP1 is a single-page app and doesn't update its URL on navigation, so the kebab menu's **Open in XIQ** link also copies the AP serial to the clipboard for paste-and-search inside EP1's filter bar. The legacy `{id}` placeholder in the device path is still substituted for sites still on the deep-linkable UI.

### Known limitations

- No bulk multi-row actions (XIQ endpoints support it; UI work only).
- No per-AP managed-state column in the table — both Managed and Unmanaged appear in every menu.
- CLI is single-device only.
- Multi-frontend Zabbix deployments must place the token file on every frontend node; it is not synchronized automatically.

The widget's own README at `xiq_ap_status/README.md` carries the full installation, token-rotation, and troubleshooting guide.

---

## Repository layout

```
ZabbixExtremeIQ/
├── apdetail/                          AP Detail widget (single-AP deep-dive)
│   ├── manifest.json
│   ├── Widget.php                     extends Zabbix\Core\CWidget
│   ├── actions/WidgetView.php
│   ├── includes/
│   │   ├── WidgetForm.php
│   │   ├── XIQClient.php              XIQ Cloud API client
│   │   └── PFClient.php               PacketFence API client
│   ├── views/{widget.view,widget.edit}.php
│   └── assets/{js/class.widget.js, css/widget.css}
├── xiq_ap_status/                     XIQ AP Status widget (fleet table + actions)
│   ├── manifest.json
│   ├── Widget.php
│   ├── actions/
│   │   ├── WidgetView.php
│   │   └── WidgetAction.php           reboot / manage / unmanage / cli / refresh
│   ├── includes/WidgetForm.php
│   ├── views/{widget.view,widget.edit}.php
│   ├── assets/{js/class.widget.js, css/widget.css}
│   └── README.md                      install + action-token + audit guide
├── templates/                         Zabbix templates (see below)
└── Zabbix Extreme/                    project plan, references, mockups
```

**Install paths:**
`/usr/share/zabbix/ui/modules/apdetail/`
`/usr/share/zabbix/modules/xiq_ap_status/`

**PHP namespaces:** `Modules\APDetail\…` and `Modules\XiqApStatus\…` — both must be globally unique across all installed Zabbix modules.

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

1. Copy each widget into Zabbix's modules location:
   ```
   /usr/share/zabbix/ui/modules/apdetail/
   /usr/share/zabbix/modules/xiq_ap_status/
   ```
2. In Zabbix → **Administration → General → Modules**, scan the directory and enable **AP Detail** and **Extreme XIQ AP Status**.
3. Import the Extreme templates from `templates/` into Zabbix.
4. Add the widgets to a dashboard:
   - **AP Detail**: pair with a **Host Navigator** so it can receive the broadcast `_hostid`. Configure XIQ + PF credentials via the macros referenced by the templates.
   - **XIQ AP Status**: set the XIQ host (the Zabbix host with the XIQ template linked). To enable per-row actions, mint an XIQ API token with the required scopes and place it at `/etc/zabbix/secrets/xiq_action_token` (owner `root:apache`, mode `0640`), then flip on the desired action toggles. Full step-by-step in `xiq_ap_status/README.md`.

After any change to either `manifest.json`, **delete and re-add every existing widget instance on every dashboard** — Zabbix only assigns broadcast `reference` values at widget-creation time.

---

## Broadcast contract

**AP Detail** (`apdetail/`):

| Wire string | Direction | Meaning |
|---|---|---|
| `_hostid` | IN | Host selected via the widget form, or pushed by an upstream Host Navigator. |
| `_timeperiod` | IN | Dashboard time-range changes. |
| `_itemids` | OUT | Emitted when the user clicks a sparkline; drives a native Graph (classic) widget. |

**XIQ AP Status** (`xiq_ap_status/`) broadcasts `_hostid` / `_hostids` outbound when an AP row is selected, so it can drive an `apdetail` widget on the same dashboard.

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
