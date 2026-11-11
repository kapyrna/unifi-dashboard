# unifi-dashboard

A single-file static page (`index.html`) that shows UniFi health and traffic across **every site your account can see**, using the cloud [Site Manager API](https://developer.ui.com/site-manager-api) (`api.ui.com`). No local controller URL is needed.

- Overall health across all sites
- WAN health (per site and combined)
- WAN traffic: the amount downloaded and uploaded through the WANs across all sites over a rolling Day (24 h), Week (7 days) or Month (30 days), chosen with the switch on the card, with a sparkline of the period
- Local traffic: the amount that stayed on your own networks over the last 24 h, shown only when a console returns traffic flows it can classify
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
| Local traffic (card shown only if data exists) | Console connector: traffic flows from the last 24 h, up to 5,000 per console. A flow counts as local when both its source and destination are private, link-local or multicast addresses; flows missing either address are skipped. If the console returns more flows than that, or ignores paging, the card says "partial sample". It follows neither the Day / Week / Month switch nor the cloud API |
| Per-WAN lines (name, IP, link state) | Console connector, gateway device (see below). Falls back to one line from the ISP metrics |
| Top clients, top destinations (cards shown only if data exists) | Console connector: client list and traffic flows from the last 24 h. Destination IPs are turned into hostnames by reverse DNS (see below) |

Site health is derived: Down if a gateway is offline, Degraded if any device is offline or there are critical alerts or internet issues, otherwise Healthy.

A console that is **offline** gets its own red Offline status instead of Degraded, and the Overall health card lists it (for example "1 console offline: Cabin"). Offline consoles are left out of the device, alert and WAN tallies, show dashes for their stale numbers, and are not sent connector requests. The overall status shows Degraded while some consoles are offline and Offline only if all of them are. The cloud API has no documented online flag, so a console counts as offline when its `reportedState.state` reads disconnected, offline or similar (an undocumented field whose values I could not confirm), or when every device at its site is offline.

## Known limitations

- **CORS:** I couldn't verify that `api.ui.com` allows requests from a browser page. If the page reports it can't reach the API, run it behind a CORS proxy and set the proxy URL under Settings, Advanced.
- **One site per console is assumed.** The connector always uses the console's `default` site, so a console with several sites would show the same WAN detail for each of them.
- **Per-WAN detail, client traffic and destination hostnames are not in the cloud API.** The page tries to reach each console's gateway through the Site Manager connector (`/v1/connector/consoles/{id}/...`). That needs console firmware 5.0.3 or later and, it seems, a key belonging to the console's owner (Ubiquiti's docs don't state the required role; one admin-only work account got errors on every console while the owner's key worked), and is limited to 100 requests per minute per console, so the result is cached for 5 minutes. The endpoints used are not part of the official API, so they may not work on every console. When the gateway call fails, the site shows a single WAN line from the ISP metrics. When the client or flow calls fail (or return nothing), the Top clients and Top destinations cards are left out and nothing is shown in their place. If several consoles succeed, the lists are combined across them.
- **Reverse DNS for destinations:** a browser page can't do DNS lookups, so IPs of the top 25 destinations are looked up with PTR queries over Cloudflare's DNS-over-HTTPS (`cloudflare-dns.com`). **This sends those destination IPs to Cloudflare.** Private and reserved addresses are never sent. Results are cached in `localStorage` (hits 7 days, misses 1 day) under `unifi-dashboard:ptr`, and you can switch the lookup off in Settings. PTR names often belong to the hosting provider (for example an `amazonaws.com` or `1e100.net` name) rather than the service you were using, and IPs with no PTR record stay as IPs.
- **Up and Down directions are assumptions.** For clients, `tx_bytes` (sent to the client) is shown as Down and `rx_bytes` (received from the client) as Up, matching how UniFi counts them from the access point or switch side. For destinations, `bytes_rx` is Down and `bytes_tx` is Up. If a flow only reports a total, its Down and Up show a dash and it is still ranked by that total. If the columns look reversed on your consoles, swap them in `clientSplit` and `parseFlows`.
- **Why a console shows only one WAN and no clients or destinations:** the line under the title says when connector calls failed, for example "console details unavailable for 8 of 8 consoles (403 Forbidden)", and hovering it lists each console's failing call. The likely causes are a key from an account that isn't the console's owner, console firmware older than 5.0.3, or a console whose site isn't named `default`. A failed call is retried after 30 seconds rather than cached for 5 minutes.
