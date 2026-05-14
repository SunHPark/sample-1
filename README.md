# FNDM Landing Page

K-POP photocard trading platform — pre-launch landing page.
**Frontend complete. Backend integration pending.**

---

## 📁 File Structure

```
fndm-site/
├── index.html              # Main landing page
├── legal.html              # Privacy Policy + Terms of Use (5 languages)
├── fndm-logo.png           # Red FNDM logo (transparent BG) — used in nav
├── fndm-logo-black.png     # Black FNDM logo (transparent BG) — used in legal page hero
├── og-image.png            # Open Graph image (1200x630)
├── favicon.ico             # Multi-size favicon (16/32/48)
├── favicon-32.png          # 32x32 favicon
├── favicon-192.png         # 192x192 (Apple touch icon)
├── favicon-512.png         # 512x512 (PWA)
├── photocard-1.jpg         # Demo card image
├── photocard-2.jpg         # Demo card image
└── photocard-3.jpg         # Demo card image
```

---

## 🌐 Internationalization (i18n)

Site supports **5 languages**: KOR, EN, JP, CN, Esp
- Browser language auto-detected on first visit
- User selection saved to `localStorage` as `fndm-locale`
- Same locale applies across `index.html` and `legal.html`

All UI text lives in the `I18N` object in `index.html` (search for `const I18N`).
Each language has the same key structure.

---

## 🔧 Backend Integration TODO

The signup form currently shows a static success screen on submit.
**No data is being saved.** Backend dev needs to wire this up.

### 1. Form submission endpoint

**Form location:** `index.html` line ~2710 (`<form id="signup-form">`)

**Current submit handler:** `function submitForm(e)` in the bottom `<script>` block.

**Fields collected:**
| Name | ID | Required | Notes |
|------|----|----|-------|
| `email` | `#email` | ✅ | Validated with regex |
| `artist` | `#artist` | ✅ | Dropdown with K-POP groups + "other" option |
| `artist_other` | `#artist-other` | conditional | Visible only when `artist === 'other'` |
| `country` | `#country` | ✅ | 25 countries with flag emojis |
| `tier` | `[name="tier"]` | ✅ (≥1) | Checkbox group: `preorder` and/or `beta` |
| `agree` | `#agree` | ✅ | Consent checkbox |

**Where to hook backend call:**
After the validation block in `submitForm()`, replace this:
```js
form.style.display = 'none';
document.getElementById('form-success').hidden = false;
```
with a `fetch('/api/register', {...})` call, then handle the response.

### 2. Required server-side logic

**A. Beta tester cap (11,000)**
- Frontend counter (`#betaCount` in `index.html`) is currently a static `75`.
- Frontend already has `checkBetaCap()` that auto-disables the beta checkbox when count ≥ 11,000.
- To make it real: backend should expose the current beta count, and frontend should fetch it on page load and update `#betaCount`.
- When 11,001st+ user tries to register for beta:
  - Server rejects beta selection, accepts pre-register only
  - Returns error code `BETA_CAP_REACHED`
  - Frontend should show: "베타테스터 모집이 마감되었습니다." (already has translations — see `form.type.beta.soldout` key in I18N)

**B. Duplicate email check**
- Before saving, check if email already exists in DB.
- If duplicate → return error code `EMAIL_EXISTS`
- Frontend should show: **"이미 팬덤과 매칭이 되신 유저입니다"** message
  - **Translations needed in i18n** (not yet added — add to all 5 langs):
    - KOR: `이미 팬덤과 매칭이 되신 유저입니다. FNDM 론칭 소식을 가장 먼저 받게 됩니다.`
    - EN: `You're already matched with the fandom. We'll be the first to send you FNDM launch news.`
    - JP: `すでにファンダムとマッチング済みのユーザーです。FNDMローンチの最新情報を最初にお届けします。`
    - CN: `您已经与粉丝群匹配。FNDM 上线消息将第一时间送达。`
    - Esp: `Ya estás matcheado con el fandom. Serás el primero en recibir las novedades de FNDM.`

### 3. Suggested API contract

