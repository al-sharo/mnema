# Email OTP Auth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace magic-link auth with Email OTP (6-digit code) and cookie-based session storage so the PWA stays logged in across opens on iPhone.

**Architecture:** All changes in `index.html`. Cookie storage adapter is passed to `supabase.createClient` so the session is stored in cookies (shared between Safari and PWA standalone mode on iOS) rather than `localStorage` (which is isolated per WebKit context). Login screen becomes a two-panel form: email input → 6-digit code entry. A `sessionHandled` flag prevents the double-fire of `onSignedIn()` that occurs when both the startup IIFE and `onAuthStateChange` fire simultaneously.

**Tech Stack:** Vanilla JS, Supabase JS SDK v2 (CDN), no build step. All verification is manual in-browser.

**Spec:** `docs/superpowers/specs/2026-04-17-pwa-auth-redesign.md`

---

## Pre-flight: Supabase dashboard

Before touching any code, verify these settings in the Supabase dashboard for project `oqubpyoceimwwputhtei`:

- [ ] Authentication → Providers → Email → **Enable Email OTP** is ON
- [ ] Authentication → Email Templates → active template type is **OTP** (not Magic Link) — if it is Magic Link, switch it to OTP
- [ ] Note the free-tier rate limit: ~3–4 OTP emails per hour per address. Use a real email during testing.

---

## Task 1: Cookie storage adapter + updated `createClient`

**Files:**
- Modify: `index.html` — Supabase init section (around line 1628)

The Supabase client currently uses the default `localStorage` storage. Replace it with a cookie adapter so sessions are readable by both Safari and the PWA.

- [ ] **Step 1.1: Locate the `createClient` call**

Find lines 1628–1630 in `index.html`:
```js
const SUPA_URL = 'https://oqubpyoceimwwputhtei.supabase.co';
const SUPA_KEY = 'sb_publishable__UpDT25FqPFn_dSscchCww_-Hmwpovv';
const supa = supabase.createClient(SUPA_URL, SUPA_KEY);
```

- [ ] **Step 1.2: Replace with cookie adapter + updated client config**

Replace those three lines with:
```js
const SUPA_URL = 'https://oqubpyoceimwwputhtei.supabase.co';
const SUPA_KEY = 'sb_publishable__UpDT25FqPFn_dSscchCww_-Hmwpovv';

const cookieStorage = {
  getItem(key) {
    const escaped = key.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    const match = document.cookie.match(new RegExp('(?:^|; )' + escaped + '=([^;]*)'));
    return match ? decodeURIComponent(match[1]) : null;
  },
  setItem(key, value) {
    const maxAge = 60 * 60 * 24 * 365;
    document.cookie = `${key}=${encodeURIComponent(value)}; max-age=${maxAge}; path=/; SameSite=Lax; Secure`;
  },
  removeItem(key) {
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

- [ ] **Step 1.3: Verify the page still loads**

Open `index.html` in a browser (or via GitHub Pages). Open DevTools → Console. Confirm no JS errors on load. The login screen should appear as before.

- [ ] **Step 1.4: Commit**

```bash
git add index.html
git commit -m "feat: use cookie storage for Supabase session (fixes iOS PWA/Safari isolation)"
```

---

## Task 2: Fix init sequence (sessionHandled flag + standalone guard)

**Files:**
- Modify: `index.html` — INIT section (around lines 1808–1835)

The current code calls `onSignedIn()` from two places (the `onAuthStateChange` listener and the startup IIFE), causing a race where it can fire twice. Fix with a `sessionHandled` flag. Also add a `!window.navigator.standalone` guard (needed when Google OAuth is added in a later phase — for OTP it has no effect since OTP always runs inside the PWA).

- [ ] **Step 2.1: Locate the INIT block**

Find the INIT section starting around line 1807. It contains:
```js
supa.auth.onAuthStateChange(async (event, session) => { ... });
(async () => { ... })();
document.getElementById('btn-send-link').addEventListener(...);
```

- [ ] **Step 2.2: Replace the entire INIT block**

Replace from `// Auth events` through `document.getElementById('btn-signout').addEventListener(...)` (keep the service worker registration below, untouched):

