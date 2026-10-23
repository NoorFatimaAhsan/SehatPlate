# SehatPlate

**An AI-powered food guide for Pakistani dishes.** A user types a dish name or uploads a photo, picks a health condition (diabetes, kidney disease or high blood pressure), answers a few follow-up questions, and gets a clear verdict (**Safe**, **Eat in moderation** or **Avoid**) with a short reason and healthier swaps.

> **Guidance only. Please consult your doctor or dietitian.** This is a hackathon prototype. The dish rules have **not** been reviewed by a doctor or dietitian and the app is **not** clinically validated.

- Live demo: `<your live URL, or delete this line>`
- Source: `<your GitHub repo URL>`

## What problem it solves

Many people in Pakistan live with diabetes, kidney disease or high blood pressure, but most food apps are built around Western meals. SehatPlate understands local dishes such as biryani, nihari, haleem, daal chawal and chai, and asks follow-up questions because the same dish differs from home to home.

## Key idea: the AI proposes, the rules decide

The AI (Groq) suggests a verdict, but **the code has the final say**. The strictest result always wins (Safe < Moderation < Avoid), so the AI can never be softer than the rules:

1. `server/dish_rules.json` can set a minimum verdict for each dish and condition.
2. Escalating answers raise it further (for example a large portion, or 2 or more teaspoons of sugar in tea).
3. Low confidence (under 0.6) or any unanswered question gives at least Moderation.
4. If the AI fails or returns unusable output, the rules answer on their own, and the app never crashes.
5. Medical wording like "dose" or "medication" in AI text is replaced with `[omitted]`.

## Features

- Dish recognition from a **typed name** (matched locally against the rules) or a **photo** (Groq vision model)
- Three conditions: diabetes, kidney disease, high blood pressure
- Dish-specific and condition-specific follow-up questions (10 curated dishes, and AI-generated questions for other dishes)
- Verdict, reason, healthy swaps and a disclaimer on every result
- Scan history for each user
- Email and password accounts, with optional two-factor authentication (authenticator app and 8 single-use recovery codes)

Curated dishes: Chicken Biryani, Nihari, Haleem, Daal Chawal, Chai, Chicken Karahi, Halwa Puri, Samosa, Paratha, Roti with Vegetable Curry.

## Tech stack

- **Frontend:** React 18, Vite 4, React Router
- **Backend:** Node.js, Express, Zod, JWT (HS256), bcryptjs, otplib, qrcode, multer, helmet, express-rate-limit
- **Database:** PostgreSQL (`users` and `scans` tables)
- **AI:** Groq API, called only from the backend (`server/ai.js`)
- **Tests:** Vitest and supertest

## Run it locally (step by step)

### 1. Requirements

- **Node.js 18 or newer** (check with `node -v`)
- **PostgreSQL** running locally, or any hosted PostgreSQL database
- A **Groq API key** (optional, see "What works without a Groq key" below)

### 2. Install

```bash
git clone <your GitHub repo URL>
cd sehatplate
npm install
```

### 3. Create the database tables

Create an empty database and run the first migration. This one file creates both tables:

```bash
createdb sehatplate
psql "postgresql://USER:PASSWORD@localhost:5432/sehatplate" -f db/migrations/001_init.sql
```

Run **only** `001_init.sql`. The other files in `db/migrations/` are older or duplicate versions, and one of them defines the 2FA tables differently.

### 4. Set the environment variables

The app reads its settings from environment variables in the **same terminal** you start it from. It does **not** load a `.env` file automatically.

macOS or Linux:

```bash
export DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/sehatplate"
export SESSION_SECRET="$(node -e "console.log(require('crypto').randomBytes(32).toString('hex'))")"
export GROQ_API_KEY="your_groq_key"            # optional
export GROQ_TEXT_MODEL="a_groq_text_model_id"   # optional
export GROQ_VISION_MODEL="a_groq_vision_model_id" # optional, for photos
```

Windows PowerShell:

```powershell
$env:DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/sehatplate"
$env:SESSION_SECRET="any-long-random-string-at-least-32-characters"
$env:GROQ_API_KEY="your_groq_key"
$env:GROQ_TEXT_MODEL="a_groq_text_model_id"
$env:GROQ_VISION_MODEL="a_groq_vision_model_id"
```

- `DATABASE_URL` must start with `postgres://` or `postgresql://`. The app exits with an error for any other database.
- `SESSION_SECRET` is required. The app exits if it is missing.
- Pick current model IDs from Groq's model list. Choose a normal (non-reasoning) model, because a "thinking" model can run out of tokens before it finishes its JSON answer.
- Leave `FRONTEND_ORIGIN` and `VITE_API_URL` **unset** for local use. The Vite dev server forwards API calls to the backend.

### 5. Start the app

```bash
npm run dev
```

This starts the API on **http://localhost:3001** and the website on **http://localhost:5173**. Open **http://localhost:5173** in your browser.

Check that the API is up: open http://localhost:5173/health. You should see `{"status":"ok","db":true}`.

## What works without a Groq key

- **Typed dish names** work. Names are matched locally against the curated rules.
- **Questions for the 10 curated dishes** work. They come from the rules file.
- **Verdicts** work. If the AI call fails, the rules decide, and the result is never softer than the rules allow.
- **Photo recognition and AI-generated questions for other dishes** need `GROQ_API_KEY` and the model variables. Without them, a photo scan shows a friendly "AI service not configured" error and you can type the dish name instead.
- A verdict of **Safe** needs a working AI proposal. With the AI off, dishes without a rule floor return Moderation, to stay on the cautious side.

## Try it: a 5-minute walkthrough for judges

