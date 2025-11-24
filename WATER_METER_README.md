# Wasserzähler mit Pulsausgang

Diese Erweiterung ermöglicht das Auslesen eines Wasserzählers mit Pulsausgang zusätzlich zum Ferraris-Stromzähler.

## Hardware-Anforderungen

- ESP-Mikrocontroller (z.B. ESP8266 oder ESP32)
- Wasserzähler mit Pulsausgang (1 Puls = 10 Liter)
- Optional: TCRT5000-Modul für Ferraris-Stromzähler

## Funktionsweise

Der Wasserzähler sendet für je 10 Liter Wasserverbrauch einen elektrischen Impuls. Diese Impulse werden vom ESP gezählt und in folgende Werte umgerechnet:

- **Wasserdurchfluss** (L/min): Berechnet aus der Zeit zwischen zwei aufeinanderfolgenden Pulsen
- **Wasserzählerstand** (L): Gesamtverbrauch durch Addition der Pulse (je 10 Liter)

## Hardware-Aufbau

### Nur Wasserzähler

Der Pulsausgang des Wasserzählers wird mit einem GPIO-Pin des ESP verbunden:

- **Pulsausgang (+)** → GPIO4 (oder anderer freier GPIO-Pin)
- **Pulsausgang (-)** → GND des ESP

**Wichtig:** Viele Wasserzähler arbeiten mit potentialfreien Kontakten (Reed-Relais). In diesem Fall sollte der interne Pull-Up-Widerstand des ESP aktiviert werden (bereits in der Konfiguration enthalten).

### Kombiniert mit Ferraris-Stromzähler

Bei gleichzeitiger Verwendung von Strom- und Wasserzähler:

- **Stromzähler (TCRT5000):**
  - VCC → 3,3V des ESP
  - GND → GND des ESP
  - A0 (analog) → GPIO17 (ADC-Pin)
  
- **Wasserzähler:**
  - Pulsausgang (+) → GPIO5 (unterschiedlicher Pin!)
  - Pulsausgang (-) → GND des ESP

**Hinweis:** Der Wasserzähler verwendet einen anderen GPIO-Pin als der Stromzähler, um Konflikte zu vermeiden.

## Verfügbare Konfigurationen

### 1. Nur Wasserzähler (`water_meter_pulse.yaml`)

Diese Konfiguration ist für den ausschließlichen Betrieb eines Wasserzählers gedacht.

**Sensoren:**
- Wasserdurchfluss (L/min)
- Wasserzählerstand (L)
- Wi-Fi Signal
- IP-Adresse
- WLAN-Status

**Funktionen:**
- Automatische Berechnung des Durchflusses
- Timeout nach 5 Minuten Inaktivität (Durchfluss → 0)
- Wiederherstellung des letzten Zählerstands nach Neustart
- Manuelles Überschreiben des Zählerstands

### 2. Kombinierte Messung (`combined_energy_water_meter.yaml`)

Diese Konfiguration ermöglicht die gleichzeitige Messung von Strom- und Wasserverbrauch.

**Stromzähler-Sensoren:**
- Momentanverbrauch (W)
- Stromzählerstand (Wh)
- Analoge Bandbreite (diagnostisch)
- Umdrehungsindikator (diagnostisch)
- Kalibrierungsstatus (diagnostisch)

**Wasserzähler-Sensoren:**
- Wasserdurchfluss (L/min)
- Wasserzählerstand (L)

**Gemeinsame Sensoren:**
- Wi-Fi Signal
- IP-Adresse
- WLAN-Status

## Einrichtung in Home Assistant

### 1. Input Helper erstellen

Für die Wiederherstellung des Zählerstands nach einem Neustart muss in Home Assistant ein Input Helper erstellt werden:

**Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Zahl**

- Name: `Wasserzähler letzter Wert`
- Entitäts-ID: `input_number.wasserzaehler_letzter_wert`
- Minimum: `0`
- Maximum: `1000000`
- Schrittweite: `10`
- Einheit: `L`

Bei kombinierter Verwendung zusätzlich für den Stromzähler:

- Name: `Stromzähler letzter Wert`
- Entitäts-ID: `input_number.stromzaehler_letzter_wert`
- Minimum: `0`
- Maximum: `1000000`
- Schrittweite: `0.01`
- Einheit: `kWh`

### 2. Automation für Zählerstand-Persistierung

Erstelle eine Automation, die den aktuellen Zählerstand in den Input Helper kopiert:

```yaml
- id: 'water_meter_cache_update'
  alias: Aktualisierung Wasserzähler-Cache
  trigger:
    - platform: state
      entity_id: sensor.wasserzaehlerstand
  condition: []
  action:
    - service: input_number.set_value
      target:
        entity_id: input_number.wasserzaehler_letzter_wert
      data:
        value: '{{ states(trigger.entity_id) }}'
  mode: single
```

