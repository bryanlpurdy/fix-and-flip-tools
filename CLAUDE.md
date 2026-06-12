# Fix & Flip Tools — CLAUDE.md

## Project Overview

A web app suite for fix-and-flip real estate investors and Realtors. Current tools:
- **`index.html`** — hub/launcher page: sign in once (or continue as guest), then choose a tool.
- **`walkthrough.html`** — Property Walkthrough: mobile-first estimator — walk a property, toggle repair items, adjust costs, attach photos, save to cloud.
- **`deal-analyzer.html`** — Deal Analyzer: desktop-primary deal analysis tool with a full responsive mobile view.
- **`net-sheet.html`** — Seller Net Sheet: Realtor-focused tool to estimate seller net proceeds at closing. Save, share, and print.

Long-term vision: expand the suite for investors and Realtors, with potential to monetize as a SaaS product beyond Bryan's own business.

Owner/user: Bryan Purdy (bryan.lee.purdy@gmail.com)

---

## Planned Tools

All planned tools are **Realtor-focused**. Remaining build order: Make Ready List → Bookkeeping.

### Make Ready List (`make-ready.html`)
A seller prep checklist agents walk through with sellers before listing. Categories: Exterior, Interior, Kitchen, Bathrooms, Systems, Staging. Items are checkable with priority levels (Must Do / Should Do / Nice to Have) and notes. Shareable link and print/PDF output. Reuses Walkthrough architecture patterns.

### Realtor Bookkeeping (`bookkeeping.html`)
Lightweight **property-centric** expense tracking. Each property tracks individual expenses:
- **Fields:** Date, category, amount, description, receipt photo
- **Categories:** Photography/Video, Marketing/Advertising, Make Ready, Mileage, Staging, Signage, Closing Gift, Misc
- **Mileage entries:** Input miles → auto-calculate dollars at current IRS standard rate
- **Receipt photos:** Reuse Supabase Storage pattern from Walkthrough
- **Reports:** Total expenses per property, breakdown by category
- Scope is intentionally narrow — not general bookkeeping, just per-property deal costs

---

## Tech Stack

- **Vanilla HTML/CSS/JS — single file per tool, no build step, no framework**
- `index.html` — hub/launcher page (sign in once, pick a tool)
- `walkthrough.html` — Property Walkthrough app (HTML + CSS + JS in one file)
- `deal-analyzer.html` — Deal Analyzer app (HTML + CSS + JS in one file)
- `net-sheet.html` — Seller Net Sheet app (HTML + CSS + JS in one file)
- **Supabase** backend via direct REST API (not the JS SDK):
  - Auth: `/auth/v1/token` and `/auth/v1/signup`
  - Database: `/rest/v1/walkthroughs`, `/rest/v1/deals`, `/rest/v1/net_sheets` (PostgREST)
  - Storage: `/storage/v1/object/walkthrough-photos/{path}`

---

## Supabase Configuration

```
Project URL (original): https://qplidmfishaclysckruq.supabase.co
Custom domain (active): https://api.bluestarrealtygroup.com
Anon public key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InFwbGlkbWZpc2hhY2x5c2NrcnVxIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzQxODc2ODMsImV4cCI6MjA4OTc2MzY4M30.YEhxwrw-HxofmV4N3KRpbgDcT_a_OyqIkx49Wu7CjR0
Plan: Pro ($25/mo) + Custom Domain add-on ($10/mo)
```

**`SUPABASE_URL` in all four files (`index.html`, `walkthrough.html`, `deal-analyzer.html`, `net-sheet.html`) is set to `https://api.bluestarrealtygroup.com`** — do not revert to the `.supabase.co` subdomain. The custom domain was added to fix AT&T ISP DNS resolution failures with Supabase's free-tier subdomain.

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

### Database — `net_sheets` table (Seller Net Sheet)

```sql
create table net_sheets (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references auth.users(id) on delete cascade,
  name text not null,
  data jsonb not null default '{}',
  net_proceeds numeric not null default 0,
  shared boolean not null default false,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
alter table net_sheets enable row level security;
create policy "Users manage own net sheets" on net_sheets
  for all using (auth.uid() = user_id);
create policy "Public read shared net sheets" on net_sheets
  for select using (shared = true);
```

The `data` column stores the full form state (all inputs). `net_proceeds` is a denormalized summary for the sidebar card display. `shared = true` enables the public read-only share view via `?share=<UUID>`.

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
- **Net Sheet:** `https://tools.bluestarrealtygroup.com/net-sheet.html`
- **Fallback:** `https://bryanlpurdy.github.io/fix-and-flip-tools/`
- **Workflow:** edit locally → `git add` → `git commit` → `git push` → GitHub Pages redeploys

