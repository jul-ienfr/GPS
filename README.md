# GPS + JK BMS Van

Un **seul ESP32** qui fait les deux :

| | |
|---|---|
| 🛰️ | **GPS Neo-6M** en UART (GPIO22 RX / GPIO23 TX, 9600 baud) → position du van |
| 🔋 | **JK BMS V11H** en Bluetooth → 8 tensions de cellule, courant, SOC, températures |
| 📡 | Remontée dans **Home Assistant** en ESPHome natif |
| 🗺️ | Position poussée vers **Traccar** par Home Assistant |

Issu de la fusion de deux projets qui tournaient chacun sur leur propre ESP32.

---

## 📁 Contenu du dossier

| Fichier | Rôle |
|---|---|
| `gps+jkbms.yaml` | La configuration ESPHome (le cœur) |
| `secrets.yaml.example` | Modèle à copier en `secrets.yaml` et à remplir |
| `ha_configuration.yaml` | Les entités Home Assistant (`input_boolean`, `device_tracker`) |
| `traccar.yaml` | La liaison Home Assistant → serveur Traccar |
| `ha_dashboard.yaml` | Le tableau de bord Lovelace |
| `.gitignore` | Empêche de committer `secrets.yaml` |

⚠️ **`secrets.yaml` ne doit jamais être commité.** Il contient votre mot de passe WiFi et votre clé OTA.

---

## 🔧 Prérequis

### Logiciel

- **ESPHome ≥ 2026.1.0** — exigé par le composant JK BMS
- Le **framework `esp-idf`**, pas `arduino`. C'est ce qu'utilise le dépôt `syssi/esphome-jk-bms`, et c'est indispensable : l'ESP32 n'a **qu'un seul radio 2,4 GHz** et doit le partager entre le WiFi et le Bluetooth.

### Matériel

- Un ESP32 (le firmware vise `esp32dev`), **alimenté en permanence**
- Un module GPS Neo-6M (ou compatible NMEA)
- Le JK BMS V11H, à portée Bluetooth (jusqu'à ~10 m en champ libre ; moins à travers un capot métallique)

### Le BMS, tel qu'il est câblé

Le JK BMS expose **deux** façons de le lire :

| | Prise | Distance | Câblage |
|---|---|---|---|
| **Bluetooth** ← *retenu ici* | BLE intégré | ~10 m | aucun fil |
| UART-TTL | JST 4 broches 1,25 mm | plusieurs mètres | filaire |

La prise UART-TTL du BMS est étiquetée **« RS485 »** mais c'est bien de l'UART TTL 3,3 V :

```
 Broche 1 : GND
 Broche 2 : RX  du BMS  → GPIO17 de l'ESP32
 Broche 3 : TX  du BMS  → GPIO16 de l'ESP32
 Broche 4 : VBAT         ⚠️ TENSION DU PACK — NE JAMAIS BRANCHER SUR L'ESP32
```

> ⚠️ **`VBAT` est la tension du pack.** Sur un 8S en LiFePO₄, ça monte à ~29 V en charge ; sur un 24S, bien plus. Brancher ça sur un GPIO de l'ESP32 le détruit immédiatement. Sur ce firmware, **GPIO16 et GPIO17 sont laissés libres exprès** pour cette option.
>
> L'avantage du filaire : plus de radio du tout à partager, donc plus de conflit WiFi/BLE, et une portée de plusieurs mètres. Si le Bluetooth s'avère capricieux à travers la cloison de la boîte à batteries, c'est la solution de repli — il faut alors changer `ble_client`/`jk_bms_ble` pour un `uart:` et un `jk_bms:`.

### Quel protocole pour le BMS ?

Le choix dépend de la **version hardware** du BMS :

| `protocol_version` | Version hardware |
|---|---|
| `JK02_32S` | **≥ 11.0 → votre V11H** |
| `JK02_24S` | ≥ 6.0 et < 11.0 |
| `JK04` | ≤ 3.0 |

C'est réglé dans `substitutions:` en haut de [`gps+jkbms.yaml`](gps+jkbms.yaml). Après le premier flash, l'entité **BMS Hardware Version** vous dira si le choix est le bon.

---

## 🔌 Câblage

### GPS (obligatoire)

```
Neo-6M          ESP32
------          -----
 VCC     ←────  3V3
 GND     ←────  GND
 TX      ────→  GPIO22   (RX de l'ESP32)
 RX      ←────  GPIO23   (TX de l'ESP32)
```

GPS fixé sur le toit, ESP32 près de la boîte à batteries : un long câble UART fait l'affaire (jusqu'à 5–10 m à 9600 bauds, ça passe).

