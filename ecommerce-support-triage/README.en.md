# Automatic triage of an online shop's customer messages (Youform, Make, Claude)

[Version française](README.md)

**[Watch the video demo (Loom)](https://www.loom.com/share/2d5e015c457c46ebb41b1e0922b9d6ba)** (in French)

Fictional case study: an online shop selling handmade candles receives all its customer messages on a single address. A [Make](https://www.make.com) scenario has Claude read each message, classify it (order tracking, return, product question or dispute), rate its urgency and write a short summary. The request is logged in a Google Sheet, then a suitable reply or alert is sent automatically.

The scenario was built step by step in Make, with Claude's help for the prompt, the formulas and error diagnosis. All test data is fictional.

## The problem (fictional)

"Maison Lumière" has two people and about 40 orders a day. Most messages are repetitive ("where is my order?", "how do I return?"). Real disputes (broken parcel, wrong item, angry customer) get lost in the pile and handled too late. Replies take 24 to 48 hours, and nobody knows which contact reasons come up most.

## What the scenario does

1. **Webhook:** Youform sends each form response to Make.
2. **Anthropic Claude, Create a Prompt:** the model reads the order number and the message and returns JSON with `categorie`, `urgence`, `numero_commande` and `resume`. The message content is treated as data, never as an instruction.
3. **JSON, Parse JSON:** splits the response into usable fields. The formula extracts what lies between the first opening brace and the first closing brace, which removes any markdown fences around the JSON.
4. **Google Sheets, Add a Row:** logs the request (date, contact, order number, message, category, urgency, summary, status `Nouveau`).
5. **Router, by category:**
   - **Tracking:** template e-mail with tracking information.
   - **Return:** template e-mail announcing the return procedure.
   - **Product:** alert to the shop owner, who replies personally (no automatic reply to the customer).
   - **Dispute:** immediate alert to the owner and a simple acknowledgement to the customer.

E-mails sent to customers are **fixed templates**. The AI writes nothing for the customer: it only analyses and classifies. It promises neither refunds nor delivery dates.

## Classification rules

| Category | Criterion |
| --- | --- |
| suivi (tracking) | Customer asks where the order is or reports a delay, in a calm tone |
| retour (return) | Customer wants to send back, exchange or be refunded, without a serious complaint |
| produit (product) | Question about a product (scent, size, burn time, stock) |
| litige (dispute) | Broken parcel, wrong or missing item, anger, threat of a bad review or legal action, delay over 15 days |

When unsure between tracking and dispute: tracking if the tone is calm, dispute if the customer is angry or mentions a refund, a review or legal action. The prompt also forbids inventing information absent from the message (the value is then `inconnu`).

## Prerequisites

- A **Youform** form with these fields, named exactly: `Prénom et nom`, `Adresse e-mail`, `Numéro de commande` (optional), `Votre message`, and a consent checkbox. Youform's **Webhook** integration must be enabled.
- A **Google Sheets** workbook with a tab named `Demandes` and the columns `Date | Nom | E-mail | N° commande | Message | Catégorie | Urgence | Résumé | Statut | Responsable`.
- An **Anthropic** account with an API key and credit. A Haiku model is enough for this task.
- In Make: a **Google Sheets** connection, a **Gmail** connection (recent module) and an **Anthropic Claude** connection.

## Setup

1. In Make, create a custom webhook and paste its address into Youform's Webhook integration. Send a test response from the form's public link so Make learns the structure.
2. Create the **Parse JSON** data structure: in the module, **Add**, then **Generator**, with this example:
   ```json
   {"categorie": "suivi", "urgence": "normale", "numero_commande": "ML-10452", "resume": "Exemple de résumé"}
   ```
3. Create an empty scenario, then **Import Blueprint** with [`blueprint.json`](blueprint.json).
4. Fill in the placeholders: the webhook, the connections, the data structure, the spreadsheet ID and the owner's address.
5. Test with one message per category, clicking **Run once** before each submission, then activate the scenario.

## What I learned along the way

- **A prompt is tuned on real cases.** A message saying "order not received after 8 days" was classified as a dispute because my "when in doubt, dispute" rule was too broad and the categories overlapped. I clarified the tone criterion (calm or angry) and the delay threshold.
- **Fallback rules must be explicit.** Telling the AI what to choose when unsure avoids inconsistent results, but an over-cautious rule produces false alerts.
- **Youform error 410.** It means Make was not listening: the address belongs to another webhook, or Run once was not started. Youform disables the webhook after failures.
- **Claude sometimes wraps the JSON in markdown fences** even when the prompt forbids it: the extraction formula between `{` and `}` fixes this.
- **Module numbers differ from one scenario to another.** The Parse JSON formula must reference the Claude module number of *your* scenario.
- **Copy and paste formulas** rather than typing them: backticks get mangled on French keyboards.
- **The scenario only runs on its own when activated** (Scheduling switch).

## Limitations

- The category comes from a language model and can be wrong on an ambiguous message. The owner must be able to correct it by hand in the Sheet.
- The data is fictional and the scenario was tested on only a few messages. It should be evaluated on real messages before real use.
- The e-mail templates do not check the real status of an order: the scenario is not connected to a shop platform (Shopify or other).
- The cost per message in Make operations and API calls was not measured precisely.
- Customers' personal data falls under GDPR: consent, retention period and restricted access to the Sheet.
- Make's Gmail connection expires after about six months and must be reauthorized.

## Possible improvements

- A weekly dashboard of contact reasons built from the Sheet.
- A draft reply written by Claude for product questions, to be approved before sending.
- A connection to the shop to fetch the real order status.

## Files

- [`blueprint.json`](blueprint.json): the scenario, ready to import, with no personal identifiers.
