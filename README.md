# Rover ESP32-S3 : véhicule télécommandé par WiFi

Firmware d'un véhicule à deux essieux (propulsion par moteur à courant continu, direction par servomoteur) basé sur la carte **Seeed Studio XIAO ESP32S3**. La carte crée son propre réseau WiFi et héberge une interface web de pilotage, accessible depuis un téléphone ou un ordinateur, sans installation ni connexion Internet.

## Sommaire

1. [Fonctionnalités](#fonctionnalités)
2. [Architecture](#architecture)
3. [Matériel requis](#matériel-requis)
4. [Brochage](#brochage)
5. [Câblage des capteurs HC-SR04](#câblage-des-capteurs-hc-sr04)
6. [Installation logicielle](#installation-logicielle)
7. [Utilisation](#utilisation)
8. [Protocole de communication](#protocole-de-communication)
9. [Paramètres de calibration](#paramètres-de-calibration)
10. [Sécurités intégrées](#sécurités-intégrées)
11. [Mode simulation](#mode-simulation)
12. [Dépannage](#dépannage)
13. [Limites connues et améliorations possibles](#limites-connues-et-améliorations-possibles)
14. [Structure du dépôt](#structure-du-dépôt)

## Fonctionnalités

- Point d'accès WiFi autonome (SSID `ROVER`), sans routeur ni Internet.
- Interface web embarquée dans la mémoire flash de la carte : un seul fichier `.ino` à téléverser, sans système de fichiers LittleFS.
- Pilotage en temps réel par WebSocket : curseur de direction, réglage de la vitesse, choix du sens (avant/arrière), bouton d'arrêt d'urgence.
- Télémétrie en direct : vitesse estimée, angle de direction, distances avant et arrière.
- Détection d'obstacles par deux capteurs ultrasons, avec blocage automatique de la marche concernée.
- Arrêt automatique en cas de perte de liaison.
- Accès par `http://192.168.4.1` ou `http://rover.local` (mDNS).
- Interface installable comme application (ajout à l'écran d'accueil) grâce à un manifeste et à une icône servis par la carte.
- Mode simulation de l'interface, utilisable sans matériel.

## Architecture

```
 Téléphone / PC                          XIAO ESP32S3
┌──────────────────┐   WiFi (AP ROVER)  ┌──────────────────────────────┐
│ Navigateur web   │◄──────────────────►│ Serveur HTTP (port 80)       │
│ Interface HTML   │   HTTP : page      │  /  /manifest.json /icon.svg │
│                  │   WebSocket : /ws  │ Serveur WebSocket (/ws)      │
└──────────────────┘   commandes JSON   │ Boucle de contrôle (100 ms)  │
                       télémétrie JSON  └──────┬───────────┬───────────┘
                                               │           │
                                  Driver moteur + moteur DC   Servo direction
                                               │
                                       2 × HC-SR04 (avant, arrière)
```

La page HTML est stockée dans la variable `INDEX_HTML` (mémoire flash, `PROGMEM`) et servie directement par la carte.

## Matériel requis

| Élément | Remarque |
|---|---|
| Seeed Studio XIAO ESP32S3 | Logique 3,3 V, GPIO non tolérants au 5 V |
| Driver moteur à pont en H (ex. L298N) | Entrées ENA (PWM), IN1, IN2 |
| Moteur à courant continu | Propulsion |
| Servomoteur | Direction, débattement utilisé : 45° à 135° |
| 2 × capteurs ultrasons HC-SR04 | Avant et arrière |
| 4 résistances : 2 × 1 kΩ, 2 × 2 kΩ | Ponts diviseurs sur les broches ECHO |
| Alimentation adaptée aux moteurs | Masse commune avec la carte |

## Brochage

| Fonction | Broche XIAO | Description |
|---|---|---|
| `ENA` | D3 | PWM de propulsion (5 kHz, 8 bits) |
| `IN1` | D1 | Sens du moteur |
| `IN2` | D2 | Sens du moteur |
| `SERVO_PIN` | D4 | Signal du servo de direction |
| `TRIG_F` | D0 | Déclencheur du capteur avant |
| `ECHO_F` | D5 | Écho du capteur avant (via pont diviseur) |
| `TRIG_R` | D8 | Déclencheur du capteur arrière |
| `ECHO_R` | D9 | Écho du capteur arrière (via pont diviseur) |

## Câblage des capteurs HC-SR04

> **Attention.** Les GPIO de la XIAO ESP32S3 ne tolèrent pas le 5 V. Le signal ECHO du HC-SR04 est à 5 V : un pont diviseur est **obligatoire** sur chaque broche ECHO, sous peine d'endommager la carte.

```
ECHO (5 V) ──[ 1 kΩ ]──┬── GPIO (D5 ou D9)
                       │
                     [ 2 kΩ ]
                       │
                      GND
```

Tension obtenue : 5 V × 2 / (1 + 2) ≈ 3,3 V. La broche TRIG, qui est une sortie de la carte, peut être reliée directement. Alimenter les capteurs en 5 V et relier toutes les masses ensemble (carte, driver, capteurs, alimentation).

## Installation logicielle

### 1. Environnement

- Installer l'[IDE Arduino](https://www.arduino.cc/en/software).
- Ajouter le support ESP32 (gestionnaire de cartes, paquet « esp32 » d'Espressif). Le code est compatible avec les versions 2.x et 3.x du paquet (gestion de l'API `ledc` selon la version).

### 2. Bibliothèques

À installer depuis le gestionnaire de bibliothèques (ou depuis GitHub pour les deux premières, selon la disponibilité) :

- `ESPAsyncWebServer`
- `AsyncTCP`
- `ArduinoJson` (version 6.x ou 7.x ; le code utilise `StaticJsonDocument`)
- `ESP32Servo`

Les bibliothèques `WiFi` et `ESPmDNS` sont fournies avec le paquet ESP32.

### 3. Téléversement

1. Ouvrir `controle_de_vehicule/controle_de_vehicule.ino`.
2. Sélectionner la carte **XIAO_ESP32S3** et le port série correspondant.
3. Cliquer sur **Téléverser**.
4. Ouvrir le moniteur série à 115200 bauds pour vérifier le démarrage (adresse IP et état du mDNS).

## Utilisation

1. Alimenter le rover.
2. Se connecter au réseau WiFi **ROVER** (mot de passe par défaut : `rover1234`).
3. Ouvrir un navigateur à l'adresse `http://192.168.4.1` (ou `http://rover.local`).
4. Vérifier que l'indicateur en haut à gauche est **cyan** (connecté). Il reste rouge si le réseau ou l'adresse est incorrect.
5. Piloter :
   - **Direction** : faire glisser le curseur horizontal ; il revient au centre au relâchement.
   - **Sens** : boutons *Avant* / *Arrière*.
   - **Vitesse** : boutons `+` et `-` (appui court : pas de 10 %, appui prolongé : incrément continu de 5 %).
   - **STOP** : remet la direction au centre, la vitesse à zéro et envoie l'ordre d'arrêt.

Le champ d'adresse de l'en-tête permet de saisir une autre adresse (par défaut : l'hôte de la page, ou `rover.local`).

**Installation en application** : depuis le navigateur du téléphone, utiliser « Ajouter à l'écran d'accueil » pour obtenir une icône dédiée et un affichage plein écran.

## Protocole de communication

Les échanges se font en JSON sur le WebSocket `ws://<adresse>/ws`.

**Navigateur → rover** (envoyé toutes les 100 ms)

```json
{ "type": "drive", "x": 0.35, "y": 0.60, "speedMax": 100 }
```

| Champ | Plage | Signification |
|---|---|---|
| `x` | -1 à 1 | Direction (négatif : gauche, positif : droite) |
| `y` | -1 à 1 | Propulsion, signe = sens, module = vitesse relative |
| `speedMax` | 0 à 100 | Plafond de PWM en pourcentage |

```json
{ "type": "stop" }
```

**Rover → navigateur**

À la connexion (poignée de main) :

```json
{ "type": "hello", "device": "rover" }
```

Télémétrie (toutes les 100 ms) :

```json
{ "distanceFront": 82, "distanceRear": 140, "speed": 1.2, "angle": 15.0 }
```

| Champ | Unité | Signification |
|---|---|---|
| `distanceFront`, `distanceRear` | cm | Distances mesurées (400 si aucun écho) |
| `speed` | m/s | Vitesse **estimée** à partir de la commande (non mesurée) |
| `angle` | degrés | Angle de direction par rapport au centre |

## Paramètres de calibration

À ajuster dans le code après les essais mécaniques :

| Constante | Valeur | Rôle |
|---|---|---|
| `SERVO_CENTER` | 90 | Angle du servo pour rouler droit |
| `SERVO_MIN` / `SERVO_MAX` | 45 / 135 | Butées de direction |
| `SAFE_DISTANCE_CM` | 25 | Seuil de détection d'obstacle |
| `CMD_TIMEOUT_MS` | 500 | Délai sans commande avant arrêt automatique |
| `AP_SSID` / `AP_PASS` | `ROVER` / `rover1234` | Identifiants du point d'accès (8 caractères minimum) |

Le seuil d'alerte de l'interface (25 cm, fonction `applyTelemetry`) doit rester cohérent avec `SAFE_DISTANCE_CM`.

## Sécurités intégrées

- **Perte de liaison** : sans commande reçue pendant 500 ms, la propulsion est coupée.
- **Obstacle avant** : la marche avant est bloquée sous 25 cm ; la marche arrière reste possible.
- **Obstacle arrière** : la marche arrière est bloquée sous 25 cm ; la marche avant reste possible.
- **Bouton STOP** : coupe immédiatement la propulsion.
- **Alerte visuelle** : bandeau rouge dans l'interface lorsqu'un obstacle est détecté.

## Mode simulation

Lorsque le fichier HTML est ouvert sans rover connecté, l'interface fonctionne en mode simulation (badge « SIMULATION » en bas à gauche) : les valeurs de télémétrie sont fictives et évoluent seules. Ce mode permet de tester l'interface sans matériel ; il se désactive dès que la poignée de main du rover est reçue.

## Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Indicateur rouge, badge SIMULATION | Téléphone connecté à un autre WiFi | Se connecter au réseau `ROVER` |
| La page ne s'ouvre pas | Mauvaise adresse | Utiliser `http://192.168.4.1` |
| `rover.local` introuvable | mDNS non pris en charge (certains Android) | Utiliser l'adresse IP |
| Distances toujours à 400 cm | Capteur mal câblé ou non alimenté | Vérifier TRIG/ECHO, le 5 V et les masses |
| Carte qui plante ou chauffe | ECHO relié directement en 5 V | Ajouter les ponts diviseurs |
| Le rover s'arrête par saccades | Liaison WiFi instable | Se rapprocher du rover, limiter les obstacles |
| Direction décentrée | Calibration mécanique | Ajuster `SERVO_CENTER`, `SERVO_MIN`, `SERVO_MAX` |
| Erreur de compilation sur `ledc` | Version du paquet ESP32 | Mettre à jour le paquet ESP32 |

## Limites connues et améliorations possibles

- La vitesse affichée est une estimation proportionnelle à la commande ; un capteur à effet Hall ou un encodeur permettrait de la mesurer.
- Le mot de passe du point d'accès est écrit en clair dans le code : le modifier avant tout usage en environnement partagé.
- Une rampe d'accélération limiterait les à-coups de propulsion.
- Une alimentation de la carte distincte de celle des moteurs limite les perturbations (chutes de tension au démarrage du moteur).

## Structure du dépôt

```
controle_de_vehicule/
├── controle_de_vehicule.ino   # Firmware et interface web embarquée
└── README.md
```
