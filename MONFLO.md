# Monflo — Ubuntu Touch App

Project overview, how the app works internally, how to run it, and the results of a deep health check. Last checked 2026-10-16 (re-verified against code, OpenStore and monflo.com) on branch `MOB-47-report-and-transaction-api-integration`.

---

## 1. What it is

**Monflo** (`monflo.monflo`) is the Ubuntu Touch (Lomiri) client for the Monflo fintech platform: an EUR IBAN account combined with a crypto wallet, an exchange (swap) aggregator, Gnosis cards, and paid subscriptions. It is written in **QML/JavaScript on Qt 5.12** (the UT 20.04 runtime), with a small **C++ host and plugin layer**, and is packaged as a `.click` with **Clickable**.

| Item | Value |
|---|---|
| App ID | `monflo.monflo` (push app id: `monflo.monflo_monflo_stg` on staging) |
| Framework | `ubuntu-sdk-20.04`, Qt 5.12.8 on device, Clickable ≥ 8.0 |
| Languages | QML / JS (~31k lines incl. C++), C++11 |
| Qt modules | Core, Gui, Qml, Quick, QuickControls2, WebEngine, Network, DBus |
| License | GPL-3.0 |
| Maintainer | Sakal Samnang |
| CI | Azure Pipelines → `.click` for amd64 / arm64 / armhf |

### Backends

| API | Staging | Production | Used for |
|---|---|---|---|
| Transaction API | `staging-api.transaction.monflo.com/api/v1` | `api.transaction.monflo.com/api/v1` | balances, transfers, exchange, reports, contacts, subscriptions, notifications |
| Onboarding / Auth API | `staging-api.onboarding.monflo.com/api/v1` | `api.onboarding.monflo.com/api/v1` | login, password reset, logout, token refresh, user information |
| Notabene | `api.notabene.dev` | (same) | VASP (counterparty exchange) info for crypto transfers |
| CoinGecko / AnyChart CDN | — | — | market sparklines (WebEngine, needs network) |

---

## 1.1 The Monflo product (monflo.com) and the OpenStore release

### Monflo the service — [monflo.com](https://www.monflo.com)
This app is one client of the Monflo platform. The website describes the service like this (checked 2026-10-16):

| Topic | What monflo.com says |
|---|---|
| Pitch | "One platform to earn, spend, swap, or save" · "Your money, your RULES" |
| Company | Founder/CEO Chhay Lin Lim, Apeldoorn (NL) |
| Web3 IBAN account | Personal IBAN holding EUR + crypto. The smart IBAN is issued by **Monerium** (EUR on/off-ramp via EURe) |
| Exchange aggregator | Swaps from the IBAN account using quotes from 15+ DEX markets → the app's **Swap / Exchange** screen |
| Personal vault | No pooled/omnibus accounts. Funds can be verified on-chain. Custody/wallet infra by **Fireblocks** |
| Debit card | Gnosis Pay card, issued by Monavate (Visa) → the app's **CardReport** / `/gnosis/*` endpoints |
| Earn | Coming soon (not in this app) |
| Fees | EUR transfers 0% · swap 0.45% (promo until 2027-01-01, normally 0.9%) · crypto deposit free (min €5) · crypto withdrawal 0.1% (min €3, max €20) |
| Subscription | €3.99/month (promo, normally €7.99) with a 30-day free trial → the app's **Subscription** screen (`/subscriptions/current`, grace period, retry payment, cancel) |
| Eligibility | Netherlands residents with a Dutch IBAN + iDIN. Businesses need a KvK registration + UBO info. EU expansion is planned |
| Regulation | MiCA CASP licence (AFM conduct, DNB prudential). Wwft and the Sanctions Act 1977 apply |
| Support | Zendesk help centre, Telegram `t.me/monfloxyz`, X `@MonfloFinance` |

The app links back to the website from Profile:
- `LegalSection.qml`: `/fees`, `/terms`, `/privacy-policy`, `/sustainability-indicator`, plus Conflicts-of-Interest, Complaints and Risk-Disclosure PDFs on Azure blob storage
- `HelpSection.qml`: `monflo.zendesk.com/hc/en-nl`

The website FAQ mentions an open-source Ubuntu version (this repo is GPL-3.0).

