# Personal setup

My own single-user Minilytics, served at **https://insights.thetestingacademy.com** and tracking
thetestingacademy.com first. The `personal-hardened` branch is:

- upstream `main` at `d665db48ad253b8f037af8459c960650cdc02fc3` (2026-10-25)
- plus open upstream PR [#3](https://github.com/surendrapandar/minilytics/pull/3) (head `fa66dce`), merged early:
  - login lockout counted before the password check, so parallel guesses can't race past it
  - revocable dashboard sessions (`admin_sessions`, migration `0004`) and `POST /api/auth/logout-all`
  - `identify()` is fill-only, so a stored email or name can't be overwritten
  - CSP, `X-Frame-Options` and other security headers on the dashboard and API
- plus my changes:
  - `wrangler.jsonc`: custom domain `insights.thetestingacademy.com`, `"workers_dev": false`, `"preview_urls": false`
  - an **Ad** tab (`utm_content`) next to Campaign and Keyword on the dashboard
  - `snippets/flexifunnels-webinar.html`, which records webinar registrations on class.thetestingacademy.com

## Before the first deploy

1. **Log in to the right Cloudflare account.** The Worker has to live in the account that holds the
   thetestingacademy.com zone, or the custom domain can't attach. `npx wrangler whoami` shows the account. If your
   login has more than one, wrangler asks which to use: pick the one with thetestingacademy.com.
2. **Leave the hostname without a DNS record.** Don't create one by hand: the first deploy creates the DNS record
   and the certificate itself, and Cloudflare refuses a custom domain on a hostname that already has a CNAME. On
   2026-10-26, `insights`, `analytics`, `stats` and `metrics` under thetestingacademy.com had no records and the
   zone had no wildcard, so any of them works. To use another name, change `pattern` in `wrangler.jsonc`.
3. **Check the Workers plan.** Open the Workers plans page (Workers & Pages -> Plans,
   `https://dash.cloudflare.com/?to=/:account/workers/plans`). It should say **Workers Paid**. "Paid Cloudflare"
   can also mean a Pro or Business plan on the zone, and that changes none of the Workers or D1 limits: Cloudflare
   says the Workers Paid plan "is separate from any other Cloudflare plan". If it doesn't say Workers Paid, add it
   there ($5/month minimum per account). If you upgrade after deploying, run `npm run deploy` once more so usage is
   tied to the new plan.

## Deploy

```bash
npm ci
npx wrangler login                                 # opens the browser, click "Allow"
npx wrangler whoami                                # must show the account with thetestingacademy.com
npx wrangler d1 create minilytics --location apac  # most visitors are in India
```

If `d1 create` asks "Would you like Wrangler to add it on your behalf?", answer **No**. Paste the printed
`database_id` into `wrangler.jsonc` in place of `00000000-0000-0000-0000-000000000000`, keeping exactly one entry
under `d1_databases`. The id is not a secret, so commit it on `personal-hardened`.

```bash
npx wrangler secret put ADMIN_PASSWORD                        # a strong password; typing is hidden
openssl rand -hex 32 | npx wrangler secret put SESSION_SECRET # piped, so it never shows on screen
npm run deploy
```

If the first `secret put` asks to create a Worker called minilytics, answer **Yes**. `npm run deploy` builds the
dashboard, applies the migrations to the live database, deploys, and creates the DNS record and certificate for
`insights.thetestingacademy.com`. If the first visit shows a certificate error, wait a few minutes. There is no
`workers.dev` URL. Log in at https://insights.thetestingacademy.com with `ADMIN_PASSWORD`.

## Plan limits

Rows written to D1 per tracker event, measured by running this repo's `ingest()` against a local D1. They apply on
both plans:

| Event | Rows written |
| --- | --- |
| New visitor, first pageview | 11 |
| Returning visitor, first pageview of a new session | 9 |
| Any later pageview, goal or outbound click | 5 |
| Each minute a page stays visible | 1 (a ping every 60 s, up to 30 per page, plus one when the tab is hidden or closed) |

A new visitor who reads one page for 5 minutes costs about 16 rows. Storage is roughly 470 bytes per pageview, and
nothing ever deletes old data.

**On Workers Paid** (what this setup expects):

- Included per month: 10M Worker requests, 50M D1 rows written, 25B rows read, 5 GB of storage. Beyond that:
  $0.30 per million requests, $1.00 per million rows written, $0.001 per million rows read, $0.75 per GB-month.
- No daily caps: queries never stop, extra usage is billed. 50M rows written covers about 3M visits a month.
- 10 GB per database, which can't be raised: about 22M pageviews.
- Because `/api/event` is public, a flood of fake events now costs money instead of stopping tracking, so keep the
  WAF rule below.

**On Workers Free** (if you don't add Workers Paid):

- 100,000 D1 rows written per day (about 6,000 of the 5-minute visits above), 5M rows read per day, 500 MB per
  database (about 1.1M pageviews), 100,000 Worker requests per day.
- The D1 limits are per account. Once one is hit, **every D1 query fails until 00:00 UTC (05:30 IST)**: tracking is
  lost for the rest of the day, the dashboard won't load, and any other D1 database on the account stops too. Past
  100k requests, `/api/*` answers with Cloudflare error 1027 until the same reset.
- A busy webinar ad day can use up a day's budget by itself, which is why Workers Paid is the default here.

Check usage with `npx wrangler d1 info minilytics` (size, rows read and written in the last 24 hours) or the D1
database's Metrics tab in the dashboard. Deletes count as writes (1 per row, measured), so prune old `events` and
`sessions` in chunks.

## Phase 1: thetestingacademy.com

### One site entry

In the dashboard, **+ Add website** with domain `thetestingacademy.com`, timezone `Asia/Kolkata` and revenue currency
**INR**. That one entry covers every subdomain (main site, `app.`, `class.`, `courses.`), and a visitor who moves
between them stays one visitor. Pages from all subdomains share one Pages list, because only the path is stored:
`/` on the main site and `/` on `app.` are one row. The Hostname tab shows which subdomain each session started on.
Copy the site id from the site's Settings page; every snippet below needs it.

The tag, for the `<head>` of every page:

```html
<script defer data-website-id="SITE_ID" src="https://insights.thetestingacademy.com/js/script.js"></script>
```

If a page sends a Content-Security-Policy, add `https://insights.thetestingacademy.com` to `script-src` and
`connect-src`.

### Install, per property

1. **thetestingacademy.com** (Dorik): paste the tag into Dorik's custom head code.
2. **app.thetestingacademy.com** (PramodDutta/sdetblueprint): add the tag to the Vite app's `index.html` (it
   follows in-app navigation by itself) and to every standalone HTML page, such as `/masterclass/*` and
   `/playwright/`, which have no analytics today. New pages come from the `publish-masterclass`,
   `publish-class-notes`, `tta-practice-page`, `publish-tuts` and `publish-ai-guide` skills, so their templates need
   the tag too.
3. **class.thetestingacademy.com** (FlexiFunnels webinars): paste
   [`snippets/flexifunnels-webinar.html`](snippets/flexifunnels-webinar.html) into the funnel's tracking/head code so
   it runs on every funnel step, with `SITE_ID` replaced. On the page that loads with `?event=completeregistration`,
   it records the goal `webinar_registered` and identifies the registrant (`email`, else `ff_email`; `name`, else
   `ff_name`). A reload in the same tab doesn't count twice. Without it, registrations are invisible: the query
   string is never stored.
4. **courses.thetestingacademy.com** (New Zenler), optional: paste the tag into New Zenler's custom head code, if
   the plan allows custom code.

**sdet.live** short links need no tag, since they redirect with JavaScript. By default the browser strips the path
from the referrer, so the dashboard shows only "sdet.live", not which link. Add `?ref=<link-name>` to each short
link's destination and the link name shows up as the Referrer. A `utm_source` on the same URL takes precedence, and
a ref that is exactly a platform alias such as `yt`, `ig` or `fb` is counted as that platform.

### Exclude yourself and verify

- **Exclude your own visits:** open `https://thetestingacademy.com/?ml_ignore=true` once in each browser you use
  (undo with `?ml_ignore=false`). The flag lives in `localStorage`, which is per origin, so the main site, `www.`,
  `app.`, `class.` and `courses.` each need their own visit.
- **Verify, and don't trust a 202:**
  - Open a page in a browser where you haven't set `ml_ignore` (a private window works), ad blocker off.
  - In DevTools, `/js/script.js` should load and a `POST /api/event` should get `202`.
  - Within 5 minutes the visit must show under **Online** (visitors seen in the last 5 minutes), with the page under
    Page. For the webinar flow, register once and check that `webinar_registered` appears under Goals.
  - A 202 alone proves nothing: the Worker also answers 202 to events it silently drops as bots (user agents
    containing `bot`, `headless`, `preview`, `monitor`, `python` and others). `403` means the page hostname doesn't
    match the site's domain, `404` an unknown site id, `400` a malformed event.
  - The tracker sends nothing on `localhost`, `127.*` or `*.local` pages (unless the tag has `data-allow-localhost`),
    when `navigator.webdriver` is true (Playwright, Selenium), or once `ml_ignore` is set.

### Dashboard setup

- **The goal:** nothing to create. `webinar_registered` appears in the Goals tab after the first registration.
- **A funnel:** Goals panel -> **Funnel** -> **+ New funnel**. Step 1: Page, the landing path, for example
  `/playwright-automation-mastery-course` (`*` works as a wildcard, as in `/*-course`). Step 2: Goal,
  `webinar_registered`. Save. It counts visitors who did the steps in order within the selected period.
- **Registrations by Source, Campaign and Ad:** in the Goals tab, click `webinar_registered`. The whole dashboard
  then shows only visitors who registered ("Completed goal is webinar_registered"). The sources panel splits them by
  Channel, Referrer, Campaign (`utm_campaign`), Keyword (`utm_term`) and Ad (`utm_content`). Click any row to stack
  another filter, for example one ad.
- **Paid traffic:** a visit lands in the Paid channel only when `utm_medium` is a paid value (`cpc`, `paid`,
  `paid_social`, `ads` and similar). Ad clicks without UTM tags show up as Organic Social (a facebook.com or
  instagram.com referrer) or Direct (in-app browsers often send no referrer). For the Ad tab, `utm_content` needs the
  ad's name, which Meta can fill in with `{{ad.name}}`.

## Phase 2 (later)

- **Pabbly payments:** send each Pabbly Subscriptions payment to
  `POST https://insights.thetestingacademy.com/api/v1/payments` with header `Authorization: Bearer <site API key>`
  (from the site's Settings) and JSON like
  `{ "transaction_id": "...", "amount": 4999, "currency": "INR", "email": "buyer@example.com" }`. Add
  `"refunded": true` for refunds and `"renewal": true` for renewals. Pabbly's own webhook body doesn't match this
  shape, so a relay has to map the fields and add the header, for example a Pabbly Connect workflow or a small
  adapter route in this Worker. Payments are matched to visitors by email, which works because the FlexiFunnels
  snippet identifies registrants, so a purchase is credited to the source that brought the registration. Keep the
  site on one currency, INR: amounts in other currencies are added up as-is.
- **Other domains:** each root domain gets its own site entry and its own site id.

## Optional hardening

The custom domain and `"workers_dev": false` are already in place, so the zone's Access and WAF rules cover the
only URL.

1. **Cloudflare Access** (Zero Trust). Create two self-hosted applications:
   - `insights.thetestingacademy.com`, with an **Allow** policy for your own email
   - a second one with these four paths on the same host and a **Bypass** policy for **Everyone**:
     `/js/script.js`, `/api/event`, `/api/webhooks/*` and `/api/v1/*`

   The more specific path wins, so visitors' browsers, webhooks and the payments relay still get through, while the
   dashboard and its API need an Access login on top of the password. Test from a private window: the dashboard asks
   for Access, and a visit to one of the sites still shows up as Online.
2. **A WAF rate-limit rule on `/api/event`.** It limits quota burn and cost from a single IP. It can't stop spoofing:
   `/api/event` trusts the page URL inside the request body, and the site id is public in the page source. Set it
   loose, because mobile carriers put many people behind one IP. A real visitor sends a pageview plus about one ping
   a minute.
   - **Free zone plan:** one rule, path only, counting per IP over 10 seconds, blocking for 10 seconds. For example,
     URI path equals `/api/event`, more than 50 requests in 10 seconds, block.
   - **Pro zone plan:** two rules, windows up to 1 minute, blocks up to 1 hour, and the expression can also use Host
     and Query. For example, host `insights.thetestingacademy.com` and path `/api/event`, more than 300 requests a
     minute, block for 10 minutes. The second rule can cover `/api/auth/login`.

## Backups

```bash
mkdir -p ~/Backups/minilytics
npx wrangler d1 export minilytics --remote --output ~/Backups/minilytics/minilytics-$(date +%F).sql
```

- Keep exports outside this repo. The fork is public, and an export holds visitor data (registrants' emails and
  names, payments).
- Cloudflare's docs say "A running export will block other database requests", so events that arrive during an
  export are lost. Run it at a quiet hour.
- For recent mistakes, D1 Time Travel is faster and free: `npx wrangler d1 time-travel info minilytics` shows the
  current bookmark (note it before risky changes), and
  `npx wrangler d1 time-travel restore minilytics --timestamp=<unix-seconds>` rolls back, overwriting the database
  in place. Workers Paid keeps 30 days of history, Free 7.
- To restore from an export, import it into a new, empty database
  (`npx wrangler d1 execute <new-db> --remote --file=<export.sql>`) and point `database_id` at it.

## Pulling upstream updates

```bash
git fetch upstream
git log --oneline personal-hardened..upstream/main      # what's new
git diff personal-hardened...upstream/main              # review it, especially src/worker, migrations, public/js
git switch personal-hardened
git merge upstream/main
npm ci && npm run typecheck && npm test && npm run build
npm run db:migrate:local
npm run dev:api                                         # in a second terminal
npm run smoke
npm run deploy
git push origin personal-hardened
```

- Read any new file in `migrations/` before deploying: `npm run deploy` applies it to the live database. Note a
  Time Travel bookmark first.
- If `wrangler.jsonc` conflicts, keep your `database_id`, `routes`, `workers_dev` and `preview_urls`. If
  `src/dashboard/pages/Dashboard.tsx` conflicts, keep the Ad tab entries.
- If upstream merges PR #3 later, those commits are already here, so the merge stays clean.
- In this clone, pushing to `upstream` is disabled on purpose (its push URL is set to
  `DISABLED-never-push-upstream`). Push only to `origin`, the fork.

## Good to know

- **One shared password.** Sessions are revocable. To log out every device, run
  `fetch('/api/auth/logout-all', { method: 'POST' })` in the DevTools console of a logged-in dashboard tab, or
  rotate the secret with `openssl rand -hex 32 | npx wrangler secret put SESSION_SECRET`, which invalidates every
  cookie.
- **Lockout:** 10 tries per IP per 15 minutes, and every try counts, so an IP that keeps trying stays locked. If
  you lock yourself out, wait 15 minutes without retrying.
- **Personal data:** the FlexiFunnels snippet stores registrants' email and name in your D1 database (that's what
  payment matching uses). Phone numbers are never stored. Cover analytics and this matching in the site's privacy
  policy.
- **`identify()` is first-wins per field.** A later call only fills fields that are still empty.
- **Revenue:** automatic attribution exists only for Dodo Payments. Everything else goes through
  `/api/v1/payments`. Amounts are summed as-is, converted only when Dodo reports a settlement amount in the site's
  currency.
- **Favicons:** the dashboard loads favicons from Google's favicon service, so Google sees your site and referrer
  domains whenever the dashboard is open.
- **`npm audit`** reports 9 issues, all in dev tooling (vitest, tailwind, wrangler's local runtime).
  `npm audit --omit=dev` is clean, so the deployed Worker isn't affected.
