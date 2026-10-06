# OpenChamber on an OCI Free-Tier Arm VM

Running [OpenChamber](https://openchamber.ai) on a free Oracle Cloud Arm
instance — reachable over HTTPS with password **and** passkey, without a
Cloudflare tunnel and without a public IP visible in the operating system.

This is the condensed version of a runbook written while setting up such an
instance. All addresses are placeholders; the findings are real.

```
Internet ──443──> OCI Security List ──> VNIC (1:1 NAT) ──> Caddy
                                                            │
                                          127.0.0.1:3000 ◄──┘
                                                        OpenChamber
```

## What this repo documents

Four failure modes that together cost close to two days and none of which
show up as an *error* in the logs:

| # | Symptom | Cause |
|---|---|---|
| 1 | `ip addr` shows no public IP | OCI implements the VNIC's public IP as **1:1 NAT at the fabric level** — it is invisible in the guest OS |
| 2 | ufw allows port 3000, but nothing arrives | the OCI image places a `REJECT` rule **above** ufw's jump, turning every allow rule into dead text |
| 3 | *„Tunnel access required"* | an active OpenChamber tunnel sets the auth scope to `unknown-public`, which hard-refuses both password login and passkeys |
| 4 | Caddy listens on `:80`, no certificate, no error | a Caddyfile **without** the `https://` scheme makes Caddy silently disable automatic HTTPS |

Number 4 is the sneakiest: the configuration is not broken, it simply is not
the one you thought you wrote — and the service starts anyway, just without
TLS. Covered in section 5 of the runbook.

## Passkeys

The actual reason for building all of this. WebAuthn requires a real domain as
`rp.id`; browsers reject IP addresses. With an externally exposed domain this
turns from

```json
{"rp": {"name": "OpenChamber", "id": "203.0.113.42"}}
```

into a working

```json
{"rp": {"name": "OpenChamber", "id": "app.example.com"}}
```

Passkeys strictly require the external reverse proxy instead of
`openchamber tunnel` — see failure mode 3.

## Requirements on the OCI side

Two things in the console, otherwise nothing gets through:

1. **Security list, not NAT gateway:** an ingress rule for TCP 443
   (`0.0.0.0/0`, stateless `No`). A security list is sufficient because the
   VNIC's public IP is passed through via 1:1 NAT.
2. **Port 80 is not needed.** Certificate validation runs over TLS-ALPN-01
   and therefore entirely through port 443.

To close the app port, deleting the rule for port 3000 is enough — there is
**no** NAT port-forward rule in play that you would need to hunt for.

## Setup in brief

```bash
# OpenChamber with password and auto-start
openchamber startup enable --port 3000 --host 0.0.0.0 \
  --ui-password "$(openssl rand -base64 18)"

# remove the OCI shadow REJECT (otherwise ufw is ineffective)
iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited

# Caddy with an explicit scheme
#   https://app.example.com {
#       reverse_proxy 127.0.0.1:3000
#   }
```

The full runbook — all verified commands, the rate limit values, the auth-scope
mechanism, and a troubleshooting table — is in
[RUNBOOK.en.md](RUNBOOK.en.md). A German translation is available as
[RUNBOOK.md](RUNBOOK.md).

## Two warnings

**`--ui-password` is a flag, not an environment variable.** Using
`OPENCHAMBER_UI_PASSWORD=… openchamber startup enable` does not write the value
to `startup.env` — the VM then boots **without authentication** on a public
address after a reboot.

**`systemctl --user restart openchamber.service` kills running sessions.** The
agent processes live in the service's cgroup. Close open chats before any
restart.

## Mobile

There is no native iOS app. The web version is explicitly built for mobile as
an installable PWA (service worker, `apple-mobile-web-app-capable`,
background notifications). Add it to the home screen from Safari to get a
full-screen experience. Note that the web app manifest is not served, which on
iOS is usually harmless but can occasionally affect the install prompt.

## Contents

| File | |
|---|---|
| [RUNBOOK.en.md](RUNBOOK.en.md) | operation, password, firewall, auth scope, HTTPS, troubleshooting (English) |
| [RUNBOOK.md](RUNBOOK.md) | dasselbe auf Deutsch (German) |

## License

See [LICENSE](LICENSE).
