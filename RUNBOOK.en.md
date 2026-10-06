# OpenChamber Runbook — vm-arm01

Quick reference for operating OpenChamber on an OCI Arm VM.
Last verified: 2026-10-05.

> **Anonymised reference.** All addresses, domains, and hostnames in this
> document are placeholders taken from the RFC documentation ranges
> (`203.0.113.0/24`, `198.51.100.0/24`, `10.0.0.0/8`) and from
> `example.com`. They describe **no** real system. The findings, commands,
> and failure modes, however, were reproduced for real.

| | |
|---|---|
| Host | `vm-arm01` (OCI, region `eu-frankfurt-1`) |
| Architecture | `VM.Standard.A1.Flex`, aarch64, 1 OCPU / 6 GB |
| Private IP | `10.0.0.87` (VNIC `enp0s6`) |
| Public IP | `203.0.113.42` (on the VNIC; **not visible in the guest OS**) |
| Domain | `app.example.com` (A record -> public IP) |
| App URL | **`https://app.example.com`** |
| TLS | Caddy 2.11.7, Let's Encrypt, automatic renewal |
| Open ports | **22 (SSH), 443 (HTTPS)** — 3000 is closed on purpose |
| Internal port | 3000 reachable only via loopback |
| Password | `cat ~/.openchamber-ui-password` |
| Passkeys | supported — `rp.id = app.example.com` |
| Service unit | `~/.config/systemd/user/openchamber.service` |

---

## 1. Day-to-day operation

```bash
openchamber status        # fastest check -> must say "password: yes"
```

```bash
systemctl --user status openchamber.service --no-pager   # details
systemctl --user is-enabled openchamber.service          # enabled -> starts at boot
systemctl --user is-active  openchamber.service          # active  -> running now
loginctl show-user ubuntu -p Linger --value              # yes -> survives logout
```

### Logs

```bash
journalctl --user -u openchamber.service -f           # follow service output
journalctl --user -u openchamber.service --since boot # since last boot
openchamber logs                                     # the app's own log
tail -f ~/.config/openchamber/logs/openchamber-3000.log
```

### Restart / stop

```bash
systemctl --user restart openchamber.service
systemctl --user stop    openchamber.service
systemctl --user start   openchamber.service
```

> **CAUTION:** `restart` also kills running sessions. The agent processes
> live in the service's cgroup — a restart takes them down with it. Close any
> open chats first.

### One-line health check

```bash
openchamber status && systemctl --user is-active openchamber.service \
  && curl -so /dev/null -w 'HTTP %{http_code}\n' --noproxy '*' http://127.0.0.1:3000/
```

Expected: `1 running runtime(s)`, `active`, `HTTP 200`.

---

## 2. Password

Show the password:

```bash
cat ~/.openchamber-ui-password
```

**Change** the password — always via the `--ui-password` flag:

```bash
openchamber startup enable --port 3000 --host 0.0.0.0 --ui-password "$(openssl rand -base64 18)"
```

This is also the clean way to invalidate **all** sessions at once: changing the
password rotates the internal `jwt-secret`, which invalidates every existing
session cookie.

> **Important:** do not delete `startup.env`!
> `~/.config/openchamber/startup.env` carries the password into the service
> via `EnvironmentFile`. If that file is missing, the VM boots **without
> authentication** on a public address after a reboot.
>
> Verify:
> ```bash
> sudo grep -q '^OPENCHAMBER_UI_PASSWORD=' ~/.config/openchamber/startup.env \
>   && echo "ok: password present" || echo "WARNING: no password!"
> ```

> **Do not** use the environment-variable form
> `OPENCHAMBER_UI_PASSWORD=... openchamber startup enable` — it does not
> reliably write the value to `startup.env`. Only the `--ui-password` flag
> does.

---

## 3. Firewall

The OCI image ships a `REJECT` rule that sits **above** ufw's jump and renders
its allow rules ineffective:

```
-A INPUT -j REJECT --reject-with icmp-host-prohibited   <- fires first
-A INPUT -j ufw-before-input                            <- the 3000 rule sits down here
```

That is why `ufw status` looked green while the rule was dead text
(0 packets). The rule was removed; ufw's own INPUT policy is `DROP`, so
nothing additional was opened.

```bash
sudo iptables -S INPUT | grep -q REJECT \
  && echo "ERROR: shadow REJECT is back" || echo "ok"
sudo iptables -S ufw-user-input          # must show the 443 ACCEPT rule
```

The boot guard restores this on every start:

```bash
systemctl is-enabled openchamber-firewall.service   # enabled
systemctl --user restart openchamber-firewall.service  # re-apply manually
```

The original rules are at `/etc/iptables/rules.v4.bak-openchamber`.

### Current state: 3000 closed, only 443 open

```bash
sudo ufw status | grep -E '22|443|3000'    # expected: 443 only
```

Port 3000 is deliberately **no longer** in ufw. OpenChamber still runs on
`0.0.0.0:3000`, but the only way in is now via Caddy over loopback:

```
Internet ──443──> Caddy ──loopback──> 127.0.0.1:3000 (OpenChamber)
```

