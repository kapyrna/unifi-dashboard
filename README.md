# unifi-dashboard

A single-file static page (`index.html`) that shows UniFi health and traffic across **every site your account can see**, using the cloud [Site Manager API](https://developer.ui.com/site-manager-api) (`api.ui.com`). No local controller URL is needed.

- Overall health across all sites
- WAN health (per site and combined)
- Total traffic (latest WAN rate, plus a 24 h sparkline)
- Top clients
- Top destination hostnames (last 24 h)

No build step. Open `index.html` in a browser or host it on any static server.

## Setup

1. Sign in at unifi.ui.com, open the API section, and create an API key.
2. Open the page, click **Settings**, and paste the key.

Settings are stored in your browser's `localStorage`. Every key is prefixed with `unifi-dashboard:` (`unifi-dashboard:apiKey`, `unifi-dashboard:refreshSec`, and optionally `unifi-dashboard:apiBase`) so they don't collide with other projects on the same origin. The key is kept in plain text, so only use this on a browser you trust.

Add `?demo=1` to the URL to preview the page with sample data.

## Where the data comes from

| Panel | Source |
| --- | --- |
| Overall health, devices, clients | `GET /v1/sites` (counts, internet issues, WAN uptime), `GET /v1/hosts` for console names |
| WAN health, total traffic | `GET /v1/isp-metrics/5m?duration=24h` (latency, packet loss, throughput) |
| Per-WAN lines (name, IP, link state, rates) | Console connector, gateway device (see below). Falls back to one line from the ISP metrics |
| Top clients, top destination hostnames | Console connector (see below) |

Site health is derived: Down if a gateway is offline, Degraded if any device is offline or there are critical alerts or internet issues, otherwise Healthy.

## Known limitations

- **CORS:** I couldn't verify that `api.ui.com` allows requests from a browser page. If the page reports it can't reach the API, run it behind a CORS proxy and set the proxy URL under Settings, Advanced.
- **One site per console is assumed.** The connector always uses the console's `default` site, so a console with several sites would show the same client, flow and WAN data for each of them.
- **Top clients, destination hostnames and per-WAN detail are not in the cloud API.** The page tries to reach each console's local Network API through the Site Manager connector (`/v1/connector/consoles/{id}/...`). That needs console firmware 5.0.3 or later and an owner or admin key, and is limited to 100 requests per minute per console, so the results are cached for 5 minutes. The client list and traffic flow endpoints used are not part of the official API, so they may not work on every console. When they fail, the panels show why and the rest of the page still works.
