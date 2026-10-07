# Elapsum → elapsum.dfault.it Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Re-home Elapsum from elapsum.com to elapsum.dfault.it — repo, docs, GitHub, local Laragon, Claude memory — and ship the first production deploy in noindex trial-run mode.

**Architecture:** No code-logic changes. Domain strings and one QR asset change in the hand-authored `_site/` web root; `.htaccess` gains a removable noindex launch toggle; local Apache gets a new vhost + cert mirroring `sub.30days.conf`; the deploy reuses `deploy.sh` with a new `DEPLOY_REMOTE`. Daniel's admin steps (folder move, hosts, certutil, Apache restart) run last because moving the folder ends this session.

**Tech Stack:** Vanilla HTML/CSS/JS (no build), Apache 2.4.62 via Laragon (`E:\vlaragon`), Git Bash `openssl`, `npx qrcode`, headless Chrome + `jsqr`/`pngjs` (scratchpad only) for the QR decode check, `gh` CLI, lftp/SFTP deploy.

**Spec:** `docs/superpowers/specs/2026-10-07-elapsum-subdomain-move-design.md`

## Global Constraints

- New canonical origin: `https://elapsum.dfault.it` (no `www`). Remote folder `__sub/elapsum`. Local folder `E:\www\sub\elapsum`.
- App name stays **Elapsum**. Deck footer legal link label: `Elapsum` (not a domain).
- Version stays **v1.0.0** (`APP_VERSION` in `_site/app.js`, `CACHE` in `_site/sw.js`); sitemap `lastmod` → `2026-10-07`.
- Nothing is deleted: old vhost → `elapsum.conf.disabled`; old cert → `elapsum.com.crt/.key`.
- `docs/superpowers/plans/2026-07-17-elapsum-iteration-1.md` and `_temp/` are history — do not edit.
- The Domeneshop username never goes into a committed file (public repo). It lives only in the gitignored `.env`.
- Tabs for indentation; educational comments in the style already in `.htaccess`.
- Commit messages end with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Commit to the current branch `dfault` (project convention). Do not push unless Daniel says so.
- Real deploy only after Daniel OKs the dry-run output.

## Review Focus

1. A stray `elapsum.com` survives anywhere in `_site/` → the deployed site points somewhere parked. Pinned by the grep gate in Task 1 Step 6.
2. The QR on the desktop deck still (or wrongly) encodes the old URL → phone opens a parking page. Pinned by the decode check in Task 1 Step 4.
3. The new `.htaccess` block has a syntax error → every request 500s on production. Pinned by the local Apache curl in Task 1 Step 7 and the prod curl in Task 6 Step 4.
4. Local cert without the right SAN/EKU → Chrome silently refuses the service worker on `https://elapsum.dfault.it` locally. Pinned by the openssl check in Task 4 Step 3.
5. The new vhost file breaks Apache's config → Laragon's Apache won't start after Daniel's restart. Pinned by `httpd -t` in Task 4 Step 5.

---

### Task 1: Domain swap in `_site/`, new QR, noindex trial-run toggle

**Files:**
- Modify: `_site/index.html` (lines 10, 16, 19, 29, 48, 71, 304, 305, 316)
- Modify: `_site/robots.txt`, `_site/sitemap.xml`, `_site/humans.txt`, `_site/.well-known/security.txt`, `_site/.htaccess`
- Regenerate: `_site/assets/img/qr-elapsum.svg`

**Interfaces:**
- Consumes: nothing.
- Produces: a `_site/` with zero `elapsum.com` strings and an `X-Robots-Tag: noindex` header; Task 6 deploys it.

- [ ] **Step 1: Swap the plain URL strings** (Git Bash, project root)

```bash
sed -i 's#https://elapsum\.com#https://elapsum.dfault.it#g' \
  _site/index.html _site/robots.txt _site/sitemap.xml _site/humans.txt \
  _site/.well-known/security.txt _site/.htaccess
```

- [ ] **Step 2: Fix the non-URL mentions in `_site/index.html`** (Edit tool, exact strings)

- `paste the elapsum.com measurement id` → `paste the elapsum.dfault.it measurement id`
- `alt="QR code linking to elapsum.com"` → `alt="QR code linking to elapsum.dfault.it"`
- `Open on your phone<br>elapsum.com</span>` → `Open on your phone<br>elapsum.dfault.it</span>`
- `<a href="https://elapsum.dfault.it">Elapsum.com</a>` → `<a href="https://elapsum.dfault.it">Elapsum</a>`

