# Netzwerkübersicht Thomas – LAN 192.168.1.0/24

Stand: 15.11.2026, ca. 13:30 Uhr. Scan nur lesend von der DS918 (192.168.1.30) aus. Methode:

- TCP/UDP-Sweep über alle Adressen 192.168.1.1–254 und Auslesen der ARP-Tabelle (`/proc/net/arp`)
- mDNS (`avahi-browse`)
- TCP-Ports 22/80/443/502/1883/8080/8443/8123, für unklare Hosts 1–1024 und weitere
- KEBA-Abfrage auf UDP 7090 (`i`)
- HTTP-Titel und Server-Header
- Hersteller (OUI) über macvendors.com

Nichts konfiguriert, nichts installiert. Auf der Syno lief nur ein Python-Skript im Speicher, über stdin übergeben.

**Plausibilitätscheck:** .20 Fronius (00:03:ac) ✓ · .206 Home Assistant ✓ · .30 DS918 ✓

**Hinweis zu den MACs:** Das TP-Link-Mesh übersetzt die MAC-Adressen von WLAN-Clients hinter den Satelliten. Der Präfix wird dabei durch `22:44:7a` bzw. `1e:3b:f3` ersetzt, die letzten 3 Bytes bleiben echt. Beleg: .224 erschien zuerst als `38:9b:73:eb:c7:0d` und später als `22:44:7a:eb:c7:0d`. .104 `22:44:7a:6f:b7:4a` ist laut mDNS Sonos `5CAAFD6FB74A`. Bei solchen Geräten lässt sich der Hersteller nicht über die OUI bestimmen. Präfixe wie `5a:`, `16:`, `d2:`, `fa:`, `56:` sind dagegen zufällige Private-MACs von Handys und Tablets.

## Wallboxen, Wechselrichter, Heizung

| IP | MAC | Hersteller | Hostname / mDNS | Offene Ports | HTTP | Einordnung |
|---|---|---|---|---|---|---|
| **192.168.1.128** | 22:44:7a:81:6d:24 (Mesh) | (Fronius, laut mDNS) | Wattpilot_32332502.local (`_Fronius-SE-Wattpilot._tcp`) | 80 | 404 "Nothing matches the given URI" | **Wallbox 1: Fronius Wattpilot**, sicher |
| 192.168.1.20 | 00:03:ac:49:44:82 | Fronius | Fronius-34054601.local, "Fronius Symo GEN24 10.0" | 80, 502 | "Fronius Inverter", Server "webserver" | PV-Wechselrichter, Status Running |
| 192.168.1.141 | 00:0e:8c:01:2c:9e | Siemens AG | – | keine | – | unklar, Kandidat für Wallbox 2 oder NIBE (siehe unten) |
| 192.168.1.239 | 10:00:3b:be:6c:0c | Espressif (ESP32) | – | keine | – | unklar, ESP32-Gerät ohne Dienste |
| 192.168.1.229 | 40:d6:3c:67:be:5c | Equitech (Hi-Link) | – | 80, 8080 | 401, realm "HLK-RM08K" | WLAN↔Seriell-Bridge (RS232/RS485), z. B. für eine Modbus-RTU-Anbindung |

**Wallbox 2:** nicht eindeutig gefunden.

- Es meldet sich nur ein Wattpilot per mDNS.
- Kein KEBA antwortet auf UDP 7090.
- Kein Gerät mit OUI eines Wallbox-Herstellers (go-e, KEBA, ABB, Easee, Zaptec, Webasto usw.) ist im Netz.

Mögliche Erklärungen und Kandidaten:

- Die zweite Box ist gerade offline oder nicht im WLAN (z. B. LAN-los oder nur Cloud/LTE).
- **.141 (Siemens AG, keine offenen Ports):** Siemens baut Wallboxen (VersiCharge).
- **.239 (Espressif/ESP32, keine Ports):** Wattpilot und go-e basieren auf ESP32. Ein Wattpilot hätte aber Port 80 offen und würde sich per mDNS melden.
- Ein Gerät hinter der Mesh-MAC-Übersetzung ohne offene Ports: .123, .207, .209.
- Eine zweite Box, die über eine Modbus-Bridge (.229 HLK-RM08K) angebunden ist.

**Heizung NIBE F1155-12 EM:** nicht eindeutig zuzuordnen. Kein Gerät mit NIBE-OUI, kein Hostname "nibe"/"F1155", Port 502 ist nur am Fronius offen. Die F1155 spricht mit myUplink nur ausgehend, lokal ist daher nichts offen zu erwarten. Wahrscheinlichste Kandidaten:

