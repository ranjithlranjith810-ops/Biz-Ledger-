# BizLedger Pro — Security Setup Guide

## ⚠️ GitHub Secret Alert: What It Means

GitHub flagged your **Firebase API key** in `index.html`. 

**This is expected for Firebase web apps.** Unlike backend secrets (passwords, private keys), Firebase web API keys are designed to be public — they identify your project, not authenticate it. Security comes from **Firebase Rules** and **API key restrictions**, not hiding the key.

However, you must complete the steps below to prevent unauthorized use.

---

## ✅ Step 1: Restrict Your API Key (Google Cloud Console)

This prevents anyone from using your key outside your website.

1. Go to → **https://console.cloud.google.com/apis/credentials**
2. Find your API key (named something like `Browser key (auto created by Firebase)`)
3. Click **Edit**
4. Under **Application restrictions** → select **HTTP referrers (websites)**
5. Add these referrers:
   ```
   https://ranjithlranjith810-ops.github.io/*
   https://ranjithlranjith810-ops.github.io/Biz-Ledger-/*
   localhost
   ```
6. Click **Save**

After this, the key will **only work from your website** — nobody else can use it.

---

## ✅ Step 2: Set Firebase Realtime Database Rules

This ensures only logged-in users can read/write their own data.

1. Go to → **https://console.firebase.google.com**
2. Select your project **vriksha-b8581**
3. Go to **Realtime Database → Rules**
4. Replace the rules with the contents of `database.rules.json` (included in this repo)
5. Click **Publish**

The rules ensure:
- Only authenticated users can access data
- Users can only read/write their own company's data
- No public/unauthenticated access

---

## ✅ Step 3: Enable Firebase Auth Authorized Domains

1. Go to **Firebase Console → Authentication → Settings → Authorized domains**
2. Make sure your GitHub Pages domain is listed:
   ```
   ranjithlranjith810-ops.github.io
   ```
3. Remove `localhost` if you're done with development

---

## ✅ Step 4: Dismiss GitHub Alert

After completing Steps 1–2, the key is properly secured. You can:
- Go to your repo → **Security → Secret scanning alerts**
- Mark the alert as **"Used in tests"** or **"False positive"**
- GitHub will stop sending alerts for this key

---

## Why the Key Stays in the Code

Firebase web apps **require** the config in the frontend JavaScript. This is by design — Firebase's security model relies on:
1. **Authentication** — only signed-in users get access
2. **Security Rules** — server-side rules block unauthorized reads/writes
3. **API key restrictions** — key only works from your domain

This is the same approach used by major apps built on Firebase.

---

## Current Security Status

| Check | Status |
|---|---|
| Firebase Auth required to access data | ✅ Implemented in app |
| Database Security Rules | ⚠️ Apply `database.rules.json` |
| API Key domain restriction | ⚠️ Do Step 1 above |
| Auth authorized domains | ⚠️ Do Step 3 above |
