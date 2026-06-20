# GPS Van Tracker

Tracker GPS pour van/camping avec deep sleep intelligent piloté par Home Assistant.

## Matériel
- ESP32dev (Arduino framework)
- Module GPS via UART (RX: GPIO22, TX: GPIO23, 9600 baud)

## Fonctionnement
- Le GPS lit les coordonnées toutes les secondes
- Les données sont publiées vers Home Assistant à un intervalle configurable (défaut: 10s)
- Deep sleep configurable (1-240 min) via slider dans HA
- **Sécurité batterie** : si HA est injoignable au boot, le GPS force le deep sleep après 1min30 pour éviter la décharge

## Contrôles HA
| Entité | Type | Description |
|--------|------|-------------|
| GPS Update Interval | number | Intervalle de publication (5-600s) |
| Sleep Duration | number | Durée du deep sleep (1-240 min) |
| HA Sleep Status | binary_sensor | Import de `input_boolean.mode_veille_gps_van` |

## Capteurs
| Entité | Unité | Description |
|--------|-------|-------------|
| Van Latitude | ° | Latitude |
| Van Longitude | ° | Longitude |
| Van Altitude | m | Altitude |
| Van Speed | km/h | Vitesse |
| GPS Satellites | - | Satellites visibles |

## Configuration
1. Copier `secrets.yaml` et remplir les vraies credentials
2. Ajuster les pins UART si nécessaire
3. Flasher via ESPHome : `esphome run GPS.YAML`