- .141 (Siemens AG, keine Ports)
- ein Gerät hinter dem Mesh

Klarheit bringen die DHCP-Client-Liste des TP-Link-Routers (.1) oder die Integrationsliste im HA (.206). Für beides ist ein Login nötig.

## Vollständige Geräteliste

| IP | MAC | Hersteller (OUI) | Hostname / mDNS | Offene Ports | HTTP-Titel / Server | Gerät (Einschätzung) |
|---|---|---|---|---|---|---|
| 192.168.1.1 | 90:9a:4a:e9:07:ec | TP-Link | – | 22, 80, 443 | TP-Link-Web-UI (ohne Titel) | Router/Gateway (TP-Link) |
| 192.168.1.5 | 34:60:f9:d1:a0:dc | TP-Link | – | 22, 80, 443 | TP-Link-Web-UI (wie .1) | TP-Link Mesh-Knoten |
| 192.168.1.6 | b8:bb:af:b4:6c:3b | Samsung | – | – | – | Samsung-Gerät (nur kurz gesehen) |
| 192.168.1.7 | 1e:3b:f3:0d:96:3e (Mesh) | – | – | 22, 80 | TP-Link | TP-Link Switch/AP |
| 192.168.1.20 | 00:03:ac:49:44:82 | Fronius | Fronius-34054601.local | 80, 502 | "Fronius Inverter" / webserver | **Fronius Symo GEN24 10.0** (PV) |
| 192.168.1.30 | – (eigene) | Synology | DS918.local | – | – | Synology DS918 (Scan-Quelle) |
| 192.168.1.50 | ec:71:db:a8:9f:ca | Reolink | – | 80, 443 | nginx/1.6.2 | Reolink-Kamera |
| 192.168.1.51 | ec:71:db:55:1f:81 | Reolink | – | 80, 443 | nginx/1.6.2 | Reolink-Kamera |
| 192.168.1.53 | ec:71:db:f1:3a:ba | Reolink | – | 80, 443 | nginx/1.6.2 | Reolink-Kamera |
| 192.168.1.54 | ec:71:db:45:18:1e | Reolink | – | 80, 443 | nginx/1.6.2 | Reolink-Kamera |
| 192.168.1.56 | cc:e5:36:06:40:c3 | ittim | – | 80, 443 | Reolink-Web-UI (CGI-API.js) | Reolink-WLAN-Kamera |
| 192.168.1.77 | 50:4f:94:10:64:f7 | Loxone | – | 80, 443 | "Webinterface" | Loxone Miniserver |
| 192.168.1.78 | 50:4f:94:a1:b4:b8 | Loxone | MiniserverGartenhaus.local | 80, 443 | "Webinterface" | Loxone Miniserver Gartenhaus |
| 192.168.1.79 | 50:4f:94:e1:3a:99 | Loxone | – | 80, 443 | "Loxone Intercom" | Loxone Intercom |
| 192.168.1.100 | 64:33:db:8c:8b:aa | Texas Instruments | – | keine | – | unbekannt (IoT-Modul) |
| 192.168.1.101 | e4:e0:c5:09:9d:24 | Samsung | – | keine | – | Samsung-Gerät |
| 192.168.1.103 | 28:7e:80:f0:a1:d6 | Gaoshengda | – | 554, 8000, 9000 | – | IP-Kamera (RTSP) |
| 192.168.1.104 | 22:44:7a:6f:b7:4a (Mesh) | (Sonos) | Sonos-5CAAFD6FB74A.local | keine | – | Sonos |
| 192.168.1.106 | b0:7f:b9:3b:7e:23 | Netgear | – | 80 | "NETGEAR GS110TP" | Netgear-Switch GS110TP |
| 192.168.1.108 | 22:44:7a:bd:0f:7a (Mesh) | – | Matter + SpotifyConnect (linux-4) | 8888 | – | Echo/Smart-Speaker (Matter) |
| 192.168.1.115 | 90:e5:b1:36:bd:60 | Espressif | Matter (90E5B136BD60.local) | keine | – | Matter-Gerät (ESP32) |
| 192.168.1.116 | d2:44:7a:0e:dd:ba (privat) | – | – | 49152 | – | Apple-Gerät (Private MAC) |
| 192.168.1.119 | c0:91:b9:fa:ee:1c | Amazon | Thread-Border-Router, Spotify | keine | – | Amazon Echo |
| 192.168.1.121 | b8:e9:37:ae:91:08 | Sonos | Sonos-B8E937AE9108.local | keine | – | Sonos |
| 192.168.1.123 | 1e:3b:f3:89:35:00 (Mesh) | – | – | 23 (Reset) | – | unbekannt |
| **192.168.1.128** | 22:44:7a:81:6d:24 (Mesh) | (Fronius) | **Wattpilot_32332502.local** | 80 | 404 | **Wallbox 1: Fronius Wattpilot** |
| 192.168.1.134 | d0:d2:b0:85:30:87 | Apple | Wohnzimmer-2.local (AirPlay) | keine | – | Apple TV Wohnzimmer |
| 192.168.1.136 | 50:d4:5c:d8:92:e2 | Amazon | Matter + Spotify | keine | – | Amazon Echo |
| 192.168.1.137 | 5c:e9:31:1b:38:9c | TP-Link | – | 22, 80, 443 | TP-Link-Web-UI | TP-Link Mesh-Knoten |
| 192.168.1.138 | 70:ee:50:af:32:26 | Netatmo | – | keine | – | Netatmo |
| 192.168.1.139 | f4:39:09:2c:d2:97 | HP | DESKTOP-M37FB31.local | keine | – | Windows-PC |
| 192.168.1.140 | 9c:c8:e9:73:ca:17 | Amazon | – | keine | – | Amazon-Gerät |
| 192.168.1.141 | 00:0e:8c:01:2c:9e | Siemens AG | – | keine | – | **unklar: Kandidat für Wallbox 2 oder NIBE** |
| 192.168.1.142 | 56:24:3a:2b:a3:78 (privat) | – | iPad-2.local | 49152 | – | iPad |
| 192.168.1.143 | fa:44:7a:d6:30:1d (privat) | – | iPad-Thomas.local | – | – | iPad Thomas |
| 192.168.1.147 | a8:ca:77:7e:66:f2 | Amazon | – | 443, 8443 | – | Amazon (Fire TV/Echo) |
| 192.168.1.150 | f8:54:b8:22:50:de | Amazon | Matter + Spotify | keine | – | Amazon Echo |
| 192.168.1.151 | 30:13:8b:94:56:dc | HP | – | 135, 443, 445 | nginx/1.26.3 | HP-Windows-Rechner |
| 192.168.1.153 | 5c:e9:31:1b:8d:98 | TP-Link | – | 22, 80, 443 | TP-Link-Web-UI | TP-Link Mesh-Knoten |
| 192.168.1.170 | 22:44:7a:20:44:7a (Mesh) | – | – | 22, 80 | TP-Link Omada-Login | TP-Link Omada AP/Switch |
| 192.168.1.171 | 2c:58:b9:c1:7d:af | HP | – | keine | – | HP-Gerät |
| 192.168.1.172 | 00:0e:58:d1:31:58 | Sonos | Sonos-000E58D13158.local | keine | – | Sonos |
| 192.168.1.174 | 20:10:b1:ea:c4:05 | Amazon | echoshow.local | keine | – | Amazon Echo Show |
| 192.168.1.176 | 22:44:7a:ae:a5:ad (Mesh) | – | – | 80, 443 | Reolink-Web-UI | Reolink-WLAN-Kamera |
| 192.168.1.177 | 40:ed:00:bd:36:ed | TP-Link | – | 22, 80, 443 | TP-Link-Web-UI | TP-Link Mesh-Knoten |
| 192.168.1.178 | 22:44:7a:97:89:03 (Mesh) | – | Matter + Spotify (linux-3) | keine | – | Echo/Smart-Speaker (Matter) |
| 192.168.1.180 | b8:e9:37:98:4a:9c | Sonos | Sonos-B8E937984A9C.local | keine | – | Sonos |
| 192.168.1.187 | 5c:aa:fd:7a:6e:d4 | Sonos | Sonos-5CAAFD7A6ED4.local | keine | – | Sonos |
| 192.168.1.191 | 00:0e:58:d0:eb:62 | Sonos | Sonos-000E58D0EB62.local | keine | – | Sonos |
| 192.168.1.202 | ec:0d:e4:88:8e:3b | Amazon | – | keine | – | Amazon-Gerät |
| 192.168.1.203 | c8:d7:78:e1:c0:dd | BSH Hausgeräte | – | 22, 443 | – | Bosch/Siemens-Hausgerät (Home Connect) |
| 192.168.1.205 | 5a:63:0c:95:53:a0 (privat) | – | – | keine | – | Handy/Tablet (Private MAC) |
| 192.168.1.206 | 02:11:32:24:dd:45 | (virtuell) | Home (`_home-assistant._tcp`), Matter | 80 | "Home Assistant" | **Home Assistant** |
| 192.168.1.207 | 22:44:7a:6f:d2:38 (Mesh) | – | – | keine | – | unbekannt |
| 192.168.1.208 | 5c:aa:fd:6f:b8:a2 | Sonos | Sonos-5CAAFD6FB8A2.local | keine | – | Sonos |
| 192.168.1.209 | 1e:3b:f3:23:84:1c (Mesh) | – | – | keine | – | unbekannt |
| 192.168.1.223 | 5c:aa:fd:7a:6e:62 | Sonos | Sonos-5CAAFD7A6E62.local | keine | – | Sonos |
| 192.168.1.224 | 22:44:7a:eb:c7:0d (Mesh; echt 38:9b:73…) | GSD Viet Nam | – | 80, 443 | Reolink-Web-UI | Reolink-WLAN-Kamera |
| 192.168.1.229 | 40:d6:3c:67:be:5c | Equitech (Hi-Link) | – | 80, 8080 | 401 realm "HLK-RM08K" | WLAN↔Seriell-Bridge (Modbus?) |
| 192.168.1.230 | 38:7a:0e:cb:60:15 | Intel | – | 135, 139, 445 | – | Windows-PC |
| 192.168.1.231 | 4c:17:44:f8:63:03 | Amazon | Matter + Spotify | keine | – | Amazon Echo |
| 192.168.1.233 | 16:40:ee:ad:75:2f (privat) | – | – | 49152 | – | Apple-Gerät (Private MAC) |
| 192.168.1.237 | 68:9a:87:27:3e:d5 | Amazon | Matter + Spotify | keine | – | Amazon Echo |
| 192.168.1.239 | 10:00:3b:be:6c:0c | Espressif | – | keine | – | ESP32-Gerät, unklar |
| 192.168.1.241 | 5c:aa:fd:a7:47:00 | Sonos | Sonos-5CAAFDA74700.local | keine | – | Sonos |
| 192.168.1.243 | b8:e9:37:ae:91:82 | Sonos | Sonos-B8E937AE9182.local | keine | – | Sonos |
| 192.168.1.244 | 94:dd:f8:62:00:b3 | Brother | BRN94DDF86200B3.local | 80 | "Brother MFC-L8690CDW series" | Drucker Brother MFC-L8690CDW |
| 192.168.1.249 | f8:fe:5e:4d:cc:5e | Intel | – | 135, 445 | – | Windows-PC |

