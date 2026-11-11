# Tri automatique des messages du service client d'une boutique en ligne (Youform, Make, Claude)

[English version](README.en.md)

**[Voir la démo vidéo (Loom)](https://www.loom.com/share/2d5e015c457c46ebb41b1e0922b9d6ba)**

Cas pratique fictif : une boutique en ligne de bougies artisanales reçoit tous ses messages clients sur une seule adresse. Un scénario [Make](https://www.make.com) fait lire chaque message par Claude, qui le classe (suivi, retour, question produit ou litige), évalue l'urgence et en tire un résumé. La demande est enregistrée dans un Google Sheet, puis une réponse ou une alerte adaptée part automatiquement.

Le scénario a été construit pas à pas dans Make, avec l'aide de Claude pour le prompt, les formules et le diagnostic des erreurs. Toutes les données de test sont fictives.

## Le problème (fictif)

La boutique « Maison Lumière » compte deux personnes et environ 40 commandes par jour. La majorité des messages sont répétitifs (« où est ma commande ? », « comment retourner ? »). Les vrais litiges (colis cassé, mauvais produit, client en colère) se perdent dans la masse et sont traités trop tard. Les réponses prennent 24 à 48 h, et personne ne sait quels motifs de contact reviennent le plus.

## Ce que fait le scénario

1. **Webhook :** Youform envoie chaque réponse au formulaire à Make.
2. **Anthropic Claude, Create a Prompt :** le modèle lit le numéro de commande et le message, et renvoie un JSON avec `categorie`, `urgence`, `numero_commande` et `resume`. Le contenu du message est traité comme une donnée, jamais comme une consigne.
3. **JSON, Parse JSON :** découpe la réponse en champs utilisables. La formule extrait ce qui se trouve entre la première accolade ouvrante et la première accolade fermante, ce qui élimine d'éventuelles balises markdown autour du JSON.
4. **Google Sheets, Add a Row :** ajoute la demande (date, contact, numéro de commande, message, catégorie, urgence, résumé, statut `Nouveau`).
5. **Routeur, selon la catégorie :**
   - **Suivi :** e-mail type avec les informations de suivi.
   - **Retour :** e-mail type annonçant la procédure de retour.
   - **Produit :** alerte au gérant, qui répond lui-même (aucun envoi automatique au client).
   - **Litige :** alerte immédiate au gérant et simple accusé de réception au client.

Les e-mails envoyés aux clients sont des **modèles fixes**. L'IA ne rédige rien pour le client : elle n'intervient que pour analyser et classer. Elle ne promet ni remboursement, ni délai.

## Règles de classement

| Catégorie | Critère |
| --- | --- |
| suivi | Le client demande où en est sa commande ou signale un retard, sur un ton calme |
| retour | Le client veut renvoyer, échanger ou être remboursé, sans reproche grave |
| produit | Question sur un produit (parfum, taille, durée, stock) |
| litige | Colis cassé, mauvais produit, produit manquant, colère, menace d'avis négatif ou de recours, retard de plus de 15 jours |

En cas de doute entre suivi et litige : suivi si le ton est calme, litige si le client est en colère ou parle de remboursement, d'avis ou de recours. Le prompt demande aussi de ne jamais inventer une information absente du message (la valeur est alors `inconnu`).

## Prérequis

- Un formulaire **Youform** avec ces champs, nommés exactement : `Prénom et nom`, `Adresse e-mail`, `Numéro de commande` (facultatif), `Votre message`, et une case de consentement. L'intégration **Webhook** de Youform doit être activée.
- Un classeur **Google Sheets** avec un onglet `Demandes` et les colonnes `Date | Nom | E-mail | N° commande | Message | Catégorie | Urgence | Résumé | Statut | Responsable`.
- Un compte **Anthropic** avec une clé API et du crédit. Un modèle Haiku suffit pour cette tâche.
- Dans Make : une connexion **Google Sheets**, une connexion **Gmail** (module récent) et une connexion **Anthropic Claude**.

## Mise en place

1. Dans Make, créez un webhook personnalisé et collez son adresse dans l'intégration Webhook de Youform. Envoyez une réponse de test depuis le lien public du formulaire pour que Make apprenne la structure.
2. Créez la structure de données de **Parse JSON** : dans le module, **Add**, puis **Generator**, avec cet exemple :
   ```json
   {"categorie": "suivi", "urgence": "normale", "numero_commande": "ML-10452", "resume": "Exemple de résumé"}
   ```
3. Créez un scénario vide, puis **Import Blueprint** avec [`blueprint.json`](blueprint.json).
4. Remplacez les valeurs à compléter : le webhook, les connexions, la structure de données, l'identifiant du classeur et `YOUR_MANAGER_EMAIL` (adresse du gérant).
5. Testez avec un message par catégorie en cliquant sur **Run once** avant chaque envoi, puis activez le scénario.

## Ce que j'ai appris en chemin

- **Un prompt se règle sur des cas réels.** Un message « commande pas reçue depuis 8 jours » était classé litige, car ma règle « en cas de doute, litige » était trop large et les catégories se recouvraient. J'ai précisé le critère de ton (calme ou en colère) et le seuil de retard.
- **Les règles de repli doivent être explicites.** Dire à l'IA quoi choisir en cas de doute évite des résultats incohérents, mais une règle trop prudente génère de fausses alertes.
- **Erreur 410 dans Youform.** Elle signifie que Make n'écoutait pas : adresse d'un autre webhook, ou Run once non lancé. Youform désactive le webhook après des échecs.
- **Claude entoure parfois le JSON de balises markdown**, même quand le prompt l'interdit : la formule d'extraction entre `{` et `}` règle le problème.
- **Les numéros de module changent d'un scénario à l'autre.** La formule du Parse JSON doit référencer le numéro du module Claude de *votre* scénario.
- **Copier-coller les formules** plutôt que les taper : les accents graves se déforment sur un clavier français.
- **Le scénario ne tourne tout seul que s'il est activé** (interrupteur Scheduling).

## Limites

- La catégorie vient d'un modèle de langage : elle peut se tromper sur un message ambigu. Le gérant doit pouvoir la corriger à la main dans le Sheet.
- Les données sont fictives et le scénario n'a été testé que sur quelques messages. Il faudrait l'évaluer sur de vrais messages avant un usage réel.
- Les modèles d'e-mails ne consultent pas l'état réel d'une commande : le scénario n'est pas relié à une boutique (Shopify ou autre).
- Le coût en opérations Make et en appels d'API par message n'a pas été mesuré précisément.
- Les données personnelles des clients relèvent du RGPD : consentement, durée de conservation et accès restreint au Sheet.
- La connexion Gmail de Make expire après environ six mois et doit être réautorisée.

## Pistes d'amélioration

- Un tableau de bord des motifs de contact par semaine à partir du Sheet.
- Un brouillon de réponse écrit par Claude pour les questions produit, à valider avant envoi.
- Une connexion à la boutique pour récupérer le vrai statut de commande.

## Fichiers

- [`blueprint.json`](blueprint.json) : le scénario, prêt à importer, sans identifiant personnel.