### Coupure d'alimentation du GPS (optionnel — voir [Économie d'énergie](#-Économie-dénergie))

Sans ce montage, le switch **GPS Power** existe dans Home Assistant mais **ne coupe rien physiquement**. Pour qu'il coupe vraiment le 3,3 V du module, un MOSFET canal P en high-side :

```
   ESP32 GPIO25 ──┬─── 1 kΩ ─── G (grille)
                  │
             [10 kΩ pull-up vers le VCC du GPS]
                  │
   VCC 3,3 V ─────┴── S (source)
   D (drain) ──────────── VCC du Neo-6M
   GND du GPS ─────────── GND (commun)
```

| GPIO25 | Vgs | MOSFET | GPS |
|---|---|---|---|
| 0 V | −3,3 V | passant | **allumé** |
| 3,3 V | 0 V | bloqué | **coupé** |

- La **résistance de pull-up de 10 kΩ** est ce qui garantit que le GPS est **coupé au boot** de l'ESP32 (sinon la grille flotte).
- Le switch est en `inverted: true` côté ESPHome : logique « ON = alimenté ».
- MOSFET conseillé : **AO3401** ou **Si2301** (canal P, Vgs(th) autour de −1 V, boîtier SOT-23).
- **Ce montage est optionnel.** Si vous ne l'avez pas soudé, commentez le bloc `- platform: gpio` du fichier : le switch apparaîtra quand même dans HA, il n'aura juste aucun effet physique.

### Rien à câbler pour le BMS

