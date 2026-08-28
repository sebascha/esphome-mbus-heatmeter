# Telegramm-Analyse: eigene Zähler anpassen

Die Sensoren dieses Projekts werden über **VIF-Codes** (Value Information Field) definiert. Diese sind in EN 13757-3 weitgehend standardisiert, unterscheiden sich zwischen Herstellern aber in Skalierung und Auswahl. Diese Seite zeigt, wie man die passenden Codes für den eigenen Zähler findet.

## 1. Dump aktivieren

In der YAML den Logger auf DEBUG stellen, das Polling-Intervall vorübergehend verkürzen und einen Dummy-Sensor ergänzen, der auf keinen realen Record passt:

```yaml
logger:
  level: DEBUG

mbus:
  - id: wmz
    uart_id: mbus_uart
    update_interval: 60s

sensor:
  - platform: mbus
    mbus_id: wmz
    name: "Dump-Dummy"
    mbus_vife: 0x00
```

Die Komponente schreibt dann bei jedem Poll alle gefundenen Records ins Log:

```
function: MBUS_INSTANT_VALUE, datatype: MBUS_BCD8, storage: 0, tariff: 0, subunit: 0, VIF(E): 0x05, value: 476520
```

Für jeden gewünschten Wert werden daraus abgelesen: **VIF(E)**, **storage** (0 = aktueller Wert; >0 sind Stichtags- und Monatswerte) und der **Rohwert**, an dem sich die Skalierung ablesen lässt.

## 2. Verifizierte Zuordnung: Rossweiner heatplus / Qundis Q heat 5.5

Aus einem Telegramm mit 232 Byte Nutzdaten, Storage 0:

| Wert | VIF | Datentyp | Rohwert | Skalierung | Ergebnis |
|---|---|---|---|---|---|
| Wärmemenge | `0x05` | BCD8 | 476520 | × 0,1 | 47.652,0 kWh |
| Volumen | `0x13` | BCD8 | 3984376 | × 0,001 | 3.984,376 m³ |
| Durchfluss | `0x3B` | INT24 | 0 | × 1 | 0 l/h |
| Leistung | `0x2B` | INT32 | 0 | × 1 | 0 W |
| Vorlauftemperatur | `0x5A` | INT16 | 289 | × 0,1 | 28,9 °C |
| Rücklauftemperatur | `0x5E` | INT16 | 236 | × 0,1 | 23,6 °C |
| Spreizung | `0x62` | INT16 | 53 | × 0,1 | 5,3 K |

Zusätzlich im Telegramm, hier nicht als Sensor angelegt:

| Inhalt | VIF | Bemerkung |
|---|---|---|
| Datum/Uhrzeit | `0x6D` | Typ-F-Zeitstempel |
| Betriebsstunden | `0x22` | BCD6 |
| Fehlerdatum | `0x6C` | `FFFF` = kein Fehler |
| Fabrikationsnummer | `0x78`, `0xFD10` | BCD8 |
| Firmware/Modelldaten | `0xFD0C`, `0xFD0B` | `0xFD0B` enthält als ASCII die Modulkennung |
| Fehlerflags | `0xFD17` | 0 = fehlerfrei |
| Stichtagswert | `0x05`, Storage 1 | Jahresstichtag |
| Monatswerte | `0x05`, Storage 8–20 | 13 Monatswerte — nützlich für die Historie |

Die Sensoren dieses Projekts filtern implizit auf `storage: 0`, greifen also immer den aktuellen Wert ab.

## 3. VIF-Bereiche nach EN 13757-3 (Auszug)

Sucht man in einem fremden Telegramm nach den passenden Records, helfen diese Bereiche. Die letzten Bits kodieren jeweils den Dezimalexponenten, weshalb je nach Zähler ein Nachbarwert auftaucht.

| Größe | VIF-Bereich | Einheiten |
|---|---|---|
| Energie | `0x00`–`0x07` | Wh (10⁻³ … 10⁴) |
| Energie | `0x08`–`0x0F` | J |
| Volumen | `0x10`–`0x17` | m³ (10⁻⁶ … 10⁻¹) |
| Masse | `0x18`–`0x1F` | kg |
| Betriebszeit | `0x20`–`0x23` | s / min / h / d |
| Leistung | `0x28`–`0x2F` | W |
| Leistung | `0x30`–`0x37` | J/h |
| Volumenstrom | `0x38`–`0x3F` | m³/h, m³/min, m³/s |
| Vorlauftemperatur | `0x58`–`0x5B` | °C (10⁻³ … 10⁰) |
| Rücklauftemperatur | `0x5C`–`0x5F` | °C |
| Temperaturdifferenz | `0x60`–`0x63` | K |
| Externe Temperatur | `0x64`–`0x67` | °C |
| Datum / Zeitstempel | `0x6C`, `0x6D` | Typ G / Typ F |
| Herstellerspezifisch | `0xFD..`, `0xFF..` | erweiterte VIFE |

Beispiel: `0x05` liegt im Energiebereich, die unteren drei Bits (`101` = 5) ergeben den Exponenten 10^(5−3) = 10² Wh = 100 Wh — daher der Faktor 0,1 für kWh. `0x13` liegt im Volumenbereich mit 10^(3−6) m³ = Liter, daher 0,001.

## 4. Sensor ergänzen

Mit VIF und Skalierung ist die Sensordefinition kurz:

```yaml
  - platform: mbus
    mbus_id: wmz
    name: "Waermemenge"
    mbus_vife: 0x05
    filters:
      - multiply: 0.1
    unit_of_measurement: kWh
    device_class: energy
    state_class: total_increasing
    accuracy_decimals: 1
```

Wichtig für die Statistik in Home Assistant: Zählerstände bekommen `state_class: total_increasing`, Momentanwerte `measurement`.

## 5. Plausibilitätsprüfung

Der Zählerstand im Dump muss zur Anzeige am Gerätedisplay passen. Beim heatplus zeigt die Display-Ebene **L2 (Momentanwerte)** Durchfluss, Vor-/Rücklauftemperatur, Spreizung, Leistung, Betriebsstunden und den hochauflösenden Zählerstand — ideal zum Abgleich. Weicht ein Wert um Faktor 10, 100 oder 1000 ab, ist die Skalierung falsch, nicht der VIF.

## 6. Fremde Telegramme beisteuern

Für einen Pull Request oder ein Issue genügen: Zählermodell, Modulbezeichnung, die Record-Liste aus dem Log und — falls vorhanden — der Roh-Hexdump. Daraus lässt sich die Sensordefinition ableiten und in die Tabelle oben aufnehmen.
