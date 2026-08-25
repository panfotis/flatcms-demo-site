# Dopamine FlatCMS site

Create a site, start DDEV, and open the panel:

```bash
composer create-project dopamine/flatcms-skeleton my-site
cd my-site
cp .env.example .env
ddev start
ddev launch /admin.php
```

The engine lives in `vendor/dopamine/flatcms`. Site-owned files live here:

- `theme/` is the site: layouts, components, `theme.yml` (global CSS/JS,
  local or CDN) and `assets/`. Everything in `theme/` is this site's own;
  the engine keeps only `@flatcms/…` (head, picture, video facade).
- `admin-theme/` brands the panel — `assets/css/admin.css` is the supported
  surface; overriding panel templates tracks engine internals.
- `content/` contains pages, globals, revisions, and uploads.
- `config.php` and `.env` configure this installation.

Styling ladder, cheapest first: add rules to `theme/assets/css/site.css`
(emitted last, wins the cascade) → copy a component's `.css` beside your own
copy of its folder → copy the whole component folder. Each step up costs you
a file that never receives an engine update again.

Never edit files in `vendor/`; Composer updates replace them.

## Production requirements

- **Cloudflare Access in front of `admin.php` — required.** The panel has no
  password of its own; Access *is* the login system, and a production boot
  refuses to start with auth off.
- **Cloudflare page caching — recommended.** The engine tags every response and
  purges the edge on each save, so pages are served from cache and never stale.
- **R2 — optional.** By default uploads live in `content/uploads/` and travel
  with the content git repo — that repo is the backup: one `git clone` restores
  pages and images together. A media-heavy site flips `R2_ENABLED=1` instead.

Setup steps for all three are in the engine README's "Cloudflare setup"
section (`vendor/dopamine/flatcms/README.md`).

`nginx-vhost-additions.conf` is **not** a virtual host — nothing here reads it.
The server runs CloudPanel, which generates the server block itself, so that
file holds only the directives to paste into CloudPanel's Vhost editor. DDEV
writes its own vhost, so locally you do not need it at all. A full standalone
vhost, for a plain-nginx or Apache box, is in the skeleton repo
(`panfotis/dopamine-flatcms-skeleton`).

Run the health check from the site root with:

```bash
bin/doctor
```

The remaining `bin/` wrappers expose backup, restore drill, form retry and
retention jobs, keeping their implementation in the versioned engine package.
They stay dormant until `content/` has its own git remote and the contact form
has `FORM_TO`/`MAIL_DSN` set.

`bin/deploy.sh` and `bin/rollback.sh` are deliberately **not** here. They drive
the atomic-release layout, and `deploy.sh` fetches a revision with
`git clone --no-checkout` plus `git checkout -- .` — which never initialises
submodules, so `theme/` would land empty on every release. This site deploys
with `git pull`; see "On a server" below.

---

## Working on this demo

