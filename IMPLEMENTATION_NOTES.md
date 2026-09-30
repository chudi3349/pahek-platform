# Implementation Notes

Running decisions log. Newest entries at the bottom. Each entry: what was decided, why, and what it affects.

Background decisions from discovery are in `PAHEK_BUILD_PLAN.md` (ETLS files) and wiki `sop/infra/pahek-server-baseline`.

---

## 2026-09-28 — Build week 1 started

### Three repositories, not one
`apps.json` gives each app its own git URL, and the image build clones each one separately. A single repo containing both `pahek_core` and `pahek_web` cannot be installed that way. So: `pahek_core`, `pahek_web` (one repo each) and `pahek-platform` (infra + docs). All three to be private and owned by PAHEK's GitHub account.

### Blog Post does not exist in Frappe v16
Checked the live site: no `Blog Post`, `Blog Category` or `Blogger` DocType, and no `Blogger` role. Frappe's own v16 tests now define a synthetic `Test Blog Post`. The build plan assumed the native Blog Post for Charles's articles and the blog index. **Open decision, not yet made** — see the options raised with Victor on 2026-09-28. Until decided, `pahek_web` has no content DocTypes.

### Name collision check
Full DocType (810) and Role lists pulled from the site. None of `PAHEK Site Settings`, `PAHEK Admin`, `PAHEK Editor`, `AI Drafter` exist. Note that core already has `Contact Us Settings`, `About Us Settings`, `Portal Settings`, `Website Settings` — hence the `PAHEK` prefix on everything.

### `pahek_core` contents
- `PAHEK Site Settings` (single): organisation name, tagline, logo, NSCDC licence number, phone, public email, office address, social links. Only identity data. Enquiry and reCAPTCHA settings will live in `pahek_web`, because they belong to PAHEK's website, not to every PAHEK site.
- Roles `PAHEK Admin`, `PAHEK Editor`, `AI Drafter` shipped as fixtures.
- No "ETLS Maintainer" role. A role with no distinct permissions adds nothing; ETLS access is a named System Manager user, disabled at the end of the 60-day support window.
- No settings fallback helper. The only default is a text field, so the `get_single_value` returns-`0` gotcha from `etls_portal` does not arise. Revisit if a numeric/check setting with a non-zero default is added.

### License
Apps are marked Proprietary, copyright PAHEK Security, not MIT. The handover promises PAHEK the source; an open licence would also give it to everyone else.

### Memory limits for db / redis (prepared, not yet applied)
The live stack had no `mem_limit` on `db`, `redis-cache`, `redis-queue`, and MariaDB used its default buffer pool. New `overrides/compose.pahek-limits.yaml`:
- `db`: 1800m, `--innodb-buffer-pool-size=1G` (baseline rule: 1–1.5 GB). The `command:` list replaces the one in `compose.mariadb.yaml`, so its three charset flags are repeated.
- `redis-cache`: 300m, `maxmemory 256mb`, `allkeys-lru` (it is a cache; eviction is correct).
- `redis-queue`: 300m, **no eviction policy** — evicting from the queue would silently drop jobs.

**Status:** pre-change backup taken on the server (`20260928_130437-pahek_ejitech_life-*`). Applying the override was stopped pending Victor's explicit approval for production changes.

### Pending server changes (need approval)
1. Apply the new limits override (`$DC config -q`, then `$DC up -d`; recreates db and both redis containers, seconds of downtime on a site that is not yet public).
2. Install `fail2ban` with an `sshd` jail (needs root; `frappe` has no passwordless sudo).
3. Set host timezone to `Africa/Lagos` so backup schedules and logs read in local time.

### Baseline wiki correction
`unattended-upgrades` is already installed and enabled on the server; the wiki's §7 lists it as outstanding.

---

## 2026-09-30 — Server hardening applied, all 5 pages live, PAHEK Insight built

### Server hardening applied
All three pending changes from 2026-09-28 applied with Victor's explicit approval: memory limits + `innodb_buffer_pool_size=1G` (verified: `1073741824` bytes exactly), fail2ban, timezone → `Africa/Lagos`. **fail2ban gotcha found and fixed:** its default `sshd` jail `journalmatch` pointed at `_SYSTEMD_UNIT=sshd.service`, which doesn't exist on this box (real unit is `ssh.service`, same class of issue as the original `systemctl restart sshd` provisioning gotcha). Without the fix it would have run and shown "active" while silently matching zero login events. Fixed with an explicit `journalmatch = _SYSTEMD_UNIT=ssh.service`. Full detail: wiki `sop/infra/pahek-server-baseline` §5a.

