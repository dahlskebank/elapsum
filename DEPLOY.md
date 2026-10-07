# Deploying Elapsum

Live at **https://elapsum.dfault.it** — Domeneshop webhotel, subdomain
`elapsum.dfault.it` → folder `__sub/elapsum`. (elapsum.com was bought
2026-07-18 but never deployed; it stays parked and is expected to lapse
July 2027. Decision record: `docs/superpowers/specs/2026-10-07-elapsum-subdomain-move-design.md`.)

## Local dev (Laragon)

- Project folder: `E:\www\sub\elapsum` (next to 30days).
- Hosts line: `127.0.0.1 elapsum.dfault.it`. While it's active, THIS PC sees
  the local copy, not production — comment it out to look at the live site
  from the PC (same as 30days).
- Apache vhost: `E:\vlaragon\etc\apache2\sites-enabled\sub.elapsum.conf` → doc root `E:/www/sub/elapsum/_site`
- HTTPS cert: `E:\vlaragon\etc\ssl\elapsum.crt` + `.key` (SAN: elapsum.dfault.it, 825 days).
  Trust it ONCE from an admin PowerShell — without this Chrome silently
  refuses the service worker (the shared laragon.crt has no matching SAN):
  `certutil -addstore Root E:\vlaragon\etc\ssl\elapsum.crt`
  …then restart Apache from the Laragon UI.
- Kept, disabled: `sites-enabled\elapsum.conf.disabled` and
  `ssl\elapsum.com.crt|key` — the original elapsum.com setup.
- Quick loop without Apache: `npx http-server _site -p 8331 -c-1`
  (service workers also register on plain-http localhost).
- **Testing on the Pixel against this PC:** Android can't edit its hosts
  file, so use Chrome's USB port forwarding instead — Pixel: enable USB
  debugging (Developer options), connect by cable; desktop Chrome:
  `chrome://inspect/#devices` → Port forwarding → `8331` →
  `localhost:8331`; then open `http://localhost:8331` ON THE PHONE.
  localhost is a secure context on Android, so the service worker,
  install prompt and offline mode all work — no certs, no DNS.
  (Or just deploy and test on https://elapsum.dfault.it.)

## Version bump — EVERY deploy after the first

1. `CACHE` in `_site/sw.js`  (e.g. `elapsum-v1.0.1`)
2. `APP_VERSION` in `_site/app.js` (same number — fills About + deck footer)
3. `lastmod` in `_site/sitemap.xml`

## Deploy

1. `.env` holds DEPLOY_HOST / DEPLOY_USER / DEPLOY_REMOTE / DEPLOY_KEY
   (gitignored; same account as 30days, remote ends in `__sub/elapsum`).
   Template: `.env.example`.
2. ALWAYS preview first: `DRY_RUN=1 ./deploy.sh _site`
3. Real deploy: `./deploy.sh _site` (mirror --delete: what's not local is removed remotely).

## Trial run (now)

First deployed 2026-10-07 as v1.0.0 in trial-run mode:

- `.htaccess` sends `X-Robots-Tag: noindex` — Google stays out until launch.
- `.htaccess` sends `Cache-Control: no-cache` on everything — fixes reach the
  phone on the next load (30days learned this the hard way: a day-long cache
  left browsers on different versions).
- Do the SPEC §10 walkthrough on the Pixel against the live URL. Fixes ship
  as v1.0.1+ (bump all three version spots above).

## Go-live checklist (after the walkthrough passes)

- [ ] Domeneshop LE certificate for elapsum.dfault.it is live (needed for the trial run too)
- [ ] Delete the "Search engines kept OUT" noindex block in `_site/.htaccess`
- [ ] Re-enable the launch cache block in `_site/.htaccess` (banner marks it), remove the global no-cache block
- [ ] Create the GA4 property for elapsum.dfault.it, paste the measurement id into `window.GA_ID` in `_site/index.html`
- [ ] Search Console: verify + submit sitemap
- [ ] Run /dd-website-launch against production
- [ ] Later: uncomment HSTS in `.htaccess` after HTTPS has been green a while
