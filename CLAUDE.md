# Fix & Flip Tools — CLAUDE.md

## Project Overview

A web app suite for fix-and-flip real estate investors. Three files:
- **`index.html`** — hub/launcher page: sign in once (or continue as guest), then choose a tool.
- **`walkthrough.html`** — Property Walkthrough: mobile-first estimator — walk a property, toggle repair items, adjust costs, attach photos, save to cloud.
- **`deal-analyzer.html`** — Deal Analyzer: desktop-primary deal analysis tool with a full responsive mobile view.

Long-term vision: expand the suite for investors and Realtors, with potential to monetize as a SaaS product beyond Bryan's own business.

Owner/user: Bryan Purdy (bryan.lee.purdy@gmail.com)

---

## Tech Stack

- **Vanilla HTML/CSS/JS — single file per tool, no build step, no framework**
- `index.html` — hub/launcher page (sign in once, pick a tool)
- `walkthrough.html` — Property Walkthrough app (HTML + CSS + JS in one file)
- `deal-analyzer.html` — Deal Analyzer app (HTML + CSS + JS in one file)
- **Supabase** backend via direct REST API (not the JS SDK):
  - Auth: `/auth/v1/token` and `/auth/v1/signup`
  - Database: `/rest/v1/walkthroughs`, `/rest/v1/deals` (PostgREST)
  - Storage: `/storage/v1/object/walkthrough-photos/{path}`

---

## Supabase Configuration

```
Project URL (original): https://qplidmfishaclysckruq.supabase.co
Custom domain (active): https://api.bluestarrealtygroup.com
Anon public key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InFwbGlkbWZpc2hhY2x5c2NrcnVxIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzQxODc2ODMsImV4cCI6MjA4OTc2MzY4M30.YEhxwrw-HxofmV4N3KRpbgDcT_a_OyqIkx49Wu7CjR0
Plan: Pro ($25/mo) + Custom Domain add-on ($10/mo)
```

**`SUPABASE_URL` in all three files (`index.html`, `walkthrough.html`, `deal-analyzer.html`) is set to `https://api.bluestarrealtygroup.com`** — do not revert to the `.supabase.co` subdomain. The custom domain was added to fix AT&T ISP DNS resolution failures with Supabase's free-tier subdomain.

### Email Templates

Supabase email templates have been customized in **Authentication → Email Templates**:
- **Confirm signup** — branded dark-green HTML template matching the app aesthetic, with "Fix & Flip Tools / by Blue Star Realty Group" header and lime green CTA button.
- **Reset Password** — same branding and layout, with password-reset–specific copy.

### Database — `walkthroughs` table

```sql
create table walkthroughs (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references auth.users(id) on delete cascade,
  name text not null,
  property_name text default '',
  bath_count integer not null default 1,
  items jsonb not null default '{}',
  subtotal numeric not null default 0,
  contingency_pct numeric not null default 10,
  total numeric not null default 0,
  notes text not null default '',
  shared boolean not null default false,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
alter table walkthroughs enable row level security;
create policy "Users manage own walkthroughs" on walkthroughs
  for all using (auth.uid() = user_id);
create policy "Public read shared walkthroughs" on walkthroughs
  for select using (shared = true);
```

### Database — `deals` table (Deal Analyzer)

```sql
create table deals (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references auth.users(id) on delete cascade,
  name text not null,
  data jsonb not null default '{}',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
alter table deals enable row level security;
create policy "Users manage own deals" on deals
  for all using (auth.uid() = user_id);
```

The `data` column stores the full deal state: `{ name, inputs: {...}, results: { profit, totalCosts }, savedAt }`.

### Storage — `walkthrough-photos` bucket

- Private bucket, accessed via signed URLs (1-year expiry = `31536000` seconds)
- Files stored at path: `{user_id}/{timestamp}_{random}.{ext}`
- **Important:** Supabase's sign endpoint returns a relative path like `/object/sign/...` (missing `/storage/v1` prefix). The code prepends it correctly:
  ```js
  if (url.startsWith('/')) url = url.startsWith('/storage/') ? SUPABASE_URL + url : `${SUPABASE_URL}/storage/v1${url}`;
  ```
- RLS policies for the storage bucket need to be applied manually in the Supabase SQL editor (allow authenticated users to insert/select their own objects under `{user_id}/` prefix).

---

## Git & Deployment

- **Repo:** `https://github.com/bryanlpurdy/fix-and-flip-tools`
- **Branch:** `main`
- **Hosting:** GitHub Pages (auto-deploys from `main` on push — takes ~1 min)
- **Hub:** `https://tools.bluestarrealtygroup.com` (index.html)
- **Walkthrough:** `https://tools.bluestarrealtygroup.com/walkthrough.html`
- **Deal Analyzer:** `https://tools.bluestarrealtygroup.com/deal-analyzer.html`
- **Fallback:** `https://bryanlpurdy.github.io/fix-and-flip-tools/`
- **Workflow:** edit locally → `git add` → `git commit` → `git push` → GitHub Pages redeploys