Le BMS est lu en Bluetooth. Il faut juste connaître son **adresse MAC** (à relever avec *nRF Connect* sur téléphone, ou dans l'application Jikong) et la mettre dans `secrets.yaml`.

---

## 🚀 Installation

### 1. Les secrets

```bash
cp secrets.yaml.example secrets.yaml
```

Puis remplissez `secrets.yaml` :

```yaml
wifi_ssid: "Mon_WiFi"
wifi_password: "mon_mot_de_passe_wifi"
fallback_password: "un_autre_mot_de_passe"
api_encryption_key: "UNE_CLE_GENEREE"     # python -c "import secrets; print(secrets.token_urlsafe(32))"
ota_password: "mon_mot_de_passe_ota"
bms0_mac_address: "C8:47:80:2F:02:81"     # ← votre adresse
```

### 2. Compiler et flasher

```bash
esphome run gps+jkbms.yaml
```

Choisissez le port série au premier flash, ensuite les mises à jour se feront en WiFi (OTA).

> **Point d'accès de secours** — si le WiFi est introuvable, l'ESP32 crée un réseau `GPS BMS Fallback Hotspot`. Connectez-vous dessus pour corriger le `wifi_ssid`.

### 3. Côté Home Assistant

Copiez le contenu de [`ha_configuration.yaml`](ha_configuration.yaml) et de [`traccar.yaml`](traccar.yaml) dans votre `configuration.yaml` (ou passez par des packages) :

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Les deux `input_boolean` (`debug_mode_gps_van` et `mode_eco_gps_van`) sont **indispensables** : l'ESP32 les lit au démarrage.

### 4. Le tableau de bord

Importez [`ha_dashboard.yaml`](ha_dashboard.yaml) dans un nouveau dashboard (⋯ → *Modifier en YAML*).

> Les entités sont préfixées par le nom de l'appareil : `sensor.gps_jkbms_van_latitude`, `sensor.gps_jkbms_bms_total_voltage`… **Si vous changez `device_name` dans `gps+jkbms.yaml`, corrigez toutes les entités du dashboard.**

### 5. Traccar

Côté serveur Traccar, créez un appareil :

| Champ | Valeur |
|---|---|
| Identifiant (*identifier*) | `van` |
| Téléphone | votre adresse e-mail |
| Mot de passe | un mot de passe quelconque (non utilisé par le protocole OsmAnd) |

Puis, dans `traccar.yaml`, renseignez l'hôte, le port (5055) et l'identifiant. Le protocole utilisé est **OsmAnd** : une simple requête HTTP GET, celle qu'attend le port 5055.

> 💡 C'est **Home Assistant** qui envoie la position à Traccar, pas l'ESP32. Conséquence pratique : pour changer d'adresse de serveur ou d'intervalle, on modifie un fichier YAML dans HA — **aucun reflash nécessaire**.

---

## 🌿 Économie d'énergie

### Ce qu'il faut comprendre d'abord

> **Un switch ESPHome coupe une connexion *logique*, jamais une alimentation *physique*.**

C'est exactement le piège dans lequel on tombe : couper `switch.gps_power` dans HA ne débranche rien. Le module GPS reste soudé sur son 3,3 V et continue de tirer ses ~45 mA. Le seul levier qui change vraiment la consommation, c'est **couper le courant qui arrive au module** — d'où le MOSFET du montage ci-dessus.

Le projet GPS d'origine économisait énormément grâce au **deep sleep**. La version fusionnée ne peut pas : le BMS doit être interrogé en continu, donc l'ESP32 doit rester réveillé. On perd cet avantage, et cette section explique ce qu'on peut récupérer.

### Niveau 1 — sans rien souder (déjà dans le fichier)

Aucune modification de câblage, ces réglages sont déjà actifs :

| Réglage | Effet |
|---|---|
| `power_save_mode: VERY_LIGHT` | WiFi endormi entre les transmissions. C'est le bon compromis pour la cohabitation avec le BLE (c'est `WIFI_PS_MIN_MODEM`, qui laisse le radio répondre). |
| `output_power: 14dBm` | ~30 mA de moins qu'à 20 dBm, suffisant si l'ESP32 n'est pas à l'autre bout de la maison. |
| `throttle: 5s` (BMS) | 1 lecture toutes les 5 s au lieu de 1 s. En usage réel, 30 s ne change rien de visible dans HA et allège encore le radio. |
| `scan_parameters: active: false` | le scan BLE est passif : moins de trafic radio, donc moins de changements de canal WiFi/BLE. |
| `BMS Display Always On` : OFF | L'écran du BMS tire sur le pack pour rien. |
| `logger.level: WARN` | Moins d'écritures flash. |

**Ce qu'il reste à durcir** si vous voulez gratter encore (rien d'obligatoire) :

```yaml
jk_bms_ble:
  throttle: 30s        # au lieu de 5s — invisible dans HA, radio plus libre
wifi:
  output_power: 10dBm  # au lieu de 14dBm, si le routeur est à portée
```

Et, si vous les supprimez, deux blocs sont encore là par sécurité — ils ne servent qu'au dépannage sur place :

- **`captive_portal:`** — le portail qui s'ouvre quand le WiFi échoue (le point d'accès de secours le rend redondant)
- **`safe_mode:`** — le mode sans échec, qui redémarre en boucle si un flash se passe mal

> ⚠️ Ne retirez pas `captive_portal:` tant que le firmware n'est pas parfaitement stabilisé : c'est lui qui vous sauve quand `wifi_ssid` est mal orthographié.

### Niveau 2 — couper le GPS (montage MOSFET)

C'est ce que fait `input_boolean.mode_eco_gps_van` :

| Au passage en mode ÉCO | Effet |
|---|---|
| `switch.gps_power_switch` → OFF | le 3,3 V du Neo-6M est coupé — **~45 mA économisés** |
| `switch.ble_client_switch0` → OFF | le client BLE est déconnecté — le radio arrête de scruter |
| `gps_status` → OFF | Home Assistant et Traccar savent que la position n'est plus valide |
| publications GPS | suspendues (pas de position périmée ni de `NaN`) |