### Homepage shipped as a stopgap Web Page, not yet Desk-editable
Built from the approved prototype, deployed as a native `Web Page` (route `home`) with raw HTML/CSS/JS in Frappe's built-in code-editor fields, `Website Settings.home_page = "home"`. This is temporary — Charles cannot edit this content from the Desk with friendly fields yet. Real editability arrives once `pahek_web`'s DocType-backed content model replaces it. `frappe.init()` needs cwd = the bench's `sites/` directory (default `sites_path='.'`), not the bench root, or it throws `IncorrectSitePath`.

### All 5 approved pages now live the same way
About, Our Approach, Security Risk Assessment and Guarding Operations shipped using the identical Web Page pipeline as the homepage. Images extracted via a forked agent (Pillow), `docker cp`'d into the container's `public/files/<slug>/` (not scp+pipe, which risks corrupting binary data through nested shells). The shared logo (`logo-mark.jpg`) is served once from `/files/homepage/` and referenced by all pages rather than duplicated per page.

**Cross-page navigation had to be rewritten.** Each prototype's nav/footer was copied verbatim from the homepage design, so every link (`#services`, `#blueprint`, `#authority`, `#insights`, `#contact`) only worked as an anchor *on the homepage itself* — on a standalone page like `/about`, clicking "Services" did nothing. Wrote `normalize_links.py` to remap these site-wide once real pages existed: `#blueprint`→`/our-approach`, `#authority`→`/about`, `#insights`→`/insights`, `#services`/`#contact`→`/#services`/`/#contact` (still valid same-page anchors from any page). Also resolved several bare `href="#"` placeholder CTAs by their visible link text (e.g. "Request an Operational Assessment" → `/#contact`, "Explore the Blueprint" → `/our-approach`) and wired up `our-approach`'s embedded cross-page references ("Explore Ground Truths on Guarding Operations" etc.) to their real target pages. Re-deployed the homepage itself with the same fix, since its own nav had the same problem. Genuine placeholders (Privacy Policy, Terms, NSCDC licence, LinkedIn, Office Address, a few unbuilt Resources links) were deliberately left as `#` — nothing exists yet for them to point to.

### PAHEK Insight — the Blog Post replacement, decided and built
Chose **Option A** from the 2026-09-28 decision list: a real DocType, not a third-party blog app. `has_web_view: 1`, `is_published_field: "published"`, controller extends Frappe's `WebsiteGenerator` (the mechanism that replaced the old `website_generators` hook — confirmed by reading `frappe/website/website_generator.py` and `Web Form`'s usage directly against this v16 install, since Blog Post — the obvious reference example — no longer exists to copy from).

- **Detail page:** `templates/generators/pahek_insight.html`, resolved automatically by route.
- **List page:** a plain `www/insights/index.py` + `index.html` (not a generator — just a normal website page that queries published `PAHEK Insight` records itself).
- **AI-draft enforcement, same pattern as the architecture plan:** `published` is at `permlevel: 1`. `PAHEK Admin`/`PAHEK Editor`/`System Manager` get an explicit permlevel-1 permission row (read+write); `AI Drafter` gets none at all — not read, not write. It can create and edit every other field on a draft but the `published` field doesn't exist for it as far as Frappe's permission system is concerned.
- **Shared styling:** rather than duplicate CSS a third time, the full homepage stylesheet was copied to `templates/includes/pahek_shared.css` and both Insight templates pull it in via Jinja `{% include %}`, then layer a handful of article/list-specific rules on top.
- No content DocType existed to reuse a pattern from, so the field list (title, route, article_number, author, published_date, featured, cover_image, excerpt, content, meta_title, meta_description, published) was designed from scratch against what the homepage's insight-card teasers already assume (article number badges, excerpt, thumbnail).

### GitHub — public repos, ETLS-owned for now
Victor: transfer to PAHEK later; for now push under ETLS's GitHub account. No `gh` CLI or credential manager was present on this machine — pushed via SSH key instead (Victor added the default `id_ed25519.pub` to his GitHub account). Repos are **public**, not private — deliberate: no secrets live in the app code (`.env`/keys are gitignored everywhere), and a public repo means the Docker image build on pahek-server needs no embedded access token at all. Simpler now; can be set private (or transferred to PAHEK, which resets the question) later with no code changes needed.