And in `_site/sitemap.xml`: `<lastmod>2026-07-18</lastmod>` → `<lastmod>2026-10-07</lastmod>`.

- [ ] **Step 3: Regenerate the QR code**

```bash
npx --yes qrcode -t svg -o _site/assets/img/qr-elapsum.svg "https://elapsum.dfault.it/"
grep -c "<svg" _site/assets/img/qr-elapsum.svg
```
Expected: `1`.

- [ ] **Step 4: Decode the QR independently** (scratchpad, `$S` = session scratchpad path)

```bash
cd "$S" && npm i --silent jsqr pngjs
"/c/Program Files/Google/Chrome/Application/chrome.exe" --headless --disable-gpu \
  --window-size=600,600 --screenshot="$S/qr.png" \
  "file:///E:/www/dev/elapsum.com/_site/assets/img/qr-elapsum.svg"
node -e "const {PNG}=require('pngjs'),jsQR=require('jsqr'),fs=require('fs');const p=PNG.sync.read(fs.readFileSync('qr.png'));const r=jsQR(new Uint8ClampedArray(p.data),p.width,p.height);console.log(r?r.data:'NO QR FOUND')"
```
Expected: `https://elapsum.dfault.it/`. If Chrome can't screenshot, report it and leave the check for Daniel's Pixel camera instead of skipping silently.

- [ ] **Step 5: Add the noindex launch toggle to `_site/.htaccess`**

Insert directly above the `# --- HTTP caching DISABLED during test-in-production ---` banner block:

```apache
# ============================================================
# --- Search engines kept OUT during the trial run ---
# X-Robots-Tag is the reliable way to keep a page out of Google's
# index. robots.txt "Disallow" only stops crawling: a blocked URL
# can still be indexed from links, and a crawler never sees a
# noindex on a page it isn't allowed to fetch. This header beats
# the "index, follow" meta tag in index.html, because when robots
# rules conflict the most restrictive one wins.
# REMOVE AT LAUNCH: delete this block once the Pixel walkthrough passes.
# ============================================================
<IfModule mod_headers.c>
	Header set X-Robots-Tag "noindex"
</IfModule>

```

- [ ] **Step 6: Grep gate — no old domain left in the web root**

```bash
grep -rn -i "elapsum\.com" _site ; echo "exit=$?"
```
Expected: no matches, `exit=1`.

