# Siechem Website — Technical Handover

For the engineer who runs this site. It covers what the system is made of, **who owns which account today**, how to operate it, and the work that is still open.

Status as of **26 Oct 2026**. For day-to-day **content editing** see [EDITING-GUIDE.md](EDITING-GUIDE.md); for a short overview see [README.md](README.md).

---

## 1. What this site is

A **static HTML site** with no framework — no React, no build-time templating, no CMS runtime. Every public page is a hand-written `.html` file committed to the repo.

A small set of **serverless functions** (in `api/`) provide the parts that can't be static: the quote-request form, and the admin panel that lets staff edit the product catalogue.

Deployment is on **Vercel**. The build step (`scripts/minify.mjs`) copies the site into `dist/` and minifies it; Vercel serves `dist/`.

```
Browser ──► Vercel CDN ──► dist/*.html          (static pages, catalogue JSON)
                    └────► api/*.js             (serverless: quote form, admin)
                                  └──► Supabase      (admin catalogue database, product images)
                                  └──► GitHub        (Publish writes catalogue JSON back)
                                  └──► SMTP mailbox  (quote notification emails)
```

### Why the catalogue is JSON, not a live database

Public pages never talk to a database. The catalogue lives in committed JSON files (`catalog/data/`), which the browser fetches as plain static files. The admin panel edits a Supabase database, and **Publish** regenerates those JSON files and commits them. This keeps customer-facing pages fast and dependency-free — if Supabase went down, the public site would be unaffected (product images hosted in Supabase Storage would not load; see §4).

The trade-off: catalogue changes are not live until someone clicks **Publish**.

---

## 2. Accounts: who owns what today

