# Neurosaur Outreach Engine — Deployment Runbook

**Time required:** ~60–90 minutes, plus up to 24h waiting for DNS propagation.
**Prerequisites:** Cloudflare account with `neurosaur.in` already on it, Node.js 18+, terminal access.

Work through the phases in order. Each ends with a verification step — don't move on until it passes.

---

## Phase 0 — Before you start (5 min)

Confirm you have:

- [ ] Cloudflare account with `neurosaur.in` nameservers active
- [ ] Ability to add DNS records in Cloudflare
- [ ] Access to whatever inbox `info@neurosaur.in` forwards to
- [ ] Node.js 18 or newer — check with `node -v`
- [ ] The project folder on your machine

```bash
node -v          # expect v18.x or higher
npm -v
```

If Node is missing, install from nodejs.org (LTS).

---

## Phase 1 — Local setup (10 min)

```bash
cd neurosaur-outreach
npm install
npx wrangler login
```

`wrangler login` opens a browser for authorisation.

> **Headless machine?** Create an API token at
> *Cloudflare dashboard → My Profile → API Tokens → Edit Cloudflare Workers template*,
> then `export CLOUDFLARE_API_TOKEN=your_token` instead of logging in.

**Verify:**
```bash
npx wrangler whoami
```
Should print your account email and ID.

---

## Phase 2 — Database (10 min)

### 2.1 Create it

```bash
npx wrangler d1 create neurosaur_outreach
```

Output ends with a block like:

```toml
[[d1_databases]]
binding = "DB"
database_name = "neurosaur_outreach"
database_id = "a1b2c3d4-5e6f-7890-abcd-ef1234567890"
```

### 2.2 Paste the ID

Open `wrangler.toml` and replace `PASTE_YOUR_D1_DATABASE_ID_HERE` with the real `database_id`.

### 2.3 Create the tables

```bash
npm run db:init
```

The `--remote` flag matters — without it you'd only create a local copy.

**Verify:**
```bash
npx wrangler d1 execute neurosaur_outreach --remote \
  --command "SELECT name FROM sqlite_master WHERE type='table'"
```
Expect: `companies`, `drafts`, `events`.

---

## Phase 3 — Email sending (20 min + DNS wait)

This is the phase where things most often go wrong. Read it fully before starting.

### 3.1 Create a Resend account

Sign up at **resend.com** — free tier, no card required.

### 3.2 Add a sending subdomain

In Resend: **Domains → Add Domain**. Enter:

```
send.neurosaur.in
```

**Use a subdomain, not the root domain.** Two reasons:

