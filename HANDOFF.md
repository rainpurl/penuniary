# Pecuniary — Project Handoff

**Purpose of this file:** hand it to a new chat (along with the zip) if the current
conversation fills up. It captures the goal, decisions already made, current state, and
what's next, so a new assistant can continue without re-litigating settled questions.

---

## 1. What this is
A private, single-user net-worth dashboard for the owner. It pulls balances from the
banks Plaid supports and lets the owner manually enter everything Plaid can't reach.
Hosted on **Cloudflare Pages** at **`pecuniary.katr.es`**; source lives in **GitHub**.
The owner works **web-only (no terminal)** — all config is via the Cloudflare dashboard.

## 2. Owner's priorities (in the order that settled the design)
1. Secure / private — data stays on infrastructure the owner controls.
2. Low/no ongoing dollar cost.
3. Low effort to use.
4. Track balances, net worth, and eventually what each holding earns / growth over time.

## 3. Decisions already made (don't re-open these without a reason)
- **Build vs. buy:** The owner chose to **build** this rather than use a hosted app
  (Empower/Monarch/Simplifi), specifically for privacy + control. Empower was the strong
  "free + zero-effort" option but was rejected because it hands account access to a third
  party. Don't keep re-pitching Empower.
- **Hosting:** GitHub Pages can't run a backend (static only, can't hold the Plaid secret).
  So: code in GitHub, **deployed via Cloudflare Pages**, which serves the static frontend
  AND runs the backend (Pages Functions) on the same domain. No separate server, no CORS.
- **No `wrangler.toml`:** removed deliberately. KV binding, secrets, and build output dir
  are all set in the **Cloudflare dashboard** (point-and-click, no terminal). An earlier
  attempt kept a wrangler.toml with a placeholder KV id and it failed the deploy
  ("Invalid KV namespace ID"); dashboard-managed config avoids that entirely.
- **Aggregator:** Plaid (free **Trial plan covers <=10 linked items**, which fits the list).
  SimpleFIN (~$15/yr, MX-backed) was considered as a cheaper-coverage alternative and set
  aside in favor of a pure-Plaid build + manual entry.
- **Storage:** Cloudflare KV for now (tokens + manual accounts). D1 is the planned upgrade
  for historical tracking.
- **Auth:** Cloudflare Access (Zero Trust) gates the whole domain to the owner's email.
  Single-user app; no in-app login system.

## 4. Account inventory and how each connects
- **Plaid (automatic):** Chase, Capital One, Chime, Discover, PayPal.
- **Manual (no aggregator reaches them — confirmed):**
  - **Fidelity Roth** — Plaid dropped Fidelity entirely; MX/SimpleFIN also can't reliably.
    (Empower *can* via Yodlee, but we're not using Empower.)
  - **Public.com** — walled brokerage; not reliably aggregatable.
  - **Gainbridge** — an annuity/insurance product (Gainbridge Life). Never aggregatable by
    anyone, in any tool. Always manual.
  - **Alphaeon (Alphaeon Credit)** — healthcare credit card issued by Comenity/Bread
    Financial; Comenity cards aggregate unreliably. Treat as manual (a liability).
- **Verify in Plaid Link:** Forbright Bank — if it doesn't appear, make it manual.
- Previously dropped from scope by the owner: Venmo, University Federal Credit Union,
  PatientFi, Klarna.

## 5. Current state — v0.1.1 (this zip)
Working scaffold, **sandbox-only** (not yet pointed at real accounts).

Files:
- `public/index.html` — UI: Plaid Link connect, net-worth hero, ledger grouped into
  assets/liabilities, manual-account add/edit. Vanilla JS + CSS, no build step.
- `functions/api/create_link_token.js` — starts Plaid Link.
- `functions/api/exchange_token.js` — swaps public_token -> access_token, stores in KV.
- `functions/api/data.js` — pulls live balances for all linked items + merges manual
  accounts + computes net worth.
- `functions/api/manual.js` — CRUD for manual accounts.
- `README.md` — full web-only deploy steps. `HANDOFF.md` — this file.

Config facts (all in the Cloudflare dashboard — no config file in the repo):
- Domain: `pecuniary.katr.es`.
- Build output directory: `public` (set in Pages build settings).
- KV binding: variable name `PECUNIARY_KV`, bound under Settings -> Bindings to a namespace
  named `pecuniary-kv`. Code reads it at `context.env.PECUNIARY_KV`.
- Secrets (Settings -> Variables and Secrets): `PLAID_CLIENT_ID`, `PLAID_SECRET`, `PLAID_ENV`.
- Plaid REST is called directly via `fetch` (no Plaid SDK — avoids Workers-runtime issues).
  Base URLs: sandbox `https://sandbox.plaid.com`, production `https://production.plaid.com`.

## 6. What the owner still needs to do
- [ ] Create a Plaid account; get sandbox client_id + secret.
- [ ] Upload repo to GitHub; connect Cloudflare Pages (build output dir `public`).
- [ ] Create KV namespace `pecuniary-kv` in the dashboard; bind it as `PECUNIARY_KV`.
- [ ] Set the three secrets; redeploy.
- [ ] Add custom domain `pecuniary.katr.es`.
- [ ] **Set up Cloudflare Access** (without it the URL is public). REQUIRED.
- [ ] Test in sandbox (`user_good` / `pass_good`) before going to production.

## 7. Roadmap (next iterations)
- **v0.2.0 — Production:** flip `PLAID_ENV=production`; register
  `https://pecuniary.katr.es/` as a Plaid allowed redirect URI; uncomment `redirect_uri`
  in `create_link_token.js`; add OAuth-return handling in `index.html`
  (re-init Link with `receivedRedirectUri`) — needed for Chase/Capital One/Discover.
  Watch cost: `transactions`/`investments` are subscription-billed per item on paid plans.
- **v0.3.0 — Holdings:** add `investments` product + `/investments/holdings/get` to show
  positions and returns for any Plaid-linked brokerage.
- **v0.4.0 — History/growth:** add a D1 database (bind via dashboard the same way as KV) and a
  Pages **Cron Trigger** that writes a daily net-worth + per-account snapshot, so net worth
  and growth can be charted over time. (This is the "what's each thing earning over time"
  feature the owner wants.)

## 8. Delivery convention (keep doing this)
- Deliver **all** files as one clean **.zip** each iteration.
- **Rename the zip per iteration** with the version, e.g. `pecuniary-v0.1.1.zip`,
  `pecuniary-v0.2.0.zip`, so the owner can track history.
- Always include an updated copy of **this HANDOFF.md** inside the zip.

## 9. Version log
- **v0.1.0** — first packaged build: Plaid Link + balances + manual accounts + net worth,
  Cloudflare Pages/Functions/KV, behind Cloudflare Access, sandbox-only.
- **v0.1.1** — removed `wrangler.toml` and the CLI dev example; switched all config to the
  Cloudflare dashboard (web-only). Fixes the "Invalid KV namespace ID" deploy error. No
  application-code changes.
