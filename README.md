# Sobra

A personal finance app built around a single question: **how much is left?**

> *Sobra* is Portuguese for "what's left over".

<img src="logo/sobra-wordmark.png" alt="Sobra" width="240">

**Phase 1, in review.** Documentation and a static mockup exist. There is no
backend, no auth and no database yet. Every number in the mockup is fake and
written by hand.

The interface and the documentation are in Portuguese, because the app is built
for one Brazilian user and the product speaks her language. This README is the
one piece written in English. The [glossary](#glossary) maps every Portuguese
term that shows up on screen.

---

## The problem

The whole thing lived in a Google Sheets file, one block per month, income on one
side and spending on the other. It worked, and it broke in three places.

**1. The main question had no answer.** How much is left this month, after
everything that still has to go out? And how much of October's salary is already
committed before October even starts? The spreadsheet showed what had been typed
in, never what was still coming.

**2. Adding an expense meant sitting at a computer.** The main credit card takes a
long time to post transactions, so every purchase was entered by hand: open the
laptop, open the sheet, find the month, type. An expense made out in the street
waited hours or days, and whatever waits is whatever gets forgotten.

**3. Installments were a hand-written counter.** "5 left", "4 left". Get it wrong
once and the number stays wrong forever, with nothing to signal that it is wrong.
It was where the spreadsheet lied most.

## What Sobra does

**An installment purchase is never stored as a single row.** A R$ 500,00 purchase
in 5x becomes one row in `entries` and five R$ 100,00 rows in `installments`,
spread across five months. "How much of January is already committed" stops being
a human counter and becomes a `SUM` grouped by month. No number needs to be
decremented by hand, so no number can go quietly wrong.

Three screens:

| Screen | Answers |
| --- | --- |
| **este mês** (this month) | How much is confirmed, how much is forecast, how much is left. Three numbers, not one. |
| **próximos meses** (next months) | How much of the next six months is already committed against expected income. |
| **dívidas** (debts) | Balance per person and history, with the balance falling on every payment. |

Beyond that:

- **Forecast and confirmed.** Every future expense is born `previsto` (forecast),
  which is what a yellow-filled cell used to mean in the spreadsheet. When it
  happens it becomes `confirmado` with the real amount.
- **Derived reimbursements.** When someone else uses her card, the entry gets a
  `reimburser_id`. What they owe is always the sum of those entries, never a
  number typed by hand.
- **Logging by Telegram.** A direct answer to problem 2: expenses are logged by
  message, from the phone, on the spot. A bot parses loose text ("mercado 80 no
  cartão azul", "vou gastar 300 de dentista mês que vem"), replies with what it
  understood and asks for confirmation. No IDs ever appear in the conversation.

## Glossary

The words that appear on screen and in the schema, and what they mean.

| Portuguese | English |
| --- | --- |
| **sobra** | what's left after everything |
| **lançamento** | an entry, something that already happened |
| **previsto** | forecast, hasn't happened yet, amount is an estimate |
| **confirmado** | confirmed, already happened, amount is real |
| **parcelado** | an installment purchase, a fixed number of payments |
| **recorrente** | recurring, no defined end (rent, utilities, subscriptions) |
| **avulso** | a one-off purchase |
| **renda prevista** | expected income for the month |
| **reembolso** | reimbursement, someone else's spending on her card |
| **dívida** | money she borrowed from a person, tracked separately |

## Design decisions

The reasoning behind the model lives in
[`docs/DECISIONS.md`](docs/DECISIONS.md), as thirteen ADRs. Each one states the
context, the decision and the **cost**, because a decision with no declared cost
was not really thought through. Some of them:

| ADR | Decision |
| --- | --- |
| [0002](docs/DECISIONS.md) | Separate `entries` from `installments`, so a counter never has to be decremented |
| [0003](docs/DECISIONS.md) | `previsto` and `confirmado` as a status on the same row, not two tables |
| [0005](docs/DECISIONS.md) | Reimbursement balances are derived by summing, never stored |
| [0008](docs/DECISIONS.md) | The home screen is always by payment date, with no accrual toggle |
| [0010](docs/DECISIONS.md) | Bars with a common baseline, never circular area |
| [0013](docs/DECISIONS.md) | Identifiers in English, domain values in Portuguese |

## Stack

| Layer | Choice |
| --- | --- |
| Front end | React, Vite, TypeScript |
| Hosting | Cloudflare Pages, installable as a PWA |
| Data and auth | Supabase, Postgres with Row Level Security, Google sign-in |
| Bot | Cloudflare Worker receiving the Telegram webhook |
| Language parsing | Anthropic API, Haiku |

The front end does not recompute business rules. It reads views and draws them.

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md), how the pieces fit together
- [`docs/SCHEMA.md`](docs/SCHEMA.md), the tables and the real migration SQL
- [`docs/CONVENTIONS.md`](docs/CONVENTIONS.md), the conventions this project holds itself to

## Security and privacy

This repository is public and the app holds personal financial data. Those two
things coexist because **no real data and no secret lives here**.

**Secrets.** What is versioned is [`.env.example`](.env.example), which carries
only the variable names. Values live in `.env.local` (git-ignored), in Cloudflare
Pages environment variables and in Worker secrets (`npx wrangler secret put`). The
Supabase service role key, the Telegram bot token, the webhook secret, the allowed
`chat_id`, the app `user_id` and the Anthropic API key never enter the repository.

**The Supabase anon key is public, and that is by design.** It ships inside the
bundle the browser downloads, so there is no way to hide it and no point trying.
What protects the data is RLS: every table has row level security on with
`user_id = auth.uid()`, so the anon key alone reads nobody's rows. The service
role is what bypasses RLS, which is exactly why it never leaves the Worker.

**Anything prefixed `VITE_` is published.** Vite inlines those variables into the
bundle at build time. Only the Supabase project URL, the anon key and the app URL
go there.

**The data.** Everything under `mockup/` is fiction written by hand, using a fixed
fictional cast. Real data lives in Supabase and is never exported here: no dumps,
no backups, no spreadsheets, no screenshots with real numbers.

## Running it

### The phase 1 mockup

Open `mockup/index.html` in a browser. No server, no install. Type comes from
Google Fonts and falls back to the system geometric sans when offline.

### The app

Does not exist yet. When it does:

```bash
cp .env.example .env.local   # fill in the variables
npm install
npm run dev
```

Supabase runs locally with `npx supabase start`, migrations apply with
`npx supabase db reset`.

## Visual direction

Very light cream background, near-white cards with no borders, generous radii,
icons always inside a thin-stroke circle. Hierarchy through type weight and size,
not through boxes and rules. Every monetary value uses
`font-variant-numeric: tabular-nums`, so digits line up in a column.

Brand colour is mauve `#B0779E`, darkened to `#7A4A68` for small text. The brand
colour is not the data colour: states use neutrals and categories get an
analogous, desaturated palette. No dark mode in v1.

The full conventions are in [`docs/CONVENTIONS.md`](docs/CONVENTIONS.md).
