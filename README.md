# OpenChamber auf einer OCI Free-Tier-Arm-VM

Betrieb von [OpenChamber](https://openchamber.ai) auf einer kostenlosen
Oracle-Cloud-Arm-Instanz — erreichbar über HTTPS mit Passwort **und**
Passkey, ohne Cloudflare-Tunnel und ohne öffentliche IP im Betriebssystem.

Das hier ist die verdichtete Fassung eines Runbooks, das beim Aufbau einer
solchen Instanz entstanden ist. Alle Adressen sind Platzhalter; die Befunde
sind echt.

```
Internet ──443──> OCI Security List ──> VNIC (1:1-NAT) ──> Caddy
                                                              │
                                        127.0.0.1:3000 ◄─────┘
                                                        OpenChamber
```

## Was dieses Repo dokumentiert

Vier Fehlerbilder, die zusammen fast zwei Tage gekostet haben und die sich
alle in den Logdateien **nicht** als Fehler darstellen:

| # | Symptom | Ursache |
|---|---|---|
| 1 | `ip addr` zeigt keine öffentliche IP | OCI setzt die VNIC-Public-IP per **1:1-NAT auf Fabric-Ebene** um — sie ist im Gast-OS unsichtbar |
| 2 | ufw erlaubt Port 3000, nichts kommt an | Das OCI-Image setzt eine `REJECT`-Regel **vor** ufws Sprung; alle Allow-Regeln werden tote Schrift |
| 3 | *„Tunnel access required"* | Ein aktiver OpenChamber-Tunnel setzt den Auth-Scope auf `unknown-public`, was Passwort-Login **und** Passkeys hart verweigert |
| 4 | Caddy lauscht auf `:80`, kein Zertifikat, kein Fehler | Eine Caddyfile **ohne** `https://`-Schema lässt Caddy Auto-HTTPS stillschweigend deaktivieren |

Nummer 4 ist die heimtückischste: die Konfiguration ist nicht kaputt, sie ist
nur nicht die, die man geschrieben hat — der Dienst startet trotzdem, eben
ohne HTTPS. Details in Abschnitt 5 des Runbooks.

## Passkeys

Der eigentliche Grund für den ganzen Aufbau. WebAuthn braucht eine echte
Domain als `rp.id`; eine IP-Adresse akzeptieren Browser nicht. Über eine
extern aufgeschaltete Domain wird aus

```json
{"rp": {"name": "OpenChamber", "id": "203.0.113.42"}}
```

das funktionierende

```json
{"rp": {"name": "OpenChamber", "id": "app.example.com"}}
```

Passkeys erfordern dabei **zwingend** den externen Reverse-Proxy statt
`openchamber tunnel` — siehe Befund 3.

## Voraussetzungen auf der OCI-Seite

Zwei Dinge in der Console, sonst kommt nichts an:

1. **Security List**, nicht NAT-Gateway: eine Ingress Rule für TCP 443
   (`0.0.0.0/0`, stateless `No`). Eine Security List ist ausreichend, weil
   die Public IP der VNIC 1:1 durchgereicht wird.
2. **Port 80 wird nicht gebraucht.** Die Zertifikatsvalidierung läuft über
   TLS-ALPN-01 und damit vollständig über 443.

Beim Schließen des App-Ports genügt es, die Rule für Port 3000 zu löschen —
es ist **keine** NAT-Port-Forward-Regel im Spiel, die man suchen müsste.

## Aufbau in Kurzform

```bash
# OpenChamber mit Passwort und Autostart
openchamber startup enable --port 3000 --host 0.0.0.0 \
  --ui-password "$(openssl rand -base64 18)"

# OCI-Schatten-REJECT entfernen (macht sonst ufw wirkungslos)
iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited

# Caddy mit explizitem Schema
#   https://app.example.com {
#       reverse_proxy 127.0.0.1:3000
#   }
```

Das vollständige Runbook mit allen verifizierten Kommandos, den Rate-Limit-Werten,
dem Auth-Scope-Mechanismus und einer Troubleshooting-Tabelle steht in
[RUNBOOK.md](RUNBOOK.md).

## Zwei Warnungen

**`--ui-password` ist ein Flag, keine Environment-Variable.** Mit
`OPENCHAMBER_UI_PASSWORD=… openchamber startup enable` wird der Wert nicht
nach `startup.env` geschrieben — die VM startet dann nach einem Reboot **ohne
Authentifizierung** auf öffentlicher Adresse.

**`systemctl --user restart openchamber.service` beendet laufende Sessions.**
Die Agent-Prozesse liegen im Cgroup des Service. Vor jedem Neustart offene
Chats schließen.

## Inhalt

| Datei | |
|---|---|
| [RUNBOOK.md](RUNBOOK.md) | Betrieb, Passwort, Firewall, Auth-Scope, HTTPS, Troubleshooting |

## Lizenz

Siehe [LICENSE](LICENSE).
