# Fork-Notizen (pulpier/wican-fw, Branch `abrp-ble`)

Dieser Fork erweitert die WiCAN-OBD-Firmware um **BLE parallel zum WLAN**, damit
derselbe Dongle zuhause per WLAN an Home Assistant liefert und unterwegs per BLE
an Apps wie Car Scanner, ABRP oder IONIQ 5 Companion.

Getestet auf: **WiCAN-OBD (ESP32-C3)**, Hardware v3.40, im Hyundai Ioniq 5.
Andere Varianten (USB, PRO) sind unberührt, aber ungetestet.

Vorbild: [meatpiHQ/wican-fw Discussion #615](https://github.com/meatpiHQ/wican-fw/discussions/615)
— dort löst jemand dasselbe Szenario mit einem Kippschalter, der zwischen zwei
Konfigurationen umschaltet. Hier laufen beide Wege gleichzeitig.

## Was dieser Fork ändert

**BLE**
- `ble.c`/`ble.h` aus dem `wican-pro`-Zweig portiert. Das Standard-Advertisement
  der OBD-Firmware ist so knapp, dass ABRP das Gerät nicht findet; die PRO-Variante
  bringt Device Information Service, Microchip-SPP und das FFF0/FFF1/FFF2-Layout
  mit, das BLE-ELM327-Adapter üblicherweise anbieten.
- `ble_enable()` wird beim Start aufgerufen. Der portierte Code trennt Vorbereitung
  (`ble_init()`) und Start (`ble_enable()`); die Original-OBD-Firmware machte beides
  in `ble_init()`, weshalb BLE nach dem Port nie hochkam.
- Die beiden Allokationen in `ble.c` nutzen internen RAM statt `MALLOC_CAP_SPIRAM` —
  der ESP32-C3 hat kein PSRAM, die Anforderung schlug immer fehl.
- WLAN bleibt bestehen, während ein BLE-Client verbunden ist (im Original wird es
  beim Connect abgeschaltet).

**WLAN**
- **Kein SoftAP mehr.** AP + Station + BLE gleichzeitig ist auf dem C3 eine Funkrolle
  zu viel: Mit aktivem AP kommt die Station nie zustande (A/B-getestet). Der Hersteller
  nennt seinen Dualmodus beim PRO ebenfalls „BLE+Station", nie BLE+AP+Station.
  **Folge: Ohne erreichbares WLAN kommt man nur noch per USB an das Gerät** (siehe unten).
  Das ist eine bewusste Entscheidung; eine AP-Rückfallebene fehlt absichtlich.
- `esp_wifi_connect()`-Fehler blockieren die Verbindungsaufgabe nicht mehr dauerhaft.

**OTA**
- `esp_ota_mark_app_valid_cancel_rollback()` stand am Anfang von `app_main()`, wo eine
  frische Firmware noch nichts bewiesen hat. Jetzt bestätigt `ota_verify_task` sie erst,
  wenn die Station wieder im Netz ist — sonst Neustart nach 5 Minuten und der Bootloader
  rollt auf die vorherige Version zurück. Mit einer absichtlich kaputten Version
  (unerreichbare SSID) end-to-end getestet.

**ELM327-Emulation** (alle Ergänzungen aus echten App-Mitschnitten entstanden)
- `ATCRA` beachtet `X`-Platzhalter. Vorher wurde `ATCRA7XX` als exakte Adresse `0x700`
  gelesen statt als Bereich `700-7FF`, wodurch **jede** ECU-Antwort verworfen wurde —
  Apps meldeten „no data" bzw. „vehicle asleep", obwohl das Fahrzeug antwortete.
- `ATCAF1`, `ATCFC1`, `ATR1` werden quittiert; sie beschreiben exakt das Verhalten, das
  die Emulation ohnehin zeigt. Die Gegenstücke `CAF0`, `CFC0`, `R0` sind **nicht**
  implementiert und antworten weiterhin `?` — ein unehrliches `OK` würde dem Client
  falsche Daten liefern statt einer klaren Absage.
- `ATAR` implementiert (setzt den `ATCRA`-Filter zurück).
- Eingaben, die weder AT-Kommando noch gültiges Hex sind, bekommen `?` statt gar keiner
  Antwort. Vorher blieben etwa die `ST*`-Befehle der STN-Chips unbeantwortet und der
  Client wartete bis zum Timeout.
- `ATI` meldet `ELM327 v1.3a` statt `OBDLink MX`. Die alte Kennung ist die eines
  STN11xx-Chips, woraufhin Apps dessen Spezialbefehle schicken, die wir nicht können.

**Diagnose**
- `GET /elm327_trace` liefert die letzten 48 Kommandos samt Antworten als JSON.
  Das ist das wichtigste Werkzeug für App-Kompatibilität: erst mitschneiden, was eine
  App wirklich schickt, dann die Folge über TCP 3333 nachspielen und das schuldige
  Kommando einkreisen. Alle oben genannten ELM327-Fehler wurden so gefunden.
- `GET /check_status` enthält zusätzlich `ble_enabled` und `ble_connected` aus dem
  `dev_status`-Bitfeld — der tatsächliche Zustand des Stacks, nicht die Konfiguration.

## Bauen

GitHub Actions (`.github/workflows/build-firmware.yml`) baut bei jedem Push auf
`abrp-ble` mit ESP-IDF v5.5.2 für `esp32c3` und legt die Binaries als Artefakt ab.
Lokal entsprechend `idf.py set-target esp32c3 build`.

## Flashen

**Über WLAN (Normalfall):**

```
curl -F "file=@wican-fw_obd_<sha>.bin;type=application/octet-stream" \
     http://<dongle>/upload/ota.bin
```

Das Gerät startet neu und muss binnen 5 Minuten wieder ins WLAN kommen, sonst rollt
es selbsttätig zurück.

**Über USB (Rettungsweg, wenn kein WLAN mehr geht):**

1. Micro-USB **direkt am Rechner**, nicht über einen Hub — über Hubs kam hier gar kein
   USB-Ereignis an.
2. Für den Download-Modus `IO9` gegen `GND` brücken (Lochfeld neben dem Micro-USB-Anschluss,
   Beschriftung auf der Platine). Die gelbe LED leuchtet, die Brücke bleibt stecken.
3. Es erscheint ein `/dev/cu.usbmodem*`; die Konsole läuft über natives USB-Serial/JTAG,
   `esptool` spricht direkt mit dem Chip.

```
esptool --port /dev/cu.usbmodemXXXX --baud 921600 write-flash \
    0x10000 wican-fw_obd_<sha>.bin \
    0xd000  ota_data_initial.bin
```

Das schreibt die App nach `ota_0` und setzt die OTA-Auswahl dorthin zurück, unabhängig
davon, in welchem Slot die kaputte Version liegt. Bootloader und Partitionstabelle
bleiben unangetastet, die Konfiguration im NVS (WLAN-Zugangsdaten) bleibt erhalten.

## Betrieb

- **BLE-Kopplung ist Pflicht** (`ESP_LE_AUTH_REQ_SC_MITM_BOND`). Der Passkey steht in der
  Konfiguration als `ble_pass`. Apps stoßen die Kopplung meist nicht selbst an — zuerst
  am Telefon koppeln, dann die App starten.
- **Nur ein BLE-Client gleichzeitig.** Für ABRP muss Car Scanner getrennt sein.
- `wifi_mode` in der Konfiguration steht ggf. noch auf `APStation`; das ist wirkungslos,
  da diese Firmware kein AP startet.
