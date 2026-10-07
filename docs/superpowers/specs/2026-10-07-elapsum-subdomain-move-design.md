# Elapsum → elapsum.dfault.it — Move + First Deploy

> Approved in conversation 2026-10-07. Amends SPEC.md §6, §8, §9 and
> resolves the §12 umbrella sidethought. Template: E:\www\sub\30days, which
> made the same kind of move (kiande.com → 30days.dfault.it) in July 2026.

## 1. Decision

- **Home:** `https://elapsum.dfault.it` — Domeneshop webhotel, folder
  `__sub/elapsum`. Local folder `E:\www\sub\elapsum`.
- **Why now:** a PWA's origin owns its localStorage, service worker and
  installed icon. Elapsum has never been deployed (elapsum.com still shows
  Domeneshop's parking page), so no data lives on any origin yet and the move
  costs nothing. After launch it would mean export → import on every device.
- **Why a subdomain, not a shared domain with subfolders:** subfolders share
  one origin (one storage bucket, "Clear site data" wipes every app on it,
  scope-carved service workers). A subdomain is its own origin, and Elapsum's
  root-relative paths (`/sw.js`, `scope: "/"`) work there unchanged.
- **Names stay.** The app stays "Elapsum" — the E-mark, icons, favicon,
  og-image and wordmark are built on the E, and the name never depended on the
  domain. 30days stays at `30days.dfault.it`: it is live with daily-use data,
  so renaming its subdomain would be a data migration for a cosmetic gain.
- **No umbrella.** The SPEC §12 "app catalog on elapsum.com" sidethought is
  closed. A birthday calendar, if it ever happens, is a view over Elapsum's
  existing yearly events, not a separate app.
- **elapsum.com** stays parked at Domeneshop and is expected to lapse at
  renewal (July 2027). Nothing redirects from it — it was never public.

## 2. State verified 2026-10-07

- `elapsum.dfault.it` → `__sub/elapsum` exists in the Domeneshop virtual-host
  table and resolves to the same webhotel IP as 30days.
- HTTP 301s to HTTPS, but HTTPS serves Domeneshop's fallback certificate
  (`*.secure.domeneshop.no`, expired 2018): no Let's Encrypt cert issued yet.
  Service worker and install are blocked on prod until it is (1–2 h after
  ordering in the panel).
- elapsum.com: parked, HTTP only, nothing on port 443.

## 3. Repo changes

| What | Change |
|------|--------|
| `_site/index.html` | canonical, og:url, og:image, twitter:image, JSON-LD `url`, GA comment, QR alt + caption → `elapsum.dfault.it`. Footer legal link: label `Elapsum`, href `https://elapsum.dfault.it` (brand, not domain, as the label). |
| `_site/robots.txt`, `sitemap.xml`, `humans.txt`, `.well-known/security.txt`, `.htaccess` banner | domain → `elapsum.dfault.it`; sitemap `lastmod` → deploy day |
| `_site/assets/img/qr-elapsum.svg` | regenerated for `https://elapsum.dfault.it` (same filename, so no reference changes) |
| `_site/.htaccess` | new launch toggle: `Header set X-Robots-Tag "noindex"`, marked REMOVE AT LAUNCH next to the existing cache banner. (robots.txt `Disallow` only stops crawling, not indexing — the header is the correct tool.) No-cache mode stays on during the trial run. |
| version | ships as **v1.0.0** (first deploy ever, nothing to bump from) |
| `.env` (gitignored) | DEPLOY_HOST / USER / KEY copied from 30days' `.env`; `DEPLOY_REMOTE` = 30days' path with `__sub/30days` → `__sub/elapsum` |
| README, HANDOFF, DEPLOY | rewritten for the new home, local folder, vhost, cert, and the trial-run → go-live sequence |
| SPEC.md | dated amendment note at the top pointing here; §6/§8/§9 domain mentions updated; §12 sidethought marked closed |
| untouched | `docs/superpowers/plans/` and `_temp/` (history) |

## 4. Local Laragon setup

- New `E:\vlaragon\etc\apache2\sites-enabled\sub.elapsum.conf` — same shape
  as `sub.30days.conf`, ROOT `E:/www/sub/elapsum/_site`, SITE
  `elapsum.dfault.it`, cert `elapsum.crt/.key`.
- New self-signed cert for `elapsum.dfault.it` (SAN), written to
  `E:\vlaragon\etc\ssl\elapsum.crt/.key`.
- Disabled, not deleted: old `elapsum.conf` → `elapsum.conf.disabled`; old
  cert → `elapsum.com.crt/.key`.
- Claude project memory (3 notes + index) copied from the
  `e--www-dev-elapsum-com` key to `e--www-sub-elapsum`, since memory is keyed
  by folder path.
- Pixel USB port-forward recipe in DEPLOY.md stays as the quick dev loop.

## 5. Deploy

GitHub repo renamed `dahlskebank/elapsum.com` → `dahlskebank/elapsum`
(`gh repo rename`; GitHub redirects the old URL) and the local `origin` URL
updated. Then `DRY_RUN=1 ./deploy.sh _site` → Daniel checks the list → real
deploy via /dd-deploy, confirmed at the moment. Deploy runs before the folder
move; it only uploads files.

## 6. Order of operations

1. **Daniel, now:** turn on SSL for `elapsum.dfault.it` in the Domeneshop
   panel (1–2 h).
2. **Claude:** repo changes (§3) → commit → GitHub rename → `.env` → dry run →
   deploy → Laragon files (§4) → memory copy. Claude says when done.
3. **Daniel, after that, in this order:**
   1. Close VS Code. Move `E:\www\dev\elapsum.com` → `E:\www\sub\elapsum`.
   2. Admin PowerShell: in hosts, comment out `127.0.0.1 elapsum.com`, add
      `127.0.0.1 elapsum.dfault.it`; run
      `certutil -addstore Root E:\vlaragon\etc\ssl\elapsum.crt`.
   3. Restart Apache in Laragon. Reopen VS Code in `E:\www\sub\elapsum`.
   4. Check `https://elapsum.dfault.it` loads locally with the SW registering.

Note: while the hosts line is active, this PC sees the local copy, not prod —
same as 30days. Comment it out to look at production from the PC.

## 7. Trial run → go-live

- **Trial run:** the SPEC §10 walkthrough on the Pixel against
  `https://elapsum.dfault.it` once the LE cert is live. Fixes deploy as v1.0.1+
  (bump `CACHE`, `APP_VERSION`, sitemap `lastmod` every deploy).
- **Go-live, after the walkthrough passes:** remove the noindex header;
  re-enable the launch cache block and drop the global no-cache block; GA4
  property for `elapsum.dfault.it` → paste into `window.GA_ID`; Search Console
  verify + submit sitemap; /dd-website-launch. HSTS later, after HTTPS has
  been green a while.

## 8. Rollback

Nothing in this plan deletes anything. Old vhost and cert are renamed, the old
hosts line is commented, the GitHub rename can be renamed back, and the remote
folder `__sub/elapsum` held nothing of value before the first deploy.

## 9. Out of scope

dfault.it listing/linking Elapsum (SPEC §11: post-launch) · elapsum.com
renewal decision (July 2027) · any change to 30days · iteration-2 features.
