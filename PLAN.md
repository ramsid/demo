# Private Finance Tracker for iOS: Phase 1 Plan (v2, for review)

> Status: **draft v2 for review**. No code yet.
> Working name: **Ledgerly** (placeholder)
> Guiding principle: **your financial data is stored and analyzed only on your devices. The only things that ever see it are your bank and the sync provider you connect. We never do.**

### What changed in v2 (based on your answers)
| Your answer | Effect on the plan |
|---|---|
| US, Wells Fargo / Chase / Citi | **Automatic bank sync through Plaid** is now the main ingestion path. Apple FinanceKit is dropped, since it only covers Apple Card / Cash / Savings in the US. |
| Automatic sync, less work | Statement import (CSV/PDF) moves to **Phase 2**. A **thin, stateless sync relay** is added (§4.2) because Plaid needs a server-side secret. |
| Stocks (optional) | **Investments** added as a *should*: holdings plus a **monthly portfolio report notification** (§5.6). |
| Support older phones | **Ship our own small local LLM inside the app** (llama.cpp + a ~1–2B open model). Apple's built-in model is used only as a bonus on newer phones (§6.2). |
| Personal / household first, App Store later | Distribution through **TestFlight**. The relay runs **in your own cloud account** for now. An App Store hardening track is added (§9). |

---

## 1. Goals and non-goals

### Goals
1. **Automatically** sync checking, savings, and credit card accounts (Wells Fargo, Chase, Citi), plus brokerage accounts if possible.
2. Analyze income and spending over **weeks, months, quarters, years, or a custom range**.
3. Use an **embedded on-device LLM** that works on older iPhones. No cloud AI.
4. **Alert proactively** about:
   - spending you probably didn't notice (unrecognized, duplicate, fee, trial-to-paid, card-testing charges)
   - **monthly subscription review**, so you can decide what to keep or cancel
   - **drastic increases** in a category, at a merchant, or in a subscription price
   - *(optional)* a **monthly stock / portfolio change** summary
5. Need very little effort: you connect once, and the app does the rest.

### Non-goals for Phase 1
- Manual CSV / OFX / PDF statement import (Phase 2)
- Net worth history beyond the accounts that sync automatically, plus goals (Phase 2)
- Bill pay or moving money (never). Access is **read-only**.
- Cloud AI (never by default)

---

## 2. Monarch Money as reference: what to borrow and what to change

| Monarch feature | Our take for Phase 1 |
|---|---|
| Account aggregation (Plaid / MX / Finicity) | ✅ **Plaid**, through a stateless relay that we (or you) control and that never stores your financial data |
| Dashboard | ✅ Simpler, with a "what changed / needs attention" card at the top |
| Transactions, rules, tags, splits | ✅ Search, edit, rules, tags. Splits are a *should*. |
| Auto-categorization | ✅ Plaid's categories, then your rules, then the on-device LLM for anything unclear. Learns from your corrections. |
| Cash flow and reports | ✅ Income vs spend for any period. Sankey is a *could*. |
| Budgets (including flex) | ✅ Simple monthly budgets plus "safe to spend". Flex budgeting is a *could*. |
| Recurring / subscriptions | ✅ **Core feature**: detection, monthly review, price-hike alerts |
| Investments (holdings, performance) | ✅ *Should*: holdings plus monthly change report. Detailed performance analytics come later. |
| AI Assistant (cloud) | ✅ **On-device** "Ask". This is our main difference from Monarch. |
| Partner collaboration | 🟡 Phase 1.5: household sharing with end-to-end encryption (see open question 1) |
| Web app | ❌ On-device by design |

---

## 3. Security and privacy (priority #1)

### 3.1 Principles
1. **Data lives on the device.** Transactions, balances, holdings, categories, and AI outputs are stored only in the app's encrypted database.
2. **Minimal, stateless server.** The only server component is a relay that adds the Plaid secret to requests. It **stores no financial data, keeps no request or response bodies in logs, and has no database of transactions** (§4.2).
3. **No third-party SDKs** except Plaid Link (required for bank login). No analytics, crash, or ad SDKs.
4. **The LLM is sandboxed.** It runs fully on device, reads data only through read-only tools, and has no network access. Any change it proposes needs your tap.
5. **Transparency.** The "Privacy Ledger" screen lists every network call: when, which host, what kind (for example "Plaid: transactions sync"), and how many bytes. The App Store privacy label is set up for "Data Not Collected" by us. The fact that Plaid is a data processor is disclosed.