Bei kombinierter Verwendung analog für den Stromzähler:

```yaml
- id: 'energy_meter_cache_update'
  alias: Aktualisierung Stromzähler-Cache
  trigger:
    - platform: state
      entity_id: sensor.stromzahlerstand
  condition: []
  action:
    - service: input_number.set_value
      target:
        entity_id: input_number.stromzaehler_letzter_wert
      data:
        value: '{{ states(trigger.entity_id) }}'
  mode: single
```

### 3. Secrets konfigurieren

Erstelle oder ergänze die Datei `secrets.yaml` im ESPHome-Verzeichnis:

```yaml
wifi_ssid: "Dein_WLAN_Name"
wifi_password: "Dein_WLAN_Passwort"
ha_api_key: "zufälliger_32_zeichen_schlüssel"
ota_password: "dein_ota_passwort"
fallback_hs_password: "fallback_passwort"
```

## Anpassung der Konfiguration

### GPIO-Pins ändern

Je nach ESP-Board können unterschiedliche Pins verwendet werden:

```yaml
# Wasserzähler Puls-Eingang
binary_sensor:
  - platform: gpio
    id: water_pulse
    pin:
      number: GPIO5  # Hier den gewünschten Pin eintragen
      mode:
        input: true
        pullup: true
```

### Liter pro Puls ändern

Falls dein Wasserzähler einen anderen Wert als 10 Liter pro Puls hat, passe dies in der Lambda-Funktion an:

```yaml
on_press:
  then:
    - lambda: |-
        // Hier den Wert ändern (z.B. 1.0 für 1 Liter pro Puls)
        id(water_volume_total) += 10.0;
        
        // Auch hier anpassen
        float flow_rate = (10.0 * 60000.0) / time_diff;
```

### Timeout für Durchflussanzeige ändern

Der Durchfluss wird nach 5 Minuten Inaktivität auf 0 gesetzt. Dies kann angepasst werden:

```yaml
sensor:
  - platform: template
    id: water_flow
    # ...
    filters:
      - timeout: 300s  # Zeit in Sekunden (300s = 5 Minuten)
        value: 0.0
```

## Entprellung

Die Konfiguration enthält bereits eine Entprellung von 50ms, um Störimpulse zu unterdrücken:

```yaml
filters:
  - delayed_on: 50ms
  - delayed_off: 50ms
```

Falls dennoch Probleme mit Mehrfachzählungen auftreten, können diese Werte erhöht werden (z.B. auf 100ms).

## Fehlersuche

### Problem: Keine Pulse werden erkannt

**Lösung:**
1. Prüfe die Verkabelung des Pulsausgangs
2. Teste mit einem Multimeter, ob der Wasserzähler tatsächlich Pulse sendet
3. Aktiviere Debug-Logging:
   ```yaml
   logger:
     level: DEBUG
   ```

### Problem: Zu viele Pulse (Zähler läuft zu schnell)

**Lösung:**
1. Erhöhe die Entprellungszeit auf 100-200ms
2. Prüfe auf elektrische Störungen in der Leitung

### Problem: Durchfluss wird nicht korrekt angezeigt

**Lösung:**
1. Prüfe, ob der Wert "Liter pro Puls" korrekt konfiguriert ist
2. Stelle sicher, dass mindestens zwei Pulse empfangen wurden (erster Puls dient als Referenz)

### Problem: Zählerstand wird nach Neustart nicht wiederhergestellt

**Lösung:**
1. Prüfe, ob der Input Helper in Home Assistant existiert
2. Prüfe, ob die Entitäts-ID in der YAML-Konfiguration korrekt ist
3. Stelle sicher, dass die Automation läuft

## Erweiterungen

### Alarm bei hohem Wasserverbrauch

In Home Assistant kann eine Automation erstellt werden, die bei ungewöhnlich hohem Durchfluss alarmiert:

```yaml
- id: 'high_water_flow_alert'
  alias: Alarm Hoher Wasserverbrauch
  trigger:
    - platform: numeric_state
      entity_id: sensor.wasserdurchfluss
      above: 20  # L/min
      for:
        minutes: 5
  action:
    - service: notify.notify
      data:
        message: "Achtung: Hoher Wasserverbrauch erkannt!"
```

### Tagesverbrauch berechnen

Mit dem Utility Meter in Home Assistant:

```yaml
utility_meter:
  water_daily:
    source: sensor.wasserzahlerstand
    cycle: daily
  
  water_monthly:
    source: sensor.wasserzahlerstand
    cycle: monthly
```

## Lizenz

Diese Erweiterung steht unter der gleichen MIT-Lizenz wie das ursprüngliche ESPHome Ferraris Meter Projekt.
