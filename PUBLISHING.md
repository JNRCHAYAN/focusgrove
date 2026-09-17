# Publishing FocusGrove

Everything that can be done from code is done. What is left is the part that
needs accounts, secrets, and a human — this file walks through it in order.

Steps 1 and 2 finish work already started in the repo. Steps 3–6 are the Play
Console submission itself. Skip to whatever you need.

---

## 1. Sign the release build

`android/app/build.gradle.kts` already reads `android/key.properties` when it
exists and falls back to the debug key when it does not. So today,
`flutter build apk --release` works and produces an APK you can sideload for
testing — but Play will reject it, because it is signed with the debug key.

### a. Generate your upload keystore

Run this **yourself**, from the project root. It is the one step nobody else
should do for you: the password you type here is the password that proves an
update is genuinely from you, and it should never leave your hands — not into
a chat, not into a file you commit.

```powershell
& "C:\Program Files\Android\Android Studio\jbr\bin\keytool.exe" -genkey -v `
  -keystore android/app/upload-keystore.jks `
  -storetype JKS `
  -keyalg RSA -keysize 2048 -validity 10000 `
  -alias upload
```

It will prompt for:

| Prompt | What to enter |
| --- | --- |
| Enter keystore password | Choose one. **Write it down somewhere safe.** |
| Re-enter new password | Same again. |
| What is your first and last name? | Your name, or your studio's name. |
| What is the name of your organizational unit? | Anything, e.g. `FocusGrove`. |
| What is the name of your organization? | Your name or studio. |
| What is the name of your City or Locality? | Your city. |
| What is the name of your State or Province? | Your state. |
| What is the two-letter country code for this unit? | e.g. `BD`, `US`, `IN`. |
| Is CN=… correct? | `yes`. |
| Enter key password for `<upload>` | Press **Enter** to reuse the keystore password. |

> `-validity 10000` is ~27 years, longer than any app store requires. Play
> wants an RSA key of at least 2048 bits, which is what `-keysize 2048` gives.

### b. Point the build at it

```powershell
Copy-Item android\key.properties.example android\key.properties
```

Then open `android/key.properties` and replace the two placeholders with the
password you just chose. Leave `keyAlias=upload` and
`storeFile=upload-keystore.jks` as they are if you ran the command above
unchanged.

That file is gitignored, along with `*.jks` and `*.keystore`. Confirm that
before you commit anything:

```powershell
git status --short android/     # key.properties and the .jks must NOT appear
```

### c. Back the keystore up — this is not optional

Put `android/app/upload-keystore.jks` and its password somewhere you will
still have them in five years: a password manager entry, an encrypted backup,
a second drive. If you lose them you cannot ship an update to an app that is
already live. Your only escape would be publishing a brand-new listing under a
new application id, and every existing install would be stranded on the old
one forever.

---

## 2. Host the privacy policy

Play Console requires a public URL for the privacy policy, and the app's
Settings screen links to the same URL. The page itself is written and needs no
editing beyond two placeholders — `docs/privacy-policy.html`.

### a. Fill in the placeholders

Open the file and replace these three, all highlighted on the rendered page so
they are hard to miss:

- `<mark>your-email@example.com</mark>` — a contact address you actually
  monitor. Play reviewers do use it.
- `<mark>Your Name or Studio</mark>` — the developer name you register on Play
  (your legal name for a personal account).
- `<mark>13 September 2026</mark>` — the `Last updated:` date. Update it
  whenever you edit the policy; it is already correct as written.

### b. Publish it with GitHub Pages

Push this repo to GitHub, then on the repository page:

**Settings → Pages → Source: `Deploy from a branch` → Branch: `main`,
folder: `/docs` → Save**

A minute or two later the policy is live at:

```
https://<your-username>.github.io/<repo-name>/privacy-policy.html
```

The `docs/` folder is exactly why the file lives there: it is the one folder
GitHub Pages can serve without extra configuration, and it keeps the page in
the same repo as the app it describes.

### c. Point the app at it

One line, in `lib/core/constants/app_constants.dart`:

```dart
static const String privacyPolicyUrl =
    'https://<your-username>.github.io/<repo-name>/privacy-policy.html';
```

That is the whole change. It is deliberately empty today, so the Settings
tile currently explains that the policy is not yet published rather than
sending users to a dead link. Once this line is filled in, the tile opens the
real page and Play Console accepts the same URL.

Rebuild after editing — the URL is compiled into the app.

---

## 3. Build the release bundle

Play takes an `.aab`, not an `.apk`. `--obfuscate --split-debug-info` shrinks
the binary and keeps a symbol file you can use to read a stack trace later.

```powershell
flutter build appbundle --release `
  --obfuscate `
  --split-debug-info=build/debug-info
```

Output: `build/app/outputs/bundle/release/app-release.aab` (57.6 MB).

