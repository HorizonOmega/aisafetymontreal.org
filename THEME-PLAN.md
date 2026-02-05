# Plan: aisafetymontreal Ghost Theme

## Overview

Fork the live Dawn theme into a new `aisafetymontreal` theme. Develop it alongside Dawn without touching the live site. The new theme integrates the current static site content (org directory, events calendar, bilingual UI) into Ghost's template system.

**Safety**: Dawn stays active and untouched. The new theme is developed in-place under `data/ghost/themes/aisafetymontreal/`. It can be activated from Ghost Admin when ready — theme switching is instant and reversible.

---

## Architecture

```
ghost-docker/data/ghost/themes/aisafetymontreal/
├── package.json                  ← renamed, updated metadata
├── gulpfile.js                   ← same build system as Dawn
├── default.hbs                   ← modified: add language toggle to header
├── home.hbs                      ← NEW: custom homepage (hero + signup + posts)
├── index.hbs                     ← post archive (kept from Dawn)
├── post.hbs                      ← single post (kept from Dawn)
├── page.hbs                      ← generic page (kept from Dawn)
├── page-directory.hbs            ← NEW: org directory
├── page-events.hbs               ← NEW: Luma calendar embed
├── custom-full-feature-image.hbs ← kept from Dawn
├── custom-narrow-feature-image.hbs
├── custom-no-feature-image.hbs
├── author.hbs                    ← kept from Dawn
├── tag.hbs                       ← kept from Dawn
├── partials/
│   ├── cover.hbs                 ← modified: bilingual hero + Ghost portal signup
│   ├── loop.hbs                  ← kept from Dawn
│   ├── content.hbs               ← kept from Dawn
│   ├── featured-posts.hbs        ← kept from Dawn
│   ├── related-posts.hbs         ← kept from Dawn
│   ├── comments.hbs              ← kept from Dawn
│   ├── content-cta.hbs           ← kept from Dawn
│   ├── pagination.hbs            ← kept from Dawn
│   ├── srcset.hbs                ← kept from Dawn
│   ├── pswp.hbs                  ← kept from Dawn
│   ├── language-toggle.hbs       ← NEW: FR/EN toggle partial
│   └── icons/                    ← kept from Dawn (all SVG icons)
├── assets/
│   ├── css/
│   │   ├── screen.css            ← add imports for new CSS files
│   │   ├── (all existing Dawn CSS files)
│   │   ├── site/directory.css    ← NEW: org directory styles (from static site)
│   │   ├── site/events.css       ← NEW: calendar embed styles
│   │   └── site/language.css     ← NEW: language toggle + .lang-en/.lang-fr styles
│   ├── js/
│   │   ├── main.js               ← existing Dawn JS (carousel, pagination)
│   │   ├── lib/directory.js      ← NEW: org data + filtering + rendering
│   │   └── lib/language.js       ← NEW: language toggle + localStorage persistence
│   ├── fonts/                    ← kept from Dawn (Mulish, Lora)
│   └── built/                    ← compiled output (rebuilt via gulp)
└── locales/
    ├── en.json                   ← UI string translations
    └── fr.json
```

---

## Step-by-step Implementation

### Step 1: Copy Dawn → aisafetymontreal

- `cp -r dawn/ aisafetymontreal/`
- Update `package.json`: name → `aisafetymontreal`, version → `1.0.0`, description
- Run `cd aisafetymontreal && npm install` to get build dependencies
- Run `npx gulp build` to verify the build works before making any changes

### Step 2: Add language toggle infrastructure

**New file: `partials/language-toggle.hbs`**
```handlebars
<div class="lang-toggle">
    <button class="lang-btn" id="btn-fr" type="button" aria-pressed="false" aria-label="Français">FR</button>
    <button class="lang-btn" id="btn-en" type="button" aria-pressed="true" aria-label="English">EN</button>
</div>
```

**New file: `assets/css/site/language.css`**
- Language toggle button styles (from static site CSS)
- `.lang-fr { display: none !important; }` / `.lang-en { display: inline !important; }` rules
- `body.french` overrides

**New file: `assets/js/lib/language.js`**
- `setLanguage(lang)` function
- localStorage persistence
- Cookie setting for server-side hints
- DOMContentLoaded listener for saved preference
- FR/EN button click handlers

**Modify: `default.hbs`**
- Add language toggle partial to header (inside `.gh-head-actions`, before member buttons)
- Add `id="html-root"` to `<html>` tag for language switching

**Modify: `assets/css/screen.css`**
- Add `@import "site/language.css";`

### Step 3: Customize the homepage

**New file: `home.hbs`** — Ghost uses `home.hbs` for the homepage if it exists, falling back to `index.hbs` for archive pages.

Same content as `index.hbs` initially. The key difference is that `default.hbs` already conditionally shows the cover/hero section and featured posts only on `{{#is "home"}}`. Having a separate `home.hbs` lets us customize the homepage independently from the post archive later.

**Modify: `partials/cover.hbs`** — Make it bilingual:

```handlebars
<div class="cover gh-outer">
    <div class="cover-content gh-inner">
        {{#if @site.icon}}
            <div class="cover-icon">
                <img class="cover-icon-image" src="{{@site.icon}}" alt="{{@site.title}}">
            </div>
        {{/if}}

        <div class="cover-description">
            <span class="lang-en">Montréal's hub for AI safety research, policy, and community</span>
            <span class="lang-fr">Le carrefour montréalais pour la recherche, les politiques et la communauté en sécurité de l'IA</span>
        </div>

        {{#if @site.members_enabled}}
            {{#unless @member}}
            <p class="cover-signup-hint">
                <span class="lang-en">Monthly: events, policy updates, research & opportunities</span>
                <span class="lang-fr">Mensuel : événements, politiques, recherche et opportunités</span>
            </p>
            <div class="cover-cta">
                <button class="button members-subscribe" data-portal="signup">
                    <span class="lang-en">Subscribe</span>
                    <span class="lang-fr">S'abonner</span>
                </button>
                <button class="button button-secondary members-login" data-portal="signin">
                    <span class="lang-en">Login</span>
                    <span class="lang-fr">Connexion</span>
                </button>
            </div>
            {{/unless}}
        {{/if}}
    </div>
</div>
```

