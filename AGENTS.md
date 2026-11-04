# Consignes Ami2MQTT

- Lire `README.md` et `PLAN.md` avant toute intervention ; consulter `docs/ami-display-analysis.md` pour l’analyse technique et les sources. Le projet est au stade du cadrage : ne pas développer de firmware avant arbitrage et instruction de l'utilisateur.
- Ne jamais modifier [AMI-Display](https://github.com/flrahe/AMI-Display). Référence analysée : commit `676572601d362af6f5e8e322788daf0afe132425`.
- CAN strictement passif : aucune requête, commande, diagnostic, réveil, ACK ou trame d'erreur émise vers le véhicule. Imposer listen-only et inhibition matérielle de l'émetteur ; vérifier le comportement au boot/reset et les errata avant branchement.
- Toute validation CAN active doit rester sur banc isolé du véhicule. Ne pas ajouter de terminaison sur le bus véhicule.
- Privilégier une architecture simple sans écran et réutiliser les décodeurs utiles après validation DLC, format, unités, signe et fraîcheur. Distinguer mesures, estimations et valeurs périmées.
- Protéger la batterie 12 V : alimentation protégée, consommation totale mesurée, coupure basse tension matérielle, temporisation et réveil justifiés. Pas de connexion à la batterie de traction.
- Ne pas assimiler publication MQTT, valeur retained et archivage durable. Prévoir journal borné, reprise après coupure, curseur et doublons ; aucune purge silencieuse de données non acquittées.
- Avant import substantiel, conserver texte MIT et copyrights flrahe/M5Stack, tracer fichiers et commit d'origine, vérifier les licences des dépendances. Ne pas changer arbitrairement la licence du dépôt.
- Ne jamais versionner les secrets Wi-Fi/MQTT. Aucun commit ni push sans accord explicite.
- Pour chaque changement, expliquer ce qui est vérifié, supposé ou restant à décider. Tests futurs ciblés : vecteurs CAN hors véhicule, stockage interrompu, réseau/broker indisponible et reprise ; ne pas annoncer une validation matérielle sans mesure.