There is no CI, no build step, no package.json. Push and it's live.

### Custom Domains

**Frontend:** `tools.bluestarrealtygroup.com` (GitHub Pages)
- GoDaddy CNAME → `bryanlpurdy.github.io`
- `CNAME` file in repo root
- HTTPS enforced via GitHub Pages + Let's Encrypt

**Supabase API:** `api.bluestarrealtygroup.com` (Supabase Custom Domain add-on)
- GoDaddy CNAME → `qplidmfishaclysckruq.supabase.co`
- TXT record `_acme-challenge.api` verified for SSL
- Configured in Supabase Settings → Custom Domains

**Standalone product domain** (e.g. `flipstacktools.com`) remains an option if the tools are ever monetized as a SaaS product outside the Blue Star brand.

---

## App Architecture — `index.html` (Hub)

The hub is the entry point for the suite. It handles auth and redirects users to tools.

### Views
- **Sign-in view** (default) — full-page centered card: email/password form, sign-up toggle, "Continue as Guest" button.
- **Hub view** (after auth) — header with brand + auth status, tool card grid.

### Auth flow
```js
initAuth()           // on load: checks sb_session in localStorage, verifies with Supabase
  → valid session    → showHub()   (user sees tool cards, signed in)
  → no/expired session → showSignin()

submitAuth()         // sign in or sign up
  → success          → saveSession() → showHub()

continueAsGuest()    // skip auth
  → showHub()        (currentUser = null, walkthrough note shown)

signOut()            // clear session
  → clearSession() → backToSignin()
```

### Shared session
All three files use the same `localStorage` key `sb_session` (`{ access_token, user }`). Signing in on the hub means both tools open already authenticated — no second login required.

### Guest mode
Deal Analyzer is fully usable as a guest. Property Walkthrough requires sign-in to save — a warning note is shown on its tool card for guest users.

### Back navigation
Both tools have a `← All Tools` / `← Fix & Flip Tools` link back to `index.html`:
- Desktop: `<a href="index.html" class="hub-back">← All Tools</a>` in each tool's `<header>`
- Deal Analyzer mobile: `<a href="index.html" class="msv-hub-back">← Fix & Flip Tools</a>` inside `#mobileSigninView` (positioned absolute top-left, since the header is hidden on mobile sign-in)

---

## App Architecture — `walkthrough.html`

### State

```js
const state = {
  propertyName: '',
  items: {},          // keyed by item id: { enabled, qty, sqft, cost, photos: [] }
  misc: { expenses: [], photos: [] },
  notes: ''           // property comments (free text)
};
```

Misc expenses/photos are saved into the `items` JSONB column under the special key `_misc_`. Notes save as a top-level `notes` column:
```js
// saving:  { items: { ...state.items, _misc_: state.misc }, notes: state.notes }
// loading: state.misc = wt.items._misc_; state.notes = wt.notes;
```

### Sections (SECTIONS config)

Section order (top to bottom in the editor):
1. **Interior & Kitchen** — toggle/count/sqft items. "Interior Demo" is the first item. Ceiling Fan and Light Fixture live here.
2. **Bathrooms** — all `count` type (per-item qty, no global bath counter).
3. **Exterior & High Ticket Items** — toggle/count items (roof, AC, electric, windows, etc.). First item is "Exterior Demo".
4. **Misc Expenses / General Photos** — `{ id: 'misc', miscSection: true, items: [] }` — freehand dollar expenses + general property photos.
5. **Property Comments** — free-text notes section, rendered after SECTIONS via `renderNotesSection()`.

### Item types

- `toggle` — on/off with adjustable cost
- `count` — integer quantity with per-unit cost
- `sqft` — square footage with per-sqft rate

### Sharing

- Each walkthrough has a `shared boolean` column. When `true`, a Supabase RLS policy allows anonymous SELECT on that row.
- Share URL format: `walkthrough.html?share=<UUID>`
- On page load, if `?share=` is present the app skips normal boot and calls `loadSharedView(id)` — fetches with anon key only, renders `renderShareReport(wt)` in `#shareView`
- The report shows only enabled items, photos/captions, misc expenses, property comments, and budget totals — no edit controls
- Owner can revoke by patching `shared = false` (Disable button on the list card)

### Auth

- Sign in / sign up via Supabase email+password
- Session stored in `localStorage` as `sb_session` (`{ access_token, user }`)
- Auth fetch pattern: `.then(r => r.json()).catch(() => ({}))` — never use AbortController timeout (breaks cold-start logins on Supabase free tier)
- No guest mode — saves require a signed-in account

### Photos

