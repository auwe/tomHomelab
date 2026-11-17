# Syno DS918+: Firewall (vorbereitet) und Support-User

Stand: 17.11.2026. Geprüft per SSH, nur lesend.

## Ist-Zustand

- DSM 7.1.1-42962, LAN 192.168.1.30 (`ovs_eth0`, Open vSwitch wegen VMM), Tailscale 100.109.143.57.
- SSH läuft auf **Port 2222**, Port 22 ist zu. SSH bietet noch `publickey,password` an, d. h. der Passwort-Login ist noch aktiv.
- Tailscale läuft mit `--tun=userspace-networking`. Zugriffe über Tailscale kommen auf der Syno als `127.0.0.1` an. Die DSM-Firewall greift dafür nicht, eine Regel für 100.64.0.0/10 hat keine Wirkung. Tailscale-Zugriffe lassen sich nur über die Tailscale-ACLs einschränken.
- Docker-Paket läuft, aber kein Container im Bridge-Netz (`docker0` DOWN, keine `br-*`). Eine Regel für 172.16.0.0/12 ist derzeit nicht nötig.
- **Firewall ist aus** (`/usr/syno/etc/firewall.d/firewall_settings.json`: `status: false`, Profil `default` ohne Regeln).
- Thomas nutzt DSM von außen über **goelles.myds.me:5001** (Synology DDNS + Portweiterleitung am Router, nicht Tailscale). Dieser Zugang muss erhalten bleiben.

## Entscheidung

Die Firewall bleibt vorerst **aus**. Sie wird erst später aktiviert, wenn Mike/Thomas es wünschen. **Offener Punkt.**

## Vorbereiteter Regelsatz (Profil „default“)

Systemsteuerung → Sicherheit → Firewall → Profil „default“ → Regeln bearbeiten.

Unter **Alle Schnittstellen**, in dieser Reihenfolge:

| # | Ports | Protokoll | Quelle | Aktion | Zweck |
|---|-------|-----------|--------|--------|-------|
| 1 | 5001 | TCP | Alle | Zulassen | DSM HTTPS von außen (goelles.myds.me:5001) |
| 2 | Alle | Alle | 192.168.1.0/255.255.255.0 | Zulassen | LAN (SMB-Backup HA .206, Cloudflared in HA-VM, Clients) |
| 3 | Alle | Alle | fe80::/10 | Zulassen | IPv6 link-local im LAN (z. B. Zugriff per DS918.local) |
| 4 | Alle | Alle | fd95:58d3:9498:99b::/64 | Zulassen | IPv6 ULA im LAN |
| 5 | Alle | Alle | 100.64.0.0/255.192.0.0 | Zulassen | Tailscale (derzeit wirkungslos, schadet nicht; greift falls Tailscale später im TUN-Modus läuft) |

Danach im Dropdown **LAN 1** und **LAN 2** wählen und „Wenn keine Regel zutrifft“ auf **Zugriff verweigern** setzen. Mit OK schließen und auf der Firewall-Seite unten **Übernehmen** klicken (ohne Übernehmen wird nichts gespeichert).

Hinweise:
- Let's Encrypt: Erneuerung des Zertifikats für `goelles.myds.me` über HTTP-01 braucht Port **80** von außen, sofern der Router 80 weiterleitet. Vor dem Aktivieren prüfen, ob Port 80 weitergeleitet wird. Falls ja: Regel „80/TCP, Quelle Alle, Zulassen“ ergänzen oder die Erneuerung beobachten. Port 443 wird für DSM selbst nicht gebraucht.
- Aussperren über Tailscale ist ausgeschlossen (Loopback, siehe oben). Über DSM 5001 aus dem LAN prüft DSM beim Übernehmen, ob die aktuelle Verbindung blockiert würde.
- Nach dem Aktivieren testen: DSM :5001 (LAN, Tailscale, goelles.myds.me), SSH :2222, Tunnel `ssh -p 2222 -L 8123:192.168.1.206:80`, SMB-Backup HA (192.168.1.206 → `/volume2/Backup/ha-backup`), `dsm-goelles.auwe.ch` über Cloudflared in der HA-VM.

## Schutz für den öffentlichen DSM-Zugang (empfohlen, unabhängig von der Firewall)

- **Auto-Block** (Sicherheit → Schutz): z. B. 5 Fehlversuche in 5 Minuten, Sperre ohne Ablauf. LAN und Tailscale in die Zulassungsliste.
- **2FA** (Benutzer → Erweitert → 2-Faktor-Authentifizierung) für alle Admin-Konten erzwingen, mindestens für Thomas' Konto.
- **Geo-IP**: nur möglich, wenn die Firewall aktiv ist. Dann Regel 1 auf Quelle „Standort: Österreich, Deutschland (ggf. Schweiz)“ statt „Alle“ einschränken.
- Standard-`admin`-Konto deaktiviert lassen, DoS-Schutz aktivieren.

## Support-User für Claude (Vorschlag)

Ziel: eigener Zugang statt Mikes Konto, nur Schlüssel-Login, möglichst kleine Rechte.

- DSM erlaubt SSH nur für Mitglieder der Gruppe **administrators**. Der User muss also dort rein. Damit hat er auch DSM-Admin-Rechte in der Oberfläche; abmildern mit 2FA und einem langen Zufallspasswort, das nur in Vaultwarden (Sammlung Homelab) liegt.
- **Schlüssel-Login** mit eingeschränktem Key: `from="127.0.0.1,192.168.1.0/24"` (Tailscale kommt als 127.0.0.1 an).
- **sudo ohne Passwort nur für eine Liste von Befehlen** über `/etc/sudoers.d/claude-support`, z. B. Lesen der Firewall-Konfiguration, `synofirewall`, `iptables -S`, `docker ps`. Hinweis: DSM-Updates können `/etc/sudoers.d` überschreiben; nach Updates prüfen.
- Passwort-Login für SSH global abschalten erst, wenn alle Admins per Schlüssel reinkommen.