So do **not** simply re-add the rule restricted to `127.0.0.0/8` — that would
close 3000 from the outside, but Caddy binds to `0.0.0.0`. The port stays
closed because ufw runs `INPUT DROP`.

Backup of the ufw rules: `/etc/ufw/user.rules.bak-openchamber`.

> **Why OpenChamber still binds to `0.0.0.0`:** `localhost` would be cleaner,
> but it requires a service restart — and with it the end of every running
> session. Only worth doing in a maintenance window. Until then the firewall
> is what provides the protection.

---

## 4. Understanding the auth scope

This is the most confusing part. Every request is classified by its `Host`
header (`server/lib/opencode/tunnel-auth.js`, `classifyRequestScope`):

| Condition | Scope | Behaviour |
|---|---|---|
| Host == active tunnel host | `tunnel` | tunnel session cookie only |
| localhost / loopback | `local` | normal UI password |
| **no tunnel active** | `local` | normal UI password |
| anything else | `unknown-public` | tunnel session cookie only |

Consequences:

- `unknown-public` and `tunnel` are hard-blocked by `requireTunnelSession()`.
- **A UI password does not help there.** Password login is explicitly refused
  in this scope (`core-routes.js`) — hence the *„Tunnel access required"*
  message as soon as the Cloudflare tunnel is running.
- `activeTunnelId` lives in RAM only. **A service restart resets it to
  `null`**, which resolves the problem as a side effect.

Password TTL (measured empirically):

| Login | `Max-Age` |
|---|---|
| normal | 43200 s = 12 hours |
| `trustDevice: true` | 604800 s = 7 days |

„Trust this device" is therefore only a longer-lived cookie. There is **no**
server-side device list and no logout endpoint — sessions are stateless JWTs
validated only by `jwtVerify` against the `jwt-secret`.

Tunnel status:

```bash
openchamber tunnel status --all
openchamber tunnel stop --port 3000     # tunnel only, server keeps running
```

---

## 5. HTTPS / Caddy

Caddy terminates TLS and forwards to `127.0.0.1:3000`.

```bash
systemctl is-active caddy                        # active
systemctl is-enabled caddy                       # enabled
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo journalctl -u caddy -f                      # watch certificate issuance
```

Current Caddyfile (`/etc/caddy/Caddyfile`):

```
https://app.example.com {
	reverse_proxy 127.0.0.1:3000
}
```

Certificate: Let's Encrypt, valid until 2027-01-03, renewed automatically
(Caddy renews roughly 30 days before expiry).

> **The explicit `https://` is mandatory.** With a bare hostname
> (`app.example.com { ... }`) Caddy judged the domain unsuitable for
> automatic HTTPS and **silently downgraded the server to `:80`** — no
> certificate, no error at startup. Log message:
> `server is listening only on the HTTP port, so no automatic HTTPS will be applied`
> The scheme forces TLS and triggers the certificate request.

Port 80 is not needed: validation runs over **TLS-ALPN-01**, which works
entirely through port 443. That is why port 80 is deliberately closed in ufw.

### OCI side: what was required

A single ingress rule in the VCN's **security list**:

| Field | Value |
|---|---|
| Stateless | No |
| Source CIDR | `0.0.0.0/0` |
| IP Protocol | TCP |
| Source Port Range | All |
| Destination Port Range | `443` |
| Description | `HTTPS via Caddy (app.example.com)` |

**No** NAT gateway port-forward rule. The VNIC's public IP is passed through
via 1:1 NAT (see section 8).

### Passkeys

Only meaningful over HTTPS — WebAuthn requires a secure origin and a domain
as `rp.id`. Verify from the VM:

```bash
# returns, among other things, "rp":{"name":"OpenChamber","id":"app.example.com"}
curl -s -X POST https://app.example.com/auth/passkey/register/options \
  -H 'Content-Type: application/json' -H 'Origin: https://app.example.com'
```

Registration: log in via the browser, then create a passkey.

---

## 6. Testing that the password is enforced

The simplest and most realistic method: a **private / incognito window**.

A private window has no cookie and behaves exactly like a new device coming
from outside — so you see the real password prompt.

Discard all sessions at once (this kills running sessions):

```bash
systemctl --user restart openchamber.service
```

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| *„Tunnel access required"* | tunnel running -> scope `unknown-public` | stop the tunnel, or restart the service (section 4) |
| Browser does not prompt / 401 | wrong password | mind the rate limit: 10 attempts / 5 min, then a **15 min** lockout. Retry after 15 min. |
| `curl https://app.example.com` -> connection error, but `:3000` works | no ingress rule for **443** in the security list | add a TCP 443 `0.0.0.0/0` ingress rule (section 5) |
| `curl https://app.example.com` -> TLS error / Caddy only listening on `:80` | Caddyfile without the `https://` scheme -> automatic HTTPS skipped | use `https://app.example.com` and run `sudo systemctl reload caddy` |
| Certificate is not issued | validation fails | `sudo journalctl -u caddy -f`. For `tls-alpn-01`, check port 443 in the security list **and** in ufw |
| Connection times out | no public IP, or private subnet / no internet gateway | OCI console: VNIC `Attached VNICs` -> `IP administration` -> *Ephemeral public IP*. Prerequisites: **public** subnet and a VCN with an **internet gateway**. Additionally **both** the NSG *and* the security list must allow the port. |
| `No route to host` from inside | hairpin test against your own public IP | Not conclusive — OCI mirrors hairpin traffic inconsistently across ports. The 3000 rule answers, 443 may not. Test from outside. |
| No password after reboot | `startup.env` missing/deleted | section 2 — set the password again |
| Port 3000 closed after reboot | boot guard missing | `systemctl enable openchamber-firewall.service` |
| `openchamber status` empty | service not running | `journalctl --user -u openchamber.service -n 50` |