```
POST /api/register
Content-Type: application/json

Request body:
{
  "email": "user@example.com",
  "artist": "NewJeans",
  "artist_other": "",           // only if artist === "other"
  "country": "KR",
  "tier": ["preorder", "beta"], // array
  "locale": "KOR"               // currently selected UI language
}

Response (success):
{
  "ok": true,
  "stats": {
    "preorder_count": 159,
    "beta_count": 76
  }
}

Response (email exists):
{
  "ok": false,
  "error": "EMAIL_EXISTS"
}

Response (beta cap reached):
{
  "ok": false,
  "error": "BETA_CAP_REACHED",
  "accepted_tier": ["preorder"],   // server saved pre-register only
  "stats": {
    "preorder_count": 159,
    "beta_count": 11000
  }
}

Response (server error):
{
  "ok": false,
  "error": "SERVER_ERROR"
}
```

### 4. Counter updates

Two counters in the UI need real-time data:

**A. Pre-register card** (`index.html` line ~2200):
```html
<div class="stat-value">158</div>           <!-- preorder count -->
<div class="stat-value" id="betaCount">75</div>  <!-- beta count -->
```

**B. Proof section** (`index.html` line ~2480):
```html
<div class="proof-stat-num">158<span class="unit">명</span></div>
<div class="proof-stat-num">75<span class="unit">명</span></div>
```

Backend should expose `GET /api/stats` returning `{ preorder_count, beta_count }`, frontend should fetch on page load and update these.

---

## 🟢 Supabase Setup Guide (Free Tier)

The backend will be built with **Supabase** — this section explains how to set up the registration system using the free tier.

### Free tier limits (more than enough for pre-launch)
- **500 MB** Postgres database
- **50,000** monthly active users
- **5 GB** bandwidth/month
- **500K** Edge Function invocations/month
- Unlimited API requests

For a pre-launch signup form, this is far more than enough.

### Step 1 — Create project at https://supabase.com
1. Sign up (free, no credit card)
2. New project → choose region close to KR (Tokyo or Seoul)
3. Set database password (save it)
4. Wait ~2 minutes for provisioning

### Step 2 — Create the `registrations` table

In Supabase SQL Editor:

```sql
create table registrations (
  id              uuid primary key default gen_random_uuid(),
  email           text not null unique,
  artist          text not null,
  artist_other    text,
  country         text not null,
  tier            text[] not null,        -- ['preorder'] or ['preorder', 'beta']
  locale          text not null default 'KOR',
  created_at      timestamptz default now()
);

create index registrations_email_idx on registrations (email);
create index registrations_tier_idx on registrations using gin (tier);
```

### Step 3 — Row Level Security (RLS) policies

```sql
alter table registrations enable row level security;

-- Anyone can INSERT (anonymous signups allowed)
create policy "anon can insert" on registrations
  for insert
  to anon
  with check (true);

-- Anyone can SELECT count only (for live counter)
-- Restrict full row visibility — use RPC functions for counts instead
```

### Step 4 — Counter functions

Expose counts safely without leaking emails:

```sql
create or replace function get_registration_stats()
returns json
language plpgsql
security definer
as $$
declare
  preorder_n int;
  beta_n int;
begin
  select count(*) into preorder_n from registrations where 'preorder' = any(tier);
  select count(*) into beta_n from registrations where 'beta' = any(tier);
  return json_build_object(
    'preorder_count', preorder_n,
    'beta_count', beta_n
  );
end;
$$;

grant execute on function get_registration_stats() to anon;
```

### Step 5 — Registration function with email + cap checks