```js
// ---- SESSION INIT ----
let sessionHandled = false;

supa.auth.onAuthStateChange(async (event, session) => {
  if (session?.user) {
    if (!window.navigator.standalone) {
      // In Safari after an OAuth redirect — show return screen, don't enter app.
      // Not reachable for Email OTP (OTP always runs inside the PWA).
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

document.getElementById('btn-signout').addEventListener('click', signOut);
```

Note: the `btn-send-link` listener line is removed here — the new event listeners are added in Task 5.

- [ ] **Step 2.3: Verify no regression**

Reload the page. Login screen should appear. Console should show no errors.

- [ ] **Step 2.4: Commit**

```bash
git add index.html
git commit -m "fix: prevent double onSignedIn() on startup, add standalone guard"
```

---

## Task 3: Add `#oauth-return` screen HTML

**Files:**
- Modify: `index.html` — HTML screens section (around line 741, after `#login`)

This screen is shown in Safari after a Google OAuth redirect (future phase). For now it just needs to exist so `showScreen('oauth-return')` doesn't silently fail.

- [ ] **Step 3.1: Add the screen after `#login`**

Find line 741 (just after the closing `</div>` of `#login`):
```html
<!-- HOME -->
```

Insert before it:
```html
<!-- OAUTH RETURN (shown in Safari after Google OAuth redirect) -->
<div class="screen hidden" id="oauth-return">
  <div style="display:flex;flex-direction:column;align-items:center;justify-content:center;height:100%;gap:1.5rem;padding:2rem;text-align:center;">
    <div style="font-family:'Playfair Display',serif;font-size:3rem;color:var(--accent);">Ελ</div>
    <div style="font-family:'Playfair Display',serif;font-size:1.4rem;color:var(--text);">Вход выполнен</div>
    <div style="color:var(--text-muted);font-size:0.95rem;line-height:1.7;">
      Вернитесь на экран «Домой»<br>и откройте <strong style="color:var(--accent);">Ελληνικά</strong> со значка.
    </div>
  </div>
</div>
```

- [ ] **Step 3.2: Verify**

Reload. In DevTools console call `showScreen('oauth-return')`. The return message should appear full-screen. Call `showScreen('login')` to restore.

- [ ] **Step 3.3: Commit**

```bash
git add index.html
git commit -m "feat: add oauth-return screen for post-OAuth Safari interstitial"
```

---

## Task 4: Redesign `#login` screen HTML and CSS

**Files:**
- Modify: `index.html` — `#login` HTML (lines 730–741) and CSS (lines 541–585)

Replace the single-input magic link form with a two-panel OTP form. Panel 1: email input + send button. Panel 2 (hidden by default): 6-digit code entry + verify button + resend timer. The subtitle has two static variants that are shown/hidden rather than setting `innerHTML` dynamically.

- [ ] **Step 4.1: Replace the `#login` HTML**

Find lines 730–741:
```html
<!-- LOGIN -->
<div class="screen" id="login">
  <div class="login-logo">Ελ</div>
  <div>
    <div class="login-title">Ελληνικά Карточки</div>
    <div class="login-sub">Введите email — мы пришлём<br>ссылку для входа</div>
  </div>
  <div class="login-form">
    <input class="field-input" id="login-email" type="email" placeholder="your@email.com" autocomplete="email">
    <button class="btn-primary" id="btn-send-link">Отправить ссылку →</button>
    <div class="login-msg" id="login-msg"></div>
  </div>
</div>
```