### 3.2 Concrete controls
| Area | Control |
|---|---|
| Data at rest | **SQLCipher**-encrypted SQLite (via GRDB). The random 256-bit key is stored in **Keychain** with `WhenUnlockedThisDeviceOnly` and wrapped with a Secure Enclave key. File protection `.complete`. |
| Plaid access tokens | **Kept only on the device** in Keychain (`ThisDeviceOnly`). The relay never stores them. The device sends the token with each request, and the relay forwards it. |
| App access | Face ID / Touch ID / passcode lock on by default. Re-authentication for export and for adding or removing accounts. |
| Network | TLS 1.3 with **certificate pinning** to the relay. ATS is strict. A single `NetworkGateway` only allows a short list of hosts (relay, Plaid Link). |
| Relay auth | App Attest / DeviceCheck plus a per-device key registered at setup, so only your app instances can use your relay. Rate limited. |
| Notifications | Redacted on the lock screen by default ("2 items need your attention"). Push payloads from the relay contain **no data at all**. They only wake the app so it can sync. |
| Background / app switcher | Content blurred in the app switcher. Optional "hide amounts" mode. |
| Backups | Database excluded from iCloud device backup. Optional passphrase-encrypted export. |
| LLM weights | Bundled with the app and checked against a SHA-256 hash before loading. |
| Prompt injection | Merchant names and memos are untrusted text. The LLM has no write, export, or network tools. Its outputs must match a schema (grammar-constrained). |
| Logging | `os.Logger` with `.private` for every value. The relay logs only status codes and timing. |
| Erase | "Erase all data" calls Plaid `/item/remove` for each bank connection, then destroys the encryption key, then deletes files. |
| Supply chain | Few, pinned dependencies (GRDB/SQLCipher, llama.cpp, Plaid Link SDK). The relay code is small (~300 lines) and open for audit. |

### 3.3 Threat model (short)
- **Lost or stolen phone:** iOS passcode + Data Protection + app lock + Keychain `ThisDeviceOnly` tokens. Erasing the app revokes Plaid access.
- **Relay compromise:** an attacker would get the Plaid secret but **no stored data or tokens**. Without a device's access token they can't pull any data. Mitigations: rotate the secret, use App Attest, set alerting.
- **Plaid / bank:** they see the data, which is unavoidable for automatic sync. This is disclosed clearly and access can be revoked from the app.
- **Malicious merchant text:** treated as data, never as instructions. Read-only tools.
- **Shoulder-surfing:** redacted notifications, blur, hide-amounts mode.

---

## 4. Automatic sync

### 4.1 Provider options (US, for Wells Fargo / Chase / Citi)
| Provider | Banks | Investments | Backend needed | Real-time-ish | Cost for personal use | Path to App Store |
|---|---|---|---|---|---|---|
| **Plaid** ✅ *recommended* | All three, via OAuth (bank-approved APIs, no screen scraping) | ✅ holdings + investment transactions | Yes: thin relay (secret + webhooks) | ✅ webhooks → silent push | Plaid's free tier for a limited number of connections (limits to confirm in M0) | ✅ Standard. Needs production approval + security questionnaire. Paid per connection. |
| **SimpleFIN Bridge** | All three (it uses MX behind the scenes) | Balances. Holdings support is limited (to verify). | **No.** The phone talks to it directly. | ❌ ~daily refresh, no push | ~$15 / year paid by the user | Weaker: users must sign up with a separate service |
| Teller | Chase, Wells Fargo, Citi | ❌ | Yes (mTLS certificate must stay server-side) | Partial | Free developer tier | OK, but no investments |
| MX / Finicity / Akoya | Yes | Yes | Yes | Yes | Enterprise contracts | Heavy for an individual developer |

**Recommendation:** build on **Plaid** behind a `SyncProvider` protocol. It covers all three banks with official OAuth connections, includes investments, supports webhooks for faster alerts, and is the standard route to the App Store. Keep **SimpleFIN** as a planned second adapter (Phase 1.5) for people who want **no relay at all**.

> M0 check: confirm that Chase, Wells Fargo, and Citi OAuth connections plus the Investments product are available on Plaid's free/limited tier for a personal account, and note the current per-connection pricing for App Store planning. US bank data-access rules (CFPB §1033, bank fees to aggregators) are still changing, so this is tracked as a risk.

