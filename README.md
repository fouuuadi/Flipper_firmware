# Flipper_firmware

Firmware **ESP32** des boutons physiques de la borne flipper HETIC. Lit les GPIO
(boutons + plunger) et publie chaque appui/relâché sur **MQTT**. Le backend
(`Flipper_back`) consomme ces events, mappe chaque bouton à une action et pilote
les 3 écrans.

> Le firmware reste **agnostique du jeu** : il envoie juste « le bouton `L1` est
> pressé ». Le mapping bouton → action vit côté backend
> (`Flipper_back/docs/BORNE_INPUT_MQTT.md`).

## Stack

- ESP32 (Arduino framework) via **PlatformIO**
- `PubSubClient` (MQTT) + `ArduinoJson`

## Contrat MQTT

Publié sur le broker partagé (le même que celui que le backend écoute) :

| Topic | Payload |
|---|---|
| `pinball/<device>/input/button` | `{"id": "<id>", "state": 0\|1, "ts": <ms>}` |
| `pinball/<device>/input/plunger` | `{"state": 0\|1, "ts": <ms>}` |

`state` : `1` = pressé / chargé, `0` = relâché.

### Mapping boutons → GPIO

| `id` | GPIO | Bouton | Rôle (backend) |
|---|---|---|---|
| `L1` | 4 | white left | flipper gauche |
| `R1` | 13 | black right | navigation droite |
| `L2` | 16 | black left | navigation gauche |
| `R2` | 25 | white right | flipper droit |
| `top` | 17 | vert | valider / start |
| `middle` | 18 | jaune | secondaire |
| `bottom` | 19 | rouge | retour / pause |
| `under_plunger` | 33 | front white | réservé |
| plunger | 32 | plunger | lanceur |

## Configuration

Les constantes matérielles (pins, timings, topics) sont dans `include/Config.h`.
Le reste — **rien en dur, rien committé** — est injecté par **variables
d'environnement** au build (via `build_flags` dans `platformio.ini`), comme le
`.env` du back. À renseigner sur le site Fliphetic (ou exportées dans le shell
avant le build) :

| Variable | Exemple | Rôle |
|---|---|---|
| `WIFI_SSID` | `HETIC-Flipper` | WiFi de la borne |
| `WIFI_PASSWORD` | `…` | mot de passe WiFi |
| `MQTT_BROKER_IP` | `192.168.1.10` | **IP réseau de la borne** |
| `MQTT_BROKER_PORT` | `1883` | port du broker |
| `DEVICE_ID` | `esp32-test` | segment `<device>` des topics |

⚠️ Variable **`MQTT_BROKER_IP`** (et non `MQTT_BROKER_HOST`) : côté backend,
`MQTT_BROKER_HOST` vaut `mqtt` (nom de service Docker). Le firmware, lui, a
besoin de l'**IP réseau de la borne** où mosquitto expose le port 1883.

Le firmware **fail-fast** : si `WIFI_SSID` ou `MQTT_BROKER_HOST` sont vides, il
bloque au boot avec un message série clair plutôt que de tourner à vide.

## Build & flash

Sur la borne du prof, le flash est **automatique au load du projet** — il suffit
que les variables ci-dessus soient configurées. En local :

```bash
pio run                 # compile
pio run -t upload       # flashe l'ESP32
pio device monitor      # logs série (115200) : "[input] button L1 state=1"
```

## Tester sans toucher au hardware

On peut simuler un appui en publiant directement sur le broker (valide la chaîne
backend → écrans sans l'ESP32) :

```bash
mosquitto_pub -h <broker> -t 'pinball/esp32-test/input/button' -m '{"id":"L1","state":1}'
mosquitto_pub -h <broker> -t 'pinball/esp32-test/input/button' -m '{"id":"L1","state":0}'
```