| Service | Owner / location now | What it does | If it is lost or down |
|---|---|---|---|
| **GitHub** | Repo `RohitDamodaran/siechem-website` (Siechem's account, **public**). The maintaining developer is a collaborator. | Source code; `catalogues-v1` release holds the 30 catalogue PDFs | Publish stops; PDF downloads break |
| **Vercel** | Project `siechem-website` in the **Siechem** team (**Hobby** plan), deploying from the repo above. Live URL `siechem-website.vercel.app`. | Hosting, build, serverless functions, env vars | Entire site |
| **Supabase** | Project `fihlhaqxpiobwfotxsxs` in **Siechem's Org** (**Free** plan), region South Asia (Mumbai). | Admin catalogue database + product image storage | Admin panel; images for families that use uploaded images |
| **Mail** | Mailboxes on the cPanel behind `mail.siechem.com` (not the web host's cPanel) | Sends and receives quote emails | Quote form emails |
| **Domain `siechem.in`** | Registered at **Square Brothers**; DNS hosted there | Intended production domain | Site unreachable on that name |
| **Domain `siechem.com`** | DNS in Cloudflare; web record still points at the previous cPanel web host | Still serves the **old** website | — |

**Legacy project (not for production use).** The original Vercel project (`siechem.vercel.app`) still exists in the previous maintainer's personal account. Its Git integration broke when the repo moved, and its Publish no longer works (its token has no access to the transferred repo). It is kept only as a rollback until the Siechem project has run stably; delete it afterwards (§8).

### Plan limits that matter

- **Vercel Hobby** is for personal, non-commercial use under Vercel's terms, and **team invitations are Pro-only**. Consequence: nobody outside the Siechem Vercel account can be added to the team, so env vars, logs and deploy hooks can only be managed by Siechem. Code changes still flow through GitHub: a push to `main` deploys automatically.
- **Supabase Free** pauses a project after about **7 days of inactivity** and has **no automatic backups**. This has already caused an outage (§4).

---

## 3. Environment variables

Set in **Vercel → Project → Settings → Environment Variables**. They are not in the repo and `.env` files are gitignored. A change only reaches the functions after a **redeploy**.

| Variable | Purpose | Where to get it |
|---|---|---|
| `SUPABASE_URL` | Supabase project API URL | Supabase → Project Settings → API |
| `SUPABASE_SERVICE_ROLE_KEY` | Server-side DB key; **bypasses row-level security** | Supabase → Project Settings → API → Legacy keys |
| `GITHUB_TOKEN` | Lets Publish commit catalogue JSON | GitHub → Settings → Developer settings → Fine-grained token |
| `VERCEL_DEPLOY_HOOK_URL` | Triggers a rebuild after Publish commits | Vercel → Project → Settings → Git → Deploy Hooks |
| `SMTP_HOST` | Outgoing mail server | `mail.siechem.com` |
| `SMTP_PORT` | `465` (SSL, default) or `587` (STARTTLS) | Mailbox's "Connect Devices" page in cPanel |
| `SMTP_USER` | Sending mailbox address | The mailbox the form sends from |
| `SMTP_PASS` | That mailbox's password | Whoever administers the mail cPanel |
| `SALES_EMAIL` | *Optional.* Where enquiries are delivered; defaults to `SMTP_USER` | — |
| `ADMIN_PASSWORD` | Admin panel login | Choose one; keep it in the company password manager |
| `ADMIN_SESSION_SECRET` | Signs admin session tokens | Any long random string (`openssl rand -hex 32`) |

`GMAIL_USER` / `GMAIL_PASS` are a legacy fallback used only when `SMTP_HOST` is unset. They can be deleted.

> **`SUPABASE_SERVICE_ROLE_KEY` bypasses all database security.** It is server-only. Never put it in client-side code and never commit it.

> **Set `SMTP_HOST`, `SMTP_USER` and `SMTP_PASS` together.** If `SMTP_HOST` is set without the other two, the code tries the mail server with the wrong credentials and quote emails stop.

### The deploy hook belongs to one Vercel project

A deploy hook is tied to the project that created it. If the Vercel project is ever recreated, create a new hook and update `VERCEL_DEPLOY_HOOK_URL`, or Publish will commit but never go live.

### The GitHub token must outlive its creator

Fine-grained tokens are scoped to specific repositories and **default to a 30-day expiry**. When a token expires, Publish fails with `GitHub API 401: Bad credentials` (or `403: Resource not accessible by personal access token` if it lacks access to the repo) while the rest of the site keeps working, so it looks like an admin bug.

The token must be created under the **GitHub account that owns the repo**, with:
- Repository access: **only** `siechem-website`
- Permissions: **Contents → Read and write**
- Expiration: **No expiration** (or a calendar reminder well before it lapses)

---

## 4. How the admin panel works

**URL:** `/admin` · **Login:** the value of `ADMIN_PASSWORD`

Sessions are stateless — a signed token valid for 12 hours, with no session table. Changing `ADMIN_SESSION_SECRET` immediately logs everyone out.

### The publish pipeline

Editing in the admin panel writes to Supabase only. Nothing is public until Publish.

```
1. Staff edits a family/series      ──► Supabase (admin_families, admin_series)
2. Change is marked as draft        ──► admin_audit_log records the action
3. Staff clicks PUBLISH
4. api/admin-publish.js reads the whole catalogue from Supabase
5. Regenerates families.json + products/<id>.json
6. Commits all files as ONE commit via the GitHub Git Data API
7. Calls VERCEL_DEPLOY_HOOK_URL
8. Vercel rebuilds and the changes go live (~1–2 minutes)
```

Deletions are tracked too — `admin_audit_log` is queried for `delete_series` and `remove_image` entries since the last publish, so a family whose only pending change is a deletion still shows as having unpublished changes.

### Database tables

| Table | Contents |
|---|---|
| `admin_families` | Product families (name, segment, summary, specs) |
| `admin_series` | Product series/variants within each family |
| `admin_audit_log` | Every admin action — the source of truth for "what changed since last publish" |
| `admin_publish_log` | Timestamp of each publish |

Product images go to the Supabase Storage bucket **`product-images`**. Only the families whose images were uploaded through the admin panel use it (`aero-44`, `aero-55`, `automotive`).

### Troubleshooting: "Failed to load" / missing images (Supabase paused)

**Symptoms:** the admin page shows `Failed to load: TypeError: fetch failed` or a wall of HTML mentioning *Cloudflare* / *Error code 521*; the `admin-families` request in the browser's Network tab returns **500** after several seconds; product images for the three families above disappear on the public site.

**Cause:** the Free-tier Supabase project paused after a week without activity. Public pages don't query the database, so visitor traffic does not keep it awake.

**Fix:** Supabase dashboard → the project → **Restore**. Restoration can take from a few minutes to several hours; the status must read *Healthy* before the admin panel works again. This happened on 1 Oct 2026.

**Prevention:** upgrade the project to **Pro** (no auto-pause, daily backups). As a stopgap, have someone open the admin panel at least once a week.

---

## 5. How quote-request emails work

A customer submits the quote form → `api/quote.js` sends **two** emails over SMTP:

1. **To the company** — the enquiry details, sent to `SALES_EMAIL` (or `SMTP_USER`)
2. **To the customer** — an acknowledgement

### Which mail server it uses

Siechem's email runs on its own cPanel mail host, not Gmail. `SMTP_HOST` is `mail.siechem.com` on port `465`, authenticating as a Siechem mailbox. Mailboxes are created in the cPanel of the **mail host** (the one that lists the real mailboxes), not in the web host's cPanel, which only has a couple of system accounts.

- **A cPanel mailbox uses its ordinary password.** There is no App Password or 2-Step Verification; those are Gmail-only.
- **Port `465`** is implicit TLS; **`587`** upgrades via STARTTLS. If sending fails after a change, check the port first.
- **If the sending mailbox's password is ever changed, the quote form stops sending.** Update `SMTP_PASS` in Vercel and redeploy. Using a dedicated mailbox just for the website (for example `website@siechem.com`, with `SALES_EMAIL` pointing at the sales inbox) avoids tying the form to a mailbox people log into every day.

### Submissions are not stored anywhere

`api/quote.js` only sends the two emails above — it does not write to Supabase, a file, or any other store. The sales inbox and the customer's own inbox are the only two places a submission exists. If a notification email is lost (deleted, misfiled, bounced), there is no way to recover it from the site.

This is a deliberate scope decision: a `quote_submissions` table in Supabase would be a natural addition if Siechem later wants a searchable record.

---

## 6. Deploying

### Normal code change

```bash
node scripts/minify.mjs        # optional: confirm the build succeeds locally
git add <files>
git commit -m "your message"
git push origin main           # Vercel builds and deploys production automatically
```

The Siechem Vercel project is connected to the repo, so a push to `main` is all that is needed. Check the **Deployments** tab if a change doesn't appear. Note that **admin Publish also commits to `main`**, so `git pull` before you push.

### After adding or removing a page

```bash
node scripts/gen-sitemap.mjs
```

Commit the regenerated `sitemap.xml` with the change.

### Publishing catalogue changes

Use the admin panel's Publish button. Don't hand-edit `catalog/data/*.json` — the next Publish regenerates those files from Supabase and overwrites manual edits.

---

## 7. Hardcoded values

These are **not** environment variables; changing them needs a code or data edit.

### a) The repo path in the publish code — correct today

`api/admin-publish.js` line 13:

```js
const REPO = 'RohitDamodaran/siechem-website';
```

If the repo is renamed or moved again, update this line or Publish will fail. Converting it to an env var would remove the need for a code change.

### b) 30 catalogue PDFs on GitHub Releases — links still use the old owner

Every download link in `catalog/data/families.json` still reads:

```
https://github.com/nareshh3105/siechem-website/releases/download/catalogues-v1/aero-44.pdf
```

The PDFs are Release assets (too large to commit — `catalogue-pdfs/` is gitignored) and now live in the transferred repo. The old-owner links **currently work only because GitHub redirects them** (verified 26 Oct 2026). That redirect disappears if the old username ever creates a repository with the same name. Rewrite the 30 URLs to `RohitDamodaran/siechem-website`, ideally through the admin panel's family fields so the next Publish doesn't overwrite them from the database.

### c) Product image URLs point at one Supabase project

Image URLs are baked into the committed JSON as absolute paths:

```
https://fihlhaqxpiobwfotxsxs.supabase.co/storage/v1/object/public/product-images/...
```

The project was **transferred**, so its reference is unchanged and the URLs still work. **Never replace it with a new Supabase project:** a new project has a new subdomain, which breaks every product image.

### d) The domain is written into several files — still `siechem.vercel.app`

| Where | What |
|---|---|
| `<link rel="canonical">` and `og:url` in 13 HTML files | `https://siechem.vercel.app/…` |
| `robots.txt` | `Sitemap:` line |
| `scripts/gen-sitemap.mjs` | `ORIGIN` constant (and the generated `sitemap.xml`) |
| `api/quote.js` | "Submitted via…" line and a footer link in the customer email |

Change all of them together to the production hostname once it is chosen (§8), then run `node scripts/gen-sitemap.mjs`. Until they change, Google is told the `.vercel.app` address is the canonical one.

---

## 8. Status and remaining work

### Done

- Supabase project transferred to Siechem's organisation (project reference unchanged, images verified)
- GitHub repo transferred to Siechem's account; `REPO` in `api/admin-publish.js` updated
- New Vercel project created by Siechem from the repo; all environment variables entered; new deploy hook created
- Publish verified end to end (Supabase → GitHub commit → Vercel rebuild)
- Quote emails verified, sending through the Siechem mailbox over SMTP
- `siechem.in` DNS configured at Square Brothers and attached to the Vercel project

### Remaining, in rough priority order

1. **Upgrade Supabase to Pro.** The Free tier already caused an outage (§4) and has no backups.
2. **Upgrade Vercel to Pro**, or accept the limits in §2: Hobby is non-commercial, and the maintainer cannot be added to the team.
3. **Finish the `siechem.in` go-live.**
   - In Vercel → Domains, confirm both `siechem.in` and `www.siechem.in` show *Valid Configuration* and have a certificate. By default the apex redirects to `www`, so `www.siechem.in` is the primary host.
   - Update the domain references in §7d, regenerate the sitemap, deploy.
   - Verify the domain in Google Search Console and submit `/sitemap.xml`.
   - Collect high-traffic URLs from the old `siechem.com` site and add redirects for any that would 404 (`vercel.json` already redirects a set of legacy paths).
   - Decide what `siechem.com` should do long term (redirect to `siechem.in`, or stay separate). **Keep the old web host running for a few weeks** as a rollback.
4. **Repoint the 30 catalogue PDF links** to the new owner's path (§7b).
5. **Decommission the legacy Vercel project** after a stable period: delete it, delete its deploy hook, revoke the previous maintainer's personal GitHub token, and remove their copies of any credentials. Rotate any credential that was shared outside a password manager during the migration.
6. **Give the form its own mailbox** (§5).
7. **Supply four partner logos** that are still missing from `assets/partners/`: Schneider Electric, Mitsubishi Electric, DRDO and Indian Railway. Only the files are needed; they have not been provided.

### DNS notes for `siechem.in` (Square Brothers)

Square Brothers has **two sets of nameservers**: the default "coming soon" parking set, and its DNS-hosting set (`mdns1`–`mdns4.squarebrothers.net`). Records entered in the DNS Management screen **only take effect when the domain's nameservers are set to the `mdns` set** (Domain → *Modify or Manage Nameservers*). While parked, the domain resolved to a parking page no matter what was entered there.

Records Vercel asked for: an `A` record on `@` and a `CNAME` on `www` pointing at the target shown under Vercel → Domains → *View DNS configuration*. Do not delete the four `NS` rows in the DNS Management table. Nameserver changes can take hours to reach every resolver, and during that window some networks see the new site while others don't.

---

## 9. Repository layout

```
siechem-redesign.html      Homepage (served at / via a rewrite in vercel.json)
company/                   About, careers, certifications, policies, contact…
products/                  Product pages and spec configurators
resources/                 Resources pages (5S, REACH, RoHS, awards…)
tech-support/              Engineering calculators and reference tables
catalogues/                Catalogue flipbook viewer
admin/                     Admin panel pages (excluded from search indexing)

api/                       Serverless functions
  _lib/adminAuth.js          Signed admin session tokens
  _lib/supabaseAdmin.js      Server-only Supabase client
  admin-publish.js           The publish pipeline (see §4)
  quote.js                   Quote form emails (see §5)
  catalogue-pdf.js           Proxies catalogue PDFs so they render inline

catalog/data/              Generated catalogue JSON — do not hand-edit
scripts/minify.mjs         Build: copies site to dist/ and minifies
scripts/gen-sitemap.mjs    Regenerates sitemap.xml
theme.css                  Sitewide styles
vercel.json                Routing, redirects, cache headers
```

### Query-param templates

`products/product-family.html`, `products/product-variant.html`, and `catalogues/catalogue-view.html` render nothing without a query string — they are templates driven by `?id=` / `?f=`. They are deliberately excluded from `sitemap.xml`.

---

## 10. Known issues and deferred work

- **The GitHub repo is public.** Source, history and this document are world-readable. No secrets are committed, but consider making it private (GitHub → Settings → Change visibility); afterwards confirm Vercel's GitHub integration still has access.
- **`REPO` should be an environment variable.** Hardcoding it means every future ownership change needs a code edit (§7a).
- **Catalogue PDFs depend on GitHub Releases.** Workable, but a dedicated file host or Supabase Storage would be less fragile (§7b).
- **The hero video has two encodes.** Screens up to 768 px wide and 2G/3G connections load `video/cable-video-hero-mobile.mp4` (640×360, ~5 MB); larger screens load `cable-video-hero.mp4`. The page plays it at 2.3× speed, so the network must deliver 2.3× the file's bitrate to avoid stalling. Replace both files if the footage changes.
- **`assets/vendor/page-flip.browser.js` is patched by hand.** The upstream library sizes its canvas in CSS pixels with no regard for `devicePixelRatio`, so catalogue pages looked soft on high-DPI screens and blurred badly under browser zoom. `resizeCanvas()` now multiplies the backing store by the pixel ratio (capped at 4) and scales the drawing context to match. This file is **not** the copy in `node_modules`; reinstalling or re-copying over it silently reverts the fix. If the library is ever upgraded, re-apply the change.
- **Admin errors are shown raw.** When an upstream service fails, `api/admin-families.js` passes the error text through unsanitised, so the admin page can show a full HTML error page. Harmless, but alarming to non-technical users.
- **Two certificate logos are oversized.** `assets/partners/cer-5.png` and `cer-18.png` are ~200 KB each but display at 44 px, and they delay the page's `load` event. Resizing them would help mobile load times.
- **No automated tests.** Verification is manual: load the site, log into `/admin`, Publish a small change, submit a test quote, open a catalogue PDF and a product page with images.
- **`catalogue-pdfs/` is gitignored** (~1.9 GB, over GitHub's file size limit). Keep a separate backup of the source PDFs; they cannot be recovered from the repo.
- **`.gitignore` blanket-ignores `*.txt` and `*.json`**, with named exceptions (`robots.txt`, `package.json`, `vercel.json`, …). A new file of either type is silently ignored unless an exception is added — `git add` gives no warning.
