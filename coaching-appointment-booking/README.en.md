# Automated coaching appointment booking (Youform, Make, Google)

[Version française](README.md)

A practical case study: a coach sells online coaching sessions (YouTube, TikTok, LinkedIn, Facebook). A client fills in a Youform form with the date, time, duration and goal. A [Make](https://www.make.com) scenario checks the calendar, replies to the client, keeps a Google Sheet up to date and proposes other slots when the requested one is taken. It was built in conversation with Claude Code, through Make's MCP server.

## What the scenario does

1. **Webhook:** Youform sends every response to the scenario.
2. **Time computation:** start = date + time, end = start + duration (`30 min` becomes 30 minutes).
3. **Google Calendar, Get events:** looks for an event overlapping the requested slot.
4. **If the slot is free:**
   - creates the event in Google Calendar (goal, email, phone and source in the description);
   - adds a row to the Sheet with status `Confirmé` (confirmed) and the event ID;
   - sends a confirmation email to the client.
5. **If the slot is taken:**
   - adds a row to the Sheet with status `À replanifier` (to reschedule);
   - generates 9 candidate slots (next 3 working days at 10:00, 14:00 and 16:00), checks each one in the calendar and keeps only the free ones;
   - emails those slots to the client, and an alert to the coach.

## Prerequisites

- A **Youform** form with these fields, named exactly: `Prénom et nom`, `Adresse e-mail`, `Téléphone`, `Date souhaitée`, `Heure souhaitée`, `Durée` (values like `30 min`), `Objectif de la séance`, `Où nous avez-vous découvert ?`, and a consent checkbox. Youform's **Webhook** integration must be enabled.
- A **Google Sheets** workbook with a tab named `Suivi de rendez-vous` and the columns `Client | E-mail | Téléphone | Date | Heure | Durée (min) | Objectif | Source | Statut | ID événement agenda`.
- In Make: a **Google** connection with Calendar access, a **Google Sheets** connection and a **Gmail** connection.

## Setup

1. In Make, create a custom webhook and paste its address into Youform's Webhook integration. Submit a test response so Make learns the structure.
2. Create an empty scenario, then **Import Blueprint** with [`blueprint.json`](blueprint.json).
3. Replace the placeholders: the webhook (`hook`), the connections (`account` and `__IMTCONN__`), `YOUR_CALENDAR_ID`, `YOUR_SPREADSHEET_ID` and `YOUR_EMAIL`.
4. Test with a free slot, then with an already booked slot before activating the scenario.

## What I learned along the way

- **Gmail and personal @gmail.com accounts.** The old Gmail module (version 2) is refused for personal accounts. The recent module (version 4, `google-email` connection) works.
- **Calendar permission.** The Google connections created for Google Sheets had no Calendar access ("Insufficient Permission"): create a connection from a Google Calendar module.
- **Hidden required fields.** The "Create an event" module required `transparency`, `visibility` and `guestsCanModify`, which the configuration validation did not flag. The error only appeared at run time.
- **Detecting a free slot without an aggregator.** `Get events` with "continue even without results" and a maximum of 1 result returns either one event or an empty bundle: an `exist` / `notexist` filter on `id` separates the two cases.
- **Webhook learning mode.** Opening the webhook address in a browser sends an empty request that ends learning with an empty structure. Let only the form send the response.

## Limitations

- The proposed slots are free at the time of sending: another client can take one before the reply. Confirming the chosen slot stays manual.
- A conflict costs about **40 Make operations** (9 slots checked), against 7 for a free slot.
- Public holidays are not excluded unless they are in the checked calendar.
- The duration computation assumes the `30 min` format (a number followed by a space).
- The consent checkbox is not checked by the scenario.
- I could not read the content of the sent emails through Make's API: the "free slot" branch was checked by hand in the calendar, the Sheet and the inbox. The new slot-proposal branch ran successfully (39 operations), but the content of the generated email still needs a manual check.
- Make's Gmail connection expires after about six months and must be re-authorized.

## Files

- [`blueprint.json`](blueprint.json): the scenario, ready to import, with no personal identifiers.
