# Ami2MQTT

Ami2MQTT est un projet de module ESP32 pour suivre la télémétrie d’une Citroën Ami et la retrouver sur un serveur local à la maison.

Le module écoutera le bus CAN du véhicule, enregistrera les mesures pendant les trajets sans Wi-Fi et les publiera vers un broker MQTT Mosquitto au retour à domicile.

## Fonctionnalités prévues

- Acquisition du niveau de batterie, de la tension et du courant de traction, de la vitesse, du kilométrage et des états du véhicule.
- Journalisation locale des mesures hors Wi-Fi.
- Publication de l’état courant et synchronisation de l’historique au retour sur le réseau domestique.
- Reprise des transferts après une interruption.

## Principes

L’interface CAN sera strictement passive, sans aucune émission vers le véhicule. L’alimentation devra protéger la batterie 12 V, avec une consommation maîtrisée et une coupure basse tension.

Le projet privilégie une architecture simple et la réutilisation du décodage d’AMI-Display lorsque cela est pertinent, dans le respect de sa licence MIT.

## État du projet

Le projet est en phase de cadrage. Aucun firmware n’est encore développé et aucun matériel n’est validé. Le choix des composants, du stockage, de l’alimentation et des garanties de synchronisation reste à arbitrer avant l’implémentation.

## Documentation

- [Analyse d’AMI-Display et architecture proposée](docs/ami-display-analysis.md)
- [Plan de développement et décisions à arbitrer](PLAN.md)
- [Consignes de contribution](AGENTS.md)

## Licence

La licence d’Ami2MQTT reste à choisir. Toute reprise de code d’AMI-Display conservera les avis de copyright et le texte de sa licence MIT, comme détaillé dans l’analyse.
