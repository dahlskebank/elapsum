# HANDOFF — Elapsum (elapsum.dfault.it)

> Orientation for continuing work in a fresh Claude session.
> Read this + README.md; SPEC.md governs design decisions;
> `_temp/ELAPSUM-HANDOFF.md` + `_temp/days-counter-v3.html` are the
> original behavior spec this was built from.

## What this is

Single-user days-counter PWA (since/until/periods/yearly). Vanilla
HTML/CSS/JS, no build step — `_site/` is hand-authored source AND web
root, committed to git. Repo: https://github.com/dahlskebank/elapsum
(public; was `elapsum.com` until 2026-10-07).

Hosted at **https://elapsum.dfault.it** (Domeneshop webhotel, folder
`__sub/elapsum`) — moved there 2026-10-07 before it ever went live, because
a PWA's origin owns its data and moving after launch means export/import on
every device. First deployed 2026-10-07 as v1.0.0 in **trial-run mode**
(noindex + no-cache). elapsum.com was bought 2026-07-18, never deployed,
stays parked, expected to lapse July 2027. Decision record:
`docs/superpowers/specs/2026-10-07-elapsum-subdomain-move-design.md`.

## Architecture

- `_site/model.js` — pure logic; `storage.js` — localStorage (key
  `days.slate.v1`) + import/export; `app.js` — DOM + state.
- `_site/sw.js` — cache-first shell. **Bump CACHE + APP_VERSION (app.js)
  + sitemap lastmod on every deploy.**
- Update lifecycle: sw.js skipWaiting+claim pairs with app.js's
  hadController-gated, overlay-deferred reload — first install never
  reloads, updates wait until no sheet is open.
- Desktop ≥992px shows the deck (landing + QR), not the app.
- Tests: `node _dev/run-tests.mjs` (35). Acceptance vs the real backup:
  `node _dev/acceptance-backup.mjs` (needs `_temp/backup.txt`, gitignored).

## Local dev

- Folder `E:\www\sub\elapsum`. hosts: `127.0.0.1 elapsum.dfault.it`.
  Vhost: `E:\vlaragon\etc\apache2\sites-enabled\sub.elapsum.conf` →
  `_site/` (sub.30days.conf clone).
- Cert `E:\vlaragon\etc\ssl\elapsum.crt|key` (SAN elapsum.dfault.it, 825
  days, same profile as 30days.crt) — must be certutil-trusted once
  (admin) + Apache restarted, or Chrome silently blocks the SW.
- Kept, disabled: `elapsum.conf.disabled` + `elapsum.com.crt|key` (the
  original elapsum.com setup).
- Quick loop: `npx http-server _site -p 8331 -c-1`. Pixel testing: the
  live URL, or Chrome USB port forwarding (recipe in DEPLOY.md).

## Open items (2026-10-07)

1. Done 2026-10-07: the move steps — folder now at `E:\www\sub\elapsum`,
   old `127.0.0.1 elapsum.com` hosts line commented out, local cert
   certutil-trusted, Apache serving `_site/` for elapsum.dfault.it. Verified
   with /dd-website-launch the same day: every production file matches the
   repo byte for byte (tested via `curl --resolve` against the webhotel IP,
   since the hosts line points this PC at the local copy).
2. Done 2026-10-07: production HTTPS is live. Domeneshop issues ONE Let's
   Encrypt cert for the whole webhotel (CN dxd.no; SANs include
   30days.dfault.it and elapsum.dfault.it), issued ~1 h after the subdomain
   was added. Verified with full TLS checking.
   First deploy (v1.0.0) was an additive SFTP push by Claude, not
   `deploy.sh` — lftp hangs without a TTY in Claude's harness (see the
   dd-deploy skill). `__sub/elapsum` was empty beforehand, so Daniel's first
   `DRY_RUN=1 ./deploy.sh _site` should show no `Removing` lines; if any
   appear, they are files Domeneshop put there — review before the real run.
3. Daniel: on-device walkthrough on https://elapsum.dfault.it (SPEC §10
   list — import ×2/dedupe, merge → 21 cards, yearly card, drag-vs-search,
   undo, formats, offline, deck, install). Gestures were ported untested
   from v3 — tuning expected (original handoff §7).
4. Go-live checklist in DEPLOY.md (noindex off, cache block on, GA_ID,
   Search Console, /dd-website-launch, HSTS later).
5. Iteration-2 ideas: SPEC §12 (flexible recurrence — every N days /
   weekday / Nth-of-month — matters for the widget-app future). A birthday
   calendar, if wanted, is a view over the existing yearly events.

## Conventions

Tabs (ported v3 regions keep their 2-space indent); educational
comments; WTFPL; version in three places (see DEPLOY.md); never add
swipe-to-delete; never render letterforms in the E-mark's bars grammar
(wordmark = E-mark + Archivo Expanded type); `_temp/` is read-only
history; `_dev/` is never deployed.
