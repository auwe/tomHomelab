# Konzept: Tailscale, Docker und Home Assistant auf Thomas' Synology NAS

| | |
|---|---|
| **Ziel** | Sicherer Fernzugriff auf Thomas' Homelab über Tailscale, ohne offene Ports. Docker (Container Manager) und Home Assistant laufen auf dem NAS. |
| **System** | Synology NAS, per DDNS erreichbar unter `goelles.myds.me` |
| **Status** | Entwurf. Die Rahmenbedingungen (Abschnitt 2) sind noch **nicht geprüft**. |
| **Stand** | 2026-11-10 |

---

## 1. Zielbild und Grundsätze

```
   Mike / Thomas (Laptop, Handy)                    Thomas' Zuhause (LAN, z. B. 192.168.x.0/24)
   ┌─────────────────────┐                          ┌───────────────────────────────────────────┐
   │ Tailscale-Client    │                          │ Synology NAS                               │
   │                     │   WireGuard (Tailscale)  │  ├─ Tailscale  (Subnet-Router, opt. Exit)  │
   │                     │ ◄──────────────────────► │  ├─ Container Manager (Docker)             │
   └─────────────────────┘   keine offenen Ports    │  │    └─ Home Assistant (:8123)            │
                                                    │  └─ DSM (:5001)                            │
                                                    │                                            │
                                                    │ Router, Drucker, IoT-Geräte, …  ◄── über   │
                                                    │                                Subnet-Route│
                                                    └───────────────────────────────────────────┘
```

**Best-Practice-Grundsätze**

1. **Keine eingehenden Portfreigaben.** Tailscale baut Verbindungen ausgehend auf. Bestehende Portweiterleitungen für DSM (5000/5001), SSH oder Home Assistant werden nach erfolgreichem Test **geschlossen**. `goelles.myds.me` bleibt als DDNS-Name bestehen, ist aber nicht mehr der Zugriffsweg.
2. **Least Privilege per ACL.** Wer welche Ziele im Netz erreichen darf, wird zentral in der Tailnet-Policy geregelt, nicht „alles für alle“.
3. **NAS als Subnet-Router.** Das NAS veröffentlicht das Heimnetz ins Tailnet. So sind auch Geräte ohne Tailscale (Router, Drucker, Kameras) erreichbar.
4. **Home Assistant mit Host-Netzwerk.** Nötig für Geräteerkennung (mDNS/SSDP, HomeKit, Chromecast usw.).
5. **Konfiguration als Code.** Docker-Compose-Dateien und die Tailnet-Policy liegen versioniert in diesem Repository (ohne Secrets).
6. **Backups vor Änderungen** und danach automatisiert nach 3-2-1.

---

## 2. Rahmenbedingungen prüfen (Checkliste)

Diese Punkte zuerst mit Thomas klären oder am NAS ablesen. Die Befehle laufen per SSH auf dem NAS (DSM → Systemsteuerung → Terminal & SNMP → SSH vorübergehend aktivieren) oder sind im DSM unter **Systemsteuerung → Infocenter** sichtbar.

### 2.1 Hardware und DSM

| ✔ | Prüfpunkt | Wie prüfen | Anforderung / Bewertung |
|---|---|---|---|
| ☐ | **NAS-Modell** | Infocenter → Allgemein | Notieren. Mit Synology-Kompatibilitätsliste für *Container Manager* und *Virtual Machine Manager* abgleichen. |
| ☐ | **CPU-Architektur** | `uname -m`, `grep -m1 "model name" /proc/cpuinfo` | `x86_64` (Intel/AMD): alles möglich. `aarch64`: Container Manager nur auf einigen Modellen, **keine** Home-Assistant-VM. `armv7`: kein Docker, Konzept nicht umsetzbar. |
| ☐ | **DSM-Version** | `cat /etc.defaults/VERSION` | **DSM 7.2 oder neuer** (Container Manager inkl. Compose-Projekte). Aktuelle Sicherheitsupdates eingespielt. |
| ☐ | **RAM** | `free -h` / Infocenter | Minimum 2 GB (nur Tailscale + HA-Container, knapp). **Empfohlen ≥ 4 GB** für HA-Container plus weitere Container. **≥ 8 GB**, wenn HA als VM laufen soll. RAM-Erweiterung beim Modell prüfen. |
| ☐ | **Speicher / Dateisystem** | Speicher-Manager, `df -h /volume1` | Volume mit **Btrfs** (für Snapshots). ≥ 20 GB frei für Docker-Images und HA-Datenbank. Möglichst SSD-Cache oder SSD-Volume für die HA-Datenbank. |
| ☐ | **TUN-Device** | `ls -l /dev/net/tun` | Muss existieren (für Subnet-Router/Exit-Node über Kernel-Netzwerk). |
| ☐ | **USB-Funkstick** (Zigbee/Z-Wave/Thread) | Welcher Stick? `ls /dev/ttyUSB* /dev/ttyACM*` | DSM 7 bringt nur wenige USB-Serial-Treiber mit. Netzwerkbasierte Koordinatoren (z. B. Ethernet-Zigbee-Gateways) sind auf Synology deutlich robuster. |
| ☐ | **USV / Stromausfall** | Systemsteuerung → Hardware & Energie | Empfehlenswert, da HA und Subnet-Router sonst bei Stromausfall ungeordnet weg sind. |