`theme/` is a **submodule** of
[panfotis/flatcms-theme-demo](https://github.com/panfotis/flatcms-theme-demo) —
it keeps its own history and is published on its own. A fresh clone therefore
needs the submodule, and two local settings that git does not carry in a repo:

```bash
git clone --recurse-submodules git@github.com:panfotis/flatcms-demo-site.git
cd flatcms-demo-site
git config push.recurseSubmodules on-demand   # one `git push` pushes both repos
git config submodule.recurse true             # one `git pull` updates both
ddev start
ddev launch /admin.php
```

### Shipping

```bash
bin/ship "hero: bigger type"
```

One command, in this order:

1. `bin/doctor` — refuses to ship a site that does not validate
2. refreshes `theme/_demo-content/` from `content/`
3. commits the theme repo, then this one
4. pushes both

**`content/` is the source of truth for the demo pages.** `theme/_demo-content/`
is the copy someone gets when they clone the theme, and nothing else syncs it —
it has drifted before. `bin/ship` rsyncs `pages/` and `uploads/` across with
`--delete`, so a page you removed here disappears there too. `.revisions/` is
never copied and is not tracked in this repo.

The catch: `content/` is your live working copy, so whatever is in it when you
ship becomes the theme's starter content. `bin/doctor` catches broken YAML, not
a half-written page. `git -C theme status --short` before shipping if you have
been mid-edit.

### Which way content flows

```
   /admin.php (laptop)          bin/ship               people cloning
        │                          │                    the theme
        ▼                          ▼                        ▲
    content/  ───── rsync ──▶ theme/_demo-content/ ──────────┘

   /admin.php (server)
        │
        ▼
  ~/content/   ← live demo, CONTENT_PATH. Flows nowhere. Deliberate.
```

`content/` on **this machine** is the source of truth for the demo, and
`bin/ship` is the only thing that syncs it into the theme.

Pages you edit in the panel **on the server** stay on the server. Nothing pulls
them back, `bin/ship` will not publish them, and the two copies simply diverge.
That is intended: the server instance is for exercising the panel, R2 and
Cloudflare Access, not for authoring. Nothing is destroyed either way —
`bin/ship` never touches the server, and `git pull` never touches
`CONTENT_PATH`.

If a page authored on the server should become part of the demo, bring it down
deliberately, then ship from here:

```bash
rsync -a <server>:/home/fotispan-dopamine-flatcms-demo/content/pages/ content/pages/
bin/ship "content: promote the page written on the demo"
```

Do the same for `uploads/` only if R2 is off; with `R2_ENABLED=1` new images
live in the bucket and are referenced by absolute URL, so they are not in
`content/uploads/` to copy.

### On a server

**There is no `.env` in a fresh clone — create it.** It is gitignored, so
nothing brings it down for you, and without it the site runs on defaults that
are wrong for a server: every canonical URL, every `og:url` and the whole
sitemap publish `http://localhost:8080`.

The panel writes to `content/`, so that directory has to live **outside the
checkout** — otherwise the first page saved in `/admin.php` makes the next
`git pull` refuse to run. Copy it out once, then point `.env` at it:

```bash
mkdir -p ~/content
cp -R content/. ~/content/
cp users.yml ~/content/users.yml     # put the real address in this copy

cat > .env <<'EOF'
SITE_BASE_URL=https://dopamine-flatcms-demo.fotispan.gr
CONTENT_PATH=/home/fotispan-dopamine-flatcms-demo/content
USERS_FILE=/home/fotispan-dopamine-flatcms-demo/content/users.yml
ADMIN_LOCALE=en
EOF

bin/doctor
```

`CONTENT_PATH` must match the `/uploads/` alias in the vhost exactly, trailing
slash included. They are two routes to one directory — PHP resolves images
through `img.php`, the browser fetches videos straight off the path — so a
mismatch shows up as working images and a missing video, not as an obvious
break.

The checkout keeps its own tracked `content/`, which the running site then
ignores once `CONTENT_PATH` is set. That copy is the **seed**: it is what
`git pull` updates when the demo gains a page or an image, and it does not
reach the live directory on its own. Bring changes across deliberately:

```bash
git pull && composer install --no-dev -o
cp -R content/pages/. ~/content/pages/      # only when the seed actually changed
cp -R content/uploads/. ~/content/uploads/
```

Pages written in the panel on the server stay on the server; `bin/ship` never
publishes them. That is the trade the split buys — the panel is free to write,
and deploys are free to run.

Both repositories are public today, so the server needs no key — but
`.gitmodules` records the theme over **SSH**, because that is the URL `bin/ship`
pushes through from a laptop. A server cloning over HTTPS therefore fails on
`git submodule update` with `Permission denied (publickey)`. Override the
submodule URL locally, once:

```bash
git clone https://github.com/panfotis/flatcms-demo-site.git
cd flatcms-demo-site
git config submodule.theme.url https://github.com/panfotis/flatcms-theme-demo.git
git submodule update --init
ls theme/components | wc -l                  # 12 — an empty theme/ is the bug below
composer install --no-dev -o
git config submodule.recurse true            # plain `git pull` updates the theme too
```

Do **not** run `git submodule sync` afterwards; it resets that URL back to SSH.

**An empty `theme/` fails loudly.** There is no engine theme underneath any
more: an uninitialised submodule means `layout.twig` resolves nowhere, and both
`bin/doctor` and the first render say so by name instead of serving a
placeholder site. Run doctor after cloning.

If this repository is ever made private, only `origin` needs an SSH key on the
GitHub **account** (a repo deploy key is scoped to one repository and would not
reach the submodule). The theme stays public, so the HTTPS override above keeps
working. Clone and pull as the same user, or the pull fails with an error that
reads like the repository disappeared.