There is no CI, no build step, no package.json. Push and it's live.

### Custom Domains

DNS is managed through **Cloudflare** (nameservers: `isaac.ns.cloudflare.com`, `nucum.ns.cloudflare.com`). GoDaddy remains the domain registrar but no longer handles DNS. All records live in Cloudflare.

**Frontend:** `tools.bluestarrealtygroup.com` (GitHub Pages)
- Cloudflare CNAME → `bryanlpurdy.github.io` — **DNS only (gray cloud)**
- `CNAME` file in repo root
- HTTPS enforced via GitHub Pages + Let's Encrypt

**Supabase API:** `api.bluestarrealtygroup.com` (Supabase Custom Domain add-on)
- Cloudflare CNAME → `qplidmfishaclysckruq.supabase.co` — **DNS only (gray cloud), never proxy**
- TXT record `_acme-challenge.api → K-fIshpAu1tvR_UWXZ0H1vC8AsCVcZLRQXIz7ljSxRM` for Supabase SSL
- Configured in Supabase Settings → Custom Domains
- **Do not switch `api` to Proxied (orange cloud)** — Cloudflare would terminate SSL instead of Supabase, breaking the custom domain cert

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
  → valid session    → showHub()
  → expired token    → refreshSession() → showHub()  (or showSignin() if refresh fails)
  → no session       → showSignin()

submitAuth()         // sign in or sign up
  → success          → saveSession() → showHub()

continueAsGuest()    // skip auth
  → showHub()        (currentUser = null, walkthrough note shown)

signOut()            // clear session
  → clearSession() → backToSignin()
```

### Shared session
All four files use the same `localStorage` key `sb_session` (`{ access_token, refresh_token, user }`). Signing in on the hub means all tools open already authenticated — no second login required.

### Guest mode
Deal Analyzer is fully usable as a guest. Property Walkthrough and Net Sheet require sign-in to save — a warning note is shown on the Walkthrough tool card for guest users. Net Sheet mobile sign-in view offers "Continue as Guest" which shows an empty editor (no save capability).

### Back navigation
Both tools have a `← All Tools` / `← Fix & Flip Tools` link back to `index.html`:
- Desktop: `<a href="index.html" class="hub-back">← All Tools</a>` in each tool's `<header>`
- Deal Analyzer mobile sign-in: `<a href="index.html" class="msv-hub-back">← Fix & Flip Tools</a>` inside `#mobileSigninView` (positioned absolute top-left, since the header is hidden on mobile sign-in)
- Deal Analyzer mobile list: `<a href="index.html" class="mlv-hub">← All Tools</a>` inside the `.mobile-list-nav` bar at the top of `#mobileListView` (the main header is hidden in list mode via `body.mobile-list`)
- Walkthrough mobile list: `<a href="index.html" class="wlv-hub">← All Tools</a>` inside the `.wt-list-nav` bar at the top of `#listView` (the main header is hidden in list mode via `body.list-mode`)
- Walkthrough desktop: `<a href="index.html" class="hub-back">← All Tools</a>` in the main `<header>` (always visible on desktop)
- Net Sheet desktop: `<a href="index.html" class="hub-back">← All Tools</a>` in the main `<header>`
- Net Sheet mobile sign-in: `<a href="index.html" class="msv-hub-back">← Fix & Flip Tools</a>` inside `#mobileSigninView` (positioned absolute top-left, header hidden via `body.mobile-signin`)
- Net Sheet mobile list: `<a href="index.html" class="mlv-hub">← All Tools</a>` inside `.mobile-list-nav` at the top of `#mobileListView` (header hidden via `body.mobile-list`)

---

## App Architecture — `walkthrough.html`

### Layout
Desktop (>1024px): two-column — resizable saved walkthroughs sidebar (`#wtSidebar`, 140–420px) | editor (`#editorView`, scrollable). Both live inside `#wtLayout` (flex). Sidebar shows walkthrough cards with active highlight, `+ New` button, and a sign-in prompt when not authenticated. Drag handle (`#wtResizer`) works identically to the Deal Analyzer resizer.

