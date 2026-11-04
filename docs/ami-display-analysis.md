# Analyse AMI-Display et architecture proposée

Module ESP32 de télémétrie pour Citroën Ami : réception CAN exclusivement passive, publication vers un Mosquitto local, journalisation sans Wi-Fi et transfert différé au retour à la maison.

État : cadrage documentaire uniquement ; aucun firmware ni matériel validé. Voir [PLAN.md](../PLAN.md) pour les arbitrages et étapes.

## Analyse de la base AMI-Display

Référence examinée : [AMI-Display](https://github.com/flrahe/AMI-Display), commit `676572601d362af6f5e8e322788daf0afe132425`. Analyse du code local, sans compilation ni essai sur véhicule. Sources : [main.cpp](https://github.com/flrahe/AMI-Display/blob/676572601d362af6f5e8e322788daf0afe132425/src/main.cpp), [configuration](https://github.com/flrahe/AMI-Display/blob/676572601d362af6f5e8e322788daf0afe132425/platformio.ini), [DBC](https://github.com/flrahe/AMI-Display/blob/676572601d362af6f5e8e322788daf0afe132425/CAN_information/Ami_Rocks-e.dbc), [SYM](https://github.com/flrahe/AMI-Display/blob/676572601d362af6f5e8e322788daf0afe132425/CAN_information/Ami_Rocks-e.sym), [documentation](https://github.com/flrahe/AMI-Display/blob/676572601d362af6f5e8e322788daf0afe132425/Readme.md).

### Architecture existante

Un seul `src/main.cpp` (~1 676 lignes) rassemble acquisition, décodage, calculs, affichage, réseau, stockage et alimentation. PlatformIO/Arduino, cibles M5Stack Core/Core2/CoreS3 ; M5Unified 0.2.25, M5GFX 0.2.32, ESP32-TWAI-CAN 1.0.1, PubSubClient 2.8, ArduinoJson ^7.3.0 ; RTClib sur le Core. `LOG_SD` est activé pour les trois profils.

`setup()` initialise CAN, SPIFFS, écran, RTC/NTP, Wi-Fi, SD et MQTT. La boucle lit une trame à la fois, décode et journalise, rafraîchit l'écran toutes les 200 ms, traite MQTT et sauvegarde l'état toutes les 5 minutes si modifié. Réseau et SD partagent cette boucle : des blocages peuvent saturer la file RX de 20 trames. Le README source évoque un câblage UART, mais le code utilise le contrôleur CAN TWAI et un transceiver externe.

### Décodage constaté

Débit configuré : 500 kbit/s. Les champs multioctets ci-dessous sont little-endian. Ces définitions issues de rétro-ingénierie restent à valider sur notre Ami.

| ID | Décodage dans le code | Réutilisation / limite |
|---|---|---|
| `0x580` | courant signé octets 0–1 × 0,1 A ; tension octets 2–3 × 0,01 V ; SOC octet 5 | Base principale. Tension castée en signé dans le code, contrairement au DBC non signé : corriger lors de l'extraction. |
| `0x581` | odomètre octets 4–6 × 0,1 km ; vitesse octet 7 en km/h | Directement extractible. |
| `0x713` | autonomie bits 0–6 ; rapport bits 7–8 (`R/N/D/X`) ; ready bit 18 ; frein à main bit 29 | Directement extractible ; DLC annoncé 7. |
| `0x582` | temps restant octets 0–1 | Incomplet : `ok=false`, horodatage non renseigné, unité non spécifiée par le DBC. Exclure du MVP. |
| `0x594` | températures signées octets 0 et 3 | Décodées/affichées mais absentes du JSON MQTT ; nommées `NOK` dans les définitions : sens et validité à confirmer. |
| `0x593` | tension 12 V, octets 0–1 × 0,01 V dans DBC/SYM seulement | Pas de décodeur ni publication existants. Disponibilité à vérifier ; ne suffit pas à protéger l'alimentation hors CAN. |
| `0x714` | frein à main bit 0 dans DBC/SYM seulement | Non exploité par le firmware. |

Puissance calculée `U × I`, avec filtrage ; courant positif interprété comme entrant dans la batterie. Énergies intégrées seulement entre trames espacées de moins de 500 ms, compteurs en Wh et publication en kWh : ce sont des estimations, pas des compteurs véhicule. Charge déduite du courant positif, frein à main et vitesse nulle, avec compteur d'hystérésis ; ce n'est pas une information de câble branché. L'heuristique peut utiliser des valeurs anciennes. Le seuil SOC ≥ 93 % sert de marqueur de « charge complète » interne, sans preuve physique de pleine charge.

Les valeurs instantanées expirent après 2 s ; SOC, odomètre et autonomie sont conservés. L'autonomie peut être estimée par une table apprise de 21 points, initialement 70 km à 100 %. Cette estimation est optionnelle, à distinguer explicitement d'une mesure.

### MQTT existant

Wi-Fi avec reconnexion automatique ; identifiants dans `secrets.h` exclu du dépôt source. MQTT TCP port 1883 via `WiFiClient`, sans TLS, client ID fixe `ami_mqtt`. Reconnexion tentée toutes les ~5 s ; `loop()` MQTT chaque seconde lorsque le Wi-Fi est connecté.

- `ami/state` : JSON toutes les 15 s et lors d'un changement de charge, non retained. SOC, autonomie, odomètre, énergies, courant, puissance, état, rapport, vitesse, ready et diagnostic M5/Wi-Fi ; tension et pics seulement si batterie valide.
- `ami/evcc/{soc,range,odometer,status}` : valeurs retained, statut `C`/`A`. Le filtre `>0` supprime aussi des zéros légitimes.
- À la reconnexion, texte non JSON `reconnected` sur `ami/state` ; message `off` avant extinction volontaire. Aucun Last Will configuré.

Pas de file hors ligne, relecture de journaux, curseur, acquittement métier ni déduplication. Les retours de publication ne garantissent pas une conservation côté maison. PubSubClient ne publie qu'en QoS 0 ([documentation officielle](https://github.com/knolleary/pubsubclient/blob/master/README.md)). La reprise fiable nécessite donc un protocole applicatif ou un autre client MQTT.

### Persistance existante

- SD : traces CAN brutes `.trc`, nom daté par RTC/NTP, temps relatif `micros()/1000` ; tampon RAM de 64 Kio, écrit/flush lorsqu'il est plein. Journalisation indépendante du Wi-Fi.
- SD : `/state.json` contient derniers SOC/odomètre/autonomie, table d'autonomie et compteurs d'énergie ; chargé au démarrage, écrit toutes les 5 minutes si modifié et après ~60 s d'inactivité CAN.
- SPIFFS : monté avec formatage automatique autorisé, mais aucun journal de mesures ou état métier n'y est effectivement stocké ; simple liste de fichiers.
- RTC M5 ou DS3231 et synchronisation NTP horaire présents. Le README source est partiellement dépassé sur la RTC intégrée.

Limites : `closeSD()` ne vide pas le reliquat du tampon CAN ; perte possible de ce reliquat à l'arrêt et du tampon à la coupure. Écriture de `state.json` non atomique, drapeau de modification effacé avant confirmation d'écriture, ouverture du fichier trace non vérifiée, pas de rotation/quota ni contrôle d'espace libre. Construction JSON manuelle dans 1 024 octets sans contrôle de capacité. Pas de validation DLC/format avant décodage ; filtre RTR par égalité de flags à revoir. Les dossiers `test`, `lib`, `include` ne contiennent que les README de squelette.

## Architecture minimale proposée

**Confirmé par le besoin :** ESP32, CAN passif, Mosquitto local, stockage hors Wi-Fi et reprise au domicile. Aucun écran, Bluetooth, diagnostic actif ou commande du véhicule nécessaire au périmètre initial.

**Proposition à arbitrer :** PlatformIO/Arduino sans dépendances M5 ; réutilisation des formules CAN après extraction, correction et tests ; pilote TWAI configuré explicitement en listen-only. Une petite boucle coopérative suffit si réception, écritures et transferts ont des budgets bornés ; surveiller les pertes RX avant d'ajouter des tâches.

```text
CAN → transceiver réception seule → TWAI → décodeur → échantillons → journal durable
                                                        ↓                ↓
                                                  état courant      reprise par curseur
                                                        └──── MQTT/Wi-Fi ───→ Mosquitto
                                                                          ↓
                                                           consommateur durable + ACK
```

Modules prévus : `can_rx`, `decoder`, `telemetry`, `journal`, `network`, `power`, `config`. Journaliser avant publication, même avec Wi-Fi ; conserver validité et âge par signal, sans remplacer une absence par une mesure zéro. Enregistrement versionné : identifiant appareil, session persistante, séquence, temps monotone, UTC facultatif et qualité de l'heure, valeurs et validité. NTP seul ne permet pas de dater absolument les données après un démarrage entièrement hors réseau ; RTC optionnelle.

Proposition simple : journal append-only segmenté de mesures décodées en JSONL, fréquence initiale 1 Hz et événements de charge ; LittleFS interne si capacité/endurance suffisantes, microSD sinon. Dimensionnement à valider : à 200 octets par échantillon, 1 Hz représente ~17,3 Mo/jour de fonctionnement continu, hors surcoûts. Quota, rotation, contrôle d'intégrité, reprise d'une dernière ligne tronquée et curseur persistant à concevoir. Pas de conservation CAN brute permanente dans le MVP.

Topics proposés `ami2mqtt/<device_id>/{state,history,availability,ack}` : état actuel retained, historique non retained avec date d'origine, disponibilité via LWT. Priorité à l'acquisition ; reprise chronologique par lots bornés, reprise après coupure et déduplication par identifiant. Un petit consommateur côté maison stocke durablement puis acquitte ; ne purger que les segments acquittés. Alternative à arbitrer : QoS 1 avec autre client, qui confirme le broker mais pas à lui seul l'archivage applicatif. Mosquitto seul ne constitue pas une base historique.

### Matériel et alimentation

Carte ESP32 avec TWAI et Wi-Fi, sans écran ni LED permanente si possible ; transceiver CAN compatible logique 3,3 V avec émission matériellement inhibée et réception maintenue, y compris au démarrage/reset/hors alimentation. Référence exacte, isolation éventuelle et protections CAN à choisir sur fiches techniques. Listen-only logiciel obligatoire ; la file TX vide du source ne prouve pas la passivité. Vérifier les errata du silicium/pilote retenus et tester l'absence d'ACK et de trames d'erreur. Voir [documentation TWAI Espressif](https://docs.espressif.com/projects/esp-idf/en/v5.4/esp32/api-reference/peripherals/twai.html).

Le source indique OBD 6=CAN-H, 14=CAN-L, 5=masse signal : hypothèse de câblage à confirmer sur le véhicule, sans extrapoler l'alimentation disponible. Dérivation courte, aucune terminaison 120 Ω supplémentaire, aucun réveil ni requête OBD/UDS.

Chaîne 12 V proposée : fusible près du prélèvement → protection inversion de polarité et transitoires/surtensions → convertisseur abaisseur adapté au véhicule → rails ESP32/transceiver/stockage. Prévoir découplage, sous-tension, coupure matérielle avec hystérésis et consommation propre faible. Aucun raccordement à la batterie de traction. Vérifier la masse et éviter les retours d'alimentation par USB ou CAN.

Mesurer la consommation totale active, veille et coupée, convertisseur inclus. L'absence de CAN ne signifie pas que le véhicule est hors tension. Le source reste éveillé sur alimentation externe et mentionne ~10 mA en veille sur un ancien M5 : ne pas reprendre cette politique. Budget à fixer selon batterie et durée de stationnement (`mA × h / 1000 = Ah` ; 10 mA pendant 30 jours = 7,2 Ah).

Une alimentation après contact ne garantit pas le temps de synchronisation au retour ni l'observation de la charge. Une alimentation permanente exige coupure basse tension, temporisation de synchronisation bornée et stratégie de réveil passive consommant peu. Seuils de coupure/reprise à décider après identification de la batterie et mesures ; ne pas dépendre uniquement de `0x593`. Prévoir flush périodique et tolérance aux coupures, sans promettre la conservation de données encore en RAM.

## Licence et périmètre

AMI-Display est sous [MIT, copyright © 2025 flrahe](https://github.com/flrahe/AMI-Display/blob/676572601d362af6f5e8e322788daf0afe132425/LICENSE) ; `main.cpp` porte aussi une attribution MIT © 2025 M5Stack Technology CO LTD. Réutilisation autorisée avec conservation des avis de copyright et du texte MIT pour toute copie ou portion substantielle. Lors de l'import, joindre la licence source, préserver les en-têtes, documenter la provenance et vérifier les licences des dépendances. Aucun code source n'est copié à ce stade ; la licence du nouveau dépôt reste à choisir. AMI-Display demeure inchangé. Aucun commit ni push sans accord explicite.
