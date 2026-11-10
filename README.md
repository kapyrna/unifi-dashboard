# unifi-dashboard

A single-file static page (`index.html`) that shows UniFi health and traffic across **every site your account can see**, using the cloud [Site Manager API](https://developer.ui.com/site-manager-api) (`api.ui.com`). No local controller URL is needed.

- Overall health across all sites
- WAN health (per site and combined)
- Total traffic: the amount downloaded and uploaded across all sites over a rolling Day (24 h), Week (7 days) or Month (30 days), chosen with the switch on the card, with a sparkline of the period
- A per-site table, with one line per WAN when a site has several
- Top clients and top destinations, below the sites list, only when at least one console returns that data. Each shows Down and Up as separate columns; the top 5 are ranked by total traffic (down plus up)

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
| WAN health, total traffic | `GET /v1/isp-metrics/5m?duration=24h` (latency, packet loss, throughput). Total traffic is each sample's average speed times the sample length (5-minute samples for Day, hourly samples from `/v1/isp-metrics/1h` for Week and Month), summed over the window, so it is an estimate and undercounts if samples are missing. Week and Month data are fetched only when selected and cached for 5 minutes; the choice is remembered in `unifi-dashboard:range` |
| Per-WAN lines (name, IP, link state) | Console connector, gateway device (see below). Falls back to one line from the ISP metrics |
| Top clients, top destinations (cards shown only if data exists) | Console connector: client list and traffic flows from the last 24 h. Destination IPs are turned into hostnames by reverse DNS (see below) |

Site health is derived: Down if a gateway is offline, Degraded if any device is offline or there are critical alerts or internet issues, otherwise Healthy.

A console that is **offline** gets its own red Offline status instead of Degraded, and the Overall health card lists it (for example "1 console offline: Cabin"). Offline consoles are left out of the device, alert and WAN tallies, show dashes for their stale numbers, and are not sent connector requests. The overall status shows Degraded while some consoles are offline and Offline only if all of them are. The cloud API has no documented online flag, so a console counts as offline when its `reportedState.state` reads disconnected, offline or similar (an undocumented field whose values I could not confirm), or when every device at its site is offline.

## Known limitations

- **CORS:** I couldn't verify that `api.ui.com` allows requests from a browser page. If the page reports it can't reach the API, run it behind a CORS proxy and set the proxy URL under Settings, Advanced.
- **One site per console is assumed.** The connector always uses the console's `default` site, so a console with several sites would show the same WAN detail for each of them.
- **Per-WAN detail, client traffic and destination hostnames are not in the cloud API.** The page tries to reach each console's gateway through the Site Manager connector (`/v1/connector/consoles/{id}/...`). That needs console firmware 5.0.3 or later and an owner or admin key, and is limited to 100 requests per minute per console, so the result is cached for 5 minutes. The endpoints used are not part of the official API, so they may not work on every console. When the gateway call fails, the site shows a single WAN line from the ISP metrics. When the client or flow calls fail (or return nothing), the Top clients and Top destinations cards are left out and nothing is shown in their place. If several consoles succeed, the lists are combined across them.
- **Reverse DNS for destinations:** a browser page can't do DNS lookups, so IPs of the top 25 destinations are looked up with PTR queries over Cloudflare's DNS-over-HTTPS (`cloudflare-dns.com`). **This sends those destination IPs to Cloudflare.** Private and reserved addresses are never sent. Results are cached in `localStorage` (hits 7 days, misses 1 day) under `unifi-dashboard:ptr`, and you can switch the lookup off in Settings. PTR names often belong to the hosting provider (for example an `amazonaws.com` or `1e100.net` name) rather than the service you were using, and IPs with no PTR record stay as IPs.
- **Up and Down directions are assumptions.** For clients, `tx_bytes` (sent to the client) is shown as Down and `rx_bytes` (received from the client) as Up, matching how UniFi counts them from the access point or switch side. For destinations, `bytes_rx` is Down and `bytes_tx` is Up. If a flow only reports a total, its Down and Up show a dash and it is still ranked by that total. If the columns look reversed on your consoles, swap them in `clientSplit` and `parseFlows`.