Replace with:
```html
<!-- LOGIN -->
<div class="screen" id="login">
  <div class="login-logo">Ελ</div>
  <div>
    <div class="login-title">Ελληνικά Карточки</div>
    <!-- Two subtitle variants; JS toggles which is visible -->
    <div class="login-sub" id="login-sub-email">Введите email — мы пришлём<br>6-значный код для входа</div>
    <div class="login-sub hidden" id="login-sub-code"></div>
  </div>

  <!-- Panel 1: email input -->
  <div class="login-form" id="login-panel-email">
    <input class="field-input" id="login-email" type="email" placeholder="your@email.com"
           autocomplete="email" inputmode="email">
    <button class="btn-primary" id="btn-send-otp">Отправить код →</button>
    <div class="login-msg" id="login-msg"></div>
  </div>

  <!-- Panel 2: code entry (hidden until code is sent) -->
  <div class="login-form hidden" id="login-panel-code">
    <div class="otp-inputs" id="otp-inputs">
      <input class="otp-digit" type="text" inputmode="numeric" maxlength="1" pattern="[0-9]">
      <input class="otp-digit" type="text" inputmode="numeric" maxlength="1" pattern="[0-9]">
      <input class="otp-digit" type="text" inputmode="numeric" maxlength="1" pattern="[0-9]">
      <input class="otp-digit" type="text" inputmode="numeric" maxlength="1" pattern="[0-9]">
      <input class="otp-digit" type="text" inputmode="numeric" maxlength="1" pattern="[0-9]">
      <input class="otp-digit" type="text" inputmode="numeric" maxlength="1" pattern="[0-9]">
    </div>
    <button class="btn-primary" id="btn-verify-otp">Подтвердить →</button>
    <div class="login-msg" id="login-msg-code"></div>
    <div class="otp-resend">
      <span id="otp-resend-timer" class="otp-resend-timer"></span>
      <button class="otp-resend-btn hidden" id="btn-resend-otp">Отправить повторно</button>
    </div>
    <button class="otp-back-btn" id="btn-back-to-email">← другой email</button>
  </div>
</div>
```

- [ ] **Step 4.2: Add CSS for the new login elements**

Find the existing login CSS block (lines ~541–585). After `.login-msg.error { color: #e74c3c; }`, add:

```css
    /* OTP digit inputs */
    .otp-inputs {
      display: flex;
      gap: 0.5rem;
      justify-content: center;
    }

    .otp-digit {
      width: 2.8rem;
      height: 3.2rem;
      text-align: center;
      font-size: 1.4rem;
      font-family: inherit;
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 8px;
      color: var(--text);
      caret-color: var(--accent);
      outline: none;
      -webkit-appearance: none;
    }

    .otp-digit:focus {
      border-color: var(--accent);
      box-shadow: 0 0 0 2px rgba(232, 197, 71, 0.2);
    }

    .otp-resend {
      font-size: 0.85rem;
      color: var(--text-muted);
      text-align: center;
      min-height: 1.4rem;
    }

    .otp-resend-timer {
      font-style: italic;
    }

    .otp-resend-btn {
      background: none;
      border: none;
      color: var(--accent2);
      font-size: 0.85rem;
      cursor: pointer;
      padding: 0;
      font-family: inherit;
      text-decoration: underline;
    }

    .otp-back-btn {
      background: none;
      border: none;
      color: var(--text-muted);
      font-size: 0.8rem;
      cursor: pointer;
      padding: 0;
      font-family: inherit;
    }
```

- [ ] **Step 4.3: Verify layout**

Reload. The login screen should show the Ελ logo, title, subtitle, email input, and "Отправить код →" button. The code panel should not be visible. No console errors.

- [ ] **Step 4.4: Commit**

```bash
git add index.html
git commit -m "feat: redesign login screen with two-panel OTP form"
```

---

## Task 5: OTP auth functions + event wiring

**Files:**
- Modify: `index.html` — AUTH section (around lines 1642–1675)

Replace `sendMagicLink()` with `sendEmailOtp()`, `verifyEmailOtp()`, OTP digit-input keyboard handling, and resend timer. Wire up all new event listeners.

- [ ] **Step 5.1: Replace `sendMagicLink` with new auth functions**

Find the AUTH section starting with `// ---- AUTH ----` (around line 1642). Replace lines 1643–1669 only (the `sendMagicLink` function). The replacement ends immediately before the `async function signOut()` definition at line 1671 — do not touch `signOut` or anything below it.

