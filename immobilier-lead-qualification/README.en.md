# Automatic triage and reply for real-estate inquiries (Youform, Make, Claude)

[Version française](README.md)

A fictional case study: a real-estate agency receives contact requests through a form. A [Make](https://www.make.com) scenario has Claude read each message, extract the key facts and assign a score (hot, warm or cold). The request is logged in a Google Sheet, then a matching reply is sent automatically. Hot requests also trigger an alert to the agent.

The scenario was built step by step in Make, with Claude's help on the prompt, the formulas and error diagnosis. All test data is fictional.

## The problem (fictional)

The agency receives inquiries from several sources. Each agent reads them when they can, sometimes the next day. A buyer who contacts several agencies often answers the first one that replies. Nobody knows which requests were handled, and viewings are sometimes booked with people who have no budget or financing.

## What the scenario does

1. **Webhook:** Youform sends each form response to Make.
2. **Anthropic Claude, Create a Prompt:** the model reads the project, the property and the message, and returns JSON with `budget`, `delai` (timeframe), `financement` (financing), `score` and `resume` (summary). The message content is treated as data, never as instructions.
3. **JSON, Parse JSON:** splits the answer into usable fields. The formula extracts the text between the first opening brace and the first closing brace, which removes any markdown fences around the JSON.
4. **Google Sheets, Add a Row:** logs the request (date, contact, message, budget, timeframe, financing, score, summary, status `Nouveau`).
5. **Router, by score:**
   - **Hot:** an alert email to the agent (contact, budget, timeframe, financing, summary) and a reply to the prospect announcing a call to arrange a viewing.
   - **Warm:** a reply asking for budget, timeframe and financing details.
   - **Cold:** a polite reply that does not involve an agent.

The emails sent to prospects are **fixed templates**. The AI writes nothing for the customer: it only analyses and classifies. It promises no price or availability.

## Scoring rules

| Score | Criterion |
| --- | --- |
| Hot | A precise budget or confirmed financing, and a wish to visit or act quickly |
| Warm | Real interest, but important information is missing |
| Cold | Mere curiosity, no specific project, or an off-topic message |

The prompt tells the model never to invent information missing from the message (the value is then `inconnu`, i.e. unknown).

## Prerequisites

- A **Youform** form with these exact field labels (in French): `Prénom et nom`, `Adresse e-mail`, `Téléphone`, `Votre projet` (Acheter or Louer), `Bien concerné`, `Votre message`, and a consent checkbox. Youform's **Webhook** integration must be enabled.
- A **Google Sheets** workbook with a `Demandes` tab and the columns `Date | Nom | E-mail | Téléphone | Projet | Bien | Message | Budget | Délai | Financement | Score | Résumé | Statut | Agent`.
- An **Anthropic** account with an API key and credit. A Haiku model is enough for this task.
- In Make: a **Google Sheets** connection, a **Gmail** connection (the recent module) and an **Anthropic Claude** connection.

## Setup

1. In Make, create a custom webhook and paste its address into Youform's Webhook integration. Send a test response from the form's public link so Make learns the structure.
2. Create the **Parse JSON** data structure: in the module, **Add**, then **Generator**, with this sample:
   ```json
   {"budget": "200000", "delai": "1 mois", "financement": "validé", "score": "chaud", "resume": "Exemple de résumé"}
   ```
3. Create an empty scenario, then **Import Blueprint** with [`blueprint.json`](blueprint.json).
4. Fill in the placeholders: the webhook (`hook`), the connections (`__IMTCONN__`), the data structure (`YOUR_DATA_STRUCTURE_ID`), `YOUR_SPREADSHEET_ID`, the tab name and `YOUR_AGENT_EMAIL`.
5. Test with three messages (hot, warm, cold), clicking **Run once** before each submission, then activate the scenario.

## What I learned along the way

- **HTTP 410 in Youform.** It means Make was not listening: the pasted address belonged to a webhook attached to no scenario, or the scenario was not waiting for data. Copy the address from the scenario's webhook, click **Run once**, then submit the form. Youform disables the webhook after failures.
- **Claude sometimes wraps the JSON in markdown fences**, even when the prompt forbids it, and Parse JSON then fails with "Source is not valid JSON". Instead of replacing the fences (Make's editor drops a replacement by an empty string), the formula extracts the text between `{` and `}`.
- **Typing a formula on a French keyboard can break backticks** (dead key). Copy and paste formulas and prompts instead.
- **Pills inside a formula.** A pill inserted inside a function can add its own braces and cause "Invalid IML". Inside a function, the reference is written without braces (`4.textResponse`).
- **Modules are only saved after clicking Save.** An added but unsaved module does not appear in the execution history.
- **After a Run once, already-processed data is no longer offered.** A new form submission is needed.
- **The scenario only runs on its own when it is switched on** (Scheduling). Otherwise, responses wait in the webhook queue.

## Limits

- The score comes from a language model: it can be wrong on an ambiguous message. It is a triage aid that the agency must be able to correct by hand in the Sheet.
- The data is fictional and the scenario was only tested on a few messages. It would need evaluating on real requests before agency use.
- The cost per request in Make operations and API calls was not measured precisely.
- Prospects' personal data falls under GDPR: consent, a retention period and restricted access to the Sheet are needed.
- Automatic replies are generic: they ignore agent availability and the specific property.
- Make's Gmail connection expires after about six months and must be re-authorised.

## Files

- [`blueprint.json`](blueprint.json): the scenario, ready to import, with no personal identifiers.
