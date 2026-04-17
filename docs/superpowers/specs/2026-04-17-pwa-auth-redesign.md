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
    // Must match all attributes of the original Set-Cookie or deletion silently fails on iOS Safari
    document.cookie = `${key}=; max-age=0; path=/; SameSite=Lax; Secure`;
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

**Cookie key namespace:** The Supabase SDK writes under `sb-<project-ref>-auth-token` (e.g. `sb-oqubpyoceimwwputhtei-auth-token`). The app's existing localStorage keys (`greek_cards_progress`, `greek_custom_words`) are in `localStorage` and will never collide with cookie keys.

**`SameSite=Lax`** is the safe default. `Strict` would also work here because all session reads happen via `document.cookie` in JS client-side code (SameSite only restricts cookies sent in HTTP request headers, not JS reads). `Lax` is chosen because it is the conventional standard and avoids any edge-case surprises in future browser updates.

**`detectSessionInUrl: true`**: Tells the Supabase SDK to automatically extract and process `#access_token` / `?code=` from the URL on page load. Required for the Google OAuth callback landing in Safari.

### 2. Google OAuth

```js
async function signInWithGoogle() {
  // signInWithOAuth internally navigates via window.location.href (not window.open),
  // which causes iOS to open the OAuth URL in Safari rather than an ephemeral in-app browser.
  // The session cookie set in Safari after the redirect is then shared with the PWA.
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
1. Tapping the button in PWA (standalone) causes iOS to open Safari for the OAuth URL (Supabase SDK uses `window.location.href` navigation, not `window.open`, so the full Safari session is used and the resulting cookie is in the shared cookie store, not an ephemeral context).
2. Google auth completes; Supabase redirects to `https://al-sharo.github.io/greek-cards#access_token=...`.
3. The page loads in Safari; `detectSessionInUrl: true` causes the Supabase SDK to process the hash and call `cookieStorage.setItem()`.
4. `onAuthStateChange` fires with `SIGNED_IN`.
5. We detect `!window.navigator.standalone` — we are in Safari, not the PWA — and render a **"return to home screen" interstitial** instead of calling `onSignedIn()`. (See standalone detection, section 4.)
6. User switches back to home screen, taps app icon. PWA loads; `getSession()` reads the cookie → session found → `onSignedIn()` called → home screen shown.

**`window.navigator.standalone`** is an iOS Safari-specific boolean. It is `true` when the page is running as an installed PWA (added to home screen, opened from there), `false` in Safari browser, and `undefined` on non-iOS platforms. The code uses `!window.navigator.standalone` (truthy check), which correctly blocks `onSignedIn()` on Safari (`false`) and non-iOS platforms (`undefined`) — both are the right behaviour for this iOS-only app.

**Supabase dashboard requirement:** Google provider must be enabled in Authentication → Providers → Google, with OAuth client credentials set. The redirect URL `https://al-sharo.github.io/greek-cards` must be in the allowed redirect URLs list.

### 3. Email OTP (6-digit code)

**Dashboard prerequisite:** In Supabase → Authentication → Providers → Email, ensure "Enable Email OTP" is on and the active email template sends a code (not a magic link). If the project currently uses magic link templates, switch the template type to OTP before deploying. Do not leave both templates active simultaneously, as the user may receive both a code and a link in the same email.

```js
// Step 1: send code
async function sendEmailOtp(email) {
  // Omitting emailRedirectTo causes Supabase to send a 6-digit OTP code
  // rather than a magic link — but ONLY if the Email OTP template is active
  // in the Supabase dashboard (see dashboard prerequisite above).
  const { error } = await supa.auth.signInWithOtp({
    email,
    options: { shouldCreateUser: true }
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
  // onAuthStateChange handles the rest — no explicit onSignedIn() call needed here
}
```

The code is valid for 1 hour. The entire flow stays inside the PWA — no Safari involvement.

### 4. Standalone detection and deduplication of `onSignedIn`

The current code calls `onSignedIn()` from two places on startup: the `getSession()` IIFE and `onAuthStateChange`. This double-fire is a pre-existing issue and must be fixed as part of this redesign since we are rewriting the auth handler.

**Fix:** Use a flag to prevent double invocation, and add the standalone gate for the OAuth interstitial case.

```js
let sessionHandled = false;

supa.auth.onAuthStateChange(async (event, session) => {
  if (session?.user) {
    if (!window.navigator.standalone) {
      // In Safari after OAuth redirect — show interstitial, don't enter the app
      showScreen('oauth-return');
      return;
    }
    if (!sessionHandled) {
      sessionHandled = true;
      await onSignedIn(session.user);
    }
  } else {
    sessionHandled = false;
    currentUser = null;
    showScreen('login');
  }
});

(async () => {
  const { data: { session } } = await supa.auth.getSession();
  if (session?.user) {
    if (!window.navigator.standalone) {
      showScreen('oauth-return');
      return;
    }
    if (!sessionHandled) {
      sessionHandled = true;
      await onSignedIn(session.user);
    }
  } else {
    showScreen('login');
  }
})();
```

**`sessionHandled` flag:** Set to `true` by whichever path (IIFE or `onAuthStateChange`) calls `onSignedIn()` first. The second path sees the flag and skips. Reset to `false` on sign-out so the next login cycle works correctly.

### 5. `localStorage` gap on first PWA open after Safari OAuth

`Store` (SRS progress) and `CustomWords` still use `localStorage`. iOS isolates `localStorage` between Safari and PWA. Therefore, on the first PWA launch after a Google OAuth that completed in Safari, the PWA's `localStorage` will be empty. This is not a bug: `onSignedIn()` calls `loadProgressFromCloud()` and `loadCustomWordsFromCloud()`, which repopulate `localStorage` from Supabase on every sign-in. The user will see a brief "↓ загружаем прогресс..." sync indicator. This is acceptable and consistent with the existing sync behaviour.

No code change is needed for this gap. It is documented here so the implementer is not surprised during testing.

### 6. Login screen HTML redesign

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

The code panel is revealed (CSS class toggle) after `sendEmailOtp` succeeds. No screen navigation — same `#login` screen, two states.

A separate `#oauth-return` screen is added (simple, full-screen message) for the interstitial shown in Safari after Google OAuth.

### 7. What is removed

- `sendMagicLink()` function
- The single email input + "Send link" button UI
- `emailRedirectTo: window.location.href` in `signInWithOtp`

### 8. What is unchanged

- All Supabase table schemas (`progress`, `custom_words`)
- `onSignedIn(user)`, `signOut()`, `syncProgressToCloud()`, `loadProgressFromCloud()`, `loadCustomWordsFromCloud()`
- All screens except `#login` (redesigned) and the new `#oauth-return` addition
- All CSS variables, fonts, and design tokens
- `sw.js`, `manifest.json`

---

## Supabase Dashboard Checklist

Before deploying:
- [ ] Enable Google provider: Authentication → Providers → Google
- [ ] Add Google OAuth client ID and secret from Google Cloud Console
- [ ] Add `https://al-sharo.github.io/greek-cards` to allowed redirect URLs (Authentication → URL Configuration)
- [ ] Confirm Email OTP template is active (Authentication → Email Templates) — must be "OTP" type, not "Magic Link"
- [ ] Disable or replace the Magic Link email template so users don't receive both a code and a link
- [ ] Note: Supabase free tier rate-limits email to ~3–4 sends/hour per address — test accordingly

---

## Out of Scope

- Offline OTP queueing (not needed; OTP requires network by definition)
- Session token size optimization (cookies accommodate current session size)
- Any changes to SRS logic, vocabulary, or sync behaviour
