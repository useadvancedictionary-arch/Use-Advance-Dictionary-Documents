# GitHub Pages (optional)

The app’s main web host is usually **Netlify** (`netlify.toml` + `scripts/netlify-build.sh`).  
This repo also has a GitHub Actions workflow that can publish the same Flutter web build to **GitHub Pages**.

## Why the workflow “failed” before

On recent runs, **build succeeded** but **deploy failed in a few seconds** because **GitHub Pages was never turned on** for the repository. The Pages API returns 404 until you enable it.

That does **not** break Netlify or Play Store builds.

## One-time enable (to stop deploy errors and publish on GitHub)

1. Open the repo on GitHub: **Settings** → **Pages**.
2. Under **Build and deployment**, set **Source** to **GitHub Actions** (not “Deploy from a branch”).
3. Push to `main` or run **Actions** → **Deploy Flutter web (GitHub Pages)** → **Run workflow**.

Public URL (user/org site):

`https://<your-github-username>.github.io/Use-Advance-Dictionary-App/`

## Firebase on GitHub Pages (optional)

Default Web App ID is already in `lib/firebase_options.dart`. To override without editing code:

- Repo **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
- Name: `FIREBASE_WEB_APP_ID`
- Value: your Firebase **Web** app ID (contains `:web:`)

Also add your GitHub Pages hostname under Firebase **Authentication** → **Authorized domains** if you use sign-in/sync on that URL.

## If you only use Netlify

You can leave Pages disabled. The workflow will **build** and **skip deploy** with a notice instead of sending a failure email.
