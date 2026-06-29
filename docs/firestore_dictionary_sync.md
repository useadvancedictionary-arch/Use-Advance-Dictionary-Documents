# Firebase Firestore dictionary sync

The app can sync dictionary entries through **Firestore** instead of (or alongside) the self-hosted HTTP sync server. When **Use Firebase Firestore** is enabled in **Multi-device sync**, the HTTP URL is ignored for sync. **New installs default to Firestore on** (the setting is only `false` if an admin turns the switch off and saves).

## Security model

- **Reads:** Only **signed-in** Firebase users may read `dictionary_entries` (`allow read: if request.auth != null` in `firebase/firestore.rules`). The app signs in **anonymously** on read-only devices when needed so users do not have to type credentials.
- **Writes:** Only users whose UID has a document in **`sync_admins`** with `{ "active": true }` may create/update/delete entries. Admins use **Firebase Authentication** (email/password) in the Multi-device sync dialog.

## One-time Firebase Console setup

1. In [Firebase Console](https://console.firebase.google.com/), open project **`use-advance-dictionary`** (or your project).
2. Enable **Firestore** (Native mode).
3. **Authentication** → Sign-in method:
   - Enable **Email/Password** (for admins).
   - Enable **Anonymous** (required so read-only devices can pull the dictionary without an account).
4. Deploy rules from this repo (install [Firebase CLI](https://firebase.google.com/docs/cli#install_the_firebase_cli)):

   ```bash
   cd use_advanced_dictionary_app
   firebase deploy --only firestore:rules
   ```

   Or paste the contents of `firebase/firestore.rules` into **Firestore → Rules** and **Publish**.

5. **Create admin Auth users:** Authentication → Users → Add user (email + password) for each person who may edit the dictionary from devices.

6. **Allow those users to write:** Firestore → collection **`sync_admins`** → for each admin user, add a document whose **Document ID** equals that user’s **UID** (from Authentication → Users → click user → UID). Field: **`active`** (boolean) = `true`.

7. **Flutter `firebase_options.dart`:** Android values are filled from `google-services.json`. For **Web** or **Windows**, add those apps under Project settings and run:

   ```bash
   dart pub global run flutterfire_cli:flutterfire configure
   ```

   Then replace `lib/firebase_options.dart` if needed.

   **Web / Netlify:** Register a **Web** app in the same project (Project settings → Your apps → **Add app** → Web). Copy its **App ID** (it contains `:web:`). The Firebase JS SDK usually **rejects** the Android App ID in the browser, which shows as “Firebase did not start” in **Multi-device sync**. Either paste that Web App ID into `lib/firebase_options.dart` (see `String.fromEnvironment('FIREBASE_WEB_APP_ID', ...)` / FlutterFire output), or set Netlify **Environment variable** `FIREBASE_WEB_APP_ID` to that value; `scripts/netlify-build.sh` passes it into `flutter build web` with `--dart-define`.

8. **Web (Netlify / custom domain):** In Firebase Console → **Authentication** → **Settings** → **Authorized domains**, add your hosting domain (e.g. `use-advance-dictionary.netlify.app` and any custom domain). Without this, sign-in and Firestore-backed sync can fail only in the browser.

9. **Web + HTTP sync:** The app on **https://** cannot call an **http://** sync server (mixed content). Prefer **Firestore** for web, or host the sync API at **https://** with CORS (see `docs/dictionary_sync_public.md`).

## On each device

1. **Readers (no edits):** Turn on **Use Firebase Firestore**, **Save**, then **Sync now**. The app signs in **anonymously** automatically if nobody is signed in yet (Anonymous must be enabled in Console).
2. **Admins:** Turn on Firestore, enter Firebase **email** / **password**, tap **Sign in**, then **Save** / **Sync now**. (Signing in replaces an anonymous session.)

If pulls fail with “permission denied”, check that **Anonymous** is enabled, rules are published, and the device has network access.

If uploads fail with “permission denied”, the signed-in UID is missing from **`sync_admins`** or **`active`** is not `true`.

### Web: `FirebaseCoreHostApi.initializeCore` / “channel-error”

If the sync dialog shows a **PlatformException** mentioning **`FirebaseCoreHostApi.initializeCore`** or **Unable to establish connection on channel**, the browser is using the wrong Firebase wiring (not your App ID). That usually means the generated **`web_plugin_registrant.dart`** did not register **`firebase_core_web`** — often from a **stale** `.dart_tool/flutter_build` after adding or upgrading Firebase packages.

**Fix locally:** from the app repo root, run `flutter clean`, then `flutter pub get`, then `flutter build web --release` and redeploy.

**Fix on Netlify:** ensure each build removes stale plugin output before `flutter build web` (this repo’s `scripts/netlify-build.sh` deletes `.dart_tool/flutter_build` before the web build). Clear the Netlify build cache once if the problem persists after a config change.

## Data layout

- Collection **`dictionary_entries`**
- Document ID = string form of numeric word `id`
- Fields mirror the app map: `word`, `englishMeaning`, …, `deleted` (bool), `updatedAt` (server timestamp)

Large dictionaries: full collection reads on every sync may be heavy; you can optimize later with queries or incremental sync.