Rückmeldungen ohne MAC (ARP unvollständig) gab es auf .55, .57 und .144. Diese Geräte sind nicht weiter zuzuordnen.

## Nächste Schritte zur Klärung von Wallbox 2 und NIBE

1. TP-Link-Router .1 (Login nötig): Die Client-Liste zeigt Gerätenamen und die echten MACs hinter dem Mesh.
2. Home Assistant .206: Stand 15.11. ist dort weder ein Wattpilot noch die NIBE eingebunden (geprüft über HA_Thomas). Auch im KNX-/ETS-Projekt gibt es keine Wallbox.
3. Wattpilot-App oder Solar.web: Ist eine zweite Box registriert? Wenn ja, welche Seriennummer und ist sie online?
4. myUplink-App: Unter Gerät → Netzwerk die IP und MAC der F1155 ablesen.

## Einbindung in Home Assistant (Vorschlag, nicht umgesetzt)

- **Fronius Wattpilot (.128):** Die Fronius-Core-Integration deckt den Wattpilot nicht ab. Möglich ist die HACS-Integration „Fronius Wattpilot“ (lokale WebSocket-API). Dafür braucht es das Wattpilot-Passwort aus der Solar.wattpilot-App. Eine feste IP (DHCP-Reservierung im TP-Link-Router) ist empfehlenswert.
- **NIBE F1155-12 EM:** Am schnellsten geht es über die Core-Integration **myUplink** (Cloud, OAuth mit Thomas' myUplink-Konto), weil der Webportal-Zugang schon aktiv ist. Lokal ginge alternativ die Core-Integration „Nibe Heat Pump“ (Modbus 40 bzw. RS485).
- **Wallbox 2:** offen, bis sie identifiziert ist.

## Hinweis für spätere Scans

Mikes Mac sieht 192.168.1.0/24 über Tailscale im Netz von **Ems** (Subnet-Route von `tailscale-ems`), nicht bei Thomas. Scans von Thomas' LAN deshalb immer auf der DS918 laufen lassen (`ssh auwe@100.109.143.57`, Schlüssel-Login seit 15.11.2026).