### 2.2 Netzwerk

| ✔ | Prüfpunkt | Wie prüfen | Anforderung / Bewertung |
|---|---|---|---|
| ☐ | **LAN-Subnetz** | `ip -4 addr`, `ip route` | Notieren (z. B. `192.168.1.0/24`). **Darf sich nicht mit anderen Netzen im selben Tailnet überschneiden** (z. B. Mikes eigene Standorte). Bei Kollision das Subnetz am Router von Thomas ändern, bevor die Route veröffentlicht wird. |
| ☐ | **Feste IP des NAS** | Router-DHCP / Systemsteuerung → Netzwerk | Statische IP oder DHCP-Reservierung. |
| ☐ | **Router-Modell** | – | Notieren. Für direkte Verbindungen: UPnP/NAT-PMP oder zumindest „normales“ NAT. Kein Pflichtpunkt, sonst läuft Tailscale über DERP-Relays (langsamer, aber funktionsfähig). |
| ☐ | **Ausgehende Ports** | – | Ausgehend **UDP 41641** und **UDP 3478** sowie **TCP 443** erlaubt. **Eingehend wird nichts benötigt.** |
| ☐ | **Bestehende Portfreigaben** | Router-Oberfläche | Liste aller Weiterleitungen (5000, 5001, 22, 8123, 443 …). Diese werden in Schritt 9 geschlossen. |
| ☐ | **Doppel-NAT / CGNAT** | Router-WAN-IP mit öffentlicher IP vergleichen | Nur zur Info. Tailscale funktioniert auch hinter CGNAT. |
| ☐ | **Upload-Bandbreite** | Speedtest bei Thomas | Relevant, falls das NAS als **Exit-Node** genutzt werden soll. |
| ☐ | **Port 8123 frei** | `netstat -tlnp \| grep 8123` | Darf auf dem NAS nicht belegt sein. |
| ☐ | **DSM-Firewall** | Systemsteuerung → Sicherheit → Firewall | Status notieren; Regeln in Schritt 9 anpassen. |

### 2.3 Organisation und Entscheidungen

