# Plan Ami2MQTT

## État initial

Analyse documentaire terminée sur AMI-Display `676572601d362af6f5e8e322788daf0afe132425`. Aucun firmware développé, aucun essai matériel réalisé. Analyse détaillée et sources dans [README.md](README.md).

Réutilisable après extraction : décodage `0x580`, `0x581`, `0x713`, connexion Wi-Fi/MQTT, principes RTC/NTP et expiration des signaux. À reprendre avec validation : charge heuristique, intégration d'énergie, autonomie estimée, températures. À développer : passivité garantie, journal de mesures robuste, transfert différé, acquittements/déduplication, supervision, protection 12 V et gestion énergétique. Les logs `.trc` et `state.json` existants ne constituent pas une file MQTT.

## Arbitrages utilisateur avant implémentation

| Décision | Proposition de départ | Conséquence / information requise |
|---|---|---|
| Matériel | ESP32 sobre sans écran + transceiver à émission inhibée | Matériel déjà disponible ? Références, brochage et errata à valider avant achat/câblage. |
| Alimentation et retour maison | 12 V permanent protégé, fenêtre de synchronisation bornée, coupure matérielle | À comparer à l'après-contact ; connaître point de prélèvement, batterie, stationnement maximal et besoin de suivi de charge. Ne pas figer les seuils sans ces données. |
| Mesures et fréquence | SOC, tension/courant traction, vitesse, odomètre, rapport, ready, autonomie mesurée ; 1 Hz + événements | Durée maximale hors Wi-Fi et perte admissible ? Températures, énergie et autonomie estimée optionnelles. |
| Stockage | LittleFS si volume faible ; microSD si autonomie hors réseau importante | Dimensionner espace, endurance, intervalle de flush, quota et politique si plein. Par défaut préserver les données non acquittées et signaler l'arrêt de journalisation ; accepter une rotation destructive nécessite décision. |
| Garantie de synchronisation | PubSubClient conservé + consommateur maison avec ACK après stockage durable | Autoriser ce petit service côté maison, ou choisir un client QoS 1 et définir la garantie attendue. Les doublons doivent rester possibles et dédupliqués. |
| Horodatage | UTC via NTP + session/séquence et temps monotone ; RTC si datation absolue hors Wi-Fi requise | Accepte-t-on une heure inconnue après démarrage hors réseau ? |
| MQTT local | Namespace par appareil, état retained, historique séparé, LWT, authentification | Adresse/port Mosquitto, ACL, besoin TLS, consommateur historique et éventuelle compatibilité evcc à préciser ; secrets hors Git. |
| Licence | MIT pour faciliter la réutilisation | Choix de licence du nouveau dépôt à confirmer ; obligations MIT du code repris conservées dans tous les cas. |

Les contraintes CAN passif, protection 12 V, source inchangée et absence de commit/push sans accord sont déjà confirmées ; elles ne sont pas soumises à arbitrage.

## Étapes après arbitrage

1. **Spécification courte** : figer nomenclature, schéma alimentation/CAN, budget de consommation, signaux, format de journal, rétention et contrat MQTT/ACK. Critère : limites de pertes et durées de fonctionnement chiffrées.
2. **Banc électrique isolé** : contrôler protections, sous-tension/hystérésis, consommation, transceiver et TWAI listen-only, boot/reset/coupure. Critère : aucune contribution dominante, ACK ou erreur sur le bus de banc ; pas de montage véhicule avant réussite.
3. **Extraction logicielle minimale** : importer uniquement les parties nécessaires avec licences ; décodage indépendant de M5, validation ID/DLC/RTR/format, unités et fraîcheur. Tester avec vecteurs connus, zéros légitimes et trames invalides ; vérifier les divergences DBC/code.
4. **Journal durable** : échantillonnage, sessions/séquences, heure valide/inconnue, segments, intégrité, quota et curseur résistant aux coupures. Critère : redémarrage après écriture tronquée sans perdre les segments antérieurs ; pertes RAM bornées et documentées.
5. **MQTT et synchronisation** : authentification, état/LWT, reprise limitée en débit, ACK durable, déduplication côté maison et purge des seuls segments acquittés. Tester absence Wi-Fi, broker arrêté, coupure pendant transfert, ACK perdu et journal plein. Critère : acquisition maintenue, reprise sans trou dans les données déjà durables.
6. **Validation sur Ami** : après validation du banc, confirmer câblage et trames en conduite/charge/arrêt, comparer les mesures, mesurer stationnement et retour maison. Critère : passivité et budget 12 V vérifiés, limites de données documentées.

Hors MVP : écran, Bluetooth, cloud, OTA, diagnostics actifs, contrôle de charge, analyse avancée de santé batterie et CAN brut permanent. Aucun calendrier engagé avant les choix matériels et énergétiques.
