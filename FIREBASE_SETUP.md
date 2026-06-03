# Firebase setup — college-only sign-in + form collection

The website now gates the "Apply" form behind **Google Sign-In** and only lets
verified **@raghuenggcollege.in** accounts submit. Submissions are written to
**Cloud Firestore**.

You only need to do the Firebase Console steps below **once**. The code in
`index.html` is already wired up — you just paste your project config and set
the security rules.

---

## 1. Create a Firebase project

1. Go to <https://console.firebase.google.com> and sign in with a college admin
   Google account.
2. Click **Add project** → name it e.g. `raghu-alp` → continue (Google Analytics
   is optional, you can disable it).

## 2. Register a Web app and copy the config

1. In the project, click the **`</>` (Web)** icon ("Add app").
2. Nickname it e.g. `ALP website`. **Do not** enable Firebase Hosting (you're on
   GitHub Pages). Click **Register app**.
3. Firebase shows a `firebaseConfig = { ... }` object. Copy those six values.
4. Open `index.html`, find the block marked
   `▼▼▼ REPLACE with your Firebase project config ▼▼▼` (near the bottom, inside
   the `<script type="module">`), and paste your real values over the
   `"REPLACE_ME"` placeholders.

> These config values are **not secret** — they're meant to ship in the browser.
> Security is enforced by the rules in step 5, not by hiding the config.

## 3. Enable Google Sign-In

1. Console → **Build → Authentication → Get started**.
2. **Sign-in method** tab → **Google** → toggle **Enable** → pick a support
   email → **Save**.

## 4. Add your authorized domains

Console → **Authentication → Settings → Authorized domains → Add domain**.
Add each of these (the sign-in popup is blocked on any domain not listed):

- `localhost`  *(usually already present — covers `localhost:3000`; the port is ignored)*
- `alp.raghuengineering.com`  *(your production domain)*
- your GitHub Pages domain if you also use it, e.g. `<user>.github.io`

> **Restrict to one Workspace domain (optional, stronger):** if
> `raghuenggcollege.in` is a Google Workspace domain, you can also go to the
> Google Cloud Console OAuth consent screen and set the app to *Internal* so
> only that Workspace can even attempt sign-in. The Firestore rule in step 5 is
> the real guarantee either way.

## 5. Create Firestore and set the security rules  ← **most important step**

1. Console → **Build → Firestore Database → Create database**.
2. Start in **production mode** (locked), pick a region close to you
   (e.g. `asia-south1` / Mumbai), → **Enable**.
3. Go to the **Rules** tab, replace everything with the rules below, → **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Each student's application document id IS their auth uid, so there can
    // only ever be ONE document per student — duplicates are impossible.
    match /applications/{userId} {

      // A student may read back their OWN application (used to pre-fill the
      // form when they sign in again).
      allow read: if request.auth != null
        && request.auth.uid == userId;

      // A VERIFIED @raghuenggcollege.in student may create or update ONLY
      // their own application. The doc id must equal their uid, and the
      // email/uid inside must match the signed-in user, so nobody can spoof
      // another address or write to someone else's record.
      allow create, update: if request.auth != null
        && request.auth.uid == userId
        && request.auth.token.email_verified == true
        && request.auth.token.email.matches('.*@raghuenggcollege[.]in')
        && request.resource.data.email == request.auth.token.email
        && request.resource.data.uid   == request.auth.uid;

      // Nobody can delete via the client. Manage/delete from the Console.
      allow delete: if false;
    }
  }
}
```

This is what makes "**strictly @raghuenggcollege.in only, no duplicates**" real
— even if someone edits the website's JavaScript in their browser, Firestore:
- rejects any write that isn't from a verified college account, and
- forces every student into a single document keyed by their uid, so a second
  submission **overwrites** the first instead of creating a duplicate.

---

## 6. Test it

1. Run locally: from this folder, `python -m http.server 3000`
   (or any static server) → open <http://localhost:3000>.
2. Click **Apply** → the form is locked behind **Sign in with college account**.
3. Sign in with a **@raghuenggcollege.in** account → the form appears with the
   email pre-filled and locked. Submit → you see the success screen.
4. **Sign in again with the same account** → the form now says "You've already
   registered" and is **pre-filled with your previous answers**. Change
   something and save → it **updates the same record** (no duplicate row).
5. Try a **personal Gmail** account → it's rejected with an error and the form
   stays locked.
6. Console → **Firestore Database → Data** → open the `applications` collection.
   Each document id is the student's uid, so there is exactly **one document per
   student**.

## Viewing / exporting submissions

- Browse them anytime under **Firestore Database → Data → `applications`**.
- To export: Console → Firestore → (⋯) **Import/Export**, or use the
  `gcloud firestore export` CLI.

## Notes & gotchas

- **Shared-device safety:** sign-in uses **session-only persistence**, so a
  student is signed out automatically when they close the tab/browser. On shared
  lab PCs the next person starts signed-out. The tradeoff is that students sign
  in again (one Google click, no password) on each visit. They can also sign out
  manually via the "Signed in as … · Sign out" bar at the top of the form.
- **Free tier (Spark plan)** is plenty for collecting form submissions.
- `localhost` without HTTPS is fine for sign-in during development. Production
  (`alp.raghuengineering.com`) must be served over **HTTPS** — GitHub Pages with
  a custom domain gives you that automatically (enable *Enforce HTTPS* in the
  repo's Pages settings).
- If sign-in fails with `auth/unauthorized-domain`, you missed the domain in
  step 4.
- The form's email field is auto-filled from the signed-in account and is
  read-only, so the stored email always matches the verified identity.
