# Rapport financier mensuel automatisé avec Make et Claude

[English version](README.en.md)

Cas pratique : un scénario [Make](https://www.make.com) qui lit un suivi de factures dans Google Sheets et envoie chaque mois un e-mail récapitulatif. Il a été construit en dialoguant avec Claude Code, via le serveur MCP de Make.

## Ce que fait le scénario

Le 1er de chaque mois à 08:00 (Europe/Paris) :

1. **Google Sheets, Search Rows** : lit l'onglet `Suivie charges` (colonnes : Clients, Montant, Statuts, Date de facturation, Date de paiement).
2. **Array aggregator** : regroupe les lignes en un seul tableau.
3. **Email, Send an Email to a Team Member** : envoie un e-mail avec trois totaux.
   - Total facturé : somme de toutes les lignes.
   - Total payé : somme des lignes dont le statut est `Payé`.
   - Total en attente : somme des lignes dont le statut est `En attente`.

Les totaux sont calculés directement dans le corps de l'e-mail :

```
{{sum(map(12.array; "1"))}}
{{sum(map(12.array; "1"; "2"; "Payé"))}}
{{sum(map(12.array; "1"; "2"; "En attente"))}}
```

## Mise en place

1. Dans Make, créez un scénario vide, ouvrez le menu `...` en bas, puis **Import Blueprint** et choisissez [`blueprint.json`](blueprint.json).
2. Dans le module Google Sheets : sélectionnez votre connexion Google, votre classeur et votre onglet (le blueprint contient le placeholder `YOUR_SPREADSHEET_ID`).
3. Dans le module Email : remplacez `YOUR_MAKE_ACCOUNT_EMAIL` par l'adresse d'un membre de votre équipe Make.
4. Réglez la planification (mensuelle, le jour 1 à 08:00) puis testez avec **Run once**.

## Ce que j'ai appris en chemin

- **Gmail et comptes @gmail.com personnels.** Le module Gmail de Make demande des autorisations « restreintes » que Google refuse aux comptes personnels. Pour envoyer à soi-même, le module *Send an Email to a Team Member* n'a besoin d'aucune connexion ni mot de passe. Pour écrire à des tiers, il faut passer par SMTP avec un mot de passe d'application.
- **Itérateurs et agrégateurs.** La première version utilisait des itérateurs et des agrégateurs enchaînés ; seul le premier total arrivait dans l'e-mail. Remplacer ces modules par des fonctions `sum(map(...))` a réglé le problème et simplifié le scénario.
- **Un module ne s'exécute que si le scénario est actif.** Pour tester, il faut l'activer, le lancer, puis le désactiver si on ne veut pas encore la planification.

## Limites

- Les statuts `Payé` et `En attente` doivent être écrits exactement ainsi dans la feuille (accent et casse).
- Les montants doivent être des nombres bruts. Un format comme « 64 300 FCFA » peut donner une somme de 0.
- Je n'ai pas pu lire le contenu des e-mails envoyés pendant les tests depuis l'API : les montants ont été contrôlés à la main dans la boîte de réception.
- Le module d'envoi ne peut écrire qu'aux membres de votre équipe Make.

## Fichiers

- [`blueprint.json`](blueprint.json) : le scénario, prêt à importer, sans identifiant personnel.

## Autres cas pratiques

- [`coaching-appointment-booking/`](coaching-appointment-booking/README.md) : prise de rendez-vous de coaching avec Youform, Google Agenda, Google Sheets et Gmail, avec proposition de créneaux libres.
- [`immobilier-lead-qualification/`](immobilier-lead-qualification/README.md) : tri et réponse automatique des demandes d'une agence immobilière avec Youform, Claude, Google Sheets et Gmail, avec un score chaud, tiède ou froid.