### 4.2 Sync relay design (stateless, zero-knowledge as far as it can be)
```
 iPhone (encrypted DB, access tokens in Keychain)
   │  1. Link: Plaid Link SDK on device → public_token
   │  2. POST /exchange {public_token}        ──► Relay adds client_id + secret ──► Plaid
   │     ◄── access_token (returned to device, NOT stored by the relay)
   │  3. POST /sync {access_token, cursor}    ──► Relay forwards ──► /transactions/sync
   │     ◄── added / modified / removed transactions (streamed back, never stored)
   │
   │  Webhook path (for faster alerts):
   │     Plaid ──webhook(item_id)──► Relay looks up item_id → APNs device token
   │                                  (the ONLY stored mapping, contains no financial data)
   │     Relay ──silent push (empty payload)──► iPhone wakes → runs /sync → local alerts
```
- **Hosting (Phase 1, personal):** a Cloudflare Worker (or Fly.io / small VPS) **in your own account**. The secret is kept in the platform's secret store. Free tier is enough.
- **Hosting (App Store):** the same open-source relay run by us, with a published no-logging policy, regular secret rotation, and an independent review.
- **Stored state:** only `{item_id → APNs token, device public key}`. No balances, transactions, or access tokens.
- **Endpoints:** `/link-token`, `/exchange`, `/transactions/sync`, `/accounts/balance`, `/investments/holdings`, `/investments/transactions`, `/item/remove`, `/webhook`.
- **Silent pushes are rate-limited by iOS and not guaranteed.** To fill the gaps the app also syncs on open, on `BGAppRefreshTask`, and when you pull to refresh.

### 4.3 Sync behavior
- First connection: pull up to **24 months** of history (as much as each bank allows). This gives baselines right away, so alerts are useful from day one.
- Ongoing: `/transactions/sync` with a cursor. Handles pending → posted transitions and changes or removals made by the bank.
- Data Plaid provides that we use directly: `merchant_name`, `personal_finance_category`, logo URL (downloaded once and cached), location, payment channel. **This reduces the work the on-device LLM has to do.**
- Connection health: detect `ITEM_LOGIN_REQUIRED` and prompt the user to re-authenticate with Plaid Link in update mode.
- **Real-time Apple Pay capture** (*could*): a Shortcuts "Transaction" automation that calls our App Intent gives an instant alert even before the bank posts the charge.

### 4.4 Ingestion pipeline (all on device after the data arrives)
```
Plaid sync → Normalize (Decimal, sign, currency, pending/posted) → Dedupe / pending-match
          → Merchant cleanup (Plaid name → rules → LLM only when unclear)
          → Categorize (user rules → Plaid category mapping → LLM for low confidence)
          → Transfer & refund matcher (e.g. "CHASE CREDIT CRD AUTOPAY" ↔ card payment)
          → Recurring detector → Anomaly engine → Insight feed → Local notifications
```

---

## 5. Phase 1 feature list

Priority: **M** = must, **S** = should, **C** = could.

### 5.1 Onboarding
- **M**: Privacy explainer ("what stays on your phone, what Plaid sees, what we never see"). Set up Face ID lock.
- **M**: Connect banks with Plaid Link (Wells Fargo, Chase, Citi), choose which accounts to include, and backfill history.
- **M**: Automatic first analysis: within a minute of connecting, show "Here's your last 3 months, and we found 14 subscriptions."
- **S**: Connect a brokerage account (if investments are enabled).
- **S**: Demo-data mode so the app can be explored before connecting a bank.

### 5.2 Accounts and transactions
- **M**: Account list with balances, last sync time, and connection health (with a "Reconnect" prompt when needed).
- **M**: Unified transaction list with search, filters, and editing. Pending transactions are labeled.
- **M**: Categories (defaults mapped from Plaid categories, customizable) and rules.
- **M**: Transfer and credit-card-payment detection, so moving money between accounts is never counted as spending twice.
- **S**: Tags, notes, splits, and a "Needs review" inbox for low-confidence items.

