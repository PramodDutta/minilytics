# Personal setup

My own single-user copy of Minilytics, deployed to my own Cloudflare account. The `personal-hardened` branch is:

- upstream `main` at `d665db48ad253b8f037af8459c960650cdc02fc3` (2026-10-25)
- plus open upstream PR [#3](https://github.com/surendrapandar/minilytics/pull/3) (head `fa66dce`), merged early:
  - login lockout counted before the password check, so parallel guesses can't race past it
  - revocable dashboard sessions (`admin_sessions`, migration `0004`) and `POST /api/auth/logout-all`
  - `identify()` is fill-only, so a stored email or name can't be overwritten
  - CSP, `X-Frame-Options` and other security headers on the dashboard and API
- plus `wrangler.jsonc` changes: `workers_dev` written out as `true`, `preview_urls: false`, and a commented
  custom-domain route

## Deploy

Needs Node 20+ and a free Cloudflare account. All commands run from the repo root.

```bash
npm ci
npx wrangler login                 # opens the browser, click "Allow"
npx wrangler d1 create minilytics
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
dashboard, applies the migrations to the live database and deploys. It prints the URL,
`https://minilytics.<subdomain>.workers.dev`. Log in there with `ADMIN_PASSWORD` and add a website.

## Free plan limits to watch

| Limit (Workers Free) | What it means here |
| --- | --- |
| 500 MB per D1 database | About 1.1M pageviews (roughly 470 bytes each). Nothing ever deletes old data. The README's "5 GB" is the account total, not the per-database cap. |
| 100,000 D1 rows written per day | The limit you'll hit first. See the table below. |
| 5M D1 rows read per day | Dashboard use counts too. The README estimates one dashboard load reads about 10 times the sessions in the selected range. |
| 100,000 Worker requests per day | Every tracker event is one request. The dashboard files and `/js/script.js` are static and free. |

Rows written per tracker event, measured by running this repo's `ingest()` against a local D1:

| Event | Rows written |
| --- | --- |
| New visitor, first pageview | 11 |
| Returning visitor, first pageview of a new session | 9 |
| Any later pageview, goal click or outbound click | 5 |
| Each minute a page stays visible | 1 (a ping every 60 s, up to 30 per page, plus one when the tab is hidden or closed) |

So a new visitor who reads one page for 5 minutes costs about 16 rows, and 100k rows a day is roughly 6,000 such
visits. The README's "about 10k pageviews per day" only holds for short visits.

**When a limit is hit:** the D1 limits are per account, and once one is reached **every D1 query fails until
00:00 UTC (05:30 IST)**. Tracking is lost for the rest of the day, the dashboard won't load, and any other D1
database on the same account stops too. Past 100k Worker requests, `/api/*` answers with Cloudflare error 1027
until the same reset.

Check usage with `npx wrangler d1 info minilytics` (database size, rows read and written in the last 24 hours), or
in the Cloudflare dashboard under the D1 database's Metrics tab.

If storage gets close to the cap, remember that deletes are writes too (1 row each, measured), so prune old
`events` and `sessions` in chunks well under 100k rows a day, or do it after moving to Paid.

## When to move to Workers Paid ($5/month)

Upgrade when any of these is true:

- `rows_written_24h` is regularly above about 70k (a few thousand visits a day)
- a launch or a post that could go viral is coming, because one big day can stop tracking until 05:30 IST
- the database is past about 400 MB
- you want more than 7 days of Time Travel history (Paid keeps 30)

Paid is $5/month minimum per account and includes 50M rows written and 25B rows read per month, 10 GB per
database, 5 GB of storage (then $0.75/GB-month) and 10M Worker requests per month. Usage beyond that is billed
instead of queries stopping. Nothing in the code or the deploy changes.

## Optional hardening

Access policies and WAF rules belong to a Cloudflare zone, so they only cover a custom domain, and only once the
`workers.dev` URL is turned off. Do these in order.

1. **Attach a custom domain, then set `"workers_dev": false`.** The domain's DNS must be on Cloudflare in the same
   account. In `wrangler.jsonc`, uncomment the `routes` block with your hostname and run `npm run deploy`. Check
   that the dashboard and `/js/script.js` load on the new host, switch the script tag on every site to it, then set
   `"workers_dev": false` and deploy again. Sites still pointing at `workers.dev` stop tracking at that moment.
   Leave `"preview_urls": false` as it is: version URLs run old code against the live database, and Cloudflare
   documents no expiry for them.
2. **Cloudflare Access** (Zero Trust). Create two self-hosted applications:
   - the whole host, e.g. `stats.example.com`, with an **Allow** policy for your own email
   - a second one with these four paths on the same host and a **Bypass** policy for **Everyone**:
     `/js/script.js`, `/api/event`, `/api/webhooks/*` and `/api/v1/*`

   The more specific path wins, so visitors' browsers, Dodo's webhooks and your backends still get through, while
   the dashboard and its API need an Access login on top of the password. Test from a private window: the dashboard
   should ask for Access, and a visit to one of your sites should still show up as Online.
3. **One WAF rate-limit rule on `/api/event`.** The Free plan allows one rule, which can match on path, counts per
   IP, and uses a 10-second window and a 10-second block. For example: URI path equals `/api/event`, more than 50
   requests in 10 seconds from one IP, block. A real visitor sends a pageview plus about one ping a minute. Don't set
   it much lower: mobile carriers put many people behind one IP. This limits quota burn from a single source. It
   can't stop spoofing, because `/api/event` trusts the page URL inside the request body and the site id is public
   in your page source, so anyone can send fake events for your domain from many IPs.

## Backups

```bash
mkdir -p ~/Backups/minilytics
npx wrangler d1 export minilytics --remote --output ~/Backups/minilytics/minilytics-$(date +%F).sql
```

- Keep exports outside this repo. The fork is public, and an export holds visitor data (identified emails and
  names, payments).
- Cloudflare's docs say "A running export will block other database requests", so events that arrive during an
  export are lost. Run it at a quiet hour.
- For recent mistakes, D1 Time Travel is faster and free: `npx wrangler d1 time-travel info minilytics` shows the
  current bookmark (note it before risky changes), and
  `npx wrangler d1 time-travel restore minilytics --timestamp=<unix-seconds>` rolls back, overwriting the database
  in place. The Free plan keeps 7 days of history.
- To restore from an export, import it into a new, empty database
  (`npx wrangler d1 execute <new-db> --remote --file=<export.sql>`) and point `database_id` at it.

## Installing on a site

1. In the dashboard, **+ Add website** with the bare domain (`example.com`). Subdomains are tracked under it.
   The Worker rejects events from pages on any other hostname.
2. Paste the tag from the site's Settings page into the `<head>` of every page:

   ```html
   <script defer data-website-id="SITE_ID" src="https://minilytics.<subdomain>.workers.dev/js/script.js"></script>
   ```

   If the site sends a Content-Security-Policy, add the Minilytics host to `script-src` and `connect-src`.
3. **Exclude your own visits:** open `https://example.com/?ml_ignore=true` once in each browser you use (undo with
   `?ml_ignore=false`). The flag lives in `localStorage`, which is per origin, so `example.com`, `www.example.com`
   and `app.example.com` each need their own visit.
4. **Verify it, and don't trust a 202:**
   - Open the site in a browser where you haven't set `ml_ignore` (a private window works), ad blocker off.
   - In DevTools, `/js/script.js` should load and a `POST /api/event` should get `202`.
   - Then open the dashboard: within 5 minutes the visit must show under **Online** (visitors seen in the last
     5 minutes), with the page listed under Page.
   - A 202 alone proves nothing: the Worker also answers 202 to events it silently drops as bots (user agents
     containing `bot`, `headless`, `preview`, `monitor`, `python` and others). Other answers: `403` means the page
     hostname doesn't match the site's domain, `404` an unknown site id, `400` a malformed event.
   - The tracker sends nothing on `localhost`, `127.*` or `*.local` pages (unless the tag has
     `data-allow-localhost`), when `navigator.webdriver` is true (Playwright, Selenium), or once `ml_ignore` is set.

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
- If `wrangler.jsonc` conflicts, keep your `database_id`, `workers_dev`, `preview_urls` and `routes`.
- If upstream merges PR #3 later, those commits are already here, so the merge stays clean.
- In this clone, pushing to `upstream` is disabled on purpose (its push URL is set to
  `DISABLED-never-push-upstream`). Push only to `origin`, the fork.

## Good to know

- **One shared password.** Sessions are revocable now. To log out every device, run
  `fetch('/api/auth/logout-all', { method: 'POST' })` in the DevTools console of a logged-in dashboard tab, or
  rotate the secret with `openssl rand -hex 32 | npx wrangler secret put SESSION_SECRET`, which invalidates every
  cookie.
- **Lockout:** 10 tries per IP per 15 minutes, and every try counts, so an IP that keeps trying stays locked. If
  you lock yourself out, wait 15 minutes without retrying.
- **`identify()` is first-wins per field.** A later call only fills fields that are still empty.
- **Revenue:** automatic attribution exists only for Dodo Payments. Other providers have to POST to `/api/v1/payments`
  from a backend. Amounts are summed as-is: the only conversion is when Dodo reports a settlement amount in the
  site's currency, so set each site's currency to the one you're paid in.
- **Favicons:** the dashboard loads favicons from Google's favicon service, so Google sees your site and referrer
  domains whenever the dashboard is open.
- **`npm audit`** reports 9 issues, all in dev tooling (vitest, tailwind, wrangler's local runtime).
  `npm audit --omit=dev` is clean, so the deployed Worker isn't affected.