| ✔ | Entscheidung | Optionen | Empfehlung |
|---|---|---|---|
| ☐ | **Wem gehört das Tailnet?** | a) Thomas hat ein eigenes Tailnet, Mike wird als Benutzer eingeladen. b) Das NAS tritt Mikes Tailnet bei. | **a)** Thomas bleibt Eigentümer seiner Infrastruktur, Mike bekommt per Einladung und ACL genau die nötigen Rechte. |
| ☐ | **Home Assistant als Container oder als VM?** | siehe Abschnitt 5.1 | Container, wenn RAM < 8 GB oder CPU ≠ x86_64. Sonst VM (HA OS) erwägen. |
| ☐ | **Exit-Node?** | ja / nein | Nur aktivieren, wenn tatsächlich gebraucht (z. B. sicheres Surfen unterwegs über Thomas' Anschluss). |
| ☐ | **Admin-Zugang** | DSM-Admin-Konto für die Einrichtung | Eigenes, benanntes Admin-Konto mit 2FA; Standardkonto `admin` deaktiviert. |
| ☐ | **Backup-Ziel** | USB-Disk, zweites NAS, Cloud (C2, S3 …) | Mindestens ein Ziel außerhalb des NAS, besser zusätzlich offsite. |

---

## 3. Vorbereitung (Phase 0)

1. **DSM aktualisieren** (Systemsteuerung → Aktualisieren & Wiederherstellen) und Neustart abwarten.
2. **Konfigurations-Backup** von DSM ziehen (Systemsteuerung → Aktualisieren & Wiederherstellen → Konfigurationssicherung → Sichern).
3. **Snapshot** des Volumes anlegen (Paket *Snapshot Replication*), falls Btrfs.
4. **Kontohygiene:** eigenes Admin-Konto, **2FA** aktiv, Konto `admin` deaktiviert, *Automatische Blockierung* aktiv (Systemsteuerung → Sicherheit → Konto).
5. **Freigegebenen Ordner `docker` anlegen** (Systemsteuerung → Freigegebener Ordner), z. B. `/volume1/docker`. Wird bei der Installation von Container Manager auch automatisch angelegt.

---

## 4. Tailscale installieren und einrichten (Phase 1)

### 4.1 Tailnet vorbereiten (Browser, admin.tailscale.com)

1. Thomas legt einen Tailscale-Account an (empfohlen mit eigenem Identity-Provider, z. B. Google/Microsoft/Apple mit 2FA).
2. Mike über **Users → Invite users** einladen (Rolle *Member*, bei Bedarf *Admin*).
3. Unter **Access controls** die Policy aus Abschnitt 4.5 einspielen, **bevor** das NAS beitritt (damit Tags von Beginn an greifen).
4. Einen **Auth-Key** erzeugen (Settings → Keys → *Generate auth key*): nicht wiederverwendbar, *Pre-approved*, Tag `tag:nas`. Alternativ interaktiv per Browser-Login anmelden.

### 4.2 Paket installieren

1. DSM → **Paket-Zentrum** → nach „Tailscale“ suchen → **Installieren**.
   *Falls nicht gelistet oder veraltet:* passendes `.spk` für die CPU-Architektur und DSM 7 von <https://pkgs.tailscale.com/stable/#spks> laden und über Paket-Zentrum → *Manuelle Installation* einspielen.
2. Paket öffnen → Anmelden (Browser-Login) **oder** per SSH mit Auth-Key:
   ```bash
   sudo tailscale up --authkey=tskey-auth-XXXXXXXX --hostname=nas-thomas
   ```

### 4.3 Kernel-Netzwerk aktivieren (wichtig für DSM 7)

Unter DSM 7 läuft das Tailscale-Paket ohne Root-Rechte und nutzt standardmäßig „Userspace Networking“. Damit sind **ausgehende Verbindungen vom NAS ins Tailnet und Subnet-Routing nicht zuverlässig**. Abhilfe laut Tailscale-Doku:

```bash
sudo /var/packages/Tailscale/target/bin/tailscale configure-host
sudo synosystemctl restart pkgctl-Tailscale.service
```

Da diese Einstellung einen Neustart nicht überlebt, als **Aufgabe beim Hochfahren** hinterlegen:

DSM → Systemsteuerung → **Aufgabenplaner** → Erstellen → *Ausgelöste Aufgabe* → *Benutzerdefiniertes Skript*
- Benutzer: `root`
- Ereignis: `Hochfahren`
- Skript:
  ```bash
  /var/packages/Tailscale/target/bin/tailscale configure-host; synosystemctl restart pkgctl-Tailscale.service
  ```

### 4.4 Subnet-Router und optional Exit-Node

```bash
# Heimnetz veröffentlichen (Subnetz aus Checkliste 2.2 einsetzen)
sudo tailscale set --advertise-routes=192.168.1.0/24

# Fremde Routen auf dem NAS NICHT annehmen (verhindert Routing-Konflikte im Heimnetz)
sudo tailscale set --accept-routes=false

# Optional: NAS als Exit-Node anbieten
sudo tailscale set --advertise-exit-node

# Automatische Updates des Tailscale-Clients
sudo tailscale set --auto-update
```

Danach in der Admin-Konsole beim Gerät `nas-thomas`:
- **Edit route settings** → Subnet-Route (und ggf. Exit-Node) **freigeben**. Mit `autoApprovers` in der Policy (siehe unten) geschieht das automatisch.
- **Disable key expiry** aktivieren (sonst verliert das NAS nach 180 Tagen die Verbindung).

> **Hinweis Subnetz-Überschneidung:** Wenn Mike in seinem eigenen Netz dasselbe Subnetz nutzt wie Thomas, ist das Heimnetz von Thomas für ihn nicht sauber erreichbar. Lösung: Subnetz bei Thomas ändern oder Tailscale *4via6* nutzen. Deshalb Checkliste 2.2 ernst nehmen.

### 4.5 Zugriffsregeln (Tailnet-Policy)

Ausgangspunkt, im Admin-Panel unter *Access controls* einfügen und anpassen. Die Mailadressen sind Platzhalter, das Subnetz ebenso.

```jsonc
{
  "groups": {
    "group:admins": ["thomas@example.com", "mike@example.com"],
    "group:family": ["thomas@example.com"]
  },

  "tagOwners": {
    "tag:nas": ["group:admins"]
  },

  "autoApprovers": {
    "routes":   { "192.168.1.0/24": ["tag:nas"] },
    "exitNode": ["tag:nas"]
  },

  "grants": [
    // Admins: Vollzugriff auf NAS und Heimnetz
    { "src": ["group:admins"], "dst": ["tag:nas", "192.168.1.0/24"], "ip": ["*"] },

    // Familie: nur Home Assistant und DSM-Weboberfläche
    { "src": ["group:family"], "dst": ["tag:nas"], "ip": ["tcp:8123", "tcp:5001", "tcp:443"] },

    // Exit-Node für Admins (nur falls aktiviert)
    { "src": ["group:admins"], "dst": ["autogroup:internet"], "ip": ["*"] }
  ],

  "tests": [
    { "src": "mike@example.com", "accept": ["tag:nas:5001", "192.168.1.1:443"] }
  ]
}
```

Weitere Empfehlungen im Admin-Panel:
- **MagicDNS** aktivieren → NAS erreichbar als `nas-thomas.<tailnet>.ts.net`.
- **HTTPS-Zertifikate** aktivieren (für `tailscale serve`, siehe 5.4).
- **Device approval** einschalten, damit neue Geräte erst freigegeben werden müssen.
- Optional **Tailnet Lock** für zusätzliche Absicherung.

### 4.6 Test

Von Mikes Gerät (Tailscale aktiv, nicht im Heimnetz von Thomas, z. B. Handy im Mobilfunk):

```bash
tailscale status                     # nas-thomas ist online
tailscale ping nas-thomas            # "direct" ist ideal, "via DERP" funktioniert auch
curl -k https://nas-thomas:5001      # DSM über Tailnet
ping 192.168.1.1                     # Router über Subnet-Route
```

---

## 5. Docker und Home Assistant (Phase 2)

### 5.1 Entscheidung: Container oder VM

| Kriterium | **HA Container** (Container Manager) | **HA OS als VM** (Virtual Machine Manager) |
|---|---|---|
| CPU | x86_64 oder unterstütztes ARM64 | nur x86_64 mit VMM-Unterstützung |
| RAM-Bedarf | ca. 0,5–1 GB | 2–4 GB für die VM, NAS insgesamt ≥ 8 GB |
| Add-ons (Apps) | **nein** (Zusatzdienste als eigene Container) | ja |
| Updates | Image neu ziehen | per Klick in HA |
| Backups | HA-Backup + Ordner `/volume1/docker/homeassistant` | HA-Backup + VM-Snapshots |
| USB-Sticks | schwierig (Treiber unter DSM) | per USB-Passthrough, meist einfacher |
| Passt zu „Docker auf dem NAS“ | ✅ | Docker läuft zusätzlich, HA aber getrennt |

**Empfehlung:** Für den Start den **Container** (unten ausführlich beschrieben). Ein späterer Umzug in eine VM ist über HA-Backup/Restore möglich.

### 5.2 Container Manager installieren

1. Paket-Zentrum → **Container Manager** → Installieren.
2. Ordnerstruktur anlegen (File Station oder SSH):
   ```
   /volume1/docker/
   └── homeassistant/
       ├── compose.yaml
       └── config/
   ```

### 5.3 Home Assistant per Compose-Projekt

`/volume1/docker/homeassistant/compose.yaml`:

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    network_mode: host           # nötig für Geräteerkennung (mDNS, SSDP, HomeKit …)
    restart: unless-stopped
    environment:
      - TZ=Europe/Vienna
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro    # Bluetooth (falls genutzt)
    # privileged: true            # nur aktivieren, wenn USB/Bluetooth es erfordern
    # devices:
    #   - /dev/ttyUSB0:/dev/ttyUSB0
```

Container Manager → **Projekt** → Erstellen → Pfad `/volume1/docker/homeassistant` → vorhandene `compose.yaml` verwenden → Erstellen.

Danach im Browser über das Tailnet: `http://nas-thomas:8123` → Onboarding durchführen, Admin-Benutzer anlegen, **2FA (TOTP) im HA-Profil aktivieren**.

**Best Practices für den Container:**
- Image-Tag `stable` nutzen und Updates bewusst einspielen (vorher HA-Backup). Bei Bedarf auf eine konkrete Version pinnen.
- Recorder-Datenbank klein halten (`purge_keep_days`), um Disk-I/O auf HDDs zu begrenzen.
- Keine weiteren Dienste auf Port 8123.

### 5.4 Home Assistant komfortabel und mit HTTPS über Tailscale

Statt Portfreigabe veröffentlicht Tailscale HA **nur im Tailnet** mit gültigem Zertifikat:

```bash
sudo tailscale serve --bg --https=443 http://127.0.0.1:8123
```

→ erreichbar als `https://nas-thomas.<tailnet>.ts.net`. Damit HA den Proxy akzeptiert, in `config/configuration.yaml` ergänzen und HA neu starten:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
```

Für die **HA-Companion-App** auf dem Handy diese URL als interne und externe URL eintragen (Tailscale auf dem Handy aktiv lassen).

> **Kein `tailscale funnel`** für Home Assistant, außer es gibt einen zwingenden Grund. Funnel macht den Dienst öffentlich im Internet erreichbar.

### 5.6 Home Assistant als MCP-Server für Claude anbinden

Stand 2026-11-11: HA ist im Tailnet als `https://homeassistant.taila5009f.ts.net` erreichbar, der MCP-Endpunkt ist `https://homeassistant.taila5009f.ts.net/api/mcp` (HA-Integration „Model Context Protocol Server“).

**Stand 2026-11-11 abends:** Funnel aktiv, Webhook-URL öffentlich erreichbar und verlangt OAuth (HA-Login). HA hört bei Thomas auf Port 80 (LAN 192.168.1.206, HAOS-VM in VMM).

**Erster Test:** Aus der Claude-Cloud ist die URL nicht erreichbar (ausgehende Verbindung vom Proxy blockiert, kein öffentlicher DNS-Eintrag sichtbar). Ein Test aus dem Tailnet steht noch aus.

**Wichtig:** Ein *Custom Connector in claude.ai* wird von Anthropics Servern aus aufgerufen, nicht vom eigenen Rechner. Eine reine Tailnet-Adresse (MagicDNS, `tailscale serve`) funktioniert dort daher nicht. Möglichkeiten:

| Variante | Erreichbarkeit | Bewertung |
|---|---|---|
| A. Claude Code / Claude Desktop auf einem Rechner im Tailnet, MCP lokal eintragen | nur Tailnet | sicherste Variante, kein öffentlicher Zugang |
| B. `tailscale funnel` nur für den MCP-Pfad + Custom Connector in claude.ai | öffentlich | nur mit Authentifizierung (HA-OAuth bzw. Long-Lived Token), widerspricht Grundsatz aus 5.4 → bewusste Entscheidung nötig |
| C. ha-mcp-Add-on mit geheimer URL, wie bei den anderen Standorten | je nach Veröffentlichung | einheitlich mit bestehenden Connectors, geheime URL wie ein Passwort behandeln |


**Entscheidung 2026-11-11: Variante C** (wie bei Karin: Add-on „Home Assistant MCP Server“ + „Webhook Proxy for HA MCP“ aus dem Repository `https://github.com/homeassistant-ai/ha-mcp`).

1. Einstellungen → Add-ons → Add-on-Store → ⋮ → Repositories → `https://github.com/homeassistant-ai/ha-mcp` hinzufügen.
2. „Home Assistant MCP Server“ installieren, starten, „Beim Start starten“ und „Watchdog“ aktivieren. Im Log steht die geheime MCP-URL.
3. „Webhook Proxy for HA MCP“ installieren und starten. Er leitet einen geheimen Webhook-Pfad von HA auf den MCP-Server weiter; die öffentliche URL steht im Log.
4. HA öffentlich per HTTPS erreichbar machen (alternativ Cloudflared oder Nabu Casa). Danach ist die HA-Anmeldeseite öffentlich → starke Passwörter und 2FA für alle HA-Benutzer. **Reihenfolge beachten:**
   1. Tailscale-Admin → DNS → HTTPS Certificates aktivieren.
   2. Tailscale-Admin → Access controls → JSON editor: `"nodeAttrs": [{"target": ["tag:tom", "autogroup:member"], "attr": ["funnel"]}]` speichern. Getaggte Geräte gehören nicht zu `autogroup:member`, daher `tag:tom` explizit.
   3. Erst dann im Tailscale-Add-on (ab 0.30) `share_homeassistant: funnel`, `share_on_port: 443` setzen und neu starten.

   > Lehre vom 2026-11-11: Wird das Add-on mit `funnel` gestartet, bevor die Policy Funnel erlaubt, bricht es mit `FATAL: Tailscale's Funnel support is disabled` ab und beendet sich komplett (Status `error`). Damit ist auch der Tailnet-Zugang zu HA weg.

5. In claude.ai → Einstellungen → Connectors → Custom Connector hinzufügen, Name `HA_Thomas`, URL = öffentliche Webhook-URL aus Schritt 3. Die URL wie ein Passwort behandeln, nicht ins Repo.

Beispiel Variante A (Claude Code auf einem Rechner im Tailnet; Token nur lokal, nie ins Repo):

```bash
claude mcp add --transport http ha-thomas https://homeassistant.taila5009f.ts.net/api/mcp \
  --header "Authorization: Bearer $HA_TOKEN"
```

Long-Lived Token in HA: Profil → Sicherheit → „Langlebige Zugriffstoken“. Eigenen HA-Benutzer ohne Admin-Rechte für Claude anlegen und unter Einstellungen → Sprachassistenten → Entitäten freigeben nur die nötigen Entitäten exponieren.

### 5.5 Weitere Container (Ausblick)

Weitere Dienste (z. B. Mosquitto, Zigbee2MQTT, Node-RED) jeweils als eigenes Compose-Projekt unter `/volume1/docker/<dienst>/` anlegen. Diese Dateien gehören (ohne Passwörter) in dieses Repository unter `docker/<dienst>/compose.yaml`.

---

## 6. Absicherung (Phase 3)

1. **Portfreigaben am Router schließen** (alle aus Checkliste 2.2), sobald der Zugriff über Tailscale getestet ist.
2. **QuickConnect** deaktivieren, falls nicht mehr benötigt (Systemsteuerung → QuickConnect).
3. **DSM-Firewall** aktivieren:
   - Erlauben: LAN-Subnetz, Tailscale-Bereich `100.64.0.0/10` (und `fd7a:115c:a1e0::/48` für IPv6).
   - Alles andere: ablehnen.
4. **SSH** am NAS wieder deaktivieren oder nur aus LAN/Tailnet zulassen.
5. **Benachrichtigungen** einrichten (Systemsteuerung → Benachrichtigung) für Updates, Disk-Fehler und Backup-Ergebnisse.
6. Regelmäßig: DSM-, Paket-, Tailscale- und HA-Updates (monatlich; Sicherheitsupdates sofort).

---

## 7. Backup-Konzept (Phase 4)

| Was | Wie | Häufigkeit | Ziel |
|---|---|---|---|
| Volume / `docker`-Ordner | **Snapshot Replication** (Btrfs) | täglich, 14 Tage Aufbewahrung | lokal |
| `/volume1/docker` + wichtige Freigaben | **Hyper Backup** | täglich | USB-Disk **und** offsite (C2, zweites NAS, S3) |
| Home Assistant | eingebautes HA-Backup (Einstellungen → System → Backups), automatisch | täglich | `config/backups`, dadurch von Hyper Backup miterfasst |
| DSM-Konfiguration | Konfigurationssicherung | nach Änderungen | lokal + extern |
| Tailnet-Policy, Compose-Dateien | dieses Git-Repository | bei Änderung | GitHub |

Mindestens **einmal eine Wiederherstellung testen** (z. B. HA-Backup in einen Test-Container einspielen).

---

## 8. Step-by-Step-Ablauf (Kurzfassung)

| # | Schritt | Wer | Erledigt |
|---|---|---|---|
| 1 | Checkliste Abschnitt 2 ausfüllen, Ergebnisse hier im Repo festhalten | Mike + Thomas | ☐ |
| 2 | Entscheidungen 2.3 treffen (Tailnet-Eigentümer, Container/VM, Exit-Node) | Mike + Thomas | ☐ |
| 3 | DSM aktualisieren, Konfig-Backup, Snapshot, Admin-Konto mit 2FA | Thomas / Mike | ☐ |
| 4 | Tailnet anlegen, Mike einladen, Policy (4.5) einspielen, Auth-Key erzeugen | Thomas | ☐ |
| 5 | Tailscale-Paket installieren und anmelden (4.2) | vor Ort oder per bestehendem Zugang | ☐ |
| 6 | `configure-host` + Boot-Aufgabe einrichten (4.3) | Admin | ☐ |
| 7 | Subnet-Route (und ggf. Exit-Node) bewerben, freigeben, Key-Expiry deaktivieren (4.4) | Admin | ☐ |
| 8 | Zugriff von außen testen (4.6) | Mike | ☐ |
| 9 | Container Manager installieren, HA-Compose-Projekt starten (5.2, 5.3) | Admin | ☐ |
| 10 | HA-Onboarding, 2FA, `tailscale serve` + `trusted_proxies` (5.4) | Admin | ☐ |
| 11 | Portfreigaben schließen, QuickConnect aus, Firewall an (6) | Admin | ☐ |
| 12 | Backups einrichten und Wiederherstellung einmal testen (7) | Admin | ☐ |
| 13 | Ist-Zustand dokumentieren (Subnetz, IPs, Versionen) und Compose/Policy ins Repo | Mike | ☐ |

**Rückfallebene:** Bis Schritt 11 bleibt der bisherige Zugang (`goelles.myds.me`) bestehen. Erst schließen, wenn Schritt 8 und 10 erfolgreich waren. Im Notfall Tailscale-Paket im Paket-Zentrum stoppen; DSM ist dann weiterhin im LAN erreichbar.

---

## 9. Offene Punkte

- [ ] Ergebnisse der Checkliste (Modell, CPU, RAM, DSM-Version, Subnetz) eintragen.
- [ ] Prüfen, ob sich Thomas' Subnetz mit Mikes Netzen überschneidet.
- [ ] Klären, ob Funk-Hardware (Zigbee/Z-Wave/Thread/Bluetooth) für HA benötigt wird.
- [ ] Backup-Ziel außerhalb des NAS festlegen.
- [x] MCP-Anbindung Variante C umgesetzt; Funnel öffentlich getestet 2026-11-11 (Webhook antwortet 401 + OAuth, Metadaten 200).
- [ ] Connector `HA_Thomas` in claude.ai anlegen und Tools testen.

## Quellen

- Tailscale auf Synology: <https://tailscale.com/kb/1131/synology>
- Subnet-Router: <https://tailscale.com/kb/1019/subnets>
- Exit-Nodes: <https://tailscale.com/kb/1103/exit-nodes>
- Tailnet-Policy / Grants: <https://tailscale.com/kb/1324/grants>
- Tailscale Serve: <https://tailscale.com/kb/1312/serve>
- Tailscale Funnel: <https://tailscale.com/kb/1223/funnel>
- HA Model Context Protocol Server: <https://www.home-assistant.io/integrations/mcp_server/>
- Home Assistant Container-Installation: <https://www.home-assistant.io/installation/linux#docker-compose>
- Synology Container Manager: <https://www.synology.com/de-de/dsm/feature/container-manager>

## Notzugang, falls HA im Tailnet ausfällt

Die Syno (100.109.143.57) erreicht HA im LAN. SSH-Weiterleitung ist auf der Syno erlaubt (`AllowTcpForwarding yes` in `/etc/ssh/sshd_config`, danach `sudo synosystemctl restart sshd`). Vom Mac:

```bash
ssh -L 8123:192.168.1.206:80 <user>@100.109.143.57
# dann http://localhost:8123
```

Achtung: 192.168.1.206 direkt vom Mac aus führt ins Ems-Netz (gleiches Subnetz 192.168.1.0/24, geroutet über tailscale-ems).