1. **Create an account.** Click **Register**, enter an email and a password of at least 8 characters. You are signed in automatically.
2. **Pick a condition.** On the **Scan** tab choose **Diabetes**.
3. **Type a dish.** Enter `biryani` and click **Identify dish**.
4. **Confirm the dish.** The app shows its best guess. Confirm or edit the name and press **Continue**.
5. **Answer the questions.** Choose **Large plate or more** for the portion and answer the rest. Press **Submit**.
6. **Read the result.** You should get **Avoid**, with a reason, healthier swaps and the disclaimer. The rules forced that verdict for a large portion, whatever the AI said. A small note, "Adjusted to be safer", appears when the rules overrode the AI.
7. **Try the rule floor.** Scan `halwa puri` with Diabetes. The result is **Avoid** whatever you answer.
8. **Try a different condition.** Scan `chai` with Diabetes and choose 2 teaspoons or more of sugar. The result is **Avoid**.
9. **See your history.** Open the **History** tab to see your scans, newest first. Use **Refresh** to reload.
10. **Turn on 2FA (optional).** Click **Enable 2FA** in the top bar. Scan the QR code with an authenticator app, save the recovery codes (they are shown only once), enter the 6-digit code, and verify. Log out and log in again to see the 2FA prompt. You can also use a recovery code once.
11. **Try a photo (needs a Groq key).** Upload a JPEG, PNG or WebP image under 5 MB instead of typing a name.

## How a scan works

1. The user types a dish name or uploads a photo, and picks a condition.
2. `POST /dish/identify` returns a dish guess with a confidence from 0 to 1.
3. The user confirms or edits the dish name.
4. `POST /dish/questions` returns 3 or 4 follow-up questions.
5. `POST /verdict` asks the AI for one proposal, then applies the safety rules in `server/verdict.js`.
6. The result is saved to the user's history and shown with the disclaimer.

## API (all errors return an HTTP status and `{ "detail": "..." }`)

- `GET /health`
- `POST /auth/register`, `POST /auth/login`
- `POST /auth/2fa/setup`, `POST /auth/2fa/verify`, `POST /auth/2fa/login`
- `GET /me`
- `POST /dish/identify` (typed `name` or an `image` file)
- `POST /dish/questions`
- `POST /verdict`
- `GET /history`

## Run the tests

The tests mock the AI, so no Groq key is needed. Set placeholder values first, because the code reads them when it loads.

macOS or Linux:

```bash
export DATABASE_URL="postgresql://user:pass@localhost:5432/test"
export SESSION_SECRET="test-secret"
npx vitest run server/verdict.test.js server/ai.test.js server/dish.test.js server/verdictRoute.test.js
```

Windows PowerShell:

```powershell
$env:DATABASE_URL="postgresql://user:pass@localhost:5432/test"
$env:SESSION_SECRET="test-secret"
npx vitest run server/verdict.test.js server/ai.test.js server/dish.test.js server/verdictRoute.test.js
```

These cover the verdict rules (floors, escalations, low confidence, AI failure, redaction), the AI response parsing, the dish routes and the `/verdict` route. `server/verdictHistory.test.js` is an end-to-end test that needs a real database, so it is not in this command.

## Project structure

- `src/`: React frontend (`pages/AuthPage.jsx`, `pages/ScanPage.jsx`, `api.js`, `styles.css`)
- `server/index.js`: Express app and middleware
- `server/auth.js`, `server/twofa.js`: login, JWT and 2FA
- `server/dish.js`: dish identification and questions
- `server/verdictRoute.js`, `server/verdict.js`: the verdict route and the safety logic
- `server/ai.js`: the only file that calls Groq
- `server/dish_rules.json`: curated dishes, questions, escalations and swaps
- `server/history.js`: scan history
- `db/migrations/`: SQL for the database tables
- `SPEC.md`: product specification

The `backend/` folder is an unused earlier FastAPI draft. The running backend is `server/`.

## Safety and privacy

- General guidance only. It is not medical advice, diagnosis or treatment, and each result carries a disclaimer.
- The AI never gives dosage, medication or diagnosis advice, and such wording is redacted.
- When unsure, the app picks the stricter verdict.
- The Groq key stays on the server and is never sent to the browser.
- Passwords are hashed with bcrypt. Every database query is parameterised, and users can read only their own scans.
- Limits: 20 login or register requests per 15 minutes per IP, 10 AI requests per minute per user, 5 failed 2FA attempts per 10 minutes, and uploads of JPEG, PNG or WebP up to 5 MB.

## Troubleshooting

- **"SESSION_SECRET is not set" or "DATABASE_URL is not set":** the variables were not exported in the same terminal where you ran `npm run dev`.
- **"Unsupported database scheme":** `DATABASE_URL` must start with `postgres://` or `postgresql://`.
- **`relation "users" does not exist`:** step 3 was skipped. Run `001_init.sql` against the database.
- **Port 5173 is busy:** the dev server will not pick another port. Set `PORT` to a free port and open that port instead.
- **Photo scan says the AI is not configured or unavailable:** set the three Groq variables, or type the dish name instead.
- **"Too many requests":** wait a minute (AI routes) or 15 minutes (login and register), then try again.
- **You are logged out after refreshing the page:** this is by design. The token is kept in memory only.

## Known limitations

- Not clinically validated, and `dish_rules.json` has not been reviewed by a doctor or dietitian.
- Only 10 dishes have curated rules. Other dishes rely on the AI and are kept at Moderation or stricter.
- No password reset, and no privacy or terms pages yet.
- The 2FA lockout counter is kept in memory, and 2FA setup and verify are not rate limited.
- English only.

## Future scope

Urdu voice and text, more dishes and regional cuisines, review by registered dietitians, weekly health insights, and a caregiver mode for managing a family member's diet.
