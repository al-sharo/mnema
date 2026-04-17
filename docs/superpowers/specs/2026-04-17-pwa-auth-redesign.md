# PWA Auth Redesign — Design Spec
Date: 2026-04-17

## Problem

iOS treats Safari browser and PWA standalone mode as completely isolated WebKit contexts with separate `localStorage`. The current magic-link auth stores the Supabase session in Safari's `localStorage` after the link is tapped; the PWA's `localStorage` is always empty, so the user is forced to log in on every app open.

The same isolation affects Google OAuth (also redirects through Safari).

## Root Cause Fix

Swap Supabase's session storage from `localStorage` to **cookies**. Cookies are shared between Safari and PWA standalone mode on the same origin at the iOS/WebKit OS level. A session written in Safari after OAuth or any auth flow is immediately readable by the PWA.

No other architectural change is required. All existing sync logic (`onSignedIn`, `onAuthStateChange`, `getSession`, `syncProgressToCloud`, etc.) is untouched.

## Scope

All changes are inside `index.html`. No new files, no build step, no new dependencies.

---

## Implementation

### 1. Cookie storage adapter

Supabase JS SDK v2 accepts a custom `storage` object on `createClient`. Replace the default (`localStorage`) with a cookie-backed adapter.

```js
const cookieStorage = {
  getItem(key) {
    const match = document.cookie.match(
      new RegExp('(?:^|; )' + key.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + '=([^;]*)')
    );
    return match ? decodeURIComponent(match[1]) : null;
  },
  setItem(key, value) {
    const maxAge = 60 * 60 * 24 * 365; // 1 year
    document.cookie = `${key}=${encodeURIComponent(value)}; max-age=${maxAge}; path=/; SameSite=Lax; Secure`;
  },
  removeItem(key) {
    document.cookie = `${key}=; max-age=0; path=/`;
  }
};

const supa = supabase.createClient(SUPA_URL, SUPA_KEY, {
  auth: {
    storage: cookieStorage,
    persistSession: true,
    autoRefreshToken: true,
    detectSessionInUrl: true,
  }
});
```

**Size budget:** A Supabase v2 session JSON (access token ~700 chars + refresh token ~50 chars + user object ~400 chars) URI-encoded is approximately 1.5–2 KB — within the 4 KB per-cookie browser limit.

**`SameSite=Lax`** (not `Strict`): Lax allows the cookie to be readable on same-origin navigations arriving from external redirects (e.g. Google OAuth callback). Strict would work for reading via `document.cookie` in JS (SameSite only restricts cookies sent in HTTP request headers, not JS reads), but Lax is the conventional safe choice here.

**`detectSessionInUrl: true`**: Tells the Supabase SDK to automatically extract and process `#access_token` / `?code=` from the URL on page load (needed for the Google OAuth callback landing in Safari).

### 2. Google OAuth

```js
async function signInWithGoogle() {
  const { error } = await supa.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo: 'https://al-sharo.github.io/greek-cards',
    }
  });
  if (error) showLoginError('Ошибка: ' + error.message);
}
```

**iOS flow:**
1. Tapping the button in PWA (standalone) causes iOS to open Safari for the OAuth URL.
2. Google auth completes; Supabase redirects to `https://al-sharo.github.io/greek-cards#access_token=...`.
3. The page loads in Safari; `detectSessionInUrl: true` causes the Supabase SDK to process the hash and call `cookieStorage.setItem()`.
4. `onAuthStateChange` fires with `SIGNED_IN`.
5. We detect `!window.navigator.standalone` — we are in Safari, not the PWA — and render a **"return to home screen" interstitial** instead of calling `onSignedIn()`.
6. User switches back to home screen, taps app icon. PWA loads; `getSession()` reads the cookie → session found → `onSignedIn()` called → home screen shown.

**Supabase dashboard requirement:** Google provider must be enabled in Authentication → Providers → Google, with the OAuth app credentials set. The redirect URL `https://al-sharo.github.io/greek-cards` must be added to the allowed redirect URLs list.

### 3. Email OTP (6-digit code)

```js
// Step 1: send code
async function sendEmailOtp(email) {
  const { error } = await supa.auth.signInWithOtp({
    email,
    options: { shouldCreateUser: true }
    // No emailRedirectTo — suppresses magic link, sends 6-digit code only
  });
  if (!error) showOtpCodePanel(email);
  else showLoginError('Ошибка: ' + error.message);
}

// Step 2: verify code
async function verifyEmailOtp(email, token) {
  const { error } = await supa.auth.verifyOtp({
    email,
    token,
    type: 'email'
  });
  if (error) showLoginError('Неверный код');
  // onAuthStateChange handles the rest
}
```

**Note:** When `signInWithOtp` is called without `emailRedirectTo`, Supabase sends a 6-digit code email (not a link). The code is valid for 1 hour.

The entire flow stays inside the PWA. No Safari. Session stored via cookie on `verifyOtp` success.

### 4. Standalone detection

```js
const isStandalone = window.navigator.standalone === true;
```

Used in `onAuthStateChange`:
- If `SIGNED_IN` fires and `isStandalone` is false → show "return to home screen" screen, do NOT call `onSignedIn()`.
- If `isStandalone` is true → call `onSignedIn()` as normal.

This prevents the full app from rendering in Safari after the OAuth redirect while still completing the session storage.

### 5. Login screen HTML redesign

Replace the current single-step magic link form. Two sections separated by a divider:

```
┌──────────────────────────────────────┐
│  [ G ]  Войти через Google           │  ← full-width, Google-branded button
│                                      │
│  ————————————  или  ————————————     │
│                                      │
│  [ email@example.com              ]  │
│  [ Отправить код →                ]  │
│                                      │
│  (code panel, hidden by default)     │
│  [ _ ] [ _ ] [ _ ] [ _ ] [ _ ] [ _ ]│
│  [ Подтвердить →                  ]  │
│  Отправить повторно через 45с        │
└──────────────────────────────────────┘
```

The code panel is revealed (CSS class toggle) after `sendEmailOtp` succeeds. No screen navigation — same `#login` screen.

A separate `#oauth-return` screen is added for the "return to home screen" interstitial shown in Safari after Google OAuth.

### 6. What is removed

- `sendMagicLink()` function
- The single email input + "Send link" button UI
- `emailRedirectTo: window.location.href` in `signInWithOtp`

### 7. What is unchanged

- All Supabase table schemas (`progress`, `custom_words`)
- `onSignedIn(user)`, `signOut()`, `syncProgressToCloud()`, `loadProgressFromCloud()`, `loadCustomWordsFromCloud()`
- `onAuthStateChange` subscription structure
- `getSession()` on load
- All screens except `#login` (which is redesigned, not replaced)
- All CSS variables, fonts, and design tokens
- `sw.js`, `manifest.json`

---

## Supabase Dashboard Checklist

Before deploying:
- [ ] Enable Google provider in Authentication → Providers
- [ ] Add Google OAuth client ID and secret
- [ ] Add `https://al-sharo.github.io/greek-cards` to allowed redirect URLs
- [ ] Confirm Email OTP is enabled (it is by default; just ensure "Confirm email" or "OTP" template is active)

---

## Out of Scope

- Offline OTP queueing (not needed; OTP requires network by definition)
- Session token size optimization (cookies accommodate current session size)
- Any changes to SRS logic, vocabulary, or sync behaviour
