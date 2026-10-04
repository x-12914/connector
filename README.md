# Connector

See all your Nigerian bank accounts in one place, send money between them, and set up
auto-save rules like *"every time money lands in Zenith, move 10% to Fidelity"*.

- **Mono** links accounts and reads balances and transactions, and debits accounts via direct-debit mandates.
- **Paystack** pays money out to any Nigerian bank account.
- Runs in **sandbox mode** by default: fake banks and fake money, no keys needed.

## Run it

Needs Node.js 22.13 or newer.

```bash
npm install
npm start
```

Open http://127.0.0.1:3000, link a couple of sandbox banks, add a rule, then click
**Simulate deposit** to watch the rule fire.

```bash
npm test
```

## How auto-save works

1. New transactions arrive (Mono webhook, a regular sync, or "Refresh").
2. For each **new incoming credit**, every enabled rule on that account is checked.
3. The rule's share is debited from the source (Mono) and paid to the destination (Paystack).

Safety rules built into the engine (see `src/services/rules.js`, covered by `test/rules.test.js`):

- **Never twice:** each (rule, deposit) pair is claimed in the database before any money moves.
- **No retroactive saving:** a rule ignores deposits from before it was created.
- **No loops:** money the app moved itself is marked "moved by app" and never triggers rules.
- **Minimum amount:** deposits below the rule's minimum are ignored.
- **Safe retries:** only failures where *nothing was debited* can be retried. If money left the source
  but the payout failed, it's marked **needs attention**; check Paystack manually.

## Going live (test keys first)

1. Copy `.env.example` to `.env`, set `USE_MOCK=false`, and add your **test** keys from
   Mono and Paystack.
2. Point the webhooks at your server (needs a public URL, e.g. via a tunnel while developing):
   - Mono → `https://<your-host>/api/webhooks/mono` (set `MONO_WEBHOOK_SECRET` to match)
   - Paystack → `https://<your-host>/api/webhooks/paystack`
3. To pull money from an account, the owner must approve a **Mono direct-debit mandate**. Store the
   mandate ID on the account (`PATCH /api/accounts/:id` with `{"mandate_id": "..."}`).
4. Check the endpoints in `src/providers/mono.js` against Mono's current docs, especially direct debit,
   which changes between API versions.

## Before letting anyone else use it

This is a personal tool right now. It has **no login**, so it only listens on `127.0.0.1`.
Before other people use it you'd need at least: user accounts and authentication, encryption of stored
bank data, a registered business (CAC) for live Mono/Paystack keys, NDPA data-protection compliance,
and legal advice on CBN licensing for moving customers' money.

## Project layout

```
src/
  server.js           start-up
  app.js              Express app, security headers
  routes.js           JSON API used by the web page
  webhooks.js         Mono + Paystack webhooks (signature-checked)
  db.js               SQLite schema (built-in node:sqlite)
  providers/          mock.js (sandbox), mono.js, paystack.js
  services/           sync.js, transfers.js, rules.js
public/               the web interface (plain HTML/CSS/JS)
test/                 rules engine tests
```
# connector
