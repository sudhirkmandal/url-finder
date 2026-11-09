# WebIntel - Website & Company Intelligence Dashboard

MERN app that analyzes **publicly accessible** company websites, stores scan history, detects changes between scans, monitors sites on a schedule and compares competitors.

> WebIntel only reads public pages. It respects `robots.txt`, never logs in to a site, never bypasses CAPTCHA or bot protection, and never fabricates data: anything a site does not publish is shown as *unavailable*.

## Quick start

Requirements: Node 18+, MongoDB (local or Atlas).

```bash
# 1. Backend
cd backend
cp .env.example .env          # set MONGODB_URI and a long random JWT_SECRET
npm install
npm run seed                  # optional demo data: demo@webintel.dev / Demo@12345
npm run dev                   # http://localhost:5000

# 2. Frontend (new terminal)
cd frontend
npm install
npm run dev                   # http://localhost:5173 (proxies /api to :5000)
```

Run backend tests: `cd backend && npm test`.

Seed data uses reserved `example.*` domains and is flagged **Demo data** in the UI. Run real scans for real information.

## Features -> where they live

| Feature | Frontend | Backend |
|---|---|---|
| Auth (JWT, bcrypt, protected routes) | `pages/Auth.jsx`, `ProtectedRoute` | `authController`, `middleware/auth` |
| Analyzer + live progress | `pages/Analyzer.jsx`, `ScanProgress` | `websiteController.analyze`, `scanService` |
| Website intelligence / company profile | `pages/WebsiteDetail.jsx` | `scraper/parser.js`, `crawler.js` |
| Activity tracker | `pages/ActivityTracker.jsx` | `diffService.js`, `activityController` |
| Monitoring + history + comparison | `pages/Monitoring.jsx` | `scheduler.js`, `scanService.js` |
| Competitors | `pages/Competitors.jsx` | `competitorController` |
| Analytics | `pages/Dashboard.jsx`, `Analytics.jsx` | `analyticsController` |
| Settings | `pages/Settings.jsx` | `authController` |

## How scanning works

1. URL validated (http/https only, no credentials, allowed ports) and DNS-resolved; every address must be public.
2. `robots.txt` loaded; if the start page is disallowed the scan stops.
3. Homepage fetched with Axios + Cheerio. If it looks like an empty JS app shell and `ENABLE_PUPPETEER=true`, it is rendered with Puppeteer.
4. Sitemap(s) parsed (index files supported, capped).
5. Bounded crawl (`SCRAPER_MAX_PAGES`, concurrency, delay, robots rules). Pages are normalized and de-duplicated.
6. Snapshot saved and diffed against the previous snapshot -> activities.

**Change detection notes**
- New/removed pages, content hash changes, title/description changes, and category detection (blog, product, job, announcement) are **URL-pattern heuristics**; unusual site structures can be missed.
- A page is only reported *removed* after re-requesting it and getting 404/410.
- Because the crawl is capped, very large sites may report different "new pages" between scans; raise `SCRAPER_MAX_PAGES` if needed.
- Content-hash changes can be triggered by dynamic text (dates, counters).

## Security

- **SSRF**: scheme/port allow-list; hostnames resolved and **all** IPs checked; private, loopback, link-local (incl. `169.254.169.254`), CGNAT, multicast and reserved ranges blocked (IPv4, IPv6, IPv4-mapped); redirects followed manually with re-validation each hop; IPs re-validated at connect time via a custom DNS lookup (DNS-rebinding defence).
- Puppeteer requests are intercepted: http(s) only, each host re-validated, images/media/fonts blocked. Chromium resolves DNS itself, so for hostile environments run it in an isolated container/network.
- Helmet, CORS allow-list, request body limit, rate limits (global, auth, scans), Zod validation on all inputs, per-user data scoping on every query, bcrypt (cost 12), JWT.
- Response size (3 MB), timeouts, retries and redirect limits are enforced.

## Environment variables

See `backend/.env.example` and `frontend/.env.example`. `JWT_SECRET` is **required** in production. AI summaries are optional: set `ANTHROPIC_API_KEY` (server-side only). The app is fully functional without it.

## API reference

Base URL `/api`. All routes except `health`, `auth/register`, `auth/login` need `Authorization: Bearer <token>`. Errors: `{ "message": "...", "details": [...] }`.

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Liveness check |
| POST | `/auth/register` | `{name,email,password}` -> `{token,user}` |
| POST | `/auth/login` | `{email,password}` -> `{token,user}` |
| GET | `/auth/me` | Current user |
| PATCH | `/auth/profile` | Update name/email |
| PATCH | `/auth/password` | `{currentPassword,newPassword}` |
| PATCH | `/auth/preferences` | `{defaultFrequency,autoMonitorNewSites,activityDigest}` |
| DELETE | `/auth/account` | Body `{password}` |
| POST | `/websites/analyze` | `{url,monitor?}` -> `{website,scan}`; poll `/scans/:id` |
| GET | `/websites` | `?search&page&limit` |
| GET | `/websites/:id` | Website + latest snapshot + counts |
| DELETE | `/websites/:id` | Remove with all history |
| POST | `/websites/:id/scan` | Queue a manual scan |
| PATCH | `/websites/:id/monitoring` | `{enabled,frequency}` |
| GET | `/websites/:id/snapshots` | Snapshot history |
| GET | `/websites/:id/compare` | Latest vs previous snapshot |
| GET | `/websites/:id/pages` | `?category=blog|product|job|announcement|page&search` |
| POST | `/websites/:id/summary` | Optional AI summary (503 if not configured) |
| GET | `/monitoring` | Websites with monitoring state |
| GET | `/scans` , `/scans/:id` | Scan history / progress |
| GET | `/activities` | `?website&type&days&page&limit` |
| GET | `/competitors` | `?website=<id>` |
| POST | `/competitors` | `{websiteId,url,notes?}` |
| DELETE | `/competitors/:id` | Remove competitor link |
| GET | `/competitors/compare/:websiteId` | Side-by-side profiles + technology matrix |
| GET | `/analytics/overview` | `?days=30` stats and chart series |

## Deployment

- **Database**: MongoDB Atlas; set `MONGODB_URI`.
- **Backend** (Render/Railway/Fly/VPS): `npm ci --omit=dev && npm start`; set `NODE_ENV=production`, `JWT_SECRET`, `CLIENT_URL` (your frontend origin). The scheduler runs inside the API process, so run **one** instance (or set `ENABLE_SCHEDULER=false` on extras). Scans run in-process; for heavy use move them to a worker/queue.
- **Frontend** (Netlify/Vercel/static host): `cd frontend && npm run build`, publish `dist/`, set `VITE_API_URL=https://your-api/api`, and add an SPA rewrite of all paths to `index.html`.
- **Puppeteer**: `npm i puppeteer` in `backend`, `ENABLE_PUPPETEER=true`, and use a container with Chromium dependencies.

## Project structure

```
backend/src  config/ models/ middleware/ validators/ controllers/ routes/ services/(scraper/) utils/ seed.js
backend/tests  ssrf + parser + diff unit tests
frontend/src  api/ context/ components/ pages/ utils/
```

## Verification status

Verified: backend unit tests (SSRF blocking, URL normalization, HTML parsing, diffing); a live crawl of a real public site (robots, sitemap, social, tech detection); SSRF blocks for metadata/localhost targets; frontend production build.
**Not yet verified**: end-to-end flows against a live MongoDB (MongoDB was unavailable in the build sandbox), Puppeteer rendering, and browser UI testing. Run `npm run seed` and walk through each page after setup.
