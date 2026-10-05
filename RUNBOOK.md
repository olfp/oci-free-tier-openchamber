# OpenChamber Runbook — vm-arm01

Kurz-Referenz für den Betrieb von OpenChamber auf einer OCI-Arm-VM.
Zuletzt verifiziert: 2026-10-05.

> **Anonymisiertes Muster.** Alle Adressen, Domains und Hostnamen in diesem
> Dokument sind Platzhalter aus den RFC-Dokumentationsbereichen
> (`203.0.113.0/24`, `198.51.100.0/24`, `10.0.0.0/8`) bzw. aus
> `example.com`. Sie beschreiben **kein** echtes System. Die Befunde,
> Kommandos und Fehlerbilder sind dagegen real reproduziert worden.

| | |
|---|---|
| Host | `vm-arm01` (OCI, Region `eu-frankfurt-1`) |
| Architektur | `VM.Standard.A1.Flex`, aarch64, 1 OCPU / 6 GB |
| Private IP | `10.0.0.87` (VNIC `enp0s6`) |
| Public IP | `203.0.113.42` (auf der VNIC; **im Gast-OS nicht sichtbar**) |
| Domain | `app.example.com` (A-Record -> Public IP) |
| App-URL | **`https://app.example.com`** |
| TLS | Caddy 2.11.7, Let's Encrypt, autom. Erneuerung |
| Offene Ports | **22 (SSH), 443 (HTTPS)** — 3000 ist bewusst zu |
| Interner Port | 3000 nur noch ueber Loopback erreichbar |
| Passwort | `cat ~/.openchamber-ui-password` |
| Passkeys | moeglich — `rp.id = app.example.com` |
| Service-Unit | `~/.config/systemd/user/openchamber.service` |

---

## 1. Täglicher Betrieb

```bash
openchamber status        # schnellster Check -> "password: yes" muss dastehen
```

```bash
systemctl --user status openchamber.service --no-pager   # Details
systemctl --user is-enabled openchamber.service          # enabled -> startet bei Boot
systemctl --user is-active  openchamber.service          # active  -> laeuft jetzt
loginctl show-user ubuntu -p Linger --value              # yes -> ueberlebt Logout
```

### Logs

```bash
journalctl --user -u openchamber.service -f           # Service-Output mitverfolgen
journalctl --user -u openchamber.service --since boot # seit letztem Boot
openchamber logs                                     # App-eigenes Log
tail -f ~/.config/openchamber/logs/openchamber-3000.log
```

### Neustart / Stoppen

```bash
systemctl --user restart openchamber.service
systemctl --user stop    openchamber.service
systemctl --user start   openchamber.service
```

> **ACHTUNG:** `restart` beendet auch laufende Sessions. Die Agent-Prozesse liegen
> im Cgroup des Service — ein Neustart riss sie mit. Vorher offene Chats schliessen.

### Health-Check in einer Zeile

```bash
openchamber status && systemctl --user is-active openchamber.service \
  && curl -so /dev/null -w 'HTTP %{http_code}\n' --noproxy '*' http://127.0.0.1:3000/
```

Erwartet: `1 running runtime(s)`, `active`, `HTTP 200`.

---

## 2. Passwort

Passwort anzeigen:

```bash
cat ~/.openchamber-ui-password
```

Passwort **ändern** — immer mit dem `--ui-password`-Flag:

```bash
openchamber startup enable --port 3000 --host 0.0.0.0 --ui-password "$(openssl rand -base64 18)"
```

Das ist gleichzeitig der saubere Weg, **alle** Sessions auf einmal zu invalidieren:
Ein Passwortwechsel rotiert intern das `jwt-secret`, wodurch alle bestehenden
Session-Cookies ungültig werden.

> **Wichtig:** `startup.env` nicht loeschen!
> `~/.config/openchamber/startup.env` traegt das Passwort per `EnvironmentFile`
> in den Service. Fehlt die Datei, startet die VM nach einem Reboot **ohne
> Authentifizierung** auf oeffentlicher Adresse.
>
> Gegenpruefung:
> ```bash
> sudo grep -q '^OPENCHAMBER_UI_PASSWORD=' ~/.config/openchamber/startup.env \
>   && echo "ok: Passwort vorhanden" || echo "WARNUNG: kein Passwort!"
> ```

> **Nicht** die Env-Var-Variante `OPENCHAMBER_UI_PASSWORD=... openchamber startup enable`
> verwenden — die schreibt den Wert nicht verlaesslich nach `startup.env`.
> Nur der `--ui-password`-Flag tut das.

