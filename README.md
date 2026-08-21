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
  local or CDN) and `assets/`. Any file here overrides the engine's copy.
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

`nginx.conf.example` is a **web server virtual host** — nothing here reads it.
Copy it to `/etc/nginx/sites-available/` and edit the certificate path, the
php-fpm socket and the deploy paths. DDEV writes its own vhost, so locally you
do not need it at all. An Apache variant, should this ever move off nginx, is
in the skeleton repo (`panfotis/dopamine-flatcms-skeleton`); it is not shipped
inside the Composer package.

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
  /srv/…-content/   ← live demo. Flows nowhere. Deliberate.
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
rsync -a <server>:/srv/dopamine-flatcms-demo-content/pages/ content/pages/
bin/ship "content: promote the page written on the demo"
```

Do the same for `uploads/` only if R2 is off; with `R2_ENABLED=1` new images
live in the bucket and are referenced by absolute URL, so they are not in
`content/uploads/` to copy.

### On a server

The panel writes to `content/`, so on a server that directory must live
**outside the checkout** — otherwise every `git pull` collides with the edits.
Set `CONTENT_PATH` (and `ROLES_FILE`, to keep real addresses out of the repo)
in the server's `.env`:

```
CONTENT_PATH=/srv/dopamine-flatcms-demo-content
ROLES_FILE=/srv/dopamine-flatcms-demo-content/roles.yml
```

The checkout keeps its own tracked `content/` directory, which the running site
then ignores. Harmless, but do not edit it on the server expecting to see a
change — it is the seed for a fresh install, not the live content.

This repository is **private**, so the server needs an SSH key on the account
(not a repo deploy key — that would not also reach the `theme` submodule):

```bash
ssh-keygen -t ed25519 -C "flatcms demo server" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub          # paste into github.com/settings/keys
ssh -T git@github.com              # expect: Hi panfotis!
```

Clone and update as the **same user** both times, or the pull fails with an
error that reads like the repository disappeared:

```bash
git clone --recurse-submodules git@github.com:panfotis/flatcms-demo-site.git
git pull && composer install --no-dev -o     # to update, later
```
