# Tri et réponse automatique des demandes d'une agence immobilière (Youform, Make, Claude)

[English version](README.en.md)

Cas pratique fictif : une agence immobilière reçoit des demandes de contact par un formulaire. Un scénario [Make](https://www.make.com) fait lire chaque message par Claude, qui en extrait les informations clés et attribue un score (chaud, tiède ou froid). La demande est enregistrée dans un Google Sheet, puis une réponse adaptée part automatiquement. Les demandes chaudes déclenchent en plus une alerte pour l'agent.

Le scénario a été construit pas à pas dans Make, avec l'aide de Claude pour le prompt, les formules et le diagnostic des erreurs. Toutes les données de test sont fictives.

## Le problème (fictif)

L'agence reçoit des demandes de plusieurs sources. Chaque agent les lit quand il peut, parfois le lendemain. Un acheteur qui contacte plusieurs agences répond souvent à la première qui lui écrit. Personne ne sait quelles demandes ont été traitées, et les visites sont parfois organisées avec des personnes sans budget ni financement.

## Ce que fait le scénario

1. **Webhook :** Youform envoie chaque réponse au formulaire à Make.
2. **Anthropic Claude, Create a Prompt :** le modèle lit le projet, le bien et le message, et renvoie un JSON avec `budget`, `delai`, `financement`, `score` et `resume`. Le contenu du message est traité comme une donnée, jamais comme une consigne.
3. **JSON, Parse JSON :** découpe la réponse en champs utilisables. La formule extrait ce qui se trouve entre la première accolade ouvrante et la première accolade fermante, ce qui élimine d'éventuelles balises markdown autour du JSON.
4. **Google Sheets, Add a Row :** ajoute la demande (date, contact, message, budget, délai, financement, score, résumé, statut `Nouveau`).
5. **Routeur, selon le score :**
   - **Chaud :** e-mail d'alerte à l'agent (contact, budget, délai, financement, résumé) et réponse au prospect annonçant un rappel pour organiser une visite.
   - **Tiède :** réponse qui demande de préciser le budget, le délai et le financement.
   - **Froid :** réponse polie, sans mobiliser un conseiller.

Les e-mails envoyés aux prospects sont des **modèles fixes**. L'IA ne rédige rien pour le client : elle n'intervient que pour analyser et classer. Elle ne promet ni prix ni disponibilité.

## Règles du score

| Score | Critère |
| --- | --- |
| Chaud | Budget précis ou financement validé, et souhait de visiter ou d'agir rapidement |
| Tiède | Intérêt réel, mais des informations importantes manquent |
| Froid | Simple curiosité, aucun projet précis ou message hors sujet |

Le prompt demande de ne jamais inventer une information absente du message (la valeur est alors `inconnu`).

## Prérequis

- Un formulaire **Youform** avec ces champs, nommés exactement : `Prénom et nom`, `Adresse e-mail`, `Téléphone`, `Votre projet` (Acheter ou Louer), `Bien concerné`, `Votre message`, et une case de consentement. L'intégration **Webhook** de Youform doit être activée.
- Un classeur **Google Sheets** avec un onglet `Demandes` et les colonnes `Date | Nom | E-mail | Téléphone | Projet | Bien | Message | Budget | Délai | Financement | Score | Résumé | Statut | Agent`.
- Un compte **Anthropic** avec une clé API et du crédit. Un modèle Haiku suffit pour cette tâche.
- Dans Make : une connexion **Google Sheets**, une connexion **Gmail** (module récent) et une connexion **Anthropic Claude**.

## Mise en place

1. Dans Make, créez un webhook personnalisé et collez son adresse dans l'intégration Webhook de Youform. Envoyez une réponse de test depuis le lien public du formulaire pour que Make apprenne la structure.
2. Créez la structure de données de **Parse JSON** : dans le module, **Add**, puis **Generator**, avec cet exemple :
   ```json
   {"budget": "200000", "delai": "1 mois", "financement": "validé", "score": "chaud", "resume": "Exemple de résumé"}
   ```
3. Créez un scénario vide, puis **Import Blueprint** avec [`blueprint.json`](blueprint.json).
4. Remplacez les valeurs à compléter : le webhook (`hook`), les connexions (`__IMTCONN__`), la structure de données (`YOUR_DATA_STRUCTURE_ID`), `YOUR_SPREADSHEET_ID`, l'onglet, et `YOUR_AGENT_EMAIL`.
5. Testez avec trois messages (un chaud, un tiède, un froid) en cliquant sur **Run once** avant chaque envoi, puis activez le scénario.

## Ce que j'ai appris en chemin

- **Erreur 410 dans Youform.** Elle signifie que Make n'écoutait pas : l'adresse collée appartenait à un webhook relié à aucun scénario, ou le scénario n'était pas en attente. Il faut copier l'adresse du webhook du scénario, cliquer sur **Run once** puis envoyer le formulaire. Youform désactive le webhook après des échecs.
- **Claude entoure parfois le JSON de balises markdown**, même quand le prompt l'interdit. Le Parse JSON échoue alors avec « Source is not valid JSON ». Plutôt que de remplacer des balises (un remplacement par une chaîne vide est supprimé par l'éditeur de Make), la formule extrait le texte entre `{` et `}`.
- **Taper une formule au clavier français peut casser les accents graves** (touche morte). Il vaut mieux copier-coller les formules et les prompts.
- **Pastilles dans une formule.** Une pastille insérée dans une fonction peut ajouter ses propres accolades et produire « Invalid IML ». À l'intérieur d'une fonction, la référence s'écrit sans accolades (`4.textResponse`).
- **Les modules ne sont enregistrés qu'après un clic sur Sauvegarder.** Un module ajouté mais non sauvegardé n'apparaît pas dans l'historique d'exécution.
- **Après un Run once, les données déjà traitées ne sont plus proposées.** Il faut envoyer une nouvelle réponse du formulaire.
- **Le scénario ne tourne tout seul que s'il est activé** (interrupteur Scheduling). Sinon, les réponses attendent dans la file du webhook.

## Limites

- Le score vient d'un modèle de langage : il peut se tromper sur un message ambigu. C'est une aide au tri, que l'agence doit pouvoir corriger à la main dans le Sheet.
- Les données sont fictives et le scénario n'a été testé que sur quelques messages. Il faudrait l'évaluer sur de vraies demandes avant un usage en agence.
- Le coût en opérations Make et en appels d'API par demande n'a pas été mesuré précisément.
- Les données personnelles des prospects relèvent du RGPD : il faut un consentement, une durée de conservation et un accès restreint au Sheet.
- Les réponses automatiques sont génériques : elles ne tiennent pas compte de la disponibilité réelle des agents ni du bien.
- La connexion Gmail de Make expire après environ six mois et doit être réautorisée.

## Fichiers

- [`blueprint.json`](blueprint.json) : le scénario, prêt à importer, sans identifiant personnel.