```js
// ---- AUTH ----
let _otpEmail = '';
let _resendTimer = null;

function startResendTimer(seconds) {
  const timerEl = document.getElementById('otp-resend-timer');
  const resendBtn = document.getElementById('btn-resend-otp');
  resendBtn.classList.add('hidden');
  let remaining = seconds;
  timerEl.textContent = `Повторно через ${remaining}с`;
  clearInterval(_resendTimer);
  _resendTimer = setInterval(() => {
    remaining--;
    if (remaining <= 0) {
      clearInterval(_resendTimer);
      timerEl.textContent = '';
      resendBtn.classList.remove('hidden');
    } else {
      timerEl.textContent = `Повторно через ${remaining}с`;
    }
  }, 1000);
}

function showOtpCodePanel(email) {
  _otpEmail = email;
  document.getElementById('login-panel-email').classList.add('hidden');
  document.getElementById('login-panel-code').classList.remove('hidden');
  document.getElementById('login-sub-email').classList.add('hidden');
  const codeSubEl = document.getElementById('login-sub-code');
  codeSubEl.classList.remove('hidden');
  codeSubEl.textContent = `Код отправлен на ${email}`;
  document.getElementById('login-msg-code').textContent = '';
  document.getElementById('login-msg-code').className = 'login-msg';
  document.querySelectorAll('.otp-digit').forEach(el => { el.value = ''; });
  document.querySelector('.otp-digit').focus();
  startResendTimer(45);
}

function showEmailPanel() {
  _otpEmail = '';
  clearInterval(_resendTimer);
  document.getElementById('login-panel-code').classList.add('hidden');
  document.getElementById('login-panel-email').classList.remove('hidden');
  document.getElementById('login-sub-code').classList.add('hidden');
  document.getElementById('login-sub-email').classList.remove('hidden');
  document.getElementById('login-msg').textContent = '';
  document.getElementById('login-msg').className = 'login-msg';
}

async function sendEmailOtp(email) {
  const msg = document.getElementById('login-msg');
  if (!email) { msg.textContent = 'Введите email'; msg.className = 'login-msg error'; return; }

  const btn = document.getElementById('btn-send-otp');
  btn.disabled = true;
  btn.textContent = 'Отправляем...';
  msg.textContent = '';

  const { error } = await supa.auth.signInWithOtp({
    email,
    options: { shouldCreateUser: true }
  });

  btn.disabled = false;
  btn.textContent = 'Отправить код →';

  if (error) {
    msg.textContent = 'Ошибка: ' + error.message;
    msg.className = 'login-msg error';
  } else {
    showOtpCodePanel(email);
  }
}

async function verifyEmailOtp() {
  const token = Array.from(document.querySelectorAll('.otp-digit'))
    .map(el => el.value.trim())
    .join('');
  const msg = document.getElementById('login-msg-code');

  if (token.length !== 6) {
    msg.textContent = 'Введите все 6 цифр';
    msg.className = 'login-msg error';
    return;
  }

  const btn = document.getElementById('btn-verify-otp');
  btn.disabled = true;
  btn.textContent = 'Проверяем...';
  msg.textContent = '';

  const { error } = await supa.auth.verifyOtp({
    email: _otpEmail,
    token,
    type: 'email'
  });

  btn.disabled = false;
  btn.textContent = 'Подтвердить →';

  if (error) {
    msg.textContent = 'Неверный код. Попробуйте ещё раз.';
    msg.className = 'login-msg error';
  }
  // On success: onAuthStateChange fires → onSignedIn() called
}
```

- [ ] **Step 5.2: Add OTP digit keyboard navigation**

After the functions above, add:

```js
// OTP digit keyboard UX: auto-advance, backspace goes back, paste fills all
function initOtpInputs() {
  const digits = Array.from(document.querySelectorAll('.otp-digit'));

  digits.forEach((el, i) => {
    el.addEventListener('input', () => {
      el.value = el.value.replace(/\D/g, '').slice(-1);
      if (el.value && i < digits.length - 1) digits[i + 1].focus();
    });

    el.addEventListener('keydown', e => {
      if (e.key === 'Backspace' && !el.value && i > 0) digits[i - 1].focus();
      if (e.key === 'Enter') verifyEmailOtp();
    });
  });

  // Paste: fill all digits from clipboard
  digits[0].addEventListener('paste', e => {
    e.preventDefault();
    const text = (e.clipboardData || window.clipboardData).getData('text').replace(/\D/g, '');
    digits.forEach((el, i) => { el.value = text[i] || ''; });
    const lastFilled = Math.min(text.length, digits.length) - 1;
    if (lastFilled >= 0) digits[lastFilled].focus();
  });
}

initOtpInputs();
```

