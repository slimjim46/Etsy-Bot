# Setup Guide (developer)

## A. Etsy developer app
1. Log in to Etsy (the shop owner's account or an account with access) and go to developers.etsy.com → "Create a New App".
2. Fill name/description; choose personal/non-commercial use if offered (it is a tool for one shop).
3. Add the redirect URI: `${PUBLIC_BASE_URL}/callback/etsy`.
4. Note the **API key (keystring)** → `ETSY_API_KEY`. Note any shared secret if shown.
5. Check app status and rate limits. If access is pending approval, record that here and plan around it.
6. Scopes needed (expected): `listings_r`, `shops_r`. Add more only if a feature requires it.

## B. Meta developer app
1. developers.facebook.com → Create App. Use case: Instagram API / content publishing (names change; follow current wizard).
2. Add the Instagram product (Instagram Login or Facebook Login, per current docs; pick one and note it in `docs/PLAN.md`).
3. Keep the app in **Development mode**.
4. App Roles: add mom as **Instagram Tester** (or Tester/Developer per current UI). She must accept the invite.
5. Add redirect URI: `${PUBLIC_BASE_URL}/callback/instagram`.
6. **Privacy Policy URL** is required: host `docs/PRIVACY_POLICY.md` (filled in) as a static page and paste the URL.
7. Note App ID/Secret → `META_APP_ID`, `META_APP_SECRET`.
8. Permissions needed (expected): `instagram_business_basic`, `instagram_business_content_publish` (add comment/insights permissions for phase 2). Names depend on login flow; verify.

## C. Mom's Instagram account
- Convert to **Business** or **Creator** (Settings → Account type and tools).
- With the Instagram Login flow, a linked Facebook Page may not be required; with Facebook Login it is. Verify current docs.

## D. Host
Follow `DEPLOYMENT.md`. Get a stable HTTPS URL first; both OAuth redirects depend on it.

## E. Mom's one-time steps (~5 minutes)
Send her these instructions:
1. Confirm your Instagram is a Business or Creator account (I'll help if not).
2. Accept the tester invite from Meta (email/notification).
3. Open the link I send for **Etsy**, log in, click **Allow**.
4. Open the link I send for **Instagram**, log in, click **Allow**.
5. Done. Nothing to install.

## F. Go-live checklist
- [ ] `DRY_RUN=true`, mode `approval`: verify seeding posts nothing.
- [ ] Add a test listing (draft → active) and confirm it appears in `/approvals` with a good caption and images.
- [ ] Approve one; confirm it publishes once and status becomes `posted`.
- [ ] Kill/restart the service; confirm no duplicates.
- [ ] Force a failure (bad token); confirm an alert arrives.
- [ ] Confirm token refresh works (simulate near-expiry).
- [ ] After 1–2 weeks of good approvals, set `POSTING_MODE=auto`.