```sql
create or replace function register_user(
  p_email text,
  p_artist text,
  p_artist_other text,
  p_country text,
  p_tier text[],
  p_locale text
)
returns json
language plpgsql
security definer
as $$
declare
  existing_id uuid;
  beta_n int;
  accepted_tier text[];
  preorder_n int;
begin
  -- Check duplicate email
  select id into existing_id from registrations where email = p_email;
  if existing_id is not null then
    return json_build_object('ok', false, 'error', 'EMAIL_EXISTS');
  end if;

  -- Check beta cap (11,000)
  accepted_tier := p_tier;
  if 'beta' = any(p_tier) then
    select count(*) into beta_n from registrations where 'beta' = any(tier);
    if beta_n >= 11000 then
      -- Strip 'beta' from tier; keep preorder if present
      accepted_tier := array_remove(p_tier, 'beta');
      if array_length(accepted_tier, 1) is null then
        accepted_tier := array['preorder'];
      end if;
    end if;
  end if;

  -- Insert
  insert into registrations (email, artist, artist_other, country, tier, locale)
  values (p_email, p_artist, p_artist_other, p_country, accepted_tier, p_locale);

  -- Return stats + whether beta was rejected
  select count(*) into preorder_n from registrations where 'preorder' = any(tier);
  select count(*) into beta_n from registrations where 'beta' = any(tier);

  if 'beta' = any(p_tier) and not ('beta' = any(accepted_tier)) then
    return json_build_object(
      'ok', false,
      'error', 'BETA_CAP_REACHED',
      'accepted_tier', accepted_tier,
      'stats', json_build_object('preorder_count', preorder_n, 'beta_count', beta_n)
    );
  end if;

  return json_build_object(
    'ok', true,
    'stats', json_build_object('preorder_count', preorder_n, 'beta_count', beta_n)
  );
end;
$$;

grant execute on function register_user(text, text, text, text, text[], text) to anon;
```

### Step 6 — Frontend integration (in index.html)

Add Supabase client to `<head>`:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

In the bottom `<script>`, initialize and wire up the form:

```js
const supabase = supabase.createClient(
  'https://YOUR_PROJECT.supabase.co',
  'YOUR_ANON_PUBLIC_KEY'   // safe to expose — RLS protects data
);

// Update form submission
async function submitForm(e) {
  e.preventDefault();
  // ... existing validation code stays ...

  const dict = (typeof I18N !== 'undefined') ? I18N[currentLocale] : null;

  const payload = {
    p_email: emailInput.value.trim(),
    p_artist: form.querySelector('#artist').value,
    p_artist_other: form.querySelector('#artist-other').value || null,
    p_country: form.querySelector('#country').value,
    p_tier: Array.from(form.querySelectorAll('[name="tier"]:checked')).map(c => c.value),
    p_locale: currentLocale
  };

  const { data, error } = await supabase.rpc('register_user', payload);

  if (error) {
    showFormToast((dict && dict['form.error.server']) || 'Something went wrong. Please try again.');
    return false;
  }

  if (!data.ok) {
    if (data.error === 'EMAIL_EXISTS') {
      showFormToast((dict && dict['form.error.dup']) || 'You\'re already matched with the fandom.');
      return false;
    }
    if (data.error === 'BETA_CAP_REACHED') {
      // Saved as pre-register only; show success with note
      showFormToast((dict && dict['form.error.betacap']) || 'Beta is closed — registered for pre-launch instead.');
      // Continue to success screen
    }
  }

  // Update counters
  if (data.stats) {
    document.getElementById('betaCount').textContent = data.stats.beta_count;
    // Update preorder counters too
  }

  form.style.display = 'none';
  document.getElementById('form-success').hidden = false;
  return false;
}

// On page load, fetch live stats
(async () => {
  const { data } = await supabase.rpc('get_registration_stats');
  if (data) {
    document.getElementById('betaCount').textContent = data.beta_count;
    // Update preorder counter too
    checkBetaCap();   // re-check cap with real number
  }
})();
```

### Step 7 — i18n keys to add (5 langs)

Add to all 5 locale blocks in `I18N`:

```js
'form.error.dup': '이미 팬덤과 매칭이 되신 유저입니다.',     // KOR
'form.error.betacap': '베타 모집이 마감되어 사전예약으로 등록되었습니다.',
'form.error.server': '일시적인 오류가 발생했어요. 다시 시도해주세요.',
```

(Translations for EN/JP/CN/Esp follow the same pattern — backend dev can ask if needed.)

### Step 8 — Email notifications (optional)

Use Supabase Database Webhooks → trigger when new row inserted to `registrations` → call Slack/email webhook. Or set up Supabase Auth Email feature for confirmation emails.