### Monflo for Ubuntu Touch — [OpenStore listing](https://open-store.io/app/monflo.monflo/)
| Item | OpenStore (2026-10-16) | This repo |
|---|---|---|
| App ID | `monflo.monflo` | `monflo.monflo` ✅ |
| Published version | **1.1.4** (updated 2026-03-11, first published 2025-07-15) | stg builds are `1.0.<commit count>` (now 1.0.150). Prod uses the latest git tag, but **no tags exist in the local clone**, so `clickable script prod` fails here. Fetch tags (`git fetch --tags`) before building a release |
| Architectures | **arm64 only** | CI builds amd64 / arm64 / armhf. Only arm64 is uploaded |
| Category | Business & Finance | — |
| Permissions | Networking, Push Notifications, Webview, Connectivity, Content Exchange, Content Exchange Source | matches `monflo.apparmor` ✅ |
| Publisher | Sakal | maintainer `Sakal Samnang <sakal@monflo.com>` |
| Downloads | 524 total, 166 for 1.1.4 | — |
| License / source | Shown as **not open source**. The "Source code" link points to www.monflo.com | `LICENSE` is GPL-3.0 ❌ mismatch |
| Changelog | "Bug fixes" | — |
| Screenshots | **none** | — |
| Description | Matches monflo.com: Web3 IBAN, EUR/crypto, EURe, DEX for 20+ coins, Fireblocks. It cites DNB registration **R194558** | website now says **MiCA CASP (AFM)** ❌ outdated |

