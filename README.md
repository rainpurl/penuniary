# Pecuniary

A private, single-user net-worth dashboard. Plaid pulls balances for the banks it
supports; everything else (Fidelity, Public.com, Gainbridge, Alphaeon) you enter
manually. Runs on Cloudflare Pages at `pecuniary.katr.es`; code lives in GitHub.
**No terminal required — everything below is the GitHub website + the Cloudflare dashboard.**

## How it's wired

```
Browser (pecuniary.katr.es)
   |   served by Cloudflare Pages    <- static files in /public
   |   gated by Cloudflare Access    <- only you can load it
   v
Pages Functions (/functions/api/*)   <- backend, holds the Plaid secret
   |
   |-- Plaid REST API   (balances for linked banks)
   `-- Cloudflare KV    (stores access tokens + manual accounts)
```

There is intentionally **no `wrangler.toml`** — all config (KV binding, secrets, build
settings) is done in the Cloudflare dashboard, so you never touch a terminal.

## Files

```
pecuniary/
├── public/index.html               # UI: Plaid Link + dashboard + manual entry
├── functions/api/
│   ├── create_link_token.js         # POST  start Plaid Link
│   ├── exchange_token.js            # POST  public_token -> access_token, store in KV
│   ├── data.js                      # GET   live balances + manual accounts + net worth
│   └── manual.js                    # GET/POST/DELETE  manual accounts
├── README.md
└── HANDOFF.md
```

## Setup (all in the browser)

### 1. Plaid keys (sandbox first)
- Sign up at dashboard.plaid.com (sandbox is free).
- From **Developers -> Keys**, copy your **client_id** and **Sandbox secret**.

### 2. Get the code into GitHub
- Create a repo and upload these files (GitHub: **Add file -> Upload files**), keeping the
  `public/` and `functions/api/` folder structure.

### 3. Create the Cloudflare Pages project
- Cloudflare dashboard -> **Workers & Pages -> Create -> Pages -> Connect to Git** -> pick the repo.
- Build settings: **Framework preset = None**, **Build command = (empty)**,
  **Build output directory = `public`**. Save & deploy.
- The `/functions` folder is detected automatically.

### 4. Create a KV namespace
- Dashboard -> **Storage & Databases -> KV** (a.k.a. Workers KV) -> **Create instance**.
- Name it `pecuniary-kv` -> **Create**. (You never need to copy the ID anywhere.)

### 5. Bind KV to the project
- Your Pages project -> **Settings -> Bindings -> Add -> KV namespace**.
- **Variable name:** `PECUNIARY_KV` -> select the `pecuniary-kv` namespace -> **Save**.

### 6. Add the secrets
Your Pages project -> **Settings -> Variables and Secrets** -> add, for Production:

| Name | Value |
|------|-------|
| `PLAID_CLIENT_ID` | your client_id |
| `PLAID_SECRET` | your **sandbox** secret (type = Secret) |
| `PLAID_ENV` | `sandbox` |

### 7. Redeploy
- **Deployments -> (latest) -> Retry deployment**, so the new binding and secrets take effect.

### 8. Custom domain
- Pages project -> **Custom domains -> Set up a domain** -> `pecuniary.katr.es`.
- Cleanest if `katr.es` is on Cloudflare (add the site, point its nameservers to Cloudflare);
  the DNS record is then created for you, and it's what makes Access work.

### 9. Cloudflare Access — REQUIRED security gate
Without this, `pecuniary.katr.es` is a public URL showing your finances. Lock it to just you:
- **Zero Trust** dashboard -> **Access -> Applications -> Add an application -> Self-hosted**.
- Application domain: `pecuniary.katr.es`.
- Policy: **Action = Allow**, **Include -> Emails ->** your email.
- Save. Loading the site (and every `/api/*` call) now requires you to authenticate. Free.

### 10. Test (sandbox)
- Open `pecuniary.katr.es`, pass the Access login.
- **Connect an account** -> search any bank -> Plaid sandbox login: username `user_good`,
  password `pass_good` (MFA `1234` if asked). Fake balances appear.
- Add a **manual account** to confirm that path.

## Going to production (real accounts)
1. In Plaid, complete your **application + company profile** and request **Production** access.
2. Set `PLAID_ENV = production` and use your Production secret (Settings -> Variables and
   Secrets), then redeploy.
3. **OAuth banks** (Chase, Capital One, Discover — most of yours): in the Plaid dashboard add
   `https://pecuniary.katr.es/` to **Allowed redirect URIs**, then uncomment the `redirect_uri`
   line in `create_link_token.js`. Returning from a bank's OAuth screen also needs Link
   re-initialized with `receivedRedirectUri` in `index.html` (Plaid's "OAuth guide" snippet).
4. **Cost:** `transactions`/`investments` are billed monthly per linked item on paid plans;
   `balance` is per-request. The free **Trial plan covers up to 10 items**, which fits your list.

## Your accounts
- **Plaid (automatic):** Chase, Capital One, Chime, Discover, PayPal.
- **Manual (no aggregator reaches them):** Fidelity Roth; Public.com; Gainbridge (annuity —
  never aggregatable); Alphaeon (Comenity/Bread card).
- **Verify in Plaid Link:** Forbright — if it doesn't appear, add it as a manual account.

## Security model
- Only you can reach the site or API — Cloudflare Access enforces login at the edge.
- Your Plaid secret and access tokens live only in Cloudflare (encrypted env var + KV, encrypted
  at rest). Never in the repo, never in the browser.
- You never type bank credentials into this app — Plaid Link handles bank login; the app holds a
  read-only token, revocable anytime at my.plaid.com.
- HTTPS is automatic.

## Roadmap (next iterations)
- **v0.2.0 — Production:** the OAuth steps above.
- **v0.3.0 — Holdings:** add `investments` + `/investments/holdings/get` for positions/returns.
- **v0.4.0 — History/growth:** add a D1 database (bind it in the dashboard the same way as KV)
  and a daily Cron Trigger that snapshots net worth, so you can chart growth over time.
