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
- **Tailscale-Identität** bleibt erhalten, wenn das Docker-Volume `tailscale-state` mit umgezogen wird (kein neuer `TS_AUTHKEY` nötig).
- **Phantom-Bluetooth-Integration (2026-08-05):** HA hatte automatisch eine Bluetooth-Integration am (nicht real vorhandenen) Host-Adapter `hci0` angelegt (`source: integration_discovery`) – Docker Desktop/macOS reicht keinen echten Bluetooth-Adapter in die Container-VM durch (`AF_BLUETOOTH` im Container nicht unterstützt, `hciconfig` scheitert mit „Address family not supported by protocol"), auch nicht mit `privileged: true`. Erzeugte im alten Log 102 Fehlerzeilen (`habluetooth.scanner: hci0 ... Failed to force stop scanner`) über 15,5 h Laufzeit. **Fix:** Config-Entry + Device-Registry-Eintrag bei gestopptem Container manuell aus `.storage/core.config_entries` und `.storage/core.device_registry` entfernt (Backups als `.bak` im selben Ordner). Kein reales Gerät betroffen, keine BLE-Mesh-Lampen laufen über HA-Bluetooth (die hängen an Alexa/Echo, siehe unten).
- **Recorder-DB-Korruption beim ersten Boot (2026-08-05):** `home-assistant_v2.db` war beim allerersten Start auf dem Mac Mini korrupt (`database disk image is malformed`, `PRAGMA integrity_check` bestätigt echte Schäden). HA hat automatisch selbst geheilt (alte Datei nach `home-assistant_v2.db.corrupt.<timestamp>` verschoben, frische DB angelegt) – Historie vor 2026-08-05 07:42 ist verloren, laufender Betrieb war nicht beeinträchtigt. Vermutete Ursache: nicht-atomarer Kopiervorgang der DB/WAL-Dateien bei der GX10→MacMini-Migration. Corrupt-Datei liegt noch in `config/` (10,7 MB), kann bei Bedarf für einen `.recover`-Versuch genutzt oder gelöscht werden.
- **Git-Vorsicht (wichtig, hat schon einmal einen Push blockiert):** `config/` (HA-Config inkl. `.storage/` mit OAuth-Tokens/Cookies/Auth-Hashes), `mosquitto/data/`, `mosquitto/log/` und `zigbee2mqtt/data/` (enthält den Zigbee-`network_key`!) sind in `.gitignore` und **dürfen nie committet werden**. Beim Umzug/Kopieren der Daten auf den Mac Mini nicht versehentlich per `git add .` wieder einfangen.

## Standorte
Zwei physische Standorte mit eigener Hardware:
1. **Dresden, Weißer Hirsch, Kurparkstr. 6** – aktueller Hauptstandort, GX10 läuft hier, Hauptteil der Geräte inkl. einfacher/älterer Hue Bridge
2. **Berlin-Wannsee, Schäferstr. 23a** – Hue Bridge Pro (Sicherheitsfunktion aktiv), SwitchBot Pan/Tilt-Innenkamera

**Geplant:** Das Haus in **Berlin-Wannsee** soll verkauft werden (Dresden bleibt dauerhafter Hauptstandort). Sobald der Verkauf erfolgt, übernimmt die Hue Bridge Pro aus Wannsee die Rolle der jetzigen älteren Dresdner Bridge (Ablösung, vermutlich durch Umzug der Pro-Bridge nach Dresden). Das relativiert die Dringlichkeit einer dauerhaften Zwei-Standorte-VPN-Lösung – ggf. wird die Standortfrage durch den Verkauf ohnehin obsolet.

**Offene Entscheidung:** Eine gemeinsame HA-Instanz mit VPN zwischen den Standorten, oder zwei getrennte Instanzen (aktuell eher letzteres empfohlen, da keine stabile VPN-Verbindung zwischen den Netzwerken besteht). Ausschlaggebend war ein früherer UUID-Konflikt zwischen der Haupt-HA-Instanz und der eingebetteten HA-Instanz im SwitchBot AI Hub, der einen Factory-Reset des Hubs nötig machte.

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