1. `neurosaur.in` already has MX records from Cloudflare Email Routing (that's how `info@` works). Adding Resend's MX at the root risks breaking your inbound mail.
2. Cold outreach reputation stays isolated on the subdomain. If deliverability ever suffers, your normal business email from `neurosaur.in` is unaffected.

Your `From` becomes `hello@send.neurosaur.in`, and `Reply-To` stays `info@neurosaur.in` — so replies still land in your normal inbox.

### 3.3 Add the DNS records

Resend shows you 3–4 records. In **Cloudflare → neurosaur.in → DNS → Records**, add each one exactly as shown. Shape will be roughly:

| Type | Name | Value | Notes |
|---|---|---|---|
| MX | `send` | `feedback-smtp.<region>.amazonses.com` | Priority 10 — handles bounces |
| TXT | `send` | `v=spf1 include:amazonses.com ~all` | SPF |
| TXT | `resend._domainkey` | *(long key from Resend)* | DKIM |

Then add DMARC manually — Resend won't give you this one:

| Type | Name | Value |
|---|---|---|
| TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:info@neurosaur.in` |

> `p=none` means monitor-only. Leave it there for the first month, then consider
> tightening to `p=quarantine` once you've confirmed nothing legitimate is failing.

**Do not proxy these records.** TXT and MX can't be proxied anyway, so this is automatic — but if Cloudflare shows an orange cloud on anything, click it to grey.

### 3.4 Verify in Resend

Back in Resend, click **Verify DNS Records**. Usually completes in minutes; allow up to 24 hours.

### 3.5 Create an API key

Resend → **API Keys → Create**. Permission: *Sending access*. Copy it now — it's shown once.

### 3.6 Update config

In `config.js`, set the outbound sender to your subdomain:

```js
outbound: {
  fromEmail: "hello@send.neurosaur.in",
  fromName: "Neurosaur AI Academy",
  replyTo: "info@neurosaur.in",
  ...
}
```

And the review sender:

```js
review: {
  inbox: "info@neurosaur.in",
  fromEmail: "engine@send.neurosaur.in",
  ...
}
```

**Verify:** Resend dashboard shows the domain as **Verified** (green).

---

## Phase 4 — Secrets (5 min)

```bash
# Generate a strong secret first
openssl rand -hex 32
```

Copy that value, then:

```bash
npx wrangler secret put RESEND_API_KEY
# paste your Resend API key

npx wrangler secret put REVIEW_SECRET
# paste the openssl output
```

**Save `REVIEW_SECRET` in your password manager** — you need it for admin endpoints, and it can't be read back from Cloudflare.

**Verify:**
```bash
npx wrangler secret list
```
Expect both names listed.

---

## Phase 5 — Deploy (10 min)

### 5.1 Safety first

Before the first deploy, open `config.js` and set:

```js
safety: {
  sendingEnabled: false,   // ← nothing reaches a prospect yet
  ...
}
```

### 5.2 First deploy

```bash
npm run deploy
```

Output includes your URL:

```
Published neurosaur-outreach
  https://neurosaur-outreach.<your-subdomain>.workers.dev
```

### 5.3 Second deploy — the chicken-and-egg step

The approve/deny links are built from `PUBLIC_BASE_URL`, which you only learn after deploying. So:

1. Copy the URL from above
2. Put it in `wrangler.toml`:
   ```toml
   [vars]
   PUBLIC_BASE_URL = "https://neurosaur-outreach.your-subdomain.workers.dev"
   ```
3. Deploy again:
   ```bash
   npm run deploy
   ```

**Skipping this step is the most common setup failure** — the buttons in your review email will point nowhere.

**Verify:**
```bash
curl https://neurosaur-outreach.YOUR-SUBDOMAIN.workers.dev/health
```
Expect: `{"ok":true,"service":"neurosaur-outreach"}`

---

## Phase 6 — Load target companies (15 min)

### 6.1 Build your list

Edit `seed-companies.json`. Format:

```json
[
  { "name": "Company Name Pvt Ltd",
    "website": "https://www.company.co.in",
    "city": "Chennai",
    "sector": "Precision machining" }
]
```

Only `name` is required, but **`website` is what makes the engine work** — without it there's nothing to enrich or score.

Free sources for Chennai manufacturers:

- **TANSTIA** — Tamil Nadu Small and Tiny Industries Association directory
- **CII Tamil Nadu** member listings
- **SIDCO / SIPCOT** industrial estate directories
- **Udyam registry** — udyamregistration.gov.in
- **ACMA / AMTMA** — auto component and machine tool associations
- Exhibitor lists from IMTEX, ACMA Automechanika

Aim for 100–300 to start. At 5/day that's 4–12 months of runway.

### 6.2 Upload

```bash
curl -X POST https://neurosaur-outreach.YOUR-SUBDOMAIN.workers.dev/admin/seed \
  -H "X-Admin-Token: YOUR_REVIEW_SECRET" \
  -H "Content-Type: application/json" \
  --data @seed-companies.json
```

Expect: `{"inserted": 150, "skipped": 0, "total": 150}`

Duplicate domains are skipped, so re-running is safe when you add more later.

**Verify:**
```bash
npx wrangler d1 execute neurosaur_outreach --remote \
  --command "SELECT COUNT(*) AS total, status FROM companies GROUP BY status"
```

---

## Phase 7 — Test run, sending still OFF (15 min)

```bash
curl -X POST https://neurosaur-outreach.YOUR-SUBDOMAIN.workers.dev/admin/run \
  -H "X-Admin-Token: YOUR_REVIEW_SECRET"
```

Returns something like:

```json
{"runId":"20260908...","enriched":25,"selected":5,"drafted":5,"errors":[]}
```

This takes 30–90 seconds — it's fetching 25 websites.

### Now check three things

**1. The review email arrived** at `info@neurosaur.in` with 5 drafts.

**2. The AI copy is actually good.** Read every draft. Ask yourself:
- Does it say something specific and true about that company?
- Would you be comfortable receiving it?
- Any invented facts or claims?

If the writing is weak, tune `generation.rules` in `config.js` and redeploy. This is worth iterating on before anything goes out.

**3. The buttons work.** Click **Modify** on one draft — the edit form should open. Click **Approve** on another — it should fail with a message about sending being disabled. That failure is the kill switch proving itself.

### Check the scoring

```bash
npx wrangler d1 execute neurosaur_outreach --remote \
  --command "SELECT name, ai_score, status, contact_email FROM companies ORDER BY ai_score DESC LIMIT 20"
```

If good prospects are being marked `ineligible`, tune `targeting` in `config.js`:
- Too many rejected → lower `minimumScore`, or raise `aiMaturityCutoff`
- Too many AI-mature companies passing → lower `aiMaturityCutoff`
- Genuine manufacturers rejected as non-manufacturing → add terms to `sectorKeywords`

Redeploy after any change, then re-run.

---

## Phase 8 — Go live (5 min)

Only once you're happy with the copy quality and the scoring.

```js
// config.js
safety: { sendingEnabled: true, ... }
```

```bash
npm run deploy
```

### First real send

Run the pipeline manually and approve **exactly one** draft. Then confirm:

- Resend dashboard → Emails → shows `Delivered`
- Not bounced, not marked spam
- If you can, check the received copy renders correctly

Wait a day before approving more. Sending a burst from a brand-new subdomain is the fastest way to damage its reputation.

### Warm-up schedule

| Week | Approvals per day |
|---|---|
| 1 | 1–2 |
| 2 | 2–3 |
| 3 | 3–4 |
| 4+ | 5 (full) |

The cron generates 5 drafts daily regardless — you simply approve fewer during warm-up.

---

## Phase 9 — Confirm the schedule (5 min)

The cron is `30 3 * * 1-5` in `wrangler.toml` = **09:00 IST, weekdays**.

**Verify it's registered:** Cloudflare dashboard → Workers & Pages → `neurosaur-outreach` → **Settings → Trigger Events**. You should see the cron listed.

**Watch it fire live:**
```bash
npm run tail
```

Leave that running at 09:00 IST, or just check the next morning for the digest email.

Changing the time — remember cron is **UTC**, IST is UTC+5:30:

| Desired IST | Cron |
|---|---|
| 09:00 weekdays | `30 3 * * 1-5` |
| 10:00 weekdays | `30 4 * * 1-5` |
| 09:00 every day | `30 3 * * *` |
| 09:00 and 16:00 weekdays | `30 3,30 10 * * 1-5` |

---

## Day-2 operations

### Daily
Open the digest email, review each draft, click Approve / Modify / Deny. That's it.

### Weekly

```bash
# How's it going?
npx wrangler d1 execute neurosaur_outreach --remote \
  --command "SELECT status, COUNT(*) FROM drafts GROUP BY status"

# How much runway is left?
npx wrangler d1 execute neurosaur_outreach --remote \
  --command "SELECT COUNT(*) FROM companies WHERE status='enriched'"
```

Check Resend for bounce rate. **Above 5% is a problem** — stop and investigate the address quality.

### Handling an unsubscribe request

Someone replies asking to be removed. Do it the same day:

```bash
npx wrangler d1 execute neurosaur_outreach --remote \
  --command "UPDATE companies SET status='unsubscribed' WHERE contact_email='them@example.com'"
```

### Adding more companies

Append to `seed-companies.json` and re-run the seed curl. Duplicates are skipped.

### Emergency stop

```js
// config.js
safety: { sendingEnabled: false }
```
```bash
npm run deploy
```

Takes about 30 seconds. Drafts keep generating so you can still see what *would* send.

To stop the cron entirely, comment out the `[triggers]` block in `wrangler.toml` and redeploy.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No review email arrives | Resend key wrong, or domain unverified | Check `npm run tail` for the error; confirm domain shows Verified in Resend |
| Buttons link to nowhere | `PUBLIC_BASE_URL` not set | Set it in `wrangler.toml`, redeploy (Phase 5.3) |
| "Link expired or invalid" | Past 72h, or `REVIEW_SECRET` changed | Re-run the pipeline for fresh links; never rotate the secret with drafts pending |
| `selected: 0` | Everything already contacted or ineligible | Seed more companies; check `status` distribution |
| All companies `ineligible` | No published emails, or scoring too strict | Query `score_detail` to see why; tune `targeting` |
| Emails land in spam | SPF/DKIM/DMARC missing or wrong | Re-verify in Resend; send yourself a test and check headers |
| `3036` / neuron limit error | Workers AI daily allocation exhausted | Resets 00:00 UTC. Don't use a 70B model on the free plan |
| Cron doesn't fire | Not deployed, or trigger not registered | Check Settings → Trigger Events; `wrangler dev` does not run crons |
| `enrich_attempts` climbing, no data | Sites blocking the bot, or bad URLs | Check `last_error` column; verify the websites load in a browser |

### Useful queries

```bash
# Why was a company rejected?
npx wrangler d1 execute neurosaur_outreach --remote \
  --command "SELECT name, ai_score, score_detail FROM companies WHERE status='ineligible' LIMIT 5"

# Recent activity
npx wrangler d1 execute neurosaur_outreach --remote \
  --command "SELECT created_at, entity_type, action FROM events ORDER BY id DESC LIMIT 20"

# Anything failed to send?
npx wrangler d1 execute neurosaur_outreach --remote \
  --command "SELECT id, error FROM drafts WHERE status='failed'"
```

---

## Cost check

Everything sits inside permanent free tiers at this volume:

| Service | Free allowance | Your usage |
|---|---|---|
| Workers | 100,000 req/day | ~20/day |
| Workers AI | 10,000 neurons/day | ~3,000/day |
| D1 | 5 GB, 5M reads/day | negligible |
| Cron Triggers | included | 1/day |
| Resend | 3,000/month, 100/day | ~10/day |

**₹0/month.** Watch the Workers AI neuron count if you raise `candidatesPerDay` much beyond 10, or if you switch to a larger model.

---

## Security notes

- `REVIEW_SECRET` protects both the approval links and the admin endpoints. Treat it like a password.
- Approval links are HMAC-signed and bound to one draft and one action — an approve link can't be replayed as a deny.
- `/dashboard` is **unauthenticated**. It exposes company names and draft subjects only. If that matters, put Cloudflare Access in front of the Worker: *Zero Trust → Access → Applications*.
- Rotating `REVIEW_SECRET` invalidates all outstanding approval links. Only do it when no drafts are pending.

---

## Quick reference

```bash
npm run deploy       # deploy changes
npm run tail         # live logs
npm run db:init      # recreate schema (destructive)

# manual run
curl -X POST $URL/admin/run -H "X-Admin-Token: $SECRET"

# add companies
curl -X POST $URL/admin/seed -H "X-Admin-Token: $SECRET" \
  -H "Content-Type: application/json" --data @seed-companies.json

# health
curl $URL/health
```

**Change something → `npm run deploy`.** Config edits do nothing until redeployed.
