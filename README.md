# 🛻 GPS Van Tracker

Tracker GPS autonome pour van/camping-cars, basé sur ESP32 + ESPHome, intégré à Home Assistant.

## 📋 Caractéristiques

- **Deep sleep intelligent** : piloté depuis Home Assistant, avec fallback sécuritaire batterie
- **Anti-jitter GPS** : filtres `delta` pour ignorer la dérive de quelques mètres
- **Throttle configurable** : intervalle de publication 5-600s via slider HA
- **Détection panne** : binary_sensor `GPS Status` si le module GPS UART tombe en panne
- **Wake GPIO** : réveil possible par contact ignition (GPIO4 par défaut)
- **Optimisé batterie** : WiFi VERY_LIGHT, output power 14dBm, safe_mode, logger WARN
- **Capteurs HDOP + Direction** : précision GPS et cap en temps réel

## 🔧 Matériel

| Composant | Détail |
|-----------|--------|
| Carte | ESP32dev (Arduino framework) |
| GPS | Module UART (RX: GPIO22, TX: GPIO23, 9600 baud) |
| Alimentation | batterie 12V van + régulateur 3.3V |
| GPIO4 (optionnel) | Contact ignition pour réveil deep sleep |

## ⚡ Optimisation énergie

Le device consomme **~10µA en deep sleep** et se réveille périodiquement pour publier les données.

### Paramètres d'économie actifs

| Option | Valeur | Effet |
|--------|--------|-------|
| `power_save_mode` | `VERY_LIGHT` | WiFi en veille entre transmissions |
| `output_power` | `14dBm` | WiFi réduit (~30mA de moins que 20dBm) |
| `fast_connect` | `true` | Connexion WiFi directe au réveil (+3s gagnées) |
| `logger.level` | `WARN` | Moins d'écritures flash |
| `safe_mode` | activé | Récupération après 5 échecs de boot |

### Tuning pour autonomie maximale

| Paramètre | Recommandation van | Effet sur batterie |
|-----------|-------------------|-------------------|
| `GPS Update Interval` | 30-60s | Moins de transmissions WiFi = moins de conso |
| `Sleep Duration` | 10-30min | Plus le device dort, moins il consomme |
| GPS `update_interval` | 1s (défaut) | Le GPS lit à 1Hz, le throttle gère la publication |

### Calcul approximatif de consommation

```
Cycle typé (30s éveil, 10min sommeil):
- Deep sleep: 10µA × 600s = 6mAh
- Éveil (WiFi + GPS): ~200mA × 30s = 1.7mAh
- Total par cycle: ~2.3mAh
- Autonomie batterie 100Ah: ~43 000 cycles = ~300 jours
```

## 🏠 Intégration Home Assistant

### Installation

1. **Cloner le repo** :
   ```bash
   git clone https://github.com/jul-ienfr/GPS.git
   cd GPS
   ```

2. **Configurer les secrets** :
   ```bash
   cp secrets.yaml.example secrets.yaml
   ```
   Éditer `secrets.yaml` avec vos vraies credentials.

3. **Adapter le GPIO wakeup** (optionnel) :
   Si vous avez un contact ignition câblé sur un GPIO différent, modifier `wakeup_pin` dans `gps.yaml`.

4. **Flasher** :
   ```bash
   esphome run gps.yaml
   ```

### Contrôles HA

| Entité | Type | Description |
|--------|------|-------------|
| GPS Update Interval | number (slider) | Intervalle de publication (5-600s) |
| Sleep Duration | number (slider) | Durée du deep sleep (1-240 min) |
| HA Sleep Status | binary_sensor | Import de `input_boolean.mode_veille_gps_van` |

### Capteurs

| Entité | Unité | Description |
|--------|-------|-------------|
| Van Latitude | ° | Latitude (6 décimales) |
| Van Longitude | ° | Longitude (6 décimales) |
| Van Altitude | m | Altitude |
| Van Speed | km/h | Vitesse |
| GPS Satellites | - | Satellites visibles |
| GPS HDOP | - | Précision (0.5-2 = bon, >5 = mauvais) |
| GPS Direction | ° | Cap (0-360°) |
| GPS Status | bool | État du module GPS (true = OK) |
| GPS Uptime | s | Temps depuis le dernier reboot |

### Automation HA exemple

**Alerte si GPS en panne** :
```yaml
automation:
  - alias: "Alerte GPS panne"
    trigger:
      - platform: state
        entity_id: binary_sensor.gps_status
        to: "off"
        for: "00:05:00"
    action:
      - service: notify.mobile_app
        data:
          title: "⚠️ GPS Van"
          message: "Le module GPS ne répond plus depuis 5 minutes."
```

**Réveil au démarrage du van** :
```yaml
# Dans configuration.yaml — input_boolean controlling deep sleep
input_boolean:
  mode_veille_gps_van:
    name: "Mode Veille GPS"
    icon: mdi:sleep
```

## 📁 Structure du projet

```
GPS/
├── gps.yaml              # Config ESPHome du tracker GPS
├── secrets.yaml          # ⚠️ NE PAS COMMITTER — credentials
├── secrets.yaml.example  # Template pour les contributeurs
├── README.md             # Ce fichier
└── .gitignore            # Exclut secrets.yaml et .esphome/
```

## 🔒 Sécurité

- Toutes les clés et mots de passe sont dans `secrets.yaml` (exclu du git via `.gitignore`)
- API chiffrée (Noise PSK)
- OTA protégé par mot de passe
- WiFi fallback AP pour configuration initiale

## 🛠️ Dépannage

| Problème | Cause probable | Solution |
|----------|---------------|----------|
| GPS Status = OFF | Module GPS débranché ou défaillant | Vérifier câblage UART |
| Pas de données dans HA | WiFi non connecté | Vérifier `secrets.yaml`, utiliser le fallback AP |
| Deep sleep immédiat | HA injoignable au boot | Vérifier que HA tourne et que `input_boolean.mode_veille_gps_van` existe |
| Position inexacte | HDOP > 5 | Attendre une meilleure visibilité satellite |
