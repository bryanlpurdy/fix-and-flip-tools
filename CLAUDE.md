# Fix & Flip Tools — CLAUDE.md

## Project Overview

A mobile-first web app for fix-and-flip real estate investors. Currently one tool: a **Property Walkthrough estimator** that lets the user walk a property, toggle repair items, adjust costs, attach photos, and save estimates to the cloud.

Owner/user: Bryan Purdy (bryan.lee.purdy@gmail.com)

---

## Tech Stack

- **Vanilla HTML/CSS/JS — single file per tool, no build step, no framework**
- `walkthrough.html` — the entire walkthrough app (HTML + CSS + JS in one file)
- `index.html` — homepage / tool launcher
- **Supabase** backend via direct REST API (not the JS SDK):
  - Auth: `/auth/v1/token` and `/auth/v1/signup`
  - Database: `/rest/v1/walkthroughs` (PostgREST)
  - Storage: `/storage/v1/object/walkthrough-photos/{path}`

---

## Supabase Configuration

```
Project URL: https://qplidmfishaclysckruq.supabase.co
Anon public key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InFwbGlkbWZpc2hhY2x5c2NrcnVxIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzQxODc2ODMsImV4cCI6MjA4OTc2MzY4M30.YEhxwrw-HxofmV4N3KRpbgDcT_a_OyqIkx49Wu7CjR0
```

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
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
alter table walkthroughs enable row level security;
create policy "Users manage own walkthroughs" on walkthroughs
  for all using (auth.uid() = user_id);
```

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
- **Workflow:** edit locally → `git add` → `git commit` → `git push` → GitHub Pages redeploys

There is no CI, no build step, no package.json. Push and it's live.

---

## App Architecture — `walkthrough.html`

### State

```js
const state = {
  propertyName: '',
  items: {},          // keyed by item id: { enabled, qty, sqft, cost, photos: [] }
  misc: { expenses: [], photos: [] }
};
```

Misc expenses/photos are saved into the `items` JSONB column under the special key `_misc_`:
```js
// saving:  items: { ...state.items, _misc_: state.misc }
// loading: if (wt.items._misc_) state.misc = wt.items._misc_;
```

### Sections (SECTIONS config)

1. **Exterior & High Ticket Systems** — toggle/count items (roof, AC, electric, windows, etc.). First item is "Exterior Demo".
2. **Interior & Kitchen** — toggle/count/sqft items. "Interior Demo" is the first item. Ceiling Fan and Light Fixture live here.
3. **Bathrooms** — all `count` type (per-item qty, no global bath counter).
4. **Misc Expenses / General Photos** — `{ id: 'misc', miscSection: true, items: [] }` — freehand dollar expenses + general property photos.

### Item types

- `toggle` — on/off with adjustable cost
- `count` — integer quantity with per-unit cost
- `sqft` — square footage with per-sqft rate

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

## Known Gotchas

- **Supabase free tier pauses** inactive projects. If DNS stops resolving (`ERR_NAME_NOT_RESOLVED`), the project needs to be manually restarted in the Supabase dashboard. It can take a few minutes to become healthy again.
- **No AbortController** on auth fetches — was tried, broke logins during Supabase cold start.
- **Signed URL path prefix** — Supabase Storage sign endpoint omits `/storage/v1` from the returned path. Fixed with the conditional prepend above.
- The `anon` key is intentionally in client-side code — it's the Supabase public anon key, not a secret. RLS policies are the security boundary.