An SSH login via port 22 remains reachable **at all times** — intended as a
fallback when OpenChamber misbehaves.

---

## 8. How all of this came about

1. OCI's `REJECT` overrode ufw -> port 3000 was blocked despite `ufw status`.
2. Active Cloudflare quick tunnel -> direct access to port 3000 was refused.
3. Service did not run under systemd -> no survival across reboot/logout.
4. Password missing (`--host 0.0.0.0` without `--ui-password`) -> an agent
   with shell access exposed to the internet.
5. Caddyfile without the `https://` scheme -> automatic HTTPS silently
   disabled.

### Trap: the public IP is invisible in the guest OS

In OCI, a public IP assigned to a VNIC is implemented as **1:1 NAT at the
fabric level**. It therefore does **not** show up in `ip addr`:

```
$ ip -br addr show enp0s6
enp0s6   UP   10.0.0.87/24        # <- public IP 203.0.113.42 is missing here
```

For the same reason, `SSH_CONNECTION` always shows the private IP as the
destination even though you connected over the public IP:

```
198.51.100.77 65118 10.0.0.87 22
```

**Consequence:** `ip addr` is **no** evidence that an instance has only a
private address. The authoritative source is the OCI console
(Instance -> Attached VNICs -> IP administration) and the security list.

Inbound traffic runs exclusively via the security list (subnet private or
public, no load balancer required):

```
Internet → Security List (Ingress) → VNIC/1:1 NAT → 10.0.0.87
```

That is why a rule in the **security list** takes effect, and why an extra
NAT gateway port-forward rule is neither needed nor present.

---

## 9. Final state (2026-10-05/06)

Verified, except for the optional hardening below and the gaps explicitly
marked underneath.

| Component | State |
|---|---|
| Reachable at | `https://app.example.com` — Let's Encrypt, TLS 1.3, HTTP 200 |
| Password login | `200` with the correct password, `401` with a wrong one |
| Auth scope | `local` (external proxy, **no** OpenChamber tunnel) |
| Passkey | registered, `rp.id = app.example.com` |
| Passkey use | logged in from an **incognito window** — no password, no cookie |
| Port 443 | open (security list + ufw) |
| Port 3000 | **closed** (loopback only, Caddy backend) |
| Port 22 | open, SSH fallback |
| Reboot resilience | `openchamber.service` and `caddy` `enabled`, `Linger=yes` |

### Mobile devices (2026-10-05/06)

| Device | What was tested | Result |
|---|---|---|
| iPad, Safari | passkey login, chat, file editor, terminal | works |
| iPhone | "Add to Home Screen" | works |
| iPhone + iPad | **the same** passkey via iCloud Keychain | works |

Home screen installation does not require a web app manifest — Safari uses
the `apple-touch-icon` tags instead (180/167/152 px, all HTTP 200) together
with the `apple-mobile-web-app-*` meta tags. See also the README section
"Mobile".

### Explicitly not verified

So that section 9 does not read as "everything verified":

- **Android and desktop** — there the install prompt relies on
  `/manifest.json`, which is answered with 404 (no
  `<link rel="manifest">` in the HTML).
- **Background push notifications** end to end. The service worker registers
  a `push` handler, but no subscription was created or tested over VAPID.

### What the operator must do in the OCI console

The **security list** still contains an ingress rule for TCP **3000**. ufw
already blocks the port, but for cleanliness the rule should be removed
there as well:

**VCN -> Security Lists -> Default Security List for vcn-example ->
Ingress Rules** -> delete the rule with destination port `3000`.

After that, port 3000 is closed at every layer.

### Optional: hardening in a maintenance window

1. **Bind OpenChamber to loopback.** Service unit `ExecStart`:
   `--host 127.0.0.1` instead of `--host 0.0.0.0`. The cleanest measure,
   because then no open socket exists at all. Requires
   `systemctl --user restart openchamber.service` -> **kills running sessions.**
2. **Restrict the ingress rule to your own IP.** Instead of `0.0.0.0/0`, use
   only `198.51.100.77/32`. Deters port scanners, but assumes a static IP.
3. **Enable ufw rate limiting** (`ufw limit 443/tcp`) — practically useless
   for passkey login, because the rate-limit layer lives inside OpenChamber
   itself (`ui-auth.js`, 10 attempts / 5 min).
