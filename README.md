# MoneyMap — Vanilla JavaScript edition

MoneyMap has been rebuilt as a static, dependency-free front end using **HTML, CSS and vanilla JavaScript modules**. It does not use React, JSX, TypeScript at runtime, Vite, Tailwind or a Node application server.

## Features included

- Public landing page with interactive feature descriptions, FAQ, privacy architecture explainer, light/dark theme switcher and quick tour.
- Email/password sign-up and sign-in, Google sign-in, Firebase Anonymous Authentication guest access.
- Responsive personal dashboard with sign-out back to the public landing page.
- Daily ledger, income/expense/net KPIs, running-balance chart, budget pace, safe-to-spend estimate and recurring watchlist.
- CSV and SMS import preview, deterministic merchant categorization, confidence labels, duplicate detection, review-first commit flow.
- Searchable transaction register, manual category review, custom merchant rules and recurring commitments.
- 30-day cash-flow Shock Radar with salary delay, unexpected bill, day-of-bill, daily spend, emergency-fund and goal-pause assumptions.
- Savings goals, progress updates and a monthly contribution what-if planner.
- Monthly P&L, budget-versus-actual report, daily ledger and CSV export.
- A website tour that moves through the landing page or each dashboard workspace and explains the current feature.
- Locally persisted theme preferences and cached guest state.

The original TypeScript finance engine and seed data were compiled into ordinary JavaScript modules in `js/core/`. The financial calculations, CSV/SMS parsing, recurring detection and monthly export logic are reused from the supplied project.

## Backend split

| Purpose | Service |
| --- | --- |
| Email/password, Google and anonymous sign-in | Firebase Authentication |
| Profile, transactions, rules, recurring items, goals, budgets and file metadata | Cloud Firestore |
| Uploaded CSV/TXT source statement files only | Supabase Storage (private bucket) |

Supabase Auth and the Supabase database are not used by the app. The Supabase service-role key is used only by the Edge Function and must never be placed in `firebase-config.js` or any browser file.

## 1. Configure Firebase

1. Open the Firebase Console and create or select a project.
2. Add a **Web App** and copy its web configuration into `firebase-config.js`.
3. In **Authentication → Sign-in method**, enable:
   - Email/Password
   - Google
   - Anonymous
4. In **Firestore Database**, create a database.
5. Open **Firestore → Rules**, paste the contents of `firestore.rules`, and publish them.
6. Add your local development domain and deployed website domain to **Authentication → Settings → Authorized domains**. For local testing, use `localhost`.

The app writes each user's records below `users/{uid}/...`. The Firestore rules restrict every nested document to the authenticated Firebase UID. Firebase anonymous users also receive a UID, so guest data is separated from other users.

### Firebase Web config

Edit `firebase-config.js` and replace:

- `YOUR_FIREBASE_API_KEY`
- `YOUR_PROJECT_ID`
- `YOUR_FIREBASE_APP_ID`
- `YOUR_MESSAGING_SENDER_ID`

Keep the domain as `<project-id>.firebaseapp.com` unless Firebase gives you a different `authDomain` value. Firebase's web config is designed to be used in a browser; Firestore Security Rules are what protect users' documents.

## 2. Configure Supabase Storage only (optional)

CSV parsing and importing work without Supabase. Supabase is only needed if you want an uploaded source statement archived in a private bucket.

1. Create or select a Supabase project.
2. Open **SQL Editor** and run `supabase/setup-storage.sql`. This creates the private `moneymap-imports` bucket, capped at 12 MB per file.
3. Install the Supabase CLI and sign in. From this project folder, link the project:

   ```bash
   supabase login
   supabase link --project-ref YOUR_SUPABASE_PROJECT_REF
   ```

4. Set function secrets (the service-role key stays on Supabase's server):

   ```bash
   supabase secrets set FIREBASE_PROJECT_ID=YOUR_FIREBASE_PROJECT_ID
   supabase secrets set SUPABASE_SERVICE_ROLE_KEY=YOUR_SUPABASE_SERVICE_ROLE_KEY
   supabase secrets set STORAGE_BUCKET=moneymap-imports
   ```

   `SUPABASE_URL` is available to hosted Supabase Edge Functions automatically. If your deployment does not expose it, set it explicitly with `supabase secrets set SUPABASE_URL=https://YOUR_PROJECT_REF.supabase.co`.

5. Deploy the upload function:

   ```bash
   supabase functions deploy storage-upload --no-verify-jwt
   ```

   The function deliberately disables Supabase's standard JWT gateway check because it verifies the Firebase ID token itself using Google's published Secure Token signing keys. It validates signature, issuer, audience and token expiry, then uploads the file to the private bucket under the verified Firebase UID. Do not remove the in-function Firebase verification.
6. In `firebase-config.js`, set `storageConfig.supabaseUrl` to `https://YOUR_PROJECT_REF.supabase.co` and `storageConfig.anonKey` to the Supabase publishable/anon key. This key can be present in the browser; the service-role key cannot.

No Supabase database table or Supabase Auth provider is required for this storage flow.

## 3. Run locally

Because the app uses JavaScript modules, open it through a local HTTP server instead of opening `index.html` as a `file://` URL.

Python option:

```bash
python3 -m http.server 5500
```

Then open `http://localhost:5500`.

VS Code option: use the Live Server extension and open the local URL it provides. Add that host to Firebase Authentication's authorized domains if you test sign-in there.

## 4. Deploy the website

This is a static site. Upload the project files to a static host such as Firebase Hosting, GitHub Pages, Netlify or Vercel. There is no build command and no `npm install` step.

For Firebase Hosting, the standard static setup is enough. Make sure `index.html`, `styles.css`, `firebase-config.js`, `js/` and `public/` (if used) are included in the deployed root.

## Notes and limitations

- The sample transactions and profile are demo records from the supplied project. Use **Profile & preferences → Reset sample data** to restore the sample workspace at any time; remove demo rows before treating a report as personal financial records.
- CSV and TXT are parsed in the browser. Other file formats are not parsed by this version. Do not upload bank statements containing unnecessary personal identifiers.
- The previous project had server-side Gemini endpoints for transaction explanations and AI-generated monthly briefs. Those endpoints were not carried into this static-only build, because a Gemini API key must not be exposed in browser JavaScript. The replacement monthly coach note is deterministic and uses only recorded figures. No AI response is claimed.
- The cash-flow simulator and safe-to-spend figure are planning estimates, not financial advice or a bank balance guarantee.
- Firebase and Supabase usage is subject to the quota and billing terms of the projects you configure. This repository itself does not add any paid third-party dependency.