**Store listing to-dos**
- Set the license to GPL-3.0 and point "Source code" to the public repo. Or, if the repo is not public, reconsider the GPL claim and the FAQ statement.
- Update the regulatory sentence from the old DNB registration to the MiCA/AFM licence.
- Add screenshots and a real changelog.
- Note in the description that the service is NL-only. A store review complains about this.
- Make sure the release build is the **prod** env with the **prod** `APP_ID`. Today `ApiController.js` is hardcoded to staging (bug #1 below), so a store build made from this branch would talk to staging.

---

## 2. How Monflo works

### 2.1 Architecture

```
┌──────────────────────────── main.cpp (C++) ─────────────────────────────┐
│ QGuiApplication + QQuickView                                            │
│ registers QML singletons:  App { Theme, FontLib, AppState, AppIcon,     │
│   Constant, AppImage, FeatureToggler, AppConfig }                       │
│   AppStorage { LocalStorage }   App.Components { CustomButton }         │
│ loads qrc:/Main.qml  (all QML/JS is compiled into the binary via qrc)   │
└─────────────────────────────────────────────────────────────────────────┘
          │                                   ▲
          ▼                                   │ import Controllers 1.0
┌─────────────── QML UI layer ───────────┐  ┌─┴──── C++ plugin "Controllers" ────┐
│ Main.qml → StackNavigation.qml          │  │ DeviceInfo   (/etc/machine-id)     │
│   └ StackView of pages (qml/ui/*)       │  │ FileUploader (multipart transfer   │
│ components/  reusable widgets           │  │   with supporting documents)       │
└──────────────┬─────────────────────────┘  └────────────────────────────────────┘
               │ .import JS
┌──────────────▼─────────────── JS controller layer ─────────────────────────────┐
│ controllers/auth, assets, profile, report, notification  (one fn per endpoint) │
│        │                                                                       │
│ ApiCache.js ── cache-or-fetch ──► DataStore.js (in-memory cache + listeners)   │
│        │                                                                       │
│ ApiController.js  sendRequest(): XMLHttpRequest, Bearer token, 401 → refresh   │
└──────────────┬─────────────────────────────────────────────────────────────────┘
               ▼
   Monflo Transaction API / Onboarding API (HTTPS, JSON)

Persistent state: storage/Storage.qml → SQLite key/value table "kv" in DB "MyAppDB"
   keys: accessToken, refreshToken, accessTokenExpiry, profile, client (decoded JWT), email
```

**Layers, in short:**
- **Pages** (`qml/ui/<Feature>/<Feature>.qml`) each have a `pageId` string, hold view state, and call controller functions with callbacks.
- **Controllers** (`qml/controllers/**.js`) are thin wrappers, one function per REST endpoint. Read endpoints go through `ApiCache.get(key, url, cb, forceRefresh)`. Write endpoints call `API.sendRequest()` directly and then **invalidate** the affected cache keys (for example, a transfer invalidates `fiatBalance`, `cryptoBalance`, `availableAssets` and `fiatTxHistory`).
- **DataStore** (`.pragma library`, so it is shared by every importer) is an in-memory cache with `subscribe(key, cb)` / `set` / `invalidate`, so widgets can react to fresh data. It is cleared on logout and when the token refresh fails.
- **Utils / parsers** turn raw API JSON into view models: `ReportParser`, `AssetParser`, `NotificationParser`, `BalanceFormatter`, `ErrorMapper` (backend error code → user message), and `Validation`.
- **AppState** (singleton) is the global event bus. It holds the `stackView` reference and the `openModal`, `openLoading` and `loginSuccess` signals, plus transient transfer state (`pendingTranfer`, `selectedDocuments`, …).

### 2.2 Startup sequence

1. `main.cpp` sets up `XDG_*` directories when running in Clickable desktop mode, registers the singletons, and loads `Main.qml`.
2. `Main.qml` (`MainView`) creates:
   - `PushClient` (appId = `AppConfig.appId`), which turns incoming push messages into an in-app `NotificationPopUp`
   - `SplashScreen` (about 1 s)
   - the global dark `overlay` and the `Loading` spinner, driven by `AppState.openLoading`
   - `pageLoader` → `StackNavigation.qml`
   - `NavBar`
3. When the splash times out, the page loader becomes visible. `StackNavigation`'s `StackView` starts with **Home**. If `LocalStorage.get("accessToken")` is null, it immediately does `replace(Login)`.
4. Every page change emits `currentPageChanged(pageId)`. `Main.qml` shows the bottom NavBar only when `pageId` is one of `Home, Markets, Portfolio, Investment`.

### 2.3 Authentication

```
Login / FirstStep           Auth.checkCredentials(email, pw)   POST onboard /auth/check-user-credential
   │ ok → push SecondStep
Login / SecondStep (2FA)    Auth.login(email, pw, otp)         POST onboard /auth/login
   │ ok → LocalStorage: accessToken, refreshToken, profile, accessTokenExpiry(+18 min),
   │      client = Utils.decodeAccessToken(jwt), email
   └ AppState.emitLoginSuccess() → Login.qml replaces stack with Home
```
- **Forgot password:** EmailStep (`POST /auth/forgot-password`) → VerificationStep (enter the code) → NewPasswordStep (`POST /auth/reset-password`) → back to Login.
- **Every API call:** `sendRequest()` adds `Authorization: Bearer <accessToken>`. Transaction API calls always get it. Onboard calls get it only for `exceptionRoutes` (`/user-information…`).
- **Expired session:** on **HTTP 401** from the transaction API, `handle401WithRefresh()`:
  1. sets `_isRefreshing` and queues any other 401'd requests
  2. calls `GET onboard /auth/refresh-token` with headers `Authorization`, `x-refresh-token`, `Origin`, `user-platform: ubuntu-touch`
  3. on success, saves the new tokens and replays the original request plus the queue
  4. on failure, `clearAllTokens()` (SQLite + DataStore) and `replace(Login)`
- **Logout** (Profile → LogoutSection): `POST onboard /auth/logout`, then clears storage and cache (the email is kept for prefill) and replaces the stack with Login.

### 2.4 Navigation

- Global `StackView` (`AppState.stackView`). Pages use `push` (sub-pages: Profile, Transfer, Deposit, reports) and `replace` (top-level tabs, login/logout).
- **NavBar tabs** (`ui/NavBar/NavBarModel.js`): Home · Markets · **Swap** (Exchange, pushed) · Portfolio · Reports. Top-level tabs use `replace`, so the stack stays shallow.
- **Modals and bottom sheets** use `AppState.openModal(true)`, which blurs the background (`shouldBlur`) and opens the drawer.

### 2.5 Main features → endpoints

| Feature (screen) | What happens | Endpoints (transaction API unless noted) |
|---|---|---|
| **Home** | `_init()` loads 4 calls in parallel with `Promise.all`, then registers the push token | `/client/balances/crypto`, `/client/balances/crypto/all`, `/client/balances/fiat`, `/deposit/fiat/iban`, `POST /notifications/register`, `/unified-transactions/recent`, `/gnosis/cards` |
| **Markets / MarketDetail** | Asset list + detail; sparkline chart is an HTML page in WebEngine (AnyChart CDN) | `/assets?page&size`, `/assets/{id}` |
| **Exchange (Swap)** | Pick tokens and amount → auto-refreshing quote with countdown timer → confirm | `/exchange/assets/available`, `POST /exchange/quote`, `POST /exchange/send` |
| **Transfer** (SEPA out) | Individual/Business form → ConfirmTransferModal (2FA code) → JSON or multipart path (see 2.6). Can save the recipient to the address book. | `POST /withdraw/fiat` (header `x-2fa-code`), `/contacts` (GET/POST/PUT/DELETE `/contacts/{iban}`), `/client/transaction-limit/daily`, `/request` |
| **Deposit** | Bank (show IBAN/BIC), Wallet, or Asset deposit; share details | `/deposit/fiat/iban`, `/client/iban`; Notabene VASP lookup |
| **Report / WalletReport** | Tabs: All (unified), Fiat, Crypto, Exchange, Billing; date/status/amount filters; paging; detail views | `/unified-transactions[?…]`, `/unified-transactions/{type}/{id}`, `/fiat-transaction/history`, `/crypto-transfers`, `/exchange/history`, `/subscription/payments` |
| **CardReport** | Gnosis card transactions | `/gnosis/cards`, `/gnosis/transactions`, `/gnosis/transactions/history` |
| **Notification** | Paged inbox | `/notifications?page&limit` |
| **Profile** | Personal info, phone/email change (2FA), legal/help links, subscription plan, logout | onboard `/user-information`, `…/request-update-phone-number`, `…/verify-phone-number`; `/subscription-plans/detail` |
| **Subscription** | Current plan, grace period, retry payment, cancel (2FA) | `/subscriptions/current`, `POST /chargings`, `POST /subscriptions/cancel` |

### 2.6 Money transfer flow (most sensitive path)

```
Transfer.qml ──form──► ConfirmTransferModal (enter 2FA code)
                          │
          amount >= DOCUMENT_REQUIRED_THRESHOLD ?
             ├─ yes, no docs ─► error "upload supporting documents"
             ├─ yes, docs  ───► C++ FileUploader.transferWithDocuments()
             │                    multipart POST {base}/withdraw/fiat
             │                    fields: amount, iban, recipientType, reference, details[…], files
             │                    signals: progressChanged / uploadFinished / uploadError
             └─ no ──────────► docs attached ? FileUploader (as above)
                               : Assets.transfer() → JSON POST /withdraw/fiat
          on success: invalidate balance/history caches → transferDone(true)
```

### 2.7 Push notifications

- `manifest.json` declares a `push` hook. `pushexec` (Python) is the push helper; it copies the payload through unchanged, so **the server must send a Lomiri-format payload** (`{"notification": {"card": …}}`).
- In the app, `PushClient.notificationsChanged` parses each message and shows `NotificationPopUp` with `title` / `text`.
- After Home loads, the device push token is sent to `POST /notifications/register`.

### 2.8 Configuration & build-time environment

`scripts/build-env.sh` (run by `clickable script stg|prod` and by CI):
1. reads `.env.<env>` (`stg` → `.env.stg`, `prod` → `.env.prod`; nothing else is accepted)
2. sets the version: stg = `1.0.<git commit count>`, prod = latest git tag (required)
3. rewrites `"version"` in `manifest.json.in` (a tracked file)
4. generates `qml/constant/AppConfig.qml` (gitignored), turning each `SNAKE_KEY` into a `camelKey` string property
5. rewrites `push.json` with `APP_ID`

In CI, Azure Pipelines builds the `.env` file from variable groups `monflo-stg` / `monflo-prod` (branch `main` → prod, everything else → stg) and builds the click for 3 architectures in the `clickable/ci-20.04-<arch>:8` containers.

---

## 3. Project layout

```
main.cpp                  Qt host: registers QML singletons, loads qrc:/Main.qml
CMakeLists.txt            Build, qrc resources, manifest, translations
clickable.yaml            Clickable config + env scripts (stg/prod)
manifest.json.in          Click manifest (version is rewritten by build-env.sh)
monflo.apparmor           networking, push, webview, connectivity, content_exchange
push.json / pushexec      Push-notification helper
scripts/build-env.sh      .env.<env> → AppConfig.qml + push.json + version
azure-pipelines.yml       CI
plugins/Controllers/      C++ QML plugin: DeviceInfo, FileUploader (FingerprintScanner disabled)
plugins/Example/          Template plugin — built and shipped, unused
po/                       Translations (gettext)
assets/                   Images/icons/fonts (assets.qrc)
qml/
  Main.qml, StackNavigation.qml
  state/  constant/  theme/  storage/
  controllers/  utils/  fileuploader/  providers/
  components/             Reusable UI
  ui/                     Screens: SplashScreen, Login, ForgotPassword, Home, Markets, Portfolio,
                          Investment, Deposit, Transfer, Exchange, Report, CardReport,
                          Notification, Profile, Subscription, Biometric, NavBar
                          (Sensor/ and CardActivity/ are empty)
```

---

## 4. Setup & running

### Prerequisites
- Docker (running) and **Clickable 8.x** (`pipx install clickable-ut`)
- `adb` + an Ubuntu Touch device with Developer Mode enabled (for on-device runs)
- `intltool` is needed only for host builds; the Clickable container already includes it

### Environment file
Copy `.env.example` → `.env.stg` (or `.env.prod`) and fill in:
```
APP_ENV=stg
APP_ID=monflo.monflo_monflo_stg      # required, missing from .env.example
API_TRANSACTION_URL=...
API_ONBOARD_URL=...
ORIGIN_URL=...
GRACE_PERIOD_DAYS=15
DOCUMENT_REQUIRED_THRESHOLD=1000
BALANCE_THRESHOLD=0.01
```

### Commands
```bash
clickable script stg-desktop          # build-env (stg) + run on desktop, no phone needed
clickable script stg                  # build-env (stg) + build/install/launch on device
clickable script prod                 # production (needs a git tag)

# manual
APP_ENV=stg bash scripts/build-env.sh
clickable build --arch amd64          # or arm64 / armhf
clickable desktop --skip-build
```
> `build-env.sh` modifies the tracked `manifest.json.in` and `push.json`. Revert them before committing (`git restore manifest.json.in`).

### Troubleshooting
- **`CMakeCache.txt directory … is different`**: `build/` was created at another path (for example `~/Monflo_Dev/`). Run `clickable clean --arch amd64` (do the same for `arm64` if you build it).
- **`libGL error: failed to load driver: iris`** in desktop mode: a host GPU issue that falls back to software rendering. Harmless.
- **Launch fails on device**: retry `clickable launch`.

---

## 5. Deep health check (2026-10-16)

### ✅ Verified working
| Check | Result |
|---|---|
| Toolchain | clickable 8.8.0, docker, adb, cmake, Qt 5.15 host / 5.12.8 container |
| `build-env.sh` (stg) | Generated `AppConfig.qml` v1.0.150 + `push.json` |
| `clickable build --arch amd64` | **Success**: `monflo.monflo_1.0.150_amd64.click`, click review **pass** |
| `clickable desktop` | Boots, splash → stack loads, 75 s with no crash |
| qrc integrity | All files listed in `qml.qrc` / `assets.qrc` exist |
| `qmllint` | No syntax errors |
| JS `const` exported from `.import` libs (`API.API_BASE_URL`) | Works on Qt 5.12.8 (tested in container) |
| `Storage.clearAll()` (nested transaction) | Works: tokens removed, `email` kept (tested in container) |

### 🔴 High: confirmed bugs that affect users or money

1. **API environment is hardcoded, so prod builds hit staging.** `qml/controllers/ApiController.js:6` sets `const ENV = "stg"` and keeps its own URL table. `AppConfig.apiTransactionUrl / apiOnboardUrl / originUrl` are never read (`AppConfig` is only used for `appId`).
   → Build the URLs from `App.AppConfig.*`.
2. **Malformed staging Origin URL in the token refresh.** `ApiController.js:29` sets `"tps://staging.onboardinght.monflo.com"`; it should be `https://staging.onboarding.monflo.com`. This value is sent as the `Origin` header to `/auth/refresh-token`. If the server checks it, every expired session ends in a forced logout.
3. **Password reset reports failure even when it succeeds.** `ui/ForgotPassword/NewPasswordStep.qml:39` calls `AppState.emitSnackMessage(...)`, which **does not exist** in `AppState.qml`. The TypeError is caught, so the user sees "Failed to change password" and stays on the screen after the backend has already changed the password. (`LogoutSection.qml:37` has the same call; logout still works only because its `catch` also redirects.)
   → Add `signal snackMessage(string msg)` + `function emitSnackMessage(m)` to AppState, or remove the calls.
4. **The payment reference typed by the user is never sent.** The UI collects `referral` (`Transfer.qml:219/226/490`, `ConfirmTransferModal.qml:124,157`), but `Assets.transfer()` reads `reference` and `plugins/Controllers/fileuploader.cpp:172` reads `params.reference`:
   - JSON path: the reference is sent as `undefined`
   - multipart path: it is replaced with an auto-generated `"Payment 123456"`
   → Pass `reference: referral`.
5. **Document upload is required from 2 EUR.** `qml/fileuploader/FileUploader.js:3` sets `DOCUMENT_REQUIRED_THRESHOLD = 2`, but the env value is `1000`. This looks like a leftover test value; read it from `AppConfig.documentRequiredThreshold`. (Also a typo on line 15: "Supporing".)
6. **Login can fail for valid users (JWT base64url).** `utils/Utils.js:5 decodeAccessToken()` passes the JWT payload straight to `Qt.atob()` without converting base64url (`-` → `+`, `_` → `/`). I tested this on Qt 5.12.8: a payload containing `-` decodes to garbage and `JSON.parse` throws. In `SecondStep.qml` this happens **after** the tokens are saved, so the user sees an error. On the next app start they are silently logged in. Whether it happens depends on the token content.
   → `payload.replace(/-/g,'+').replace(/_/g,'/')` before padding.

### 🟠 Medium

7. **Access and refresh tokens are printed to the log.** `ui/Login/SecondStep.qml:42-43` uses `console.log("Access token:" …)`. On device these logs go to the journal and `clickable logs`. Remove them.
8. **The password rule can block legitimate passwords.** `Validation.isPasswordFormatValid` (used at login) only allows `[A-Za-z0-9]` plus some symbols, and trims the input. Passwords with spaces, `~`, `` ` ``, `€` or other non-ASCII characters (allowed by web onboarding?) cannot log in on this app.
9. **`ApiCache.get()` drops its `headers` argument**, so the `x-subscription-plan` header that `Profile.getSubscriptionPlan()` sets is never sent.
10. **Upload retry config is never passed, and retrying would be unsafe.** `ConfirmTransferModal` doesn't pass `maxRetries`, `retryDelay` or `documentThreshold`, so they are 0. The retry branch in `fileuploader.cpp` is therefore dead code today. If it is ever enabled: it retries on any non-4xx network error, including **5xx**, on a non-idempotent `POST /withdraw/fiat`, which could create a **duplicate transfer**. Add an idempotency key before enabling it.
11. **The address-book contact is saved before the transfer result.** `ConfirmTransferModal.qml` calls `addNewContact` / `updateContact` right after starting the transfer, even if the transfer later fails.
12. **`Theme.primaryText` is undefined** (`ui/Home/Home.qml:313`). The runtime warns `Unable to assign [undefined] to QColor`.
13. **Binding loop** at `components/ProgressBar.qml:30`. Use `width: control.visualPosition * parent.width`.
14. **NavBar shows a "Reports" tab, but the Report page hides the NavBar.** `hasNavBarPageIds` in `Main.qml` contains `Investment` (not a tab) but not `Report`.
15. **Notabene uses the `.dev` host** (`Assets.js:24`, `api.notabene.dev`), even in production builds. Confirm that this is the intended environment.
16. **Config drift:** the `clickable.yaml` `dev` / `dev-desktop` scripts always fail (only `stg|prod` are accepted). `.env.example` is missing `APP_ID`. `GRACE_PERIOD_DAYS` and `BALANCE_THRESHOLD` are generated but unused.
17. **The token-expiry check is commented out** (`StackNavigation.qml`, `ApiController.js`); the app relies only on 401 → refresh. Tokens are stored unencrypted in the app-confined SQLite database.

### 🟡 Low: cleanup
- `xgettext` "unterminated string" warnings at `Home/Widgets/Card.qml:46,96` and `Subscription/Widgets/SubscriptionCard.qml:196`. Harmless, but those strings won't be translated.
- `plugins/Example` is shipped but unused. `DummyData.js` is imported but unused. Also leftover: `LOCAL_IP = 192.168.100.120` and `testRefreshToken()`.
- Tracked junk files: `.DS_Store`, an empty `from`, `qml/constant/Constant.qml.autosave`.
- `ui/Report/Widgets/BillingHistorySection.qml` and `FiatTxSection.qml` are not in `qml.qrc`, so they are dead code.
- Open TODOs: QR download (`ShareDetail.qml:114`) and email/phone OTP sending (`EmailModal.qml:169, 246`).
- Clickable 8.10 is available (8.8.0 installed).
- No git tags in the local clone (`git tag` is empty), so prod versioning in `build-env.sh` cannot work until tags are fetched. The store is at 1.1.4.

---

## 6. Suggested fix order
1. #3 password reset, #4 transfer reference, #6 JWT decode, #7 token logging. Each is a small, contained change.
2. #1 + #2: wire `ApiController.js` to `AppConfig` (removes the hardcoded env and the bad Origin).
3. #5 threshold from `AppConfig`; #11 save the contact only on success.
4. #9, #12–#14 UI and cache fixes; #16 config cleanup.
5. Before enabling upload retries (#10): add an idempotency key on the backend.
