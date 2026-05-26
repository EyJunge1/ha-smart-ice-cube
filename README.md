# HA Smart Ice Cube

Home Assistant Automation-Blueprint für eine smarte Eiswürfelmaschine.

**Blueprint-Datei:** [`blueprints/automation/ha-smart-ice-cube.yaml`](blueprints/automation/ha-smart-ice-cube.yaml)  
**Repository:** [github.com/EyJunge1/ha-smart-ice-cube](https://github.com/EyJunge1/ha-smart-ice-cube)

Hardware:

- **NEO NAS-WR01B** Smart Plug (Strom + Leistungsmessung)
- **Tuya TS0001_fingerbot** Fingerbot (Knopf drücken)
- **input_boolean** für HomeKit-Steuerung

Steuerung über Home Assistant App, Apple HomeKit oder physischen Touch am Fingerbot.

## Ablauf

| Auslöser | Aktion |
|----------|--------|
| HomeKit/App: **Ein** | Strom an → warten → Fingerbot drückt Start-Knopf |
| HomeKit/App: **Aus** | Fingerbot drückt Aus-Knopf → warten → Strom aus |
| **Touch** (Steckdose aus) | Strom an → warten → drücken → Status sync |
| **Touch** (Steckdose an) | Kein zweiter Druck – Power-Sensor prüft ob Maschine läuft, Status wird synchronisiert |

Der Power-Sensor erkennt, ob der Kompressor wirklich läuft (nicht nur ob Strom an der Steckdose liegt).

## Blueprint importieren (URL)

**Falsch** (führt zu „Unknown error“):

```text
https://github.com/EyJunge1/ha-smart-ice-cube.git/blueprint/automations/ha-smart-ice-cube.yaml
```

**Richtig** – in HA unter *Einstellungen → Automatisierungen → Blueprints → Blueprint importieren*:

```text
https://github.com/EyJunge1/ha-smart-ice-cube/blob/main/blueprints/automation/ha-smart-ice-cube.yaml
```

Wichtig:

- `blob/main` (nicht `.git`)
- `blueprints/automation` (nicht `blueprint/automations`)

**One-Click-Import** (My Home Assistant):

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FEyJunge1%2Fha-smart-ice-cube%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fha-smart-ice-cube.yaml)

Alternativ die Datei manuell kopieren (siehe unten).

## Installation

### 1. Blueprint nach Home Assistant kopieren

Kopiere die Blueprint-Datei in dein HA-Konfigurationsverzeichnis:

```text
/config/blueprints/automation/ha-smart-ice-cube.yaml
```

Danach: **Einstellungen → Automatisierungen & Szenen → Blueprints** → Blueprint neu laden (oder HA neu starten).

Alternativ per Samba/SSH/VS Code Add-on in `config/blueprints/automation/` ablegen, oder den Ordner `blueprints/` aus diesem Repo kopieren.

### 2. Zigbee2MQTT – Geräte konfigurieren

#### Smart Plug (NEO NAS-WR01B)

In Zigbee2MQTT unter deinem Smart-Plug-Gerät:

| Parameter | Wert | Warum |
|-----------|------|-------|
| `power_outage_memory` | `off` | Nach Stromausfall bleibt die Steckdose aus – Maschine startet nicht unkontrolliert |

#### Fingerbot (Tuya TS0001_fingerbot)

Unter deinem Fingerbot-Gerät:

| Parameter | Wert | Warum |
|-----------|------|-------|
| `mode` | `click` | Ein Klick und zurück – kein dauerhaftes Drücken |
| `touch` | `ON` | Physischer Touch am Fingerbot wird aktiviert |

Optional (empfohlen):

| Parameter | Wert | Warum |
|-----------|------|-------|
| `state_action` | `true` | Zusätzliche Action-Events bei State-Änderungen |

### 3. input_boolean anlegen

**Einstellungen → Geräte & Dienste → Helfer → Schalter erstellen**

- Name: `HA Smart Ice Cube`
- Entity-ID (empfohlen): `input_boolean.ha_smart_ice_cube`

Dieser Helfer ist die **einzige** Steuerung in HomeKit – nicht die Steckdose direkt exponieren.

### 4. Automation aus Blueprint erstellen

**Einstellungen → Automatisierungen → Blueprint erstellen → „HA Smart Ice Cube“**

| Blueprint-Input | Entity (Beispiel) |
|-----------------|-------------------|
| Fingerbot | `switch.0xa4c13845ef27b2ea` |
| Steckdose | `switch.ha_smart_ice_cube_plug` |
| Leistungssensor | `sensor.ha_smart_ice_cube_power` |
| Status-Schalter | `input_boolean.ha_smart_ice_cube` |
| Leistungsschwellwert | `5` W (anpassen nach Messung) |
| Anlaufzeit | `10` s |
| Nachlaufzeit | `10` s |
| Power-Stabilisierung | `10` s |

Bestehende Entity-IDs (z. B. `switch.eiswurfelmaschine_steckdose`) können weiterverwendet werden – die Namen sind nur Vorschläge.

### 5. HomeKit

In der HomeKit-Bridge-Konfiguration (`configuration.yaml` oder UI) den `input_boolean` exponieren:

```yaml
homekit:
  filter:
    include_entities:
      - input_boolean.ha_smart_ice_cube
```

Die **Steckdose nicht** in HomeKit zeigen – sonst kann Strom ohne sauberes Ausschalten abgeschaltet werden.

### 6. Alte Scripts entfernen

Nach erfolgreichem Test können alte Scripts gelöscht werden (z. B. `script.eiswurfelmaschine_drucken`, `script.eiswurfelmaschine_on_off`).

## Leistungsschwellwert kalibrieren

1. Maschine **aus** (Power-Button nicht gedrückt), Steckdose **an** → Power-Wert notieren (nur Standby/Display).
2. Maschine **an** (Kompressor läuft) → Power-Wert notieren.
3. Schwellwert zwischen beiden Werten setzen (z. B. 5 W, wenn Standby < 2 W und Betrieb > 20 W).

**Hinweis:** Bei „Wasser leer“ kann der Kompressor weiterlaufen und hohe Leistung ziehen – die Automation erkennt das korrekt als „läuft“. Das ist gewollt.

## Fehlerbehebung

| Problem | Lösung |
|---------|--------|
| Blueprint nicht sichtbar | Datei muss exakt `ha-smart-ice-cube.yaml` heißen und unter `config/blueprints/automation/` liegen |
| Touch am Fingerbot löst nichts aus | `touch: ON` und `mode: click` in Z2M prüfen |
| Fingerbot drückt doppelt | Blueprint nutzt `mode: single` – während einer Sequenz keine neuen Trigger |
| `on_time` funktioniert nicht | In Z2M prüfen ob Firmware `on_time` unterstützt; ggf. `switch.turn_on` → 1 s warten → `switch.turn_off` manuell testen |
| HomeKit zeigt falschen Status | Nur `input_boolean.ha_smart_ice_cube` exponieren, nicht die Steckdose |
| Maschine startet nach Stromausfall | `power_outage_memory: off` an der Steckdose setzen |

## Lizenz

Siehe [LICENSE](LICENSE).