### 5.3 Analysis (the core)
- **M**: Period picker: week / month / quarter / year / **custom**, plus a custom month start (e.g. payday) and week start.
- **M**: Cash flow: income, spending, net, savings rate, and a trend across periods.
- **M**: Breakdown by category and merchant, compared with the previous period and the N-period average.
- **M**: Drill down from any number to the transactions behind it.
- **S**: Income sources and paycheck detection. Fixed vs variable vs discretionary split.
- **C**: Sankey diagram, spending calendar heat map.

### 5.4 Subscriptions and recurring payments
- **M**: Detection of weekly, monthly, quarterly, and annual charges, with the next expected date and amount. (Plaid's recurring endpoint can serve as a second opinion.)
- **M**: Subscriptions screen showing the monthly total and **annualized cost**.
- **M**: **Monthly Subscription Review** notification on a day you choose, leading to a swipe flow: *Keep / Cancel / Later*.
  - Choosing Cancel shows a how-to-cancel guide (bundled offline). For Apple subscriptions it opens StoreKit's manage-subscriptions sheet. The app then **checks the next cycle that the charge actually stopped**.
- **M**: Alerts for **new subscriptions**, **free trial → paid**, and **price increases**.
- **S**: Upcoming bills for the next 30 days, plus a low-balance warning using live balances.

### 5.5 Alerts and anomaly detection
These are computed on device after each sync and delivered as local notifications that are redacted on the lock screen by default.

| Alert | Detection (deterministic first, AI to explain) |
|---|---|
| **Unrecognized / unexpected spend** | Merchant never seen before, unusual location, foreign currency, unusual time |
| **Duplicate charge** | Same merchant and amount within a short window, and not a known recurring charge |
| **Card-testing pattern** | Several tiny charges ($0.01–$2) from unknown merchants |
| **Fees** | Overdraft, ATM, foreign transaction, late fee, interest charged (using Plaid's category + keywords) |
| **Trial → paid** | A new recurring merchant whose first real charge follows a $0 / $1 charge |
| **Category spike** | Spending in the period is more than baseline (median of the last 3–6 periods) × threshold **and** more than a minimum dollar amount |
| **Merchant spike** | A transaction above the p95 for that merchant, or more than 3× its median |
| **Subscription price hike** | Recurring amount changed by more than 5% |
| **Pace alert** | "80% of your usual dining spend by the 12th" |
| **Large deposit / income missing** | Expected paycheck didn't arrive, or an unusual deposit appeared |

- **M**: Sensitivity setting (Low / Medium / High). "This is normal" feedback tunes future alerts. Quiet hours.
- **M**: Insight feed that keeps every alert.
- **S**: Daily or weekly digest instead of individual pings (high-severity alerts still arrive right away).

### 5.6 Investments (*should*, as requested; optional)
- **S**: Holdings by account: ticker, shares, value, cost basis (when the broker provides it), allocation.
- **S**: **Monthly Portfolio Report** notification, for example: "October: portfolio +3.2% (+$1,840). Biggest gainer NVDA +12%, biggest drop INTC −8%. Dividends $120. You added $500 in contributions."
  - The report **separates market gains from contributions and withdrawals**, so new deposits aren't mistaken for growth.
- **S**: Optional alerts for a large single-month change in a holding (e.g. more than ±15%) and concentration (one stock over 25% of the portfolio).
- Prices: use **Plaid's holding prices** (usually the previous close). Calling a separate market-data API would reveal your tickers to another provider, so it is deferred. If it's added later, the app could request a broad set of tickers so your actual holdings aren't revealed.
- Informational only. Not investment advice (there will be a disclaimer).

### 5.7 Budgets and reports
- **S**: Monthly per-category budgets, with AI-suggested starting amounts based on history. "Safe to spend this month."
- **S**: Monthly recap screen and notification: top changes, subscriptions, savings, portfolio.
- **S**: CSV / PDF export (after re-authentication).

### 5.8 Settings
- **M**: App lock, notification redaction, hide-amounts mode, Privacy Ledger, manage or remove connections, encrypted backup, erase all data.

---

## 6. AI design: embedded local model

### 6.1 Rule: "Math is deterministic, language is AI"
The LLM never calculates or invents numbers. SQL and Swift compute every figure. The LLM **understands messy text, routes questions to the right query, and writes explanations**. Every AI answer links to the transactions behind it.

### 6.2 Model strategy for older phones
| Component | Choice | Why |
|---|---|---|
| **Runtime** | **llama.cpp** (Metal on GPU, called from Swift), GGUF weights | Broadest device support (iPhone 11 / A13 and newer). Mature. Supports **grammar / JSON-schema constrained output**, which makes small models reliable. |
| **Model (default)** | Benchmark in M0: **Qwen3-1.7B** (Apache-2.0), **Llama 3.2 1B / 3B**, **Gemma 3 1B**, all 4-bit quantized | ~0.7–1.2 GB. Fits comfortably within the memory limits of 4 GB-RAM phones. |
| **Tiering** | 4 GB RAM phones get the ~1B model. 6 GB+ phones get the ~1.7–3B model. Apple Intelligence devices may use **Apple Foundation Models** instead (saves battery). | Same `LanguageModelProvider` protocol in every case |
| **No-LLM fallback** | Rules + Plaid categories + a small **Core ML text classifier** (a few MB) | Every core feature keeps working if the LLM can't load |
| **Delivery** | Phase 1 (TestFlight): **bundled in the app**. App Store: download after install via **Background Assets**, verified by hash, so the initial download stays small. | |
| **Battery / thermal** | Heavy batch jobs (backfill categorization) run in `BGProcessingTask` with `requiresExternalPower`. Interactive requests are short and capped. | |
| **Quality** | An **eval set** of ≥500 labeled real merchant strings and ~100 "Ask" questions picks the model and catches regressions. A LoRA fine-tune for categorization is optional later. | |

**Minimum iOS:** iOS 17 (iPhone XS / XR and newer), to be confirmed by the M0 benchmark. If the oldest phones are too slow, the floor becomes iPhone 11 / iOS 18.

### 6.3 Making a small model reliable
- **Constrained output:** every AI task outputs a `Codable` struct enforced by a JSON-schema grammar, so the output is never malformed.
- **Narrow tasks:** one small, focused prompt per job (clean up a merchant name, categorize, classify a question, phrase a summary), not one general chatbot.
- **"Ask" pipeline:** question → **intent + slots** (constrained JSON: `{intent: compareSpending, category: "Dining", range: "last_month", vs: "avg_6m"}`) → deterministic query → numbers → the LLM writes 1–3 sentences around the **given** numbers. If the confidence is low, the app shows a "Did you mean…" chip instead of guessing.
- **Caching:** results for each raw merchant string are cached, so the LLM only runs on new merchants. In practice that's a few per day after the backfill.

### 6.4 AI features (Phase 1)
1. **Merchant cleanup** when Plaid's name is missing or unclear.
2. **Categorization of uncertain transactions plus learning:** after you correct something, it suggests "Always categorize X as Y?"
3. **"Ask" your money:** "How much did we spend on groceries this quarter vs last?", "Which subscriptions went up this year?", "Why was October high?", "How did my stocks do last month?"
4. **Narrative summaries** for the period screen, the monthly recap, and the portfolio report.
5. **Alert explanations:** a one-line *why* plus a suggested action.
6. **Subscription review helper:** groups overlapping services ("3 streaming services: $46/mo"), flags upcoming annual renewals, and calculates potential savings.
7. **Budget suggestions** from your history.

### 6.5 Later
Cash-flow forecast with an explanation, goal coaching, Siri / App Intents ("How much have I spent on dining this month?"), receipt itemization, and a LoRA fine-tune of the categorization model.

---

## 7. Ease of use
- **Connect once:** after Plaid Link everything is automatic, and the first insights appear within a minute.
- **One-glance home:** "This month: $X spent · $Y left · 3 things need attention."
- **Two-minute weekly review queue:** swipe through uncertain categorizations, alerts, and subscription decisions.
- **Calm notifications:** digests by default, instant only for high-severity alerts, quiet hours.
- **Explainable numbers:** tap any number to see its transactions.
- **Widgets** (*should*): month-to-date spend, next bill. They respect hide-amounts mode.
- **Accessibility:** Dynamic Type, VoiceOver on charts (audio graphs), color-blind-safe palette.

---

## 8. Technical architecture

```
iOS app (SwiftUI, iOS 17+)
├── Features/       Home · Transactions · Analysis · Subscriptions · Investments · Insights · Ask · Settings
├── Domain/         Period math, RecurringDetector, AnomalyEngine, PortfolioReport, Budgets (pure Swift, unit-tested)
├── Data/           GRDB + SQLCipher, migrations, repositories, FTS5 search
├── Sync/           SyncProvider protocol → PlaidProvider (via relay) · SimpleFINProvider (1.5) · MockProvider
├── Intelligence/   LanguageModelProvider → LlamaCppProvider · AppleFMProvider · NoneProvider; grammars; eval harness
├── Notifications/  Local scheduling, digests, redaction; silent-push handler → sync
├── Security/       KeyManager, AppLock, NetworkGateway + Privacy Ledger, App Attest
└── Background/     BGAppRefresh (sync), BGProcessing (categorization, recompute)

relay/ (TypeScript, Cloudflare Worker, ~300 lines)
└── link-token · exchange · sync proxy · investments · item/remove · webhook → APNs silent push
```
- Money is stored as `Decimal` plus a currency code. Swift 6 strict concurrency. Swift Charts.
- Testing: `MockProvider` with a synthetic data generator (realistic merchants, subscriptions, anomalies, holdings), Plaid **Sandbox** for integration tests, unit tests for every detector, and the LLM eval harness.

---

## 9. Milestones

| # | Milestone | Outcome |
|---|---|---|
| **M0** | Spikes + decisions (≈1–2 wks) | **LLM benchmark** on the oldest target iPhone (speed, memory, accuracy). Plaid account + tier check for Chase/WF/Citi + Investments. Wireframes for core screens. |
| **M1** | Secure foundation + sync | Encrypted DB, app lock, relay deployed to your account, Plaid Link, transaction sync with backfill, accounts and transactions screens |
| **M2** | Analysis | Period engine, cash flow, breakdowns, comparisons, drill-down, transfer detection |
| **M3** | Subscriptions | Recurring detector, subscriptions screen, monthly review flow, price-hike / new-subscription alerts |
| **M4** | Alerts | Anomaly engine, insight feed, webhook → silent push → sync → local notifications, settings |
| **M5** | On-device AI | llama.cpp integration, model tiering, merchant cleanup, categorization + learning, Ask, narratives, evals |
| **M6** | Investments (optional) | Holdings sync, investments screen, monthly portfolio report |
| **M7** | Household beta | Polish, accessibility, widgets, Privacy Ledger, security review, **TestFlight** for household |
| *Later* | App Store track | Plaid production approval, our own hosted relay + audit, Background Assets model delivery, privacy policy, pricing (Plaid charges per connection), SimpleFIN adapter, statement import |

---

## 10. Open questions (round 2)
1. **Household meaning:** (a) one phone that shows the whole household's accounts, or (b) two or more phones (you + partner) seeing the **same shared data**? Option (b) needs end-to-end encrypted sync (CloudKit encrypted fields + sharing), which I'd put in Phase 1.5.
2. **Brokerages:** where are the stocks held (Chase / J.P. Morgan, WellsTrade, Fidelity, Schwab, Robinhood…)? Coverage differs by broker.
3. **Relay hosting for now:** OK to run the small relay in **your own** Cloudflare (free) account? It's the most private option until the App Store phase.
4. **Oldest iPhone** in your household that must be supported? This sets the model size and the minimum iOS version.
5. **Apple Developer Program** ($99/yr): do you already have one? It's needed for TestFlight, push notifications, and App Attest.

---

## 11. Risks
| Risk | Mitigation |
|---|---|
| Plaid tier limits, pricing, or bank OAuth availability | Check in M0. `SyncProvider` abstraction. SimpleFIN as a fallback. |
| US bank data-access rules changing (CFPB §1033, bank fees) | Use official OAuth channels through Plaid. Keep the provider swappable. |
| Small LLM accuracy on older phones | Plaid categories first, constrained outputs, narrow tasks, rules that learn, eval harness, no-LLM fallback |
| App size (~1 GB model) | Bundled for TestFlight. Background Assets download for the App Store. 1B model on low-RAM phones. |
| Silent pushes delayed or dropped by iOS | Also sync on open and in background refresh, plus pull to refresh. Set expectations in the UI. |
| The relay is a new attack surface | Stateless design, no stored tokens or data, App Attest, pinned TLS, small auditable code, secret rotation |
| Recurring / anomaly false positives | Confidence scores, minimum dollar amounts, "this is normal" feedback, sensitivity setting |
| Portfolio numbers misread as advice | Clear "informational only" wording. Contributions separated from market gains. |