- [ ] **Step 5.3: Update the event listeners in the INIT block**

After `document.getElementById('btn-signout').addEventListener('click', signOut);`, add:

```js
// Email OTP login form
document.getElementById('btn-send-otp').addEventListener('click', () => {
  sendEmailOtp(document.getElementById('login-email').value.trim());
});
document.getElementById('login-email').addEventListener('keydown', e => {
  if (e.key === 'Enter') sendEmailOtp(document.getElementById('login-email').value.trim());
});
document.getElementById('btn-verify-otp').addEventListener('click', verifyEmailOtp);
document.getElementById('btn-resend-otp').addEventListener('click', () => {
  sendEmailOtp(_otpEmail);
});
document.getElementById('btn-back-to-email').addEventListener('click', showEmailPanel);
```

- [ ] **Step 5.4: Verify no console errors**

Reload. Open DevTools. No errors. The "Отправить код →" button should be clickable.

- [ ] **Step 5.5: Commit**

```bash
git add index.html
git commit -m "feat: implement Email OTP auth (send code + verify 6-digit code)"
```

---

## Task 6: End-to-end manual test

No automated test runner exists in this project. Verify the full flow manually.

- [ ] **Step 6.1: Push and deploy**

```bash
git push
```

Wait ~60 seconds for GitHub Pages to deploy `https://al-sharo.github.io/greek-cards`.

- [ ] **Step 6.2: Test in desktop browser**

1. Open `https://al-sharo.github.io/greek-cards` in Chrome or Safari.
2. Login screen appears. Email field and "Отправить код →" visible. Code panel hidden.
3. Enter your email and tap "Отправить код →". Button shows "Отправляем..." then reverts. Code panel appears, email panel hides. Subtitle updates to "Код отправлен на [email]".
4. Resend countdown shows "Повторно через 45с" and counts down.
5. Check inbox for a 6-digit code.
6. Enter digits. Auto-advance between inputs works. Backspace goes back one digit.
7. Tap "Подтвердить →". Home screen appears.
8. DevTools → Application → Cookies: `sb-oqubpyoceimwwputhtei-auth-token` cookie is present.
9. Reload the page. Home screen appears immediately — no login prompt.

- [ ] **Step 6.3: Test session persistence across tab close**

1. Close the tab. Open a new tab to the same URL.
2. Home screen appears immediately without re-logging in.

- [ ] **Step 6.4: Test wrong code**

1. Sign out. Send a code. Enter `000000`.
2. Error message appears: "Неверный код. Попробуйте ещё раз."
3. Enter the real code. Login succeeds.

- [ ] **Step 6.5: Test "← другой email" back button**

1. Sign out. Enter email, send code. Code panel appears.
2. Tap "← другой email". Email panel reappears, subtitle resets.

- [ ] **Step 6.6: Test paste**

1. Sign out. Send a new code. Copy the 6 digits from the email.
2. On the code panel, paste into the first digit input. All 6 should fill automatically.

- [ ] **Step 6.7: Test expired/invalid code recovery**

1. Sign out. Send a code. Enter `000000` (wrong code).
2. Error "Неверный код. Попробуйте ещё раз." appears.
3. The "Подтвердить →" button must re-enable — it must not stay stuck as "Проверяем...".
4. Enter the real code. Login succeeds normally.

- [ ] **Step 6.8: Test on iPhone (critical — the whole point)**

1. Open `https://al-sharo.github.io/greek-cards` in iPhone Safari.
2. If app is already on home screen, remove it first (long-press → Remove App).
3. Add to home screen: Share → Add to Home Screen → Add.
4. Open from the home screen icon (no browser chrome — this is standalone mode).
5. Complete the OTP login flow.
6. Press home button. Re-open the app from the icon.
7. **Home screen should appear immediately — no login prompt.** This is the success criterion.

---

## Done

Email OTP auth is fully implemented. Session persists via cookies, shared between Safari and the PWA. Google OAuth can be added in a separate phase — the cookie adapter, standalone guard, and `#oauth-return` screen are already in place.
