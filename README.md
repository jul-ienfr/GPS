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
- **Mode Debug / Production** : toggle HA pour basculer entre les deux modes

## 🔧 Matériel nécessaire

### Liste de courses

| Composant | Quantité | Prix indicatif | Où trouver |
|-----------|----------|----------------|------------|
| **ESP32-DevKitC** (WROOM-32) | 1 | ~8-15 € | AliExpress, Amazon |
| **Module GPS Neo-6M** (ou Neo-7M) | 1 | ~8-12 € | AliExpress, Amazon |
| **Antenne GPS externe** (câble magnétique) | 1 | ~3-5 € | Souvent incluse avec le Neo-6M |
| **Batterie LiPo 3.7V** (2000-5000 mAh) | 1 | ~5-10 € | Amazon, AliExpress |
| **Module chargeur TP4056** (avec protection LiPo) | 1 | ~1-2 € | AliExpress |
| **Switch ON/OFF** (optionnel) | 1 | ~1 € | Magasin électronique |
| **Boîtier** (imprimé 3D ou universel) | 1 | ~3-5 € | Thingiverse (impression 3D) |
| **Câbles jumper mâle-mâle** | Several | ~2 € | AliExpress |

**Total estimé : ~30-50 €**

### Détails des composants

#### ESP32-DevKitC
- **Pourquoi l'ESP32** : WiFi intégré, deep sleep ~10 µA, suffisant pour ESPHome
- **Version recommandée** : Module WROOM-32 (pas le WROOM-U)
- **Alimentation** : 3.3V ou 5V (régulateur intégré)

#### Module GPS Neo-6M
- **Protocole** : UART série, 9600 baud
- **Temps de premier fix** : ~30s extérieur, ~2-5 min intérieur
- **Antenne** : Placer l'antenne magnétique sur le toit du van (vue ciel dégagée)
- **Consommation** : ~45 mA en fonctionnement, ~5 µA en veille

#### Batterie LiPo
- **Capacité** : 2000-5000 mAh selon autonomie souhaitée
- **Autonomie estimée** (deep sleep 30 min) :
  - 2000 mAh → ~3-5 jours
  - 5000 mAh → ~7-12 jours
- **Important** : Utiliser une batterie avec protection intégrée OU un module TP4056 séparé

### Schéma de câblage

```
┌─────────────────────┐
│      ESP32          │
│                     │
│  GPIO22 (RX) ◄──────┼────── GPS TX (broche 2)
│  GPIO23 (TX) ───────┼──────► GPS RX (broche 3)
│                     │
│  GPIO4 (WAKE) ──────┼────── Switch → GND
│                     │       (INPUT_PULLUP, inverted)
│                     │
│  5V ────────────────┼──────► VCC GPS
│  GND ───────────────┼──────► GND GPS
│                     │
└─────────────────────┘

Alimentation:
  Batterie LiPo → TP4056 → 5V ESP32
                           → VCC GPS
```

> ⚠️ **Ne pas confondre TX/RX** : Le TX du GPS va sur le RX de l'ESP32 (GPIO22) et inversement.

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

## 📖 Utilisation

### Mode normal (van en route)
1. L'ESP32 se réveille automatiquement toutes les X minutes (configurable via `Sleep Duration`)
2. Il envoie les coordonnées GPS à Home Assistant via WiFi
3. Les données apparaissent sur la carte HA en temps réel
4. Si le van bouge, la position se met à jour à chaque cycle

### Mode veille (économie de batterie)
1. **Activer** : Dans HA, activer `input_boolean.mode_veille_gps_van`
2. **Effet** : L'ESP32 entre en deep sleep au prochain cycle (pas de réveil périodique)
3. **Réveiller** : Désactiver l'input boolean OU activer le switch GPIO4 (si câblé)
4. **Usage typique** : Van stationné, pas besoin de tracker en continu

### Créer l'input boolean dans HA

```yaml
# Ajouter dans configuration.yaml
input_boolean:
  mode_veille_gps_van:
    name: "Mode Veille GPS Van"
    icon: mdi:sleep
```

### Changer les paramètres
- **Intervalle de publication** : Slider `GPS Update Interval` dans HA (5-600s)
- **Durée de veille** : Slider `Sleep Duration` dans HA (1-240 min)
- Les valeurs sont sauvegardées automatiquement (restore_value)

### Automatisations utiles

**Alerte GPS en panne :**
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
          message: "Le module GPS ne répond plus depuis 5 min."
```

**Logger position van :**
```yaml
automation:
  - alias: "Logger position van"
    trigger:
      - platform: numeric_state
        entity_id: sensor.van_latitude
    action:
      - service: notify.mobile_app
        data:
          message: "Van à {{ states('sensor.van_latitude') }}°, {{ states('sensor.van_longitude') }}°"