---

## 3. Firewall

Das OCI-Image liefert eine `REJECT`-Regel mit, die **vor** ufws Sprung hängt und
dessen Allow-Regeln wirkungslos macht:

```
-A INPUT -j REJECT --reject-with icmp-host-prohibited   <- feuert zuerst
-A INPUT -j ufw-before-input                            <- 3000er Regel hier unten
```

Deshalb war `ufw status` zwar gruen, die Regel aber tote Schrift
(0 Pakete). Die Regel wurde entfernt; ufws eigene INPUT-Policy ist `DROP`,
es wurde also nichts zusaetzlich geoeffnet.

```bash
sudo iptables -S INPUT | grep -q REJECT \
  && echo "FEHLER: Shadow-REJECT zurueck" || echo "ok"
sudo iptables -S ufw-user-input          # muss die 443er ACCEPT-Regel zeigen
```

Der Boot-Guard stellt das bei jedem Start wieder her:

```bash
systemctl is-enabled openchamber-firewall.service   # enabled
systemctl --user restart openchamber-firewall.service  # manuell neu anwenden
```

Originalregeln liegen unter `/etc/iptables/rules.v4.bak-openchamber`.

### Aktueller Stand: 3000 zu, nur 443 offen

```bash
sudo ufw status | grep -E '22|443|3000'    # erwartet: nur 443
```

Port 3000 ist bewusst **nicht** mehr in ufw. OpenChamber laeuft weiter auf
`0.0.0.0:3000`, aber die einzige Erreichbarkeit ist jetzt der Weg ueber
Caddy auf Loopback:

```
Internet ──443──> Caddy ──loopback──> 127.0.0.1:3000 (OpenChamber)
```

Deshalb **nicht** einfach die Regel wieder auf `127.0.0.0/8` setzen — das
wuerde zwar 3000 von aussen schliessen, aber Caddy laeuft auf `0.0.0.0`.
Der Port bleibt zu, weil ufw `INPUT DROP` fährt.

Backup der ufw-Regeln: `/etc/ufw/user.rules.bak-openchamber`.

> **Warum OpenChamber trotzdem auf `0.0.0.0` laeuft:** `localhost` als Bind
> wäre sauberer, erfordert aber einen Neustart des Service — und damit das
> Ende aller laufenden Sessions. Der Versuch lohnt erst in einem
> Wartungsfenster. Bis dahin ist die Firewall die Absicherung.

---

## 4. Auth-Scope verstehen

Das ist der verwirrendste Teil. Jeder Request wird nach dem `Host`-Header
klassifiziert (`server/lib/opencode/tunnel-auth.js`, `classifyRequestScope`):

| Bedingung | Scope | Verhalten |
|---|---|---|
| Host == aktiver Tunnel-Host | `tunnel` | nur Tunnel-Session-Cookie |
| localhost / Loopback | `local` | normales UI-Passwort |
| **kein Tunnel aktiv** | `local` | normales UI-Passwort |
| alles andere | `unknown-public` | nur Tunnel-Session-Cookie |

Konsequenzen:

- `unknown-public` und `tunnel` werden von `requireTunnelSession()` hart gesperrt.
- **Ein UI-Passwort hilft dort nicht.** Passwort-Login wird in diesem Scope
  explizit verweigert (`core-routes.js`) — daher die Meldung
  *„Tunnel access required"*, sobald der Cloudflare-Tunnel laeuft.
- `activeTunnelId` liegt nur im RAM. **Ein Neustart des Service setzt ihn auf
  `null`** und loest das Problem nebenbei.

Passwort-TTL (empirisch gemessen):

| Login | `Max-Age` |
|---|---|
| normal | 43200 s = 12 Stunden |
| `trustDevice: true` | 604800 s = 7 Tage |

„Trustthis device" ist also nur ein laenger lebendes Cookie. Es gibt **keine**
serverseitige Geraete-Liste und keinen Logout-Endpoint — die Sessions sind
stateless JWTs, die nur per `jwtVerify` gegen `jwt-secret` geprueft werden.

Tunnel-Status:

```bash
openchamber tunnel status --all
openchamber tunnel stop --port 3000     # nur Tunnel, Server laeuft weiter
```

---

## 5. HTTPS / Caddy

Caddy terminiert TLS und leitet auf `127.0.0.1:3000` weiter.

```bash
systemctl is-active caddy                        # active
systemctl is-enabled caddy                       # enabled
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo journalctl -u caddy -f                      # Zertifikatsausstellung beobachten
```