Mobile (≤1024px): same two-view JS-controlled pattern as before — `#listView` (list/sign-in) and `#wtLayout` (editor only, sidebar hidden via `body.editor-mode`). `#editorNav` (← My Walkthroughs | + New) shown on mobile editor, hidden on desktop.

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
1. **Interior & Kitchen** — toggle/count/sqft items. "Interior Demo" is the first item. Carpet and Subfloor repair are at the bottom.
2. **Bathrooms** — all `count` type (per-item qty, no global bath counter).
3. **Garage & Electric Panel** — Water Heater, Electric Panel Service/Replace, Garage Door, Insulation.
4. **Exterior & High Ticket Items** — toggle/count items (roof, AC, windows, etc.). Permit Allowance is last.
5. **Misc Expenses / General Photos** — `{ id: 'misc', miscSection: true, items: [] }` — freehand dollar expenses + general property photos.
6. **Property Comments** — free-text notes section, rendered after SECTIONS via `renderNotesSection()`.

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

### View management

Three views, switched by JS inline styles (same philosophy as Deal Analyzer):
- **List view** (`#listView`) — shown on load and after `showList()`. Mobile only. Contains `.wt-list-nav` (hub link + title + `+ New`) at the top, then `#wlvAuthRow` (email + Sign Out, signed-in only), then either `#listSignin` (not signed in) or `#listContent` (signed in). `body.list-mode` hides the main header.
- **Editor view** (`#wtLayout`) — shown by `showEditor()`. On desktop this is the full two-column layout (sidebar + editor). On mobile the sidebar is hidden via `body.editor-mode`. Main header reappears. `#editorNav` (← My Walkthroughs | + New) shown only on mobile.
- **Share view** (`#shareView`) — shown by `showShareView()` for `?share=<id>` URLs. Both `body.list-mode` and `body.editor-mode` are removed.

`showList()` and `showEditor()` both branch on `isMobile()`:

```js
function isMobile() { return window.innerWidth <= 1024; }

function showList() {
  // ...
  if (!isMobile()) {
    // Desktop: show layout with sidebar, call renderWtSidebar() + loadWalkthroughs()
    document.getElementById('wtLayout').style.display = 'flex';
    document.body.classList.remove('list-mode');
    return;
  }
  // Mobile: hide layout, show #listView, add body.list-mode
  document.body.classList.add('list-mode');
  if (currentUser) { /* show auth row */ } else { /* show #listSignin */ }
}

function showEditor() {
  // ...
  document.getElementById('wtLayout').style.display = isMobile() ? 'block' : 'flex';
  if (isMobile()) {
    document.getElementById('editorNav').style.display = 'flex';
    document.body.classList.add('editor-mode');    // hides sidebar on mobile
  } else {
    document.getElementById('editorNav').style.display = 'none';
    document.body.classList.remove('editor-mode'); // sidebar visible on desktop
  }
}
```

`renderWtSidebar(walks)` populates `#wtSidebarList` with clickable cards (active highlight on `activeId`). It shows `#wtSidebarAuth` (sign-in prompt) when not signed in. `_cachedWalks` stores the last fetched list so `newWalkthrough()` and `openWalkthrough()` can refresh the active highlight without a re-fetch. `loadWalkthroughs()` re-fetches, updates `_cachedWalks`, and calls both `renderList()` (mobile) and `renderWtSidebar()` (desktop).

### Auth

- Sign in / sign up via Supabase email+password
- Session stored in `localStorage` as `sb_session` (`{ access_token, refresh_token, user }`)
- Auth fetch pattern: `.then(r => r.json()).catch(() => ({}))` — never use AbortController timeout (breaks cold-start logins on Supabase free tier)
- No guest mode — saves require a signed-in account
- **Token refresh:** `refreshSession()` calls `POST /auth/v1/token?grant_type=refresh_token`. Called on boot if token verification fails (before kicking to sign-in) and every 45 minutes via `setInterval` while the user is active.
- Boot verifies the stored token against `/auth/v1/user` in the background. If expired: tries refresh; if refresh fails, clears session and re-renders list as signed-out.

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
- **List view** (`#mobileListView`) — custom nav bar (`.mobile-list-nav`: `← All Tools` | `Deal Analyzer` | `+ New`) + auth row (`#mlvAuthRow`: email + Sign Out) + deal cards. The main header is hidden via `body.mobile-list`. Shown when signed in on mobile.
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
  if (signedIn) {
    document.body.classList.remove('mobile-signin');
    document.body.classList.add('mobile-list');
    const authRow = document.getElementById('mlvAuthRow');
    authRow.style.display = 'flex';
    document.getElementById('mlvEmail').textContent = currentUser.email || '';
  } else {
    document.body.classList.add('mobile-signin');
    document.body.classList.remove('mobile-list');
  }
}

