# unifi-dashboard

A single-file static page (`index.html`) that shows UniFi health and traffic across all sites:

- Overall health across all sites
- WAN health (per site and combined)
- Total traffic (live WAN rate with a short history sparkline)
- Top clients
- Top destination hostnames (last 24 h)

No build step. Open `index.html` in a browser or host it on any static server.

## Setup

1. In UniFi Network, go to Settings, Control Plane, Integrations and create an API key.
2. Open the page, click **Settings**, and enter the controller URL (for example `https://192.168.1.1`) and the key.

Settings are stored in your browser's `localStorage`. Every key is prefixed with `unifi-dashboard:` (`unifi-dashboard:baseUrl`, `unifi-dashboard:apiKey`, `unifi-dashboard:refreshSec`) so they don't collide with other projects on the same origin. The key is kept in plain text, so only use this on a browser you trust.

Add `?demo=1` to the URL to preview the page with sample data.

## Known limitations

- **CORS / certificates:** UniFi controllers generally don't send CORS headers, and use self-signed certificates. If the page can't connect, open the controller URL in the browser once to trust the certificate, and use a CORS proxy or a browser CORS extension if requests are still blocked.
- **Top destination hostnames** come from the controller's Traffic Flows data, which is not part of the official API. It is best-effort and shows a notice if your controller doesn't provide it.
- **Health, WAN and client data** use the controller's Network API (`/proxy/network/...`) with the `X-API-KEY` header.
