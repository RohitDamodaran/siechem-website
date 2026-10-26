# Siechem Wires & Cables — Website

Corporate website for Siechem Technologies Pvt. Ltd.: product catalogue, certifications, tech-support calculators, resources and a quote-request form, with a small admin panel for editing the catalogue.

It is a **static HTML site with no framework**. Every public page is a hand-written `.html` file. A few serverless functions (`api/`) cover the parts that cannot be static: the quote form and the admin panel.

## Tech stack

| Part | Technology |
|---|---|
| Pages | Plain HTML, CSS (`theme.css`), vanilla JS |
| Hosting / functions | Vercel |
| Admin database + product images | Supabase (Postgres + Storage) |
| Publishing | GitHub API (commits catalogue JSON back to this repo) |
| Quote emails | SMTP via `nodemailer` |
| Runtime | Node 24.x |

## How it fits together

```
Browser ─► Vercel ─► dist/*.html              static pages + catalogue JSON
                └──► api/*.js                 serverless functions
                        ├─► Supabase          admin catalogue database, product images
                        ├─► GitHub            Publish commits catalog/data/*.json
                        └─► SMTP mailbox      quote-request emails
```

Public pages never query a database. The catalogue lives in committed JSON (`catalog/data/`). Staff edit in the admin panel (`/admin`), which saves **drafts to Supabase**. Clicking **Publish Changes** regenerates the JSON, commits it to this repo in one commit, and triggers a Vercel rebuild. Edits are not public until someone publishes.

## Repository layout

```
siechem-redesign.html     Homepage (served at / via a rewrite in vercel.json)
company/                  About, careers, certifications, policies, contact…
products/                 Product pages and cable configurators
resources/                5S, REACH, RoHS, awards…
tech-support/             Engineering calculators and reference pages
catalogues/               Catalogue flipbook viewer
admin/                    Admin panel pages
api/                      Serverless functions (admin-*, quote, catalogue-pdf)
catalog/data/             Generated catalogue JSON — do not hand-edit
assets/  video/           Images, partner logos, vendored libraries, video
scripts/
  minify.mjs              Build: copies the site into dist/ and minifies it
  gen-sitemap.mjs         Regenerates sitemap.xml
theme.css                 Site-wide styles
vercel.json               Routing, redirects, cache headers
```

## Running locally

Pages use root-absolute paths (`/theme.css`), so serve from the repository root:

```bash
python -m http.server 8321
# open http://localhost:8321/siechem-redesign.html
```

`/` is only rewritten to the homepage on Vercel, so open `/siechem-redesign.html` directly. The `api/` functions and admin panel need `vercel dev` and the environment variables below.

## Build and deploy

```bash
node scripts/minify.mjs     # builds dist/ (what Vercel runs)
```

Vercel runs `node scripts/minify.mjs` and serves `dist/` (see `vercel.json`). Pushing to `main` deploys to production.

After adding or removing a page, regenerate the sitemap:

```bash
node scripts/gen-sitemap.mjs
```

## Environment variables

Set in **Vercel → Project → Settings → Environment Variables**. Never commit them.

| Variable | Purpose |
|---|---|
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Server-side database key (bypasses row-level security; server only) |
| `GITHUB_TOKEN` | Fine-grained token, Contents read/write on this repo (used by Publish) |
| `VERCEL_DEPLOY_HOOK_URL` | Deploy hook that rebuilds the site after Publish |
| `ADMIN_PASSWORD` | Admin panel login |
| `ADMIN_SESSION_SECRET` | Signs admin session tokens (changing it logs everyone out) |
| `SMTP_HOST`, `SMTP_PORT` | Outgoing mail server (port `465` = SSL, `587` = STARTTLS) |
| `SMTP_USER`, `SMTP_PASS` | Mailbox the quote form sends from |
| `SALES_EMAIL` | *Optional.* Where enquiries are delivered; defaults to `SMTP_USER` |

Environment variable changes only reach the functions after a redeploy.

## Things to know

- **Do not hand-edit `catalog/data/*.json`.** The next Publish regenerates it from Supabase and overwrites manual edits.
- **The repo path is hardcoded** in `REPO` at the top of `api/admin-publish.js`. Update it if the repository is ever renamed or moved, or Publish will fail.
- **Product images** live in the Supabase Storage bucket `product-images`, and their absolute URLs are stored in the catalogue JSON. Keep the same Supabase project, or every product image breaks.
- **Quote submissions are not stored.** `api/quote.js` only sends two emails (to the sales inbox and to the customer).
- **`assets/vendor/page-flip.browser.js` is patched** to respect `devicePixelRatio` (sharp flipbook on high-DPI screens). Do not overwrite it with the `node_modules` copy.
- **The hero video has two encodes.** Screens up to 768 px wide (and 2G/3G connections) use `video/cable-video-hero-mobile.mp4`; larger screens use `cable-video-hero.mp4`. Replace both if the footage changes.
- **`.gitignore` ignores `*.txt` and `*.json`** except named exceptions (`robots.txt`, `package.json`, `vercel.json`…). A new file of either type is silently ignored unless an exception is added.
- **Markdown files are not published.** `scripts/minify.mjs` skips `.md` files so documentation never reaches the live site.

## More documentation

- [`HANDOVER.md`](HANDOVER.md) — architecture, account ownership, credential rotation and the transfer checklist
- [`EDITING-GUIDE.md`](EDITING-GUIDE.md) — how to edit catalogue content by hand