### Step 4: Create directory page

**New file: `page-directory.hbs`**
- Auto-used when a Ghost page has slug `directory`
- Contains the filter bar HTML (all 6 category filter buttons, bilingual)
- Contains `<ul id="directory-list">` container
- Includes count display

**New file: `assets/js/lib/directory.js`**
- Organization data array (all 16 orgs, copied from static site)
- `renderDirectory()` function
- `updateDirectoryCount()` function
- Filter button click handlers
- XSS escaping helper

**New file: `assets/css/site/directory.css`**
- All directory styles from the static site: `.directory-filter`, `.filter-btn` (with category colors), `.directory-list`, `.directory-card`, `.entry-*`, `.tag-*`
- Responsive breakpoints
- Dark mode overrides for filter buttons and tags

### Step 5: Create events page

**New file: `page-events.hbs`**
- Auto-used when a Ghost page has slug `events`
- Contains the Luma calendar iframe embed
- Bilingual heading

**New file: `assets/css/site/events.css`**
- Calendar embed styles (border-radius, border, responsive height)

### Step 6: routes.yaml

No changes needed. The `/directory/` and `/events/` pages are Ghost pages — they automatically use `page-directory.hbs` and `page-events.hbs` by slug convention.

### Step 7: Build and verify

- `cd aisafetymontreal && npx gulp build`
- Verify `assets/built/screen.css` and `assets/built/main.min.js` are generated
- Ghost should auto-detect the new theme in Admin → Design → Change theme

### Step 8: Create Ghost pages (in Ghost Admin)

Before activating the theme, create two pages in Ghost Admin:
1. **Page** with title "Directory" (slug: `directory`) — can have minimal content or be empty (the template handles everything)
2. **Page** with title "Events" (slug: `events`) — same

### Step 9: Update Ghost navigation (in Ghost Admin)

Set primary navigation to:
- Events → `/events/`
- Directory → `/directory/`
- Newsletter → `/` (or `/blog/` if we separate later)

### Step 10: Activate theme

In Ghost Admin → Design → Change theme → select `aisafetymontreal`. Instant, reversible. If anything looks wrong, switch back to Dawn immediately.

---

## What stays from Dawn (unchanged)

- Post rendering (`post.hbs`, `content.hbs`)
- Post feed/loop (`loop.hbs` — the calendar-date-based list)
- Featured posts carousel
- Related posts
- Comments
- Author/tag pages
- Dark mode system (Auto/Light/Dark setting)
- Font selection (sans-serif/serif)
- Navigation layout options (left/middle/stacked)
- PhotoSwipe lightbox
- Pagination
- All icon SVGs

## What changes from Dawn

| File | Change |
|------|--------|
| `package.json` | Rename, update metadata |
| `default.hbs` | Add language toggle to header, add `id` to html tag |
| `partials/cover.hbs` | Bilingual hero text + signup hint |
| `assets/css/screen.css` | Add 3 new CSS imports |

## What's new

| File | Purpose |
|------|---------|
| `home.hbs` | Dedicated homepage template |
| `page-directory.hbs` | Organization directory page |
| `page-events.hbs` | Events/calendar page |
| `partials/language-toggle.hbs` | FR/EN toggle widget |
| `assets/css/site/directory.css` | Directory styles (~350 lines, from static site) |
| `assets/css/site/events.css` | Calendar embed styles (~30 lines) |
| `assets/css/site/language.css` | Language toggle + display rules (~80 lines) |
| `assets/js/lib/directory.js` | Org data + filtering logic (~180 lines) |
| `assets/js/lib/language.js` | Language switching (~40 lines) |
| `locales/en.json` | English UI strings |
| `locales/fr.json` | French UI strings |

---

## Bilingual scope

- **Theme chrome** (nav labels, buttons, footer): Bilingual via `.lang-en`/`.lang-fr` spans
- **Directory page**: Fully bilingual (org descriptions, filter labels, counts)
- **Events page heading**: Bilingual
- **Cover/hero**: Bilingual
- **Ghost content** (posts, pages body): Single-language (whatever language the author writes in). Ghost doesn't support per-post bilingual content natively. This is a known limitation — acceptable for a newsletter.

---

## Risk assessment

| Risk | Mitigation |
|------|------------|
| Theme breaks the live site | Theme is developed separately; Dawn stays active. Activation is done manually in Ghost Admin. |
| Build fails (missing node_modules) | Run `npm install` + `npx gulp build` before activating |
| Directory data gets stale | Org data lives in `directory.js` — same maintenance model as the static site. Can be migrated to Ghost Content API later. |
| Ghost upgrade breaks theme | Theme targets `ghost >= 5.0.0` (same as Dawn). Ghost 6 is current. |
| Language toggle doesn't work with Ghost portal | Portal (signup/signin modals) is always English. This is a Ghost limitation, not theme-specific. |

---

## Later (not in this phase)

- DNS cutover: `aisafetymontreal.org` → this VPS
- Ghost URL change: `newsletter.aisafetymontreal.org` → `aisafetymontreal.org`
- Caddy redirect: `newsletter.aisafetymontreal.org` → `aisafetymontreal.org`
- Retire GitHub Pages static site
- Migrate org directory data to Ghost Content API (editable in admin)
- Evaluate Mailgun → SES for transactional email