- Each repair item has a `photos: []` array: `{ path, url, caption }`
- Misc section has its own `photos: []`
- Upload: `POST /storage/v1/object/walkthrough-photos/{path}`
- Sign: `POST /storage/v1/object/sign/walkthrough-photos/{path}` — see URL fix note above
- Delete: `DELETE /storage/v1/object/walkthrough-photos` with `{ prefixes: [path] }`
- Lightbox viewer: `#photoViewer` overlay, triggered by `viewPhoto(url)`

---

## App Architecture — `deal-analyzer.html`

### Layout
Desktop (>1024px): three-column — saved deals sidebar (resizable) | inputs panel | results panel (sticky).

Mobile (≤1024px): three-view pattern controlled entirely by **JavaScript** (not CSS media queries — see gotcha below):
- **Sign-in view** (`#mobileSigninView`) — full-screen landing: app icon, title, Sign In button, Continue as Guest. Shown when not signed in on mobile.
- **List view** (`#mobileListView`) — deal cards showing name, ARV, color-coded profit. Shown when signed in on mobile.
- **Editor view** (`.layout`) — single-column scrolling inputs + sticky bottom bar with live Expected Profit + Save button.

### Mobile View Switching
```js
function isMobile() { return window.innerWidth <= 1024; }

function showMobileList() {
  const signedIn = !!currentUser;
  document.getElementById('mobileSigninView').style.display = signedIn ? 'none' : 'block';
  document.getElementById('mobileListView').style.display  = signedIn ? 'block' : 'none';
  document.querySelector('.layout').style.display = 'none';
  document.body.classList.remove('mobile-editor');
  if (signedIn) document.body.classList.remove('mobile-signin');
  else          document.body.classList.add('mobile-signin');
}

function showMobileEditor() {
  document.getElementById('mobileSigninView').style.display = 'none';
  document.getElementById('mobileListView').style.display   = 'none';
  document.querySelector('.layout').style.display = '';
  document.body.classList.remove('mobile-signin');
  document.body.classList.add('mobile-editor');
  window.scrollTo(0, 0);
}
```
`body.mobile-editor` is used as a CSS hook for editor-mode styling (single-column grid, sticky profit bar, etc.) but show/hide is always driven by JS inline styles, not CSS selectors. **Do not switch this back to a CSS-only approach** — it proved unreliable across browsers and screen sizes.

`body.mobile-signin` hides the header (`header { display: none !important }`) during the sign-in landing screen. `showMobileEditor()` **must remove this class** — forgetting to do so was a previous bug where the sign-in screen remained visible on top of the editor.

On init: `loadSession()` runs synchronously first, then `if (isMobile()) showMobileList()`, then `initAuth()` (async).

### Auth
Same Supabase email+password pattern as the walkthrough. Session in `localStorage` as `sb_session`. Guest mode is supported in the Deal Analyzer (no local storage fallback — guest users just can't save). Signed-in users get cloud sync via the `deals` Supabase table.

### Deal Data Flow
Inputs → `calculate()` → results panel updates live. Save → `openSaveModal()` → `confirmSave()` → Supabase. Load → `loadCloudDeal(id)` → `populateFields(inputs)` → `calculate()`.

---

## AI Coding Guidelines

### Think Before Coding
Don't assume. Don't hide confusion. Before implementing, explicitly state assumptions, flag ambiguity, and ask for clarification when the request could be interpreted multiple ways. Surface tradeoffs before writing code, not after.

### Surgical Changes
Touch only what you must. When editing any of these large single-file apps, don't improve unrelated sections or refactor working code while fixing something else. Match the existing style. Remove only what the current change made unused.

---

## Known Gotchas

- **Supabase plan is Pro** — no free-tier pausing. The project stays active 24/7.
- **No AbortController** on auth fetches — was tried, broke logins during Supabase cold start.
- **Signed URL path prefix** — Supabase Storage sign endpoint omits `/storage/v1` from the returned path. Fixed with the conditional prepend above.
- The `anon` key is intentionally in client-side code — it's the Supabase public anon key, not a secret. RLS policies are the security boundary.
- **Deal Analyzer mobile view — use JS, not CSS** — CSS `body:not(.mobile-editor)` media query selectors proved unreliable across browsers/screen sizes for show/hide. The current approach uses `element.style.display` inline styles controlled by `showMobileList()` / `showMobileEditor()`. Do not revert to a CSS-only approach.
- **`showMobileEditor()` must hide `#mobileSigninView` and remove `body.mobile-signin`** — previously missing these caused the sign-in screen to remain visible on top of the editor when clicking "Continue as Guest".
- **All three files must use the custom Supabase URL** — always use `https://api.bluestarrealtygroup.com` in `index.html`, `walkthrough.html`, and `deal-analyzer.html`. Never revert to `qplidmfishaclysckruq.supabase.co`.