```

---

## 🐛 Mode Debug / Production

Le tracker supporte deux modes opératoires, commutables depuis Home Assistant.

### Créer l'input boolean de debug

Le fichier `ha_configuration.yaml` contient les deux input booleans nécessaires. Copier son contenu dans `configuration.yaml` de HA :

### Comparaison des modes

| Paramètre | 🔧 Debug | 🏭 Production |
|-----------|---------|--------------|
| **Deep sleep** | Désactivé (reste éveillé) | Activé (1-240 min configurable) |
| **Intervalle GPS** | 1 seconde (pas de throttle) | 5-600s (configurable via slider) |
| **Logger level** | VERBOSE (tout détaillé) | WARN (erreurs uniquement) |
| **Capteurs debug** | WiFi Signal, Free Heap | Cachés |
| **Usage** | Développement, test, dépannage | Utilisation en van |

### Quand utiliser chaque mode

- **Debug** : Premier flash, test du câblage, dépannage, développement de nouvelles features
- **Production** : Utilisation quotidienne en van, optimisation batterie

> ⚠️ **Ne pas oublier de repasser en Production** après les tests ! Le mode Debug consomme beaucoup plus de batterie.

---

## 🔒 Sécurité batterie

**Problème** : Si Home Assistant est injoignable (panne réseau, HA éteint...), l'ESP32 pourrait rester éveillé et vider la batterie.

**Solution** : Au boot, l'ESP32 attend max ~95 secondes pour se connecter à HA. Si échec → deep sleep forcé.

```
Boot → 30s attente → 60s attente API → 5s tampon
  │
  └─ Si pas de réponse HA :
      └─ +30s → Deep sleep forcé (sécurité)
```

> Ce mécanisme protège la batterie même si HA est totally injoignable.

---

## 🖥️ Prérequis logiciels

### Sur votre ordinateur
1. **Python 3.8+** : [python.org](https://www.python.org/downloads/)
2. **ESPHome** :
   ```bash
   pip install esphome
   ```
3. **Git** (optionnel, pour cloner le dépôt)

### Dans Home Assistant
- **ESPHome integration** : Installer via HACS ou intégration native
- **HACS** (Home Assistant Community Store) : [hacs.xyz](https://hacs.xyz/)

---

## 🏠 Intégration Home Assistant

### Installation

#### Étape 1 : Préparer les secrets
```bash
cp secrets.yaml.example secrets.yaml
```
Éditer `secrets.yaml` avec vos vraies credentials WiFi et générer une clé API :
```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

#### Étape 2 : Câbler le matériel
Suivre le schéma de câblage dans la section [Matériel](#schéma-de-câblage). Vérifier les connexions avant de brancher.

#### Étape 3 : Flasher l'ESP32
1. Connecter l'ESP32 en USB
2. Lancer :
   ```bash
   esphome run gps.yaml
   ```
3. Sélectionner le port COM quand demandé
4. Attendre la compilation et le flash (~2-3 min)

#### Étape 4 : Vérifier le fonctionnement
```bash
esphome logs gps.yaml
```
Vérifier que les données GPS apparaissent (Latitude, Longitude, Satellites).

#### Étape 5 : Intégrer dans Home Assistant
1. Paramètres → Intégrations → Ajouter → ESPHome
2. Entrer l'IP de l'ESP32 (trouvé dans le dashboard ESPHome)
3. Saisir la clé API (`api_encryption_key` de secrets.yaml)

#### Étape 6 : Créer les input booleans dans HA
Copier le contenu de `ha_configuration.yaml` dans `configuration.yaml` de Home Assistant, ou ajouter manuellement :
```yaml
input_boolean:
  mode_veille_gps_van:
    name: "Mode Veille GPS Van"
    icon: mdi:sleep
  debug_mode_gps_van:
    name: "Debug Mode GPS Van"
    icon: mdi:bug
```

### Contrôles HA

| Entité | Type | Description |
|--------|------|-------------|
| GPS Update Interval | number (slider) | Intervalle de publication (5-600s) |
| Sleep Duration | number (slider) | Durée du deep sleep (1-240 min) |
| HA Sleep Status | binary_sensor | Import de `input_boolean.mode_veille_gps_van` |
| HA Debug Mode | binary_sensor | Import de `input_boolean.debug_mode_gps_van` |

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
| GPS WiFi Signal | dBm | Force du signal WiFi (mode debug) |
| GPS Free Heap | kB | Mémoire libre (mode debug) |

## 📁 Structure du projet

```
GPS/
├── gps.yaml              # Config ESPHome du tracker GPS
├── ha_configuration.yaml # Entités HA à copier dans configuration.yaml
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
