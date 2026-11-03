# Prise de rendez-vous de coaching automatisée (Youform, Make, Google)

[English version](README.en.md)

Cas pratique : un entrepreneur vend des séances de coaching en ligne (YouTube, TikTok, LinkedIn, Facebook). Un client remplit un formulaire Youform avec la date, l'heure, la durée et son objectif. Un scénario [Make](https://www.make.com) vérifie l'agenda, répond au client, tient à jour un Google Sheet et propose d'autres créneaux si celui demandé est pris. Il a été construit en dialoguant avec Claude Code, via le serveur MCP de Make.

## Ce que fait le scénario

1. **Webhook :** Youform envoie chaque réponse au scénario.
2. **Calcul des horaires :** début = date + heure, fin = début + durée (`30 min` devient 30 minutes).
3. **Google Agenda, Get events :** cherche un événement qui chevauche le créneau demandé.
4. **Si le créneau est libre :**
   - crée l'événement dans Google Agenda (objectif, e-mail, téléphone, source en description) ;
   - ajoute une ligne au Sheet, statut `Confirmé`, avec l'ID de l'événement ;
   - envoie un e-mail de confirmation au client.
5. **Si le créneau est pris :**
   - ajoute une ligne au Sheet, statut `À replanifier` ;
   - génère 9 créneaux candidats (3 jours ouvrés à venir, à 10h, 14h et 16h), vérifie chacun dans l'agenda et ne garde que les libres ;
   - envoie ces créneaux au client dans un e-mail, et une alerte à l'entrepreneur.

## Prérequis

- Un formulaire **Youform** avec ces champs, nommés exactement : `Prénom et nom`, `Adresse e-mail`, `Téléphone`, `Date souhaitée`, `Heure souhaitée`, `Durée` (valeurs du type `30 min`), `Objectif de la séance`, `Où nous avez-vous découvert ?`, et une case de consentement. L'intégration **Webhook** de Youform doit être activée.
- Un classeur **Google Sheets** avec un onglet `Suivi de rendez-vous` et les colonnes `Client | E-mail | Téléphone | Date | Heure | Durée (min) | Objectif | Source | Statut | ID événement agenda`.
- Dans Make : une connexion **Google** avec accès Agenda, une connexion **Google Sheets** et une connexion **Gmail**.

## Mise en place

1. Dans Make, créez un webhook personnalisé et collez son adresse dans l'intégration Webhook de Youform. Envoyez une réponse de test pour que Make apprenne la structure.
2. Créez un scénario vide, puis **Import Blueprint** avec [`blueprint.json`](blueprint.json).
3. Remplacez les valeurs à compléter : le webhook (`hook`), les connexions (`account` et `__IMTCONN__`), `YOUR_CALENDAR_ID`, `YOUR_SPREADSHEET_ID` et `YOUR_EMAIL`.
4. Testez avec un créneau libre, puis avec un créneau déjà occupé avant d'activer le scénario.

## Ce que j'ai appris en chemin

- **Gmail et comptes @gmail.com personnels.** Le module Gmail ancien (version 2) est refusé pour les comptes personnels. Le module récent (version 4, connexion `google-email`) fonctionne.
- **Autorisation Agenda.** Les connexions Google créées pour Google Sheets n'avaient pas l'accès Agenda (erreur « Insufficient Permission ») : il faut créer une connexion depuis un module Google Calendar.
- **Champs obligatoires cachés.** Le module « Create an event » exigeait `transparency`, `visibility` et `guestsCanModify`, que les validations de configuration ne signalaient pas. L'erreur n'est apparue qu'à l'exécution.
- **Détecter un créneau libre sans agrégateur.** `Get events` avec « continuer même sans résultat » et un maximum de 1 résultat renvoie soit un événement, soit un bundle vide : un filtre `exist` / `notexist` sur `id` sépare les deux cas.
- **Webhook en mode apprentissage.** Ouvrir l'adresse du webhook dans un navigateur envoie une requête vide qui termine l'apprentissage avec une structure vide. Il faut laisser seul le formulaire envoyer la réponse.

## Limites

- Les créneaux proposés sont libres au moment de l'envoi : un autre client peut en prendre un avant la réponse. La confirmation du créneau choisi reste manuelle.
- Un conflit coûte environ **40 opérations Make** (9 créneaux vérifiés), contre 7 pour un créneau libre.
- Les jours fériés ne sont pas exclus, sauf s'ils sont dans l'agenda vérifié.
- Le calcul de la durée suppose le format `30 min` (nombre suivi d'un espace).
- La case de consentement n'est pas contrôlée par le scénario.
- Je n'ai pas pu lire le contenu des e-mails envoyés depuis l'API de Make : la branche « créneau libre » a été vérifiée à la main dans l'agenda, le Sheet et la boîte e-mail. La nouvelle branche de proposition de créneaux a tourné avec succès (39 opérations), mais le contenu de l'e-mail généré reste à vérifier à la main.
- La connexion Gmail de Make expire après environ six mois et doit être réautorisée.

## Fichiers

- [`blueprint.json`](blueprint.json) : le scénario, prêt à importer, sans identifiant personnel.