**Comment le rallumer :** repassez `input_boolean.mode_eco_gps_van` sur OFF dans Home Assistant. Les deux switchs se rallument automatiquement (c'est le `on_state` du capteur `ha_eco_mode`). Comptez ~2–5 s pour que le BLE se reconnecte au BMS, et **~30 s** pour que le GPS retrouve un fix (hot start).

Au boot, l'ESP32 applique aussi l'état du mode ÉCO reçu depuis HA — donc après une coupure de courant, il redémarre dans le mode où vous l'aviez laissé.

> Si vous coupez le switch **GPS Power** à la main (sans le mode ÉCO), même comportement : `gps_status` passe à OFF et les publications s'arrêtent. C'est un interrupteur manuel indépendant.

### Niveau 3 — deep sleep cyclique (volontairement désactivé)

C'est le seul réglage qui change l'ordre de grandeur. Le principe : dormir, se réveiller, publier, se rendormir.

```yaml
# ⚠️  À N'ACTIVER QU'EN TOUTE CONNAISSANCE DE CAUSE.
#     Le BMS n'est plus suivi en continu : entre deux cycles, HA n'a aucune
#     donnée batterie. C'est incompatible avec une surveillance BMS.
deep_sleep:
  id: deep_sleep_1
  wakeup_pin: GPIO4        # un interrupteur ou un simple bout de fil
  wakeup_pin_mode: HIGH    # à adapter au câblage
  sleep_duration: 10min
```

```yaml
# Dans substitutions : la durée de sommeil devient réglable
deep_sleep_duration: 10min
```

Conséquence incontournable : pendant le sommeil, **plus rien ne remonte** — ni GPS, ni BMS. Et le reveil par GPIO4 exige un fil sur GPIO4.

**Ma recommandation : ne l'activez pas.** Vous avez une alimentation permanente sur la batterie de service, et la valeur du projet est justement le suivi continu du BMS. Le gain (quelques dizaines de mA) est faible au regard d'une batterie de service qui se recharge au solaire ou à l'alternateur.

### Tableau de consommation

Mesures indicatives sur un ESP32 sous 5 V :

| État | Courant | Par jour | Sur une batterie de 100 Ah |
|---|---|---|---|
| **Ancien projet GPS** (deep sleep, cycle 10 min) | ~10 µA endormi / ~200 mA éveillé | ~240 mAh | **~400 jours** |
| **Version fusionnée**, GPS allumé | ~80 mA | ~1,9 Ah | **~50 jours** |
| **Version fusionnée**, mode ÉCO (GPS coupé) | ~35 mA | ~0,84 Ah | **~120 jours** |
| Deep sleep (jamais en pratique ici) | ~10 µA | ~0,24 mAh | plusieurs années |

Calcul de la ligne « ancien projet », parce que celle de l'ancien README était fausse :

```
Cycle de 10 minutes (600 s de sommeil, 30 s éveillé)
  Sommeil :  10 µA × 600 s        = 0,0017 mAh
  Éveillé  : 200 mA ×  30 s       = 1,67 mAh
  Total par cycle                 ≈ 1,67 mAh
  144 cycles par jour             ≈ 240 mAh/jour
```

> L'ancien README annonçait « 6 mAh » pour le sommeil et « ~300 jours » sur 100 Ah. Le sommeil coûte en réalité **0,0017 mAh**, pas 6 mAh, et la ligne « 100 Ah » masquait un calcul fait sur 100 mAh. D'où l'écart.

### Ce qui consomme, en clair

1. **Le WiFi** — c'est le premier poste. `VERY_LIGHT` le divise par 3 à 5.
2. **Le GPS** — ~45 mA en continu. C'est le second poste, et le seul qu'on peut couper proprement.
3. **Le partage radio WiFi/BLE** — un coût réel mais difficile à chiffrer ; il justifie le framework esp-idf et `active: false` dans `scan_parameters`.
4. **Le BMS lui-même** — alimenté par le pack, ~10–20 mA selon que l'écran est allumé. Hors de votre portée, sauf en éteignant l'écran (`BMS Display Always On`).
5. **L'ESP32 au repos** — ~5 mA, plancher incompressible.

---

## 🐛 Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| `Failed to connect to BLE device` | adresse MAC erronée, BMS endormi | vérifier la MAC avec *nRF Connect* |
| L'ESP32 redémarre en boucle | **stack overflow** — issue connue du composant | le firmware force déjà `CONFIG_ESP_MAIN_TASK_STACK_SIZE: "8192"` ; si ça suffit pas, montez à `12288` |
| Aucune entité `bms_*` dans HA | la connexion BLE échoue au boot | vérifier la portée ; le BMS doit être réveillé (il diffuse peu, par intermittence) |
| `Hardware Version` incohérente avec V11H | mauvais `protocol_version` | corriger dans `substitutions:` |
| OTA échoue | le BLE tourne encore | le firmware coupe `ble_client_switch0` au début de l'OTA ; si ça bloque, flasher par USB |
| Le switch GPS Power ne coupe rien | MOSFET non soudé | c'est normal (voir le montage), ou commenter le bloc `- platform: gpio` |
| Position jamais mise à jour | `gps_status` OFF | vérifier le câblage TX/RX (croisé), et laisser le GPS voir le ciel 1 à 2 min au premier démarrage (cold start) |
| La carte du dashboard est vide | `device_tracker.van` absent | le bloc `template:` de `ha_configuration.yaml` est-il bien rechargé ? |

### Un mot sur le WiFi et le Bluetooth

L'ESP32 n'a **qu'un seul radio 2,4 GHz**, utilisé à tour de rôle par le WiFi et le BLE. Ce n'est pas un vrai parallèle : chaque changement de canal coûte quelques centaines de ms. C'est pourquoi ce firmware est en **esp-idf** (le seul framework où la coexistence est correctement gérée) et pourquoi le scan BLE est passif (`active: false`).

En pratique : une latence WiFi un peu irrégulière, jamais de coupure. Si vous voyez des `WiFi disconnected` fréquents, éloignez l'ESP32 du routeur ou baissez encore `output_power` — un signal trop fort près du routeur sature aussi son récepteur.

---

## 📊 Les entités ESPHome

Toutes préfixées par `gps_jkbms_`.

**GPS** — `sensor.gps_jkbms_van_latitude`, `_longitude`, `_altitude`, `_speed`, `_gps_satellites`, `_gps_hdop`, `_gps_direction`
**BMS** — `sensor.gps_jkbms_bms_total_voltage`, `_current`, `_power`, `_state_of_charge`, `_capacity_remaining`, `_full_charge_capacity`, `_min_cell_voltage`, `_max_cell_voltage`, `_delta_cell_voltage`, `_average_cell_voltage`, `_temperature_1`, `_temperature_2`, `_mosfet_temperature`, `_balancing_current`, `_charging_cycles`, `_total_cycle_capacity`, `_total_runtime_hours`, `_cell_voltage_1` … `_cell_voltage_8`
**Diagnostic** — `sensor.gps_jkbms_gps_uptime`, `_gps_wifi_signal`, `_gps_free_heap`, `binary_sensor.gps_jkbms_bms_online`
**Réglages** — `number.gps_jkbms_gps_update_interval`
**Interrupteurs** — `switch.gps_jkbms_gps_power` (alimentation du module GPS), `switch.gps_jkbms_bms_bluetooth` (client BLE), `text_sensor.gps_jkbms_mode_eco` (état du mode éco)
**Commandes BMS ⚠️ écrivent dans le BMS** — `switch.gps_jkbms_bms_charging`, `_bms_discharging`, `_bms_balancer`, `_bms_display_always_on`, `button.gps_jkbms_bms_retrieve_settings`, `_bms_retrieve_device_info`

Le pack fait **8 cellules** : `cell_voltage_1` à `cell_voltage_8` sont les seuls capteurs de cellule activés. Les `cell_voltage_9` à `cell_voltage_32` du composant sont commentés dans le fichier — c'est aussi ce qui évite le stack overflow de l'issue #63.

> Les `switch.gps_jkbms_bms_charging` / `_discharging` **coupent réellement la charge ou la décharge du pack**. Ne les laissez pas à portée d'un clic distrait sur un téléphone.

---

## 🔗 Liens

- [syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms) — le composant BMS
- [ESPHome — GPS](https://esphome.io/components/gps.html)
- [ESPHome — ble_client](https://esphome.io/components/ble_client.html)
- [Traccar](https://www.traccar.org/)
