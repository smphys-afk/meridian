# Preserve client IPs with Caddy socket activation

Systemd opens Caddy's public TCP/UDP sockets so apps receive the visitor's IP in
`X-Forwarded-For` over HTTP/1.1, HTTP/2 and HTTP/3. Podman's `rootlessport` replaces
that IP with a bridge address. Caddy still uses `meridian-proxy` for container DNS
and blue/green routing.

## Requirements

- Caddy 2.11.6+ for ACME challenges on activated sockets. The default image
  qualifies; setup and `check` reject older versions.
- `net.ipv4.ip_unprivileged_port_start` at or below the lowest proxy port.
  Bootstrap sets it to `80`, covering the default 80/443 ports.
- Inbound TCP on both ports and UDP on the HTTPS port for HTTP/3.

## Upgrade an existing host

Upgrading from `v0.1.0-alpha.1` to `v0.1.0-alpha.2` or newer is a breaking change.
Setup briefly interrupts **every** app sharing Caddy.
Use the same current Meridian version for all services on the host.

Before setup, back up old `ssl: false` route fragments outside
`~/.config/containers/meridian-caddy/routes/`, then remove those originals.
These fragments lack `bind fd/3`: they can pass validation but use the wrong
listener, leaving HTTP requests redirected to HTTPS. Setup creates a 503 route
for its service when the fragment is missing; deploy restores the app route.
Existing HTTPS fragments work until the next deploy rewrites them.

During the maintenance window, run:

```sh
meridian setup --config path/to/environment.yml
meridian deploy --config path/to/environment.yml   # repeat for each service
meridian check --config path/to/environment.yml    # repeat for each service
```

Setup validates the configuration before stopping Caddy. A successful validation
does not prove that legacy HTTP fragments use the correct activated socket.

## Socket layout

`meridian quadlet` exports `meridian-caddy.socket`. Install it in
`~/.config/systemd/user`.

| Descriptor | Host socket | Caddy binding |
| --- | --- | --- |
| 3 | TCP `proxy.http_port` | `fd/3`, HTTP/1.1 |
| 4 | TCP `proxy.https_port` | `fd/4`, HTTP/1.1 and HTTP/2 |
| 5 | UDP `proxy.https_port` | `fdgram/5`, HTTP/3 |

HTTP routes bind FD 3; HTTPS sites use the FD 4/5 defaults. Automatic redirects
cannot use activated sockets, so the root Caddyfile provides a 308 HTTPS redirect.
Explicit HTTP routes take precedence; ACME HTTP challenges run before routing.

On Podman 4.9, network helpers can inherit the public sockets and hold them open.
The Quadlet starts them in `ExecStartPre`, which receives no activation FDs.

## Checks

`meridian check` verifies the active socket unit, configured bindings, Caddy's
inherited descriptors 3/4/5, low-port permission and Caddy version. It rejects
`rootlessport` listeners. Caddy's descriptors remain visible when systemd's
process details are hidden.

The check cannot prove external UDP reachability or client IP forwarding by a CDN.