### Notes
- **Anon key is safe to expose in frontend** — RLS + RPC functions prevent unauthorized data access
- Service Role Key must **never** be in frontend code
- For admin dashboard (viewing all registrations): use Supabase Studio (built-in) or build a separate authenticated page
- Free tier auto-pauses after 1 week of inactivity — pings the API endpoint daily via cron-job.org keeps it warm

---

## 🎨 Design Tokens

```css
--bg: #FFFFFF;
--ink: #0A0E1A;        /* primary text */
--ink-3: #5A6172;       /* secondary text */
--primary: #FF1F3D;     /* FNDM red */
```

Fonts: **Inter** (display, English), **Pretendard** (display+body, Korean), **JetBrains Mono** (mono labels).

---

## 📧 Contact / Brand

- Company: **CultureCode Inc.** (㈜컬처코드)
- Business reg: **158-87-03985**
- Email: `fndm@culturecode.co.kr`
- Personal Information Manager: 임형중 (Hyungjoong Lim)
- Web: https://www.culturecode.co.kr
- Target domain: **www.thefndm.com**

---

## 🚀 Deployment Notes

1. **Static deployment** — works on Vercel, Netlify, Cloudflare Pages out of the box.
2. **Custom domain** — OG meta tags point to `https://www.thefndm.com/og-image.png`. After domain connection, social previews (KakaoTalk, X, Facebook) will work automatically. If using a Vercel preview URL for testing, update the 3 og:image URLs in `<head>` of `index.html` to that URL.
3. **No build step** — pure HTML/CSS/JS, no bundler needed.
4. **MIME type for `.ico`** — make sure your host serves `image/x-icon`. Vercel handles this automatically.

### Recommended cache headers (Vercel `vercel.json`)
```json
{
  "headers": [
    {
      "source": "/(.*).(png|jpg|jpeg|ico|svg)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    },
    {
      "source": "/(.*).(html)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=0, must-revalidate" }
      ]
    }
  ]
}
```

---

## 📋 Page Sections (index.html)

1. **NAV** — sticky FNDM logo + language switcher (gradient fade)
2. **HERO** — "포카 교환, 이제 그만 기다려요."
3. **AUTO SWIPE DEMO** — iPhone mockup with auto-swiping cards + "매칭되었습니다!" banner
4. **PRE-REGISTER CARD** — counters (158/75) + 2 action buttons
5. **PROBLEM** — 4 pain point cards
6. **HOW IT WORKS** — 3-step (scan / swipe / safe check)
7. **SOLUTION** — 4 axis cards
8. **TRADE LOOP** — Globe with city dots (Seoul, Tokyo, Shanghai, Dubai, LA, NY, Paris, Jakarta, Sydney, São Paulo)
9. **PROOF** — Stat cards + 9-chip interests carousel (draggable horizontal scroll)
10. **APP NAV PREVIEW** — 5 boxes (CONTENT / SHOP / TRADE-NEW / EVENTS / COMMUNITY)
11. **CTA / FORM** — Signup form
12. **FOOTER** — Socials + tagline + legal links + copyright

## 📋 Legal Page (legal.html)

- Sticky nav with red FNDM logo + back link
- Big black FNDM logo + "For the Real Ones." headline
- Language switcher (5 langs)
- Tab switcher (Privacy ↔ Terms)
- URL hash for direct linking: `#privacy` or `#terms`
- Same footer as main page
- Business registration number included

---

## ✅ What's Already Working

- All UI, animations, transitions
- 5-language i18n with browser auto-detect
- Form client-side validation (email format, tier selection, consent)
- Custom centered toast for form errors (replaces native `alert()`)
- Page-to-page fade transitions
- Mailto link with auto-localized subject
- Share link copy-to-clipboard
- "Other" artist write-in field
- Country selector (25 countries)
- Beta cap check (frontend-only, ready for backend)
- SEO meta tags + Open Graph + Twitter Card + JSON-LD
- Hreflang for 5 languages
- Mobile responsive (all sections)
- Light-mode forced (no dark mode override)

## ⏳ What's Pending (Backend)

- Real form submission (currently shows success screen statically)
- Email duplicate detection
- Beta tester cap enforcement (currently visual only)
- Counter live updates
- Email notifications to admin

---

Last updated: May 14, 2026