> **Ignore the `--strip` warning.** The build prints
> `Warning: The generated ELF library contains unobfuscated DWARF debugging
> information. To avoid this, use --strip` several times. Checked directly
> against a built bundle: all 15 native libraries in it are already stripped
> and contain no `.debug_info`, `.debug_str`, or `.debug_line` sections. Adding
> `--strip` changes nothing except how long the build takes. Do check it again
> if you add a plugin with native code, because for that plugin it might be
> real.

Keep `build/debug-info/` somewhere permanent — it is gitignored, and without
it an obfuscated stack trace from a crash report is unreadable. It is also
**per build**: symbols from release 1.0.0 will not symbolicate 1.0.1. Archive
the folder next to the version it belongs to.

To verify the whole thing compiles and installs on a real device first:

```powershell
flutter build apk --release
flutter install --release
```

An APK is for your own testing only — never upload one to Play.

---

## 4. Play Console submission

1. **Account** — a Google Play developer account, one-time registration fee.
   New *personal* accounts must run a closed test with a minimum number of
   testers for a minimum number of days before they can reach Production.
   Budget for that delay; it is the single most common surprise.
2. **Create the app** — application id `com.jnrchayan.focusgrove`, name
   `FocusGrove`, type App, Free.
3. **Store listing** — copy is ready in [../store/listing.md](../store/listing.md).
   Upload `store/play-icon-512.png` and
   `store/feature-graphic-1024x500.png`; both are generated by
   `tool/generate_icons.py`, so they are original artwork and match the app
   icon exactly. You still need to supply phone screenshots — take them on a
   real device or an emulator with the app running.
4. **Data safety** — answer **"No data collected"** and **"No data shared"**.
   This is not a judgement call. The release build requests no network
   permission at all, so it is incapable of transmitting anything, and
   everything else on the form follows from that answer.

   > Do not be thrown by `INTERNET` appearing in
   > `android/app/src/debug/AndroidManifest.xml`. Flutter puts it there so the
   > tool can talk to a running app for hot reload. The `debug/` and
   > `profile/` manifests are not merged into a release build — check
   > `android/app/src/main/AndroidManifest.xml`, which is the only one that
   > ships, and you will find `POST_NOTIFICATIONS` and nothing else.

5. **Privacy policy** — paste the GitHub Pages URL from step 2.
6. **Permissions** — `POST_NOTIFICATIONS` should be the only entry, and it
   should be declared as requested in context. If the console lists anything
   else, stop and check `android/app/src/main/AndroidManifest.xml`.
7. **Content rating** — complete the questionnaire. FocusGrove has no user
   content, no ads, no purchases, and no data sharing, so it lands in the
   lowest rating band.
8. **Upload** — the `.aab` from step 3 to **Internal testing** first. Install
   it from the opt-in link on a real device and run through a session before
   promoting anything.
9. **Promote** — Internal → Closed → Production, with a staged rollout. Start
   low; you can always increase the percentage, but you cannot un-ship a bad
   build to people who already have it.

**Play App Signing** is enabled automatically on your first upload. Google
holds the app signing key and yours becomes the *upload* key — which is why
losing your keystore is a serious problem but not always a fatal one. Keep the
backup anyway.

---

## 5. Before you submit — a self-check

Work through BUILD_SPEC.md §11. The boxes that are already done in code:

- [x] Only permission is `POST_NOTIFICATIONS`, requested contextually on first
      session completion
- [x] `targetSdk` 37 (above Play's current minimum)
- [x] Loading, empty, error, and success states on Forest and Stats
- [x] 8dp spacing grid, one type scale, one color scheme, light and dark
- [x] Touch targets ≥48dp, semantic labels on icon-only controls
- [x] Onboarding is 3 screens and skippable
- [x] Icon and feature graphic are original generated art
- [x] No ads, no analytics, no crash reporting, no network permission
- [x] App name is 10 characters and describes the app
- [x] Release signing wired (activates once `key.properties` exists — step 1)

The boxes that need you:

- [ ] Upload keystore generated and backed up (step 1)
- [ ] Privacy policy hosted and `privacyPolicyUrl` filled in (step 2)
- [ ] Data Safety form completed honestly (step 4.4)
- [ ] Screenshots captured on a real device
- [ ] Tested on a real device at compact-phone, large-phone, and tablet widths
- [ ] Play Console account, listing, and release tracks configured
- [ ] Internal test run before any Production rollout

---

## Appendix — which file does what

| File | Role |
| --- | --- |
| `android/key.properties` | Your keystore passwords. Gitignored. Never commit. |
| `android/key.properties.example` | The template. Safe to commit. |
| `android/app/upload-keystore.jks` | The upload key itself. Gitignored. Back it up. |
| `docs/privacy-policy.html` | The policy. Served by GitHub Pages. |
| `lib/core/constants/app_constants.dart` | Where the published policy URL goes. |
| `store/listing.md` | Store listing copy, within Play's length limits. |
| `store/play-icon-512.png` | Store icon (512×512). |
| `store/feature-graphic-1024x500.png` | Feature graphic (1024×500). |
| `tool/generate_icons.py` | Regenerates every icon and both store images. |
| `tool/generate_ambient_audio.py` | Regenerates `assets/audio/*.mp3`. |
