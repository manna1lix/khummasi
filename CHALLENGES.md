# Engineering Deep Dives

Building Khummasi single-handedly meant acting as the Product Manager, Lead Designer, Mobile Engineer, and Backend Architect simultaneously. Below are the six hardest technical challenges I solved during the 6+ months of development.

---

## 1. The Dual-Path Data Model

**The Problem:**
Sports booking platforms require two things that naturally fight each other:
1. **Ultra-low latency:** Players need to see match statuses and chat messages instantly.
2. **Ironclad security:** Financial transactions (bookings, wallets) cannot trust the client.

**The Solution:**
I designed a "Dual-Path" architecture:
- **Path 1 (Reads):** Mobile clients connect directly to Cloud Firestore using `onSnapshot` listeners. This provides real-time updates for feeds, chat, and match statuses with zero backend overhead.
- **Path 2 (Mutations):** Firestore Security Rules strictly deny all client writes to collections like `bookings`, `walletTransactions`, and `match_stats`. Instead, the mobile app makes REST calls to the Express 5 API. The backend validates the logic, executes the transaction using the Firebase Admin SDK (bypassing the rules securely), and updates Firestore. The mobile client's `onSnapshot` listener then instantly picks up the change.

---

## 2. Adversarial Test Harnesses ("Integrity Court")

**The Problem:**
When you manage real-world turf bookings and calculate financial debt (-5 LYD commission per booking), a race condition or double-booking bug isn't just an error—it costs people real money and ruins reputations. I needed a way to guarantee my booking logic was bulletproof.

**The Solution:**
I built two internal simulation scripts: **Integrity Court** and **Rules Court**.
Instead of relying solely on Jest unit tests, I wrote an adversarial test harness that actively tries to break the backend. It fires concurrent booking requests for the exact same slot, attempts to manipulate wallet balances, and tries to accept bookings without enough debt headroom. 

By utilizing deterministic slot locking (`pitches/{id}/schedule/{date}_{time}`) inside a Firestore transaction, the backend successfully rejects all concurrent race conditions. Integrity Court proves the system works under pressure.

---

## 3. RBAC Without a Microservice

**The Problem:**
Khummasi has four distinct user roles: Player, Owner, Staff, and Admin (with three sub-tiers: support, standard, super). Building a dedicated authentication microservice for routing these roles would have significantly increased hosting costs and latency.

**The Solution:**
I engineered an access control system entirely within Firestore Security Rules (21KB, 498 lines of code).
For Staff who manage pitches on behalf of turf owners, I implemented a delegated `effectiveOwnerId` pattern. When a Staff member logs in, their token grants them read/write access to the specific Owner's pitches, completely contained within the Firebase rules layer. The admin dashboard consumes a separate `admin_roles` collection to selectively render UI components based on the admin's tier.

---

## 4. Bypassing Native Map Limitations (Leaflet-in-WebView)

**The Problem:**
The native `react-native-maps` library (which uses Google Maps or Apple Maps) was too restrictive for the highly customized, dark-themed "Explore" map I designed. The visual markers didn't match the Neo-Surface UI, and map tile customization was limited and expensive.

**The Solution:**
I implemented a hybrid mapping strategy. For precise pinpointing during Owner registration, I use the native `react-native-maps`. However, for the main Player Explore map, I built a custom component that injects **Leaflet.js** into a `react-native-webview`. 

I pass coordinates between the React Native thread and the WebView via injected JavaScript and `postMessage`. This allowed me to use free CartoCDN dark-mode tiles, custom HTML/CSS markers that perfectly match the app's aesthetic, and bounded panning to specifically keep the user within Libya's coordinates.

---

## 5. The RTL "Double-Flip" Dilemma

**The Problem:**
Khummasi is fully bilingual (English and Arabic). When switching to Arabic (RTL), React Native's `I18nManager` automatically flips layouts that use `flexDirection: 'row'`. However, in early development, I was manually reversing rows in the code when Arabic was active. This resulted in a "double-flip"—the native engine flipped my already-flipped row, completely breaking the UI.

**The Solution:**
I audited all ~100 components to remove manual RTL overrides. Instead, I strictly adhered to using `start` and `end` (e.g., `paddingStart`, `alignItems: 'flex-start'`) rather than `left` and `right`. I built a global layout system that trusts the native engine to handle the mirroring, resulting in a flawless transition between LTR and RTL that required significantly less code.

---

## 6. Disabled-by-Default Enterprise Auth

**The Problem:**
Ghost accounts and bot signups can inflate database size and skew Typesense search results. I needed to ensure that only real humans with verified email addresses could access the app.

**The Solution:**
I implemented a "disabled-by-default" authentication flow. When a user creates an account, Firebase Auth registers them as `disabled: true`. A Cloud Function triggers on their creation and emails a 6-digit OTP using `nodemailer`. The client is locked on a verification screen until they provide the correct OTP to the `/api/auth/verify-otp` endpoint, which then enables the account. A daily `janitorService` cron job sweeps the database and deletes any account that remains disabled for over 24 hours.
