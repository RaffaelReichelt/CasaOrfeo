# Home Assistant Neuaufbau – Projektkontext

## Umgebung
- Home Assistant läuft als **Docker-Container** (nicht HA OS/Supervised), Host: **Mac Mini** (umgezogen von einem GX10, siehe „Migration" unten) – Docker-Runtime dort: **Docker Desktop**
- Kein Add-on Store verfügbar – zusätzliche Dienste (MQTT, Zigbee2MQTT) müssen als eigene Docker-Container laufen
- Ziel: kompletter Neuaufbau der Konfiguration, saubere Struktur von Anfang an
- Stack: `homeassistant` + `tailscale` (Sidecar, `network_mode: host`) + `mosquitto` + `zigbee2mqtt`, siehe [docker-compose.yml](docker-compose.yml)

### Migration GX10 → Mac Mini (2026-08-05)
- **Grund:** GX10 ist für Privatemind reserviert, HA gehört fachlich nicht dorthin.
- **Docker-Desktop-Settings, die auf dem Mac Mini gesetzt sein müssen** (sonst Instabilität/Ausfälle):
  - *„Enable Resource Saver"* **deaktiviert** – sonst pausiert Docker Desktop die VM bei Inaktivität und HA reagiert nicht mehr auf Automationen.
  - *„Start Docker Desktop when you log in"* **aktiviert**, plus automatischer macOS-Login – anders als beim GX10 läuft der Docker-Daemon hier nicht ohne eingeloggten Nutzer.
  - macOS-Ruhezustand deaktiviert (`pmset -a sleep 0`), feste IP/DHCP-Reservierung für den Mac Mini.
- **Netzwerk:** `network_mode: host` für den `homeassistant`-Service ist unter Docker Desktop/macOS **tatsächlich instabil aufgetreten** (2026-08-05): Container lief sauber und antwortete intern (`docker exec homeassistant curl localhost:8123` → 200), war aber vom Host aus über `localhost:8123` nicht erreichbar (Connection refused) – klassische Docker-Desktop-Host-Networking-Falle, kein HA-Fehler. **Fix angewendet:** `homeassistant`-Service läuft jetzt auf Bridge-Networking mit `ports: ["8123:8123"]` statt `network_mode: host`, siehe [docker-compose.yml](docker-compose.yml). `privileged: true` wurde testweise entfernt und war NICHT die Ursache (Hang bestand auch ohne); wieder auf `true` gesetzt, unklar ob nötig, nicht weiter angefasst. Multicast (mDNS/SSDP für Hue-/Apple-TV-Discovery) verhält sich unter Bridge-Networking anders als unter Linux-Host-Networking. Alle bestehenden Integrationen (Hue, Samsung TV, Apple TV) nutzen gespeicherte IPs/Credentials aus `.storage/` und brauchen daher im laufenden Betrieb keine erneute Discovery – nur bei Neu-Pairing relevant, dort ggf. erneut prüfen. Zigbee2MQTT/Tailscale sind von der Umstellung nicht betroffen (Z2M: reine TCP-Verbindung zur SLZB-06 ohne Broadcast, eigener Container ohne `network_mode: host`; Tailscale: eigener Sidecar-Container, behält `network_mode: host`).
- **Tailscale-Identität** bleibt erhalten, wenn das Docker-Volume `tailscale-state` mit umgezogen wird (kein neuer `TS_AUTHKEY` nötig). **Aber Vorsicht bei Container-Neuerstellung ohne sauber übernommenen State:** Beim Umzug entstand einmal ein Duplikat-Node (`homeassistant-1` neben einem toten `homeassistant`, altes Gerät ging offline) – alte Maschine im Tailscale-Admin (login.tailscale.com/admin/machines) löschen, dann `tailscale set --hostname=homeassistant` im Container ausführen. Nach Löschen/Rename kann der **lokale macOS-DNS-Cache** (`dscacheutil`) noch länger die alte IP für den MagicDNS-Namen liefern, obwohl `dig`/Tailscale selbst schon korrekt auflösen – Abhilfe: `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` auf dem Mac.
- **HTTPS-Zugriff über Tailscale (`https://homeassistant.tail5a2ccd.ts.net`)** läuft über `tailscale serve --bg --https=443 http://host.docker.internal:8123` (im `tailscale`-Container ausgeführt, persistiert im `tailscale-state`-Volume – kein Compose-Eintrag). Grund für `host.docker.internal` statt Container-Name: `tailscale` läuft mit `network_mode: host`, `homeassistant` seit der Bridge-Networking-Umstellung nicht mehr im selben Netz-Namespace, daher kein Container-DNS zwischen beiden. **Wichtig:** Traffic kommt dadurch über Docker Desktops internes Gateway `192.168.65.1` bei HA an – dieses IP muss in `configuration.yaml` unter `http.trusted_proxies` stehen, sonst blockt HA mit `Received X-Forwarded-For header from an untrusted proxy` (400 Bad Request, kein sichtbarer Fehler im Browser außer endlosem Laden).
- **Phantom-Bluetooth-Integration (2026-08-05):** HA hatte automatisch eine Bluetooth-Integration am (nicht real vorhandenen) Host-Adapter `hci0` angelegt (`source: integration_discovery`) – Docker Desktop/macOS reicht keinen echten Bluetooth-Adapter in die Container-VM durch (`AF_BLUETOOTH` im Container nicht unterstützt, `hciconfig` scheitert mit „Address family not supported by protocol"), auch nicht mit `privileged: true`. Erzeugte im alten Log 102 Fehlerzeilen (`habluetooth.scanner: hci0 ... Failed to force stop scanner`) über 15,5 h Laufzeit. **Fix:** Config-Entry + Device-Registry-Eintrag bei gestopptem Container manuell aus `.storage/core.config_entries` und `.storage/core.device_registry` entfernt (Backups als `.bak` im selben Ordner). Kein reales Gerät betroffen, keine BLE-Mesh-Lampen laufen über HA-Bluetooth (die hängen an Alexa/Echo, siehe unten).
- **Recorder-DB-Korruption beim ersten Boot (2026-08-05):** `home-assistant_v2.db` war beim allerersten Start auf dem Mac Mini korrupt (`database disk image is malformed`, `PRAGMA integrity_check` bestätigt echte Schäden). HA hat automatisch selbst geheilt (alte Datei nach `home-assistant_v2.db.corrupt.<timestamp>` verschoben, frische DB angelegt) – Historie vor 2026-08-05 07:42 ist verloren, laufender Betrieb war nicht beeinträchtigt. Vermutete Ursache: nicht-atomarer Kopiervorgang der DB/WAL-Dateien bei der GX10→MacMini-Migration. Corrupt-Datei liegt noch in `config/` (10,7 MB), kann bei Bedarf für einen `.recover`-Versuch genutzt oder gelöscht werden.
- **Git-Vorsicht (wichtig, hat schon einmal einen Push blockiert):** `config/` (HA-Config inkl. `.storage/` mit OAuth-Tokens/Cookies/Auth-Hashes), `mosquitto/data/`, `mosquitto/log/` und `zigbee2mqtt/data/` (enthält den Zigbee-`network_key`!) sind in `.gitignore` und **dürfen nie committet werden**. Beim Umzug/Kopieren der Daten auf den Mac Mini nicht versehentlich per `git add .` wieder einfangen.

## Standorte
Zwei physische Standorte mit eigener Hardware:
1. **Dresden, Weißer Hirsch, Kurparkstr. 6** – aktueller Hauptstandort, GX10 läuft hier, Hauptteil der Geräte inkl. einfacher/älterer Hue Bridge
2. **Berlin-Wannsee, Schäferstr. 23a** – Hue Bridge Pro (Sicherheitsfunktion aktiv), SwitchBot Pan/Tilt-Innenkamera

**Geplant:** Das Haus in **Berlin-Wannsee** soll verkauft werden (Dresden bleibt dauerhafter Hauptstandort). Sobald der Verkauf erfolgt, übernimmt die Hue Bridge Pro aus Wannsee die Rolle der jetzigen älteren Dresdner Bridge (Ablösung, vermutlich durch Umzug der Pro-Bridge nach Dresden). Das relativiert die Dringlichkeit einer dauerhaften Zwei-Standorte-VPN-Lösung – ggf. wird die Standortfrage durch den Verkauf ohnehin obsolet.

**Offene Entscheidung:** Eine gemeinsame HA-Instanz mit VPN zwischen den Standorten, oder zwei getrennte Instanzen (aktuell eher letzteres empfohlen, da keine stabile VPN-Verbindung zwischen den Netzwerken besteht). Ausschlaggebend war ein früherer UUID-Konflikt zwischen der Haupt-HA-Instanz und der eingebetteten HA-Instanz im SwitchBot AI Hub, der einen Factory-Reset des Hubs nötig machte.

### VPN-Zugang zum Wannsee-Router (WireGuard, 2026-08-05)
- **Router:** Telekom Speedport (Smart 4/Pro-Klasse) mit eingebautem WireGuard-Server unter **Netzwerk → Virtuelles Netz (VPN)**. Endpoint: `berlin.privatemind.de:53280` (DynDNS-Hostname, nicht die rohe IP verwenden – die kann wechseln).
- **Speedport-Eigenheiten, die vom Standard-WireGuard-Workflow abweichen:**
  - Es gibt **kein „Peer mit eigenem Public Key hinzufügen“** – der Router generiert pro VPN-Zugang ein komplettes Client-Config (inkl. Private Key) selbst, ausgegeben als QR-Code + einmaliger Download-Link. Eigene Schlüssel lassen sich nicht einspeisen.
  - Die Config-Datei ist **nur direkt bei der Erstellung herunterladbar**, danach nicht erneut abrufbar – bei Verlust muss ein neuer Zugang angelegt werden.
  - Speedport Smart 3 erlaubt nur **einen** gleichzeitigen VPN-Zugang; Smart 4/Pro mehrere parallel (unser Fall).
  - Generierte Configs sind standardmäßig **Full-Tunnel** (`AllowedIPs = 0.0.0.0/0`) – auf Split-Tunnel (`AllowedIPs = 192.168.1.0/24, 10.200.200.0/24`, Wannsee-LAN + Tunnelnetz) umgestellt, sonst reißt ein toter/fehlkonfigurierter Tunnel den gesamten Internetzugriff des Clients mit.
- **Fallstrick, der zum Debugging-Anlass wurde:** Ein WireGuard-Profil 1:1 (gleicher Private Key) vom MacBook auf den Mac Mini kopiert funktionierte scheinbar (Interface „up“), aber **kein einziges Paket kam zurück** (`Ipkts: 0`) – der Router kennt pro Config einen eigenen Peer und schickt Antworten an den zuletzt aktiven Endpoint (MacBook), nicht an den Klon. Jedes Gerät braucht einen **eigenen, separat am Router erzeugten VPN-Zugang** (eigene Adresse im Tunnelnetz, z.B. MacBook `10.200.200.1`, Mac Mini `10.200.200.2`).
- **iPhone-Zugang** funktionierte durchgehend, weil er ein eigenes Profil nutzt – kein Konflikt.
- Aktiver Tunnel-Name auf dem Mac Mini in der WireGuard-App: **„MacMini“**. Router-Interface darüber erreichbar unter `https://192.168.1.1`.

## Geräteinventar & Integrationsstrategie

### Zigbee-Heizkörperthermostate (9 Stück)
- **Koordinator:** SMLIGHT SLZB-06 (Ethernet/PoE/WiFi Zigbee 3.0 Gateway, TI CC2652P-Chip, kein Vendor-Lock-in) — IP: 192.168.2.207, Socket-Port 6638, Z2M-Adapter-Typ: `zstack` (nicht "ember"!)
- **Empfohlener Stack:** Zigbee2MQTT (statt ZHA) wegen ausgereifterer Ventilsteuerung/Kalibrierung bei Thermostaten
- Benötigt: Mosquitto-Container (MQTT-Broker) + Zigbee2MQTT-Container im selben Docker-Netzwerk
- Verbindung zum Koordinator über `socket://<IP-der-SLZB-06>:6638`, feste IP/DHCP-Reservierung empfohlen
- **To-do:** Nach Pairing alle 9 Thermostate kalibrieren (Ventilstellung, Temperatur-Offset)

### Philips Hue
- Hue-Lampen am Hauptstandort + **Hue Bridge Pro in Wannsee** (Sicherheitsfunktion aktiv, inkl. Bewegungs-/Kontaktsensoren)
- Integration: native HA-Hue-Integration, lokal über die jeweilige Bridge (kein Cloud-Umweg)
- Offene Frage: Hue-eigene Sicherheitslogik (Alarm scharf/unscharf) beibehalten oder in HA-Automationen nachbauen

### SwitchBot
- **Geräte:** AI Hub, 2× Hub Mini (IR-Steuerung diverser Geräte), Pan/Tilt-Innenkamera (Wannsee)
- Integration: offizielle SwitchBot-Integration über Cloud-API (Token + Secret aus der App)
- AI Hub war bereits an die Haupt-HA-Instanz gekoppelt (nach Factory-Reset neu gepairt)
- Alle IR-gelernten Geräte über die Hub Minis kommen darüber als HA-Entities

### Bluetooth-Lampen (aktuell über Alexa gesteuert)
- **Ergebnis der Prüfung (2026-08-04):** Alexa-Gruppe "Vitrine", Hersteller Hangzhou Broadlink Technology Co. Ltd, reine BLE-Mesh-Lampen, gekoppelt über Raffaels Echo Dot als Bluetooth-Bridge — kein WLAN/Matter/Thread, kein direkter HA-Zugriff möglich.
- **Empfehlung:** Ersatz durch Zigbee-Lampen, da Zigbee2MQTT/SLZB-06-Koordinator bereits läuft (direkte lokale Steuerung ohne Alexa-Umweg). Kaufentscheidung/Zeitpunkt noch offen.

### Samsung The Frame TVs (32" & 65")
- Integration: native Samsung-TV-Integration (lokal über WebSocket-API)
- Steuerbar: Ein/Aus, Lautstärke, Quellenwahl, Kunstmodus-Umschaltung

### Apple TV (aktuelles Modell)
- Integration: native HA-Integration über `pyatv`, lokal, Pairing-Code-Verfahren
- Liefert: Power/Play/Pause/Skip, aktuelle App/Inhalt als Zustand, Fernbedienungs-Befehle
- **Hinweis:** Bei tvOS 17+ ggf. „Automatisch weiterleiten erlauben“ unter AirPlay-Einstellungen manuell aktivieren, sonst bricht Pairing ab
- Automationsidee: Apple TV-Wiedergabe startet → Frame TV auf HDMI-Quelle umschalten, Kunstmodus verlassen, Lichtszene aktivieren

### Amazon Echo-Geräte (2× Echo Show, 2× Echo, 1× Echo Dot)
- **HA → Alexa:** Alexa Smart Home Skill, um HA-Entities für Sprachsteuerung freizugeben
- **Alexa → HA:** inoffizielle „Alexa Media Player“-Integration über HACS für TTS-Ansagen, Musiksteuerung, teilweise Routine-Trigger

### CZEview Außenkamera (neu)
- **Ergebnis der Prüfung (2026-08-04):** Kein ONVIF/RTSP-Menüpunkt in der App auffindbar, kein Tuya-Branding erkennbar. Netzwerk-Scan der unbekannten LAN-Hosts auf Port 554/80/8000 (typische Kamera-Ports) ergab überall geschlossen — Kamera bietet keine lokale Schnittstelle.
- **Fazit:** Rein cloud-/app-gebunden, kein bekannter Unterbau. Aktuell keine HA-Integration möglich, bleibt außerhalb von HA (nur über eigene App nutzbar). Bei Bedarf später erneut prüfen, falls Hersteller-Firmware-Update lokale Schnittstelle nachreicht.

## Priorisierter Fahrplan
1. Docker-Compose-Stack neu aufsetzen (HA + Mosquitto + Zigbee2MQTT), persistente Volumes
2. SLZB-06 einbinden, 9 Thermostate pairen und kalibrieren
3. Hue-Bridges (beide Standorte) einbinden
4. SwitchBot-Cloud-Integration einrichten (IR-Geräte, Pan/Tilt-Kamera)
5. Samsung-TVs lokal einbinden
6. Apple TV einbinden
7. Alexa Media Player via HACS für die Echos
8. CZEview-Kamera auf ONVIF/RTSP/Tuya prüfen
9. Bluetooth-Lampen-Bestandsaufnahme, ggf. Ersatz
10. Wannsee-Standort-Strategie (eine vs. zwei Instanzen) final entscheiden

## Präferenzen
- YAML-Configs und andere nutzbare Artefakte statt reiner Erklärungen bevorzugt
- Direktes, technisches Feedback während der Fehlersuche