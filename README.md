# 🔐 MFA / OTP / CAPTCHA Automation: What I Learned

A QA learning project about the **hardest blockers in login automation**: **Email OTP, SMS OTP, CAPTCHA and MFA (Google Authenticator)**.

I built a small authentication app with each of these security checks, then tried to automate them with **Playwright**.
These checks are **designed to stop bots**, and an automation script *is* a bot. So the real lesson was **how a QA engineer tests secure flows without breaking the security itself.**

> 📌 **Where the code is:** the full app and QA tests are on the **`captcha-stable`** branch (latest). `main` has only the first version of the app.
>
> | Branch | What's in it |
> |---|---|
> | `main` | First version: register → email OTP → Google Authenticator MFA |
> | `captcha` / `phone` | CAPTCHA screen and SMS OTP (Twilio) added |
> | `qa` / `otp-stable` | ✅ Email OTP automation working |
> | `captcha-stable` | ✅ CAPTCHA automation + separate OTP / CAPTCHA / MFA servers |

---

## 🧩 The app under test

A Node.js + Express app with three separate "modes", so each blocker can be practised on its own:

| Server | Port | Security check | How the user passes it |
|---|---|---|---|
| `otp-server` | 3000 | 📧 Email OTP / 📱 SMS OTP | 6-digit code sent by email (Nodemailer) or SMS (Twilio) |
| `captcha-server` | 3000 / 5000 | 🧩 CAPTCHA | Type the distorted text from an image (`svg-captcha`) |
| `mfa-server` | 8000 | 🔑 MFA / TOTP | Scan a QR code with Google Authenticator and enter the 6-digit code that changes every 30 s (`otplib`) |

Tech: Node.js, Express, express-session, Nodemailer, Twilio, otplib, qrcode, svg-captcha, Swagger · **QA:** Playwright, imap-simple, mailparser, axios.

```
qa/
├── playwright.config.js     # baseURL from .env, screenshots/videos/trace on failure
├── page/
│   ├── emailHelper.js       # reads the OTP from a real Gmail inbox (IMAP)
│   └── captchaHelper.js     # gets past CAPTCHA through the API and returns a session cookie
└── tests/
    ├── otp.spec.js          # full register flow with a real email OTP
    └── captcha.spec.js      # skips CAPTCHA, injects the session, tests the dashboard
```

---

## 🚧 Blocker 1: Email OTP

### The problem
After registering, the app emails a **random 6-digit code**. The test can't know the code in advance, and Playwright only controls the browser, not my inbox.

### What I did ✅
I made the test **read the real inbox**, the way a human would:

1. Playwright fills the register form and clicks **Register**.
2. `emailHelper.js` connects to Gmail over **IMAP** (`imap.gmail.com:993`) with a Gmail **App Password**.
3. It searches **UNSEEN** emails sent **TO** the test address.
4. It **retries up to 10 times, every 3 s**, because email doesn't arrive instantly.
5. It parses the email with `mailparser` and pulls out the code with the regex `/\b\d{6}\b/`.
6. It marks the email as read (so the next run doesn't pick up an old OTP), then types the code into the page.

```js
const otp = await getOTPFromGmail(TEST_EMAIL);
await page.fill('#reg-otp-input', otp);
await page.click('#btn-reg-otp-verify');
```

### What I learned
- 🕐 **Email is asynchronous.** A single check fails randomly; **polling with retry + wait** is required. Connection timeouts also had to be raised (10 s → 30 s).
- 🔑 Gmail blocks normal passwords for IMAP. You need **2-Step Verification + an App Password**.
- 📬 Use a **dedicated test inbox** and **only read UNSEEN mail**, otherwise an old OTP gets reused and the test fails.
- 🐢 Real-inbox tests are **slow and depend on an outside service**. They're good for a few end-to-end checks, not for hundreds of tests. Faster alternatives are a **test email API** (Mailosaur, MailSlurp, Mailtrap) or a **dev mode** where the server logs or returns the OTP. My server already logs `[DEV MODE] OTP for …` when email isn't configured.

---

## 🚧 Blocker 2: SMS OTP

### The problem
The code is sent to a **real phone** through Twilio. A test can't read a phone's SMS app.

### What I learned
- A test needs a way to read SMS **programmatically**, for example a Twilio test number plus the Twilio API to fetch received messages, or an SMS-testing service.
- In **dev/test environments** the simplest option is to **not send a real SMS at all**. My server falls back to logging the OTP to the console when Twilio isn't configured, which keeps tests free and fast.
- SMS costs money per message and is rate-limited, so it should **never run on every commit**.

> ⏳ SMS OTP is built into the app, but its automated test is not written yet.

---

## 🚧 Blocker 3: CAPTCHA

### The problem
CAPTCHA exists *specifically* to prove "you are not a bot". The text is distorted, noisy and random.
Trying to "solve" it with OCR or AI in tests is **unreliable, slow, and the wrong approach**. If a test can solve your CAPTCHA, so can attackers.

### What I did ✅: **bypass it in the test environment, then test what comes after it**
1. **Test-only hook on the server:** in the test environment the captcha server also sends the answer in a response header:
   ```js
   // Only acceptable in non-prod environments
   res.setHeader('x-e2e-captcha', captcha.text);
   ```
2. **Register through the API, not the UI** (`captchaHelper.js` with axios). This creates a unique user every run (`testuser_<timestamp>@gmail.com`), so there's no "already registered" error.
3. Read the `x-e2e-captcha` header and the **session cookie**, then call `/api/verify-captcha` with the right answer, **sending the same cookie back** (the answer is stored in *that* session).
4. **Inject the logged-in session cookie into Playwright**, then open the dashboard directly:
   ```js
   await page.context().addCookies(cookies);
   await page.goto('/dashboard.html');
   await expect(page.locator('#dash-welcome')).toContainText('स्वागतम्');
   ```

### What I learned
- 🧠 **Don't automate the CAPTCHA, automate *around* it.** Common real-world options:
  - turn CAPTCHA **off** in QA/staging with a config flag,
  - use the provider's **test keys** (Google reCAPTCHA has official test keys that always pass),
  - whitelist test IPs/users, or use a **test-only header/hook** like mine.
- 🔒 Any bypass **must never reach production.** Exposing the answer in a header in prod would make the CAPTCHA useless. It needs an environment guard (e.g. only when `NODE_ENV=test`).
- 🍪 **Sessions matter.** The CAPTCHA answer lives in the server session, so the verify request must send the **same session cookie**. Without it, the server sees a brand-new session and rejects the answer.
- ⚡ **API login + cookie injection** is much faster than clicking through the UI, and it's how real teams skip login/CAPTCHA in most UI tests. Only *one* test needs to cover the login UI itself.
- 🧪 It's still worth **manually testing** the real CAPTCHA (wrong answer, refresh, case-insensitivity). My server compares answers in lowercase.

---

## 🚧 Blocker 4: MFA / Google Authenticator (TOTP)

### The problem
The 6-digit code comes from an **app on a phone** and **changes every 30 seconds**. The test can't open the phone.

### The key idea I learned 💡
A TOTP code is **not random**. It's calculated from a **shared secret + the current time**.
When the user scans the QR code, the phone stores that secret. **If the test also knows the secret, it can generate the exact same code**:

```js
const { authenticator } = require('otplib');
const code = authenticator.generate(TOTP_SECRET);   // same code as the phone shows
await page.fill('#mfa-otp-input', code);
```

My app already shows the **manual setup key** (`Manual Code: XXXX…`) under the QR code, so a test can read the secret from the setup screen, or a test account can be created with a known secret stored safely in `.env`.

### What I learned
- ⏱️ Watch the **30-second window**: a code generated at second 29 can expire before it's submitted. Wait for a fresh window if needed, and keep the test machine's **clock in sync**.
- 🔁 The same code usually **can't be reused** twice, so be careful when retrying.
- 🔐 The TOTP secret is a **real credential**. Keep it in `.env` / CI secrets, never in code.

> ⏳ The MFA server is built, but its automated test is not written yet. The `otplib` approach above is the plan.

---

## 📚 Overall lessons

1. **Security features are supposed to block automation.** The job is to **test them smartly, not defeat them.**
2. Ask devs for **testability hooks**: test mode, test keys, or a way to read OTPs. These must be enabled **only outside production**.
3. **Split the testing:**
   - a few **true end-to-end** tests that go through the real OTP / CAPTCHA screens,
   - **most** tests log in through the **API + cookie injection** and then focus on the feature itself.
4. Use **unique test data** every run (timestamped emails) so tests don't collide.
5. Expect **waiting** (email delivery, TOTP windows) and build in **polling and retries** instead of fixed sleeps.
6. **Secrets management:** App Passwords, Twilio tokens and TOTP secrets belong in `.env` (git-ignored) or CI secrets.
   ⚠️ I learned this one the hard way: a `.env` file with real credentials got committed early in this repo. **Removing the file in a later commit does not remove it from git history**, so the passwords had to be changed.

---

## 🚀 How to run (from the `captcha-stable` branch)

```bash
git checkout captcha-stable

# 1. App
npm install
cp .env.example .env                 # add email / Twilio settings (optional: without them OTPs are printed in the console)
npm run start:otp                    # or start:captcha / start:mfa / start:modes (all three)

# 2. Tests (second terminal)
cd qa
npm install
npx playwright install
cp .env.example .env                 # APP_URL, TEST_PASSWORD, GMAIL_USER, GMAIL_APP_PASSWORD
npx playwright test tests/otp.spec.js       # needs the OTP server + a real Gmail inbox
npx playwright test tests/captcha.spec.js   # needs the CAPTCHA server
npm run report
```

Swagger API docs are available at `http://localhost:<port>/api-docs` on the CAPTCHA server.

---

## 🔮 Next steps

- [ ] Automate **MFA** with `otplib` + a known test secret
- [ ] Automate **SMS OTP** (Twilio test API or dev-mode OTP)
- [ ] Guard the `x-e2e-captcha` header with `NODE_ENV === 'test'`
- [ ] Add `axios`, `imap-simple` and `mailparser` to `qa/package.json`. Right now they only resolve from the root `node_modules`
- [ ] Replace fixed `waitForTimeout` waits with proper `expect(...)` waits
- [ ] Run the suite in **GitHub Actions** with secrets