function showMobileEditor() {
  document.getElementById('mobileSigninView').style.display = 'none';
  document.getElementById('mobileListView').style.display   = 'none';
  document.querySelector('.layout').style.display = '';
  document.body.classList.remove('mobile-signin');
  document.body.classList.remove('mobile-list');
  document.body.classList.add('mobile-editor');
  window.scrollTo(0, 0);
}
```
`body.mobile-editor` is used as a CSS hook for editor-mode styling (single-column grid, sticky profit bar, etc.) but show/hide is always driven by JS inline styles, not CSS selectors. **Do not switch this back to a CSS-only approach** — it proved unreliable across browsers and screen sizes.

`body.mobile-signin` hides the header during the sign-in landing screen. `body.mobile-list` hides the header during the list view — the list view renders its own nav bar (`.mobile-list-nav`) inside `#mobileListView` instead. `showMobileEditor()` **must remove both classes** — failing to do so leaves the header hidden when entering the editor.

On init: `loadSession()` runs synchronously first, then `if (isMobile()) showMobileList()`, then `initAuth()` (async).

### Auth
Same Supabase email+password pattern as the walkthrough. Session in `localStorage` as `sb_session` (`{ access_token, refresh_token, user }`). Guest mode is supported in the Deal Analyzer (no local storage fallback — guest users just can't save). Signed-in users get cloud sync via the `deals` Supabase table.

**Token refresh:** `initAuth()` tries `refreshSession()` before clearing session on a 401. A 45-minute `setInterval` proactively refreshes the token while the user is active.

### Deal Data Flow
Inputs → `calculate()` → results panel updates live. Save → `openSaveModal()` → `confirmSave()` → Supabase. Load → `loadCloudDeal(id)` → `populateFields(inputs)` → `calculate()`.

---

## App Architecture — `net-sheet.html`

### Layout
Desktop (>1024px): two-column — resizable saved net sheets sidebar (`#nsSidebar`, CSS variable `--ns-sidebar-w: 400px`) | editor (`#nsEditorView`, scrollable). Both inside `.ns-layout` (flex). Sticky footer bar (`#nsFooter`) is `position: fixed; left: var(--ns-sidebar-w); right: 0` and updates via the resizer. Header `btn-save` and `btn-print` are hidden on desktop via `@media (min-width: 1025px)` — footer buttons are used instead.

Mobile (≤1024px): three-view JS-controlled pattern matching Deal Analyzer:
- **Sign-in view** (`#mobileSigninView`) — full-screen landing with Sign In / Continue as Guest.
- **List view** (`#mobileListView`) — nav bar (`.mobile-list-nav`: `← All Tools` | `Net Sheet` | `+ New`) + auth row + net sheet cards. Header hidden via `body.mobile-list`.
- **Editor view** (`.ns-layout`) — single-column form + `#mobileEditorNav` (← Net Sheets | + New) + fixed mobile footer with net proceeds + Save/Print.

### Mobile View Switching
Same JS pattern as Deal Analyzer — `showMobileList()` and `showMobileEditor()` control `element.style.display` inline and the `mobile-signin`, `mobile-list`, `mobile-editor` body classes. `showMobileEditor()` must remove all three classes before adding `mobile-editor` or the header stays hidden.

On init: `loadSession()` → `updateAuthUI()` → `if (isMobile()) showMobileList()` → `initAuth()` (async).

### Form State
All form inputs are read at calculate/save time via `collectState()`. No central state object — inputs live in the DOM. `populateForm(data)` restores all fields from saved `data` jsonb. `resetForm()` calls `populateForm({})` with default cost values.

Misc fee items (`miscItems[]`) are stored in memory as `{ id, label, amount }` and rendered by `renderMiscItems()`. They are serialized into `data.miscItems` on save.

### Costs & Calculation
`calculate()` reads all inputs, computes prorated tax, agent fees, and misc, then updates the results section, footer, and mobile footer. Returns `{ sp, lp, totalCosts, netProceeds }`.

- **Prorated tax:** `(city.rate / 100) × salePrice × (daysFromJan1 / daysInYear)` — city dropdown references `TAX_CITIES[]` array. **Cities and rates are stubbed — populate with real values before use.**
- **Owner's Title Policy:** manual input only — formula stub pending rate table from user.
- **Buyer agent fee:** three modes — `Seller Pays` (full fee from seller), `Buyer Pays` ($0 from seller), `Split` (seller pays their stated % of sale price directly). Type toggle: `%` of sale price or fixed `$`.
- **Seller agent fee:** `%` of sale price or fixed `$`.

### Auth
Same Supabase email+password pattern. Session in `sb_session`. Guest mode: shows editor but can't save. **Token refresh:** `refreshSession()` on boot if `/auth/v1/user` returns non-200, plus 45-min `setInterval`.

### Sharing
- `shared boolean` column. `enableShare(id)` patches `shared = true`, copies URL to clipboard. `disableShare(id)` patches `shared = false`.
- Share URL: `net-sheet.html?share=<UUID>`
- On load, if `?share=` present: hides layout/footer, fetches with anon key, calls `renderShareReport(sheet)` into `#shareView`.
- Share report recalculates all values from stored `data` jsonb (does not trust `net_proceeds` column for display, recomputes for accuracy).

### Pending wiring
- **`TAX_CITIES` array** — replace stubs with actual city names and annual % rates. Format: `{ name: 'CityName', rate: 1.25 }`. The `<select>` option `value` must match `city.name` exactly.
- **Owner's Title Policy formula** — user to provide rate table; replace manual input with calculated field.
- **Agent branding (v2)** — `profiles` table in Supabase; white-labeled PDF/share report with agent name, brokerage, logo.

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
- **`showMobileEditor()` must hide `#mobileSigninView`, remove `body.mobile-signin`, and remove `body.mobile-list`** — missing either class removal leaves the main header hidden when entering the editor. The `mobile-signin` omission was a prior bug (guest login didn't navigate into the app); `mobile-list` must also be cleared when opening a deal from the list.
- **Walkthrough `body.list-mode` hides the main header** — same pattern as `body.mobile-list` in the Deal Analyzer. `showEditor()` must remove it, or the header stays hidden in the editor. `showShareView()` must also remove it.
- **Walkthrough `body.editor-mode` hides the sidebar on mobile** — `body.editor-mode .wt-sidebar, body.editor-mode .wt-resizer { display: none }`. On desktop, `showEditor()` removes `editor-mode` so the sidebar stays visible. Never set `body.editor-mode` on desktop or the sidebar disappears.
- **All four files must use the custom Supabase URL** — always use `https://api.bluestarrealtygroup.com` in `index.html`, `walkthrough.html`, `deal-analyzer.html`, and `net-sheet.html`. Never revert to `qplidmfishaclysckruq.supabase.co`.
- **Walkthrough footer must use `display: flex` on desktop** — `showEditor()` sets `mainFooter.style.display = isMobile() ? 'block' : 'flex'`. Using `'block'` kills the flex layout and breaks `margin-left: auto` right-alignment of the buttons.
- **`renderAll()` removed from walkthrough boot** — sections are only rendered when a walkthrough is opened or created. Calling `renderAll()` on boot caused a flash of section dropdowns before the empty-state overlay appeared.
- **Walkthrough sidebar width: 480px; Deal Analyzer sidebar width: 446px** — CSS variables `--wt-sidebar-w` and `--da-sidebar-w` are updated by the resizer drag handler to keep the sticky footer `left` offset in sync.
- **Empty state overlays** — `#wtEmptyState` (walkthrough, inside `#editorView`) and `#daEmptyState` (deal analyzer, inside `.inputs-panel`) are shown when no walkthrough/deal is open on desktop. Controlled by `_editorActive` / `_daEditorActive` flags. DA also hides `.results-panel` via `visibility: hidden` (not `display: none`, to preserve `position: sticky`).
- **Deal Analyzer desktop footer** — `.da-footer` is a fixed bottom bar (`left: var(--da-sidebar-w); right: 0`) containing Expected Profit (left) and Save Deal + Export PDF buttons (right). On desktop, `header .btn-save` and `header .btn-pdf` are hidden with `display: none !important` — scoped to `header` to avoid hiding the footer buttons.
- **Deal Analyzer `+ New` button** — lives in the sidebar header (`.da-sidebar-hdr`) on desktop, not in the main header.
- **Net Sheet `TAX_CITIES` and title policy are stubs** — `TAX_CITIES[]` in `net-sheet.html` contains placeholder entries. Prorated tax will calculate $0 until real city names and rates are populated. Owner's Title Policy is a manual input until the formula rate table is provided.
- **Net Sheet `showMobileEditor()` must remove `mobile-signin` and `mobile-list`** — same pattern as Deal Analyzer. Missing either removal leaves the header hidden in the editor.
- **Net Sheet sidebar CSS variable `--ns-sidebar-w`** — updated by the resizer drag handler to keep the sticky footer `left` offset in sync. Default: 400px.