Aktuelle Caddyfile (`/etc/caddy/Caddyfile`):

```
https://app.example.com {
	reverse_proxy 127.0.0.1:3000
}
```

Zertifikat: Let's Encrypt, gueltig bis 2027-01-03, automatische Erneuerung
(Let's_Enschluessel erneuert automatisch ~30 Tage vor Ablauf).

> **Das explizite `https://` ist Pflicht.** Mit blossem Hostnamen
> (`app.example.com { ... }`) hat Caddy die Domain fuer Auto-HTTPS als ungeeignet
> eingestuft und den Server **still auf `:80` heruntergestuft** — ohne
> Zertifikat, ohne Fehler beim Start. Log-Meldung:
> `server is listening only on the HTTP port, so no automatic HTTPS will be applied`
> Das Schema erzwingt TLS und loest die Zertifikatsanfrage aus.

Port 80 wird nicht gebraucht: die Validierung laeuft ueber **TLS-ALPN-01**,
das vollstaendig ueber Port 443 geht. Deshalb ist Port 80 in ufw bewusst zu.

### OCI-Seite: was noetig war

Eine einzige Ingress Rule in der **Security List** der VCN:

| Feld | Wert |
|---|---|
| Stateless | No |
| Source CIDR | `0.0.0.0/0` |
| IP Protocol | TCP |
| Source Port Range | All |
| Destination Port Range | `443` |
| Description | `HTTPS via Caddy (app.example.com)` |

**Keine** NAT-Gateway-Port-Forward-Regel. Die Public IP der VNIC wird per
1:1-NAT durchgelassen (siehe Abschnitt 8).

### Passkeys

Erst ueber HTTPS sinnvoll — WebAuthn verlangt eine sichere Herkunft und eine
Domain als `rp.id`. Gegenprobe von der VM:

```bash
# liefert u.a.  "rp":{"name":"OpenChamber","id":"app.example.com"}
curl -s -X POST https://app.example.com/auth/passkey/register/options \
  -H 'Content-Type: application/json' -H 'Origin: https://app.example.com'
```

Registrierung: im Browser einloggen → Passkey anlegen.

---

## 6. Passwort-Pflicht testen

Am einfachsten und realistischsten: **Privates/Incognito-Fenster**.

Ein privates Fenster hat kein Cookie und verhaelt sich exakt wie ein neues
Geraet von aussen — damit sieht man die echte Passwort-Abfrage.

Alle Sessions gleichzeitig wegwerfen (beendet laufende Sessions):

```bash
systemctl --user restart openchamber.service
```

---

## 7. Troubleshooting

| Symptom | Ursache | Loesung |
|---|---|---|
| *„Tunnel access required"* | Tunnel laeuft → Scope `unknown-public` | Tunnel stoppen bzw. Service neu starten (Abschnitt 4) |
| Browser zeigt Passwort nicht / 401 | falsches Passwort | Rate-Limit beachten: 10 Versuche / 5 Min, dann **15 Min** Sperre. Nach 15 Min erneut. |
| `curl https://app.example.com` → Verbindungsfehler, aber `:3000` geht | Security List ohne Ingress Rule fuer **443** | Ingress Rule TCP 443 `0.0.0.0/0` in der Security List ergaenzen (Abschnitt 5) |
| `curl https://app.example.com` → TLS-Fehler / Caddy laeuft nur auf `:80` | Caddyfile ohne `https://`-Schema → Auto-HTTPS uebersprungen | `https://app.example.com` eintragen und `sudo systemctl reload caddy` |
| Zertifikat wird nicht ausgestellt | Validierung scheitert | `sudo journalctl -u caddy -f`. Bei `tls-alpn-01` Port 443 in Security List **und** ufw pruefen |
| Verbindung laeuft in Timeout | Public IP fehlt, oder Subnet privat / kein Internet Gateway | OCI-Konsole: VNIC `Attached VNICs` → `IP administration` → *Ephemeral public IP*. Voraussetzungen: Subnet **public** und VCN mit **Internet Gateway**. Zusaetzlich muessen **NSG *und*** Security List den Port erlauben. |
| `No route to host` von innen | Hairpin-Test gegen die eigene oeffentliche IP | Nicht aussagekraeftig — OCI spiegelt Hairpin je nach Port unterschiedlich. Die 3000er-Regel antwortet, 443 moeglicherweise nicht. Extern testen. |
| Nach Reboot ohne Passwort | `startup.env` fehlt/geloescht | Abschnitt 2 — Passwort neu setzen |
| Nach Reboot Port 3000 zu | Boot-Guard fehlt | `systemctl enable openchamber-firewall.service` |
| `openchamber status` leer | Service laeuft nicht | `journalctl --user -u openchamber.service -n 50` |

Ein SSH-Login ist ueber Port 22 weiterhin **immer** erreichbar — als
Rueckfallweg bei Problemen mit OpenChamber gedacht.

---

## 8. Woher das alles kam

1. OCI-`REJECT` ueberschrieb ufw → Port 3000 trotz `ufw status` zu.
2. Cloudflare-Quick-Tunnel aktiv → direkter Zugriff auf Port 3000 gesperrt.
3. Service lief nicht unter systemd → kein Ueberleben von Reboot/Logout.
4. Passwort fehlte (`--host 0.0.0.0` ohne `--ui-password`) → Agent mit
   Shell-Zugriff offen im Internet.
5. Caddyfile ohne `https://`-Schema → Auto-HTTPS stillschweigend deaktiviert.

### Falle: Public IP ist im Gast-OS unsichtbar

In OCI wird eine der VNIC zugewiesene Public IP per **1:1-NAT auf Fabric-Ebene**
umgesetzt. Sie taucht deshalb **nicht** in `ip addr` auf:

```
$ ip -br addr show enp0s6
enp0s6   UP   10.0.0.87/24        # <- Public IP 203.0.113.42 fehlt hier
```

Aus demselben Grund zeigt `SSH_CONNECTION` immer die private IP als Ziel,
obwohl du dich ueber die oeffentliche IP verbindest:

```
198.51.100.77 65118 10.0.0.87 22
```

**Konsequenz:** `ip addr` ist **kein** Beleg dafuer, dass eine Instanz nur eine
private Adresse hat. Die massgebliche Quelle ist die OCI-Console
(Instance → Attached VNICs → IP administration) bzw. die Security List.

Erhoehungsweg laeuft ausschliesslich ueber die Security List
(Subnet private oder public, kein loadbalancer noetig):

```
Internet → Security List (Ingress) → VNIC/1:1-NAT → 10.0.0.87
```

Deshalb wird eine Regel in der **Security List** wirksam und eine
zusaetzliche NAT-Gateway-Port-Forward-Regel ist weder noetig noch vorhanden.
---

## 9. Abschluss-Status (2026-10-05)

Alles verifiziert, nichts offen ausser der optionalen Härtung unten.

| Komponente | Zustand |
|---|---|
| Erreichbar | `https://app.example.com` — Let's Encrypt, TLS 1.3, HTTP 200 |
| Passwort-Login | `200` bei korrektem, `401` bei falschem Passwort |
| Auth-Scope | `local` (externer Proxy, **kein** OpenChamber-Tunnel) |
| Passkey | registriert, `rp.id = app.example.com` |
| Port 443 | offen (Security List + ufw) |
| Port 3000 | **geschlossen** (nur noch Loopback, Caddy-Backend) |
| Port 22 | offen, SSH-Rückfallweg |
| Reboot-Festigkeit | `openchamber.service` und `caddy` `enabled`, `Linger=yes` |

### Was der Nutzer in der OCI-Console tun muss

Die **Security List** hat weiterhin eine Ingress Rule für TCP **3000**. ufw
blockt den Port bereits, aber sauber ist es, die Regel dort auch zu entfernen:

**VCN → Security Lists → Default Security List for vcn-example →
Ingress Rules** → Regel mit Destination Port `3000` löschen.

Danach ist Port 3000 auf allen Ebenen zu.

### Optional: Härtung in einem Wartungsfenster

1. **OpenChamber auf Loopback binden.** Service-Unit `ExecStart`:
   `--host 127.0.0.1` statt `--host 0.0.0.0`. Sauberste Massnahme, weil dann
   gar kein offener Socket mehr existiert. Erfordert
   `systemctl --user restart openchamber.service` → **beendet laufende Sessions.**
2. **Ingress Rule auf die eigene IP beschränken.** Statt `0.0.0.0/0` nur
   `198.51.100.77/32`. Wirkt gegen Portscanner, setzt aber eine statische
   IP voraus.
3. **UFW-Ratenbegrenzung aktivieren** (`ufw limit 443/tcp`) — bei
   Passkey-Login praktisch nutzlos, weil die Rate-Limit-Schicht in
   OpenChamber selbst sitzt (`ui-auth.js`, 10 Versuche / 5 Min).