- [ ] **Step 7: Apache parses the new `.htaccess`** (the old `elapsum.com` vhost still serves `_site/` locally until Daniel's steps)

```bash
curl -sk -I https://elapsum.com/ | grep -i -E "^HTTP|x-robots-tag"
```
Expected: `HTTP/1.1 200 OK` and `X-Robots-Tag: noindex`. If Apache isn't serving that vhost (connection refused), note it; Task 6 Step 4 still checks production.

- [ ] **Step 8: Unit tests still green**

```bash
node _dev/run-tests.mjs
```
Expected: all pass (the two error lines in the header note are expected output).

- [ ] **Step 9: Commit**

```bash
git add _site docs/superpowers/specs docs/superpowers/plans/2026-10-07-elapsum-subdomain-move.md
git commit -m "Move to elapsum.dfault.it: domain strings, QR, noindex trial-run toggle"
```

---

### Task 2: Docs for the new home

**Files:**
- Modify: `README.md`, `HANDOFF.md`, `DEPLOY.md`, `SPEC.md`

**Interfaces:**
- Consumes: Task 1's final state (noindex toggle, v1.0.0, folder/vhost/cert names from the Global Constraints).
- Produces: docs that the next session (opened in `E:\www\sub\elapsum`) orients from.

- [ ] **Step 1: README.md** — add a `Live: https://elapsum.dfault.it` line under the tagline; Develop section names the Laragon vhost `sub.elapsum.conf`.
- [ ] **Step 2: HANDOFF.md** — title `# HANDOFF — Elapsum (elapsum.dfault.it)`; "What this is" says hosted at `https://elapsum.dfault.it` (Domeneshop `__sub/elapsum`), first deployed 2026-10-07 in noindex trial-run mode, elapsum.com parked and expected to lapse July 2027, decision record = the 2026-10-07 spec; Local dev section: folder `E:\www\sub\elapsum`, hosts `127.0.0.1 elapsum.dfault.it`, vhost `sub.elapsum.conf`, cert `E:\vlaragon\etc\ssl\elapsum.crt|key` (SAN elapsum.dfault.it, 825 days), old `elapsum.conf.disabled` + `elapsum.com.crt|key` kept; Open items: (1) Daniel's move steps if not yet done, (2) LE cert on prod, (3) Pixel walkthrough on the live URL, (4) go-live list in DEPLOY.md, (5) iteration-2 ideas.
- [ ] **Step 3: DEPLOY.md** — Local dev section uses the new hosts line / vhost / cert / certutil command and notes that while the hosts line is active this PC sees the local copy (comment it out to view prod); Deploy section: remote is `__sub/elapsum` on the Domeneshop webhotel; new "Trial run" section (noindex + no-cache, fixes ship as v1.0.1+); go-live checklist becomes: remove the noindex block, re-enable the cache block + drop the global no-cache block, GA4 property for elapsum.dfault.it → `window.GA_ID`, Search Console, /dd-website-launch, HSTS later. Drop the DNS line (done).
- [ ] **Step 4: SPEC.md** — add under the header quote:

```markdown
> **Amended 2026-10-07:** hosted at `https://elapsum.dfault.it`, not
> elapsum.com — see `docs/superpowers/specs/2026-10-07-elapsum-subdomain-move-design.md`.
> Domain mentions below are updated; the decision record lives there.
```
Then: §2 GitHub remote → `github.com/dahlskebank/elapsum`; §4 tree root → `E:\www\sub\elapsum\`; §6 QR → `https://elapsum.dfault.it/`, legal line → `© 2026 ⌁ <year> → Elapsum · v1.0.0`; §8 canonical → `https://elapsum.dfault.it/`; §9 rewritten for the new hosts/vhost/cert and the Domeneshop subdomain; §12 sidethought gets `**Closed 2026-10-07:** no umbrella domain — see the move spec.`
- [ ] **Step 5: Gate** — `grep -n "elapsum\.com" README.md HANDOFF.md DEPLOY.md SPEC.md` shows only intentional historical mentions (parked domain, old cert name, the July acquisition note).
- [ ] **Step 6: Commit** — `git commit -am "Docs: Elapsum lives at elapsum.dfault.it"`

---

### Task 3: GitHub rename + deploy config

**Files:**
- Create: `.env` (gitignored)

- [ ] **Step 1: Rename the repo** (approved in the design)

```bash
gh repo rename elapsum --repo dahlskebank/elapsum.com --yes
git remote set-url origin https://github.com/dahlskebank/elapsum.git
git remote -v && git ls-remote --heads origin | head -2
```
Expected: origin shows `elapsum.git`; ls-remote lists `refs/heads/dfault`.

- [ ] **Step 2: Write `.env` from 30days'** (values never printed in full)

```bash
sed 's#__sub/30days#__sub/elapsum#' /e/www/sub/30days/.env > .env
grep -c "" .env; grep DEPLOY_REMOTE .env | sed -E 's#(/home/[^/]+/[^/]+/)[^/]+#\1<user>#'
git check-ignore .env
```
Expected: DEPLOY_REMOTE ends in `/__sub/elapsum`; `git check-ignore` prints `.env`.

---

### Task 4: Local Laragon vhost + cert

**Files (outside the repo):**
- Rename: `E:\vlaragon\etc\apache2\sites-enabled\elapsum.conf` → `elapsum.conf.disabled`
- Rename: `E:\vlaragon\etc\ssl\elapsum.crt|.key` → `elapsum.com.crt|.key`
- Create: `E:\vlaragon\etc\apache2\sites-enabled\sub.elapsum.conf`, `E:\vlaragon\etc\ssl\elapsum.crt|.key`

- [ ] **Step 1: Disable the old vhost, keep the old cert**

```bash
cd /e/vlaragon/etc && mv apache2/sites-enabled/elapsum.conf apache2/sites-enabled/elapsum.conf.disabled \
  && mv ssl/elapsum.crt ssl/elapsum.com.crt && mv ssl/elapsum.key ssl/elapsum.com.key
```

- [ ] **Step 2: New cert, same profile as 30days.crt** (`MSYS_NO_PATHCONV` stops Git Bash rewriting `/O=…` into a Windows path)

```bash
cd /e/vlaragon/etc/ssl && MSYS_NO_PATHCONV=1 openssl req -x509 -nodes -newkey rsa:2048 -days 825 \
  -keyout elapsum.key -out elapsum.crt \
  -subj "/O=Laragon local dev/CN=elapsum.dfault.it" \
  -addext "basicConstraints=CA:FALSE" \
  -addext "keyUsage=digitalSignature,keyEncipherment" \
  -addext "extendedKeyUsage=serverAuth" \
  -addext "subjectAltName=DNS:elapsum.dfault.it"
```

- [ ] **Step 3: Verify the cert**

```bash
openssl x509 -in /e/vlaragon/etc/ssl/elapsum.crt -noout -subject -ext subjectAltName,extendedKeyUsage -enddate
```
Expected: `CN=elapsum.dfault.it`, `DNS:elapsum.dfault.it`, `TLS Web Server Authentication`, notAfter ≈ 2029-01.

- [ ] **Step 4: New vhost** — write `E:\vlaragon\etc\apache2\sites-enabled\sub.elapsum.conf`:

```apache
define ROOT "E:/www/sub/elapsum/_site"
define SITE "elapsum.dfault.it"

<VirtualHost *:80>
    DocumentRoot "${ROOT}"
    ServerName ${SITE}
    <Directory "${ROOT}">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:443>
    DocumentRoot "${ROOT}"
    ServerName ${SITE}
    <Directory "${ROOT}">
        AllowOverride All
        Require all granted
    </Directory>

    SSLEngine on
    SSLCertificateFile      E:/vlaragon/etc/ssl/elapsum.crt
    SSLCertificateKeyFile   E:/vlaragon/etc/ssl/elapsum.key

</VirtualHost>
```

- [ ] **Step 5: Config test with the running Apache's binary**

```bash
A="E:/vlaragon/bin/apache/httpd-2.4.62-240904-win64-VS17"
"$A/bin/httpd.exe" -d "$A" -t 2>&1 | tail -3
```
(`-d` sets ServerRoot so httpd finds Laragon's `conf/httpd.conf` instead of its compiled-in default.)
Expected: `Syntax OK`. A warning that `E:/www/sub/elapsum/_site` doesn't exist is expected until Daniel moves the folder — anything else is a failure.

---

### Task 5: Carry Claude's project memory to the new folder key

- [ ] **Step 1: Copy**

```bash
P="/c/Users/Jack Steel/.claude/projects"
mkdir -p "$P/e--www-sub-elapsum/memory" && cp "$P/e--www-dev-elapsum-com/memory/"*.md "$P/e--www-sub-elapsum/memory/"
ls "$P/e--www-sub-elapsum/memory/"
```
Expected: `MEMORY.md` + the 3 notes. (Key pattern confirmed by the existing `e--www-sub-30days`.)

---

### Task 6: First deploy

- [ ] **Step 1: Dry run**

```bash
DRY_RUN=1 ./deploy.sh _site
```
- [ ] **Step 2: STOP — show Daniel the dry-run summary** (uploads, and especially any `Removing` lines for files already in `__sub/elapsum`). Wait for his OK.
- [ ] **Step 3: Real deploy** — `./deploy.sh _site`
- [ ] **Step 4: Verify production** (hosts has no `elapsum.dfault.it` line yet, so this hits Domeneshop)

```bash
curl -sk -I https://elapsum.dfault.it/ | grep -i -E "^HTTP|x-robots-tag|cache-control"
curl -sk https://elapsum.dfault.it/ | grep -o '<link rel="canonical"[^>]*>'
for f in sw.js manifest.webmanifest assets/img/qr-elapsum.svg robots.txt; do echo "$f $(curl -sk -o /dev/null -w '%{http_code}' https://elapsum.dfault.it/$f)"; done
```
Expected: `200`, `X-Robots-Tag: noindex`, `Cache-Control: no-cache`, canonical `https://elapsum.dfault.it/`, all four files `200`. (`-k` because the LE cert may still be issuing; also report which cert prod serves.)

---

### Task 7: Hand over Daniel's steps

- [ ] **Step 1:** Final message with the copy-paste checklist from spec §6.3 (close VS Code → move folder → admin PowerShell: hosts + certutil → restart Apache → reopen VS Code in `E:\www\sub\elapsum` → check `https://elapsum.dfault.it` locally), the LE-cert check, and the push question.
