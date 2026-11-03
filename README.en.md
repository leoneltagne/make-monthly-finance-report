# Automated monthly financial report with Make and Claude

[Version française](README.md)

A practical case study: a [Make](https://www.make.com) scenario that reads an invoice tracker in Google Sheets and emails a monthly summary. It was built in conversation with Claude Code, through Make's MCP server.

## What the scenario does

On the 1st of every month at 08:00 (Europe/Paris):

1. **Google Sheets, Search Rows**: reads the `Suivie charges` tab (columns: Client, Amount, Status, Invoice date, Payment date).
2. **Array aggregator**: gathers the rows into a single array.
3. **Email, Send an Email to a Team Member**: sends an email with three totals.
   - Total invoiced: sum of all rows.
   - Total paid: sum of rows whose status is `Payé` (paid).
   - Total pending: sum of rows whose status is `En attente` (pending).

The totals are computed directly in the email body:

```
{{sum(map(12.array; "1"))}}
{{sum(map(12.array; "1"; "2"; "Payé"))}}
{{sum(map(12.array; "1"; "2"; "En attente"))}}
```

## Setup

1. In Make, create an empty scenario, open the `...` menu at the bottom, choose **Import Blueprint** and select [`blueprint.json`](blueprint.json).
2. In the Google Sheets module: pick your Google connection, spreadsheet and sheet (the blueprint holds the placeholder `YOUR_SPREADSHEET_ID`).
3. In the Email module: replace `YOUR_MAKE_ACCOUNT_EMAIL` with the address of a member of your Make team.
4. Set the schedule (monthly, day 1 at 08:00), then test with **Run once**.

## What I learned along the way

- **Gmail and personal @gmail.com accounts.** Make's Gmail module requires "restricted" Google scopes, which Google refuses for personal accounts. To email yourself, the *Send an Email to a Team Member* module needs no connection or password. To email other people, use SMTP with an app password.
- **Iterators and aggregators.** The first version chained iterators and aggregators, and only the first total reached the email. Replacing them with `sum(map(...))` functions fixed it and simplified the scenario.
- **A scenario only runs when it is active.** To test, activate it, run it, then deactivate it if you do not want the schedule yet.

## Limitations

- The statuses `Payé` and `En attente` must be written exactly like that in the sheet (accents and case).
- Amounts must be plain numbers. A format like "64 300 FCFA" may sum to 0.
- The API did not let me read the content of the emails sent during testing, so the amounts were checked by hand in the inbox.
- The sending module can only email members of your Make team.

## Files

- [`blueprint.json`](blueprint.json): the scenario, ready to import, with no personal identifiers.

## Other case studies

- [`coaching-appointment-booking/`](coaching-appointment-booking/README.en.md): coaching appointment booking with Youform, Google Calendar, Google Sheets and Gmail, with free-slot proposals.
