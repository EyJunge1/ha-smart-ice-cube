# Eiswürfelmaschine – Home Assistant Automation

Automation-Blueprint für eine Eiswürfelmaschine mit:

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

## Installation

### 1. Blueprint nach Home Assistant kopieren

Kopiere den Ordner `blueprints/` in dein HA-Konfigurationsverzeichnis:

```text
/config/blueprints/automation/ha-smart-ice-cube.yaml
```

Danach: **Einstellungen → Automatisierungen & Szenen → Blueprints** → Blueprint neu laden (oder HA neu starten).

Alternativ per Samba/SSH/VS Code Add-on in `config/blueprints/automation/` ablegen.

### 2. Zigbee2MQTT – Geräte konfigurieren

#### Steckdose (NEO NAS-WR01B)

In Zigbee2MQTT unter dem Gerät **Küche-Eiswürfelmaschine-Steckdose**:

| Parameter | Wert | Warum |
|-----------|------|-------|
| `power_outage_memory` | `off` | Nach Stromausfall bleibt die Steckdose aus – Maschine startet nicht unkontrolliert |

#### Fingerbot (Tuya TS0001_fingerbot)

Unter **Küche-Eiswürfelmaschine-Drücker**:

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

- Name: `Eiswürfelmaschine läuft`
- Entity-ID: `input_boolean.eiswuerfelmaschine_laeuft` (oder ähnlich)

Dieser Helfer ist die **einzige** Steuerung in HomeKit – nicht die Steckdose direkt exponieren.

### 4. Automation aus Blueprint erstellen

**Einstellungen → Automatisierungen → Blueprint erstellen → „Eiswürfelmaschine“**

| Blueprint-Input | Entity (Beispiel) |
|-----------------|-------------------|
| Fingerbot | `switch.0xa4c13845ef27b2ea` |
| Steckdose | `switch.eiswurfelmaschine_steckdose` |
| Leistungssensor | `sensor.eiswurfelmaschine_steckdose_power` |
| Status-Schalter | `input_boolean.eiswuerfelmaschine_laeuft` |
| Leistungsschwellwert | `5` W (anpassen nach Messung) |
| Anlaufzeit | `10` s |
| Nachlaufzeit | `10` s |
| Power-Stabilisierung | `10` s |

### 5. HomeKit

In der HomeKit-Bridge-Konfiguration (`configuration.yaml` oder UI) den `input_boolean` exponieren:

```yaml
homekit:
  filter:
    include_entities:
      - input_boolean.eiswuerfelmaschine_laeuft
```

Die **Steckdose nicht** in HomeKit zeigen – sonst kann Strom ohne sauberes Ausschalten abgeschaltet werden.

### 6. Alte Scripts entfernen

Nach erfolgreichem Test können diese gelöscht werden:

- `script.eiswurfelmaschine_drucken`
- `script.eiswurfelmaschine_on_off`

## Leistungsschwellwert kalibrieren

1. Maschine **aus** (Power-Button nicht gedrückt), Steckdose **an** → Power-Wert notieren (nur Standby/Display).
2. Maschine **an** (Kompressor läuft) → Power-Wert notieren.
3. Schwellwert zwischen beiden Werten setzen (z. B. 5 W, wenn Standby < 2 W und Betrieb > 20 W).

**Hinweis:** Bei „Wasser leer“ kann der Kompressor weiterlaufen und hohe Leistung ziehen – die Automation erkennt das korrekt als „läuft“. Das ist gewollt.

## Fehlerbehebung

| Problem | Lösung |
|---------|--------|
| Touch am Fingerbot löst nichts aus | `touch: ON` und `mode: click` in Z2M prüfen |
| Fingerbot drückt doppelt | Blueprint nutzt `mode: single` – während einer Sequenz keine neuen Trigger |
| `on_time` funktioniert nicht | In Z2M prüfen ob Firmware `on_time` unterstützt; ggf. in HA `switch.turn_on` → 1 s warten → `switch.turn_off` manuell testen |
| HomeKit zeigt falschen Status | Nur `input_boolean` exponieren, nicht die Steckdose |
| Maschine startet nach Stromausfall | `power_outage_memory: off` an der Steckdose setzen |

## Lizenz

Siehe [LICENSE](LICENSE).
