# Private Finance Tracker for iOS: Phase 1 Plan (for review)

> Status: **draft for review**. No code yet.
> Working name: **Ledgerly** (placeholder)
> Guiding principle: **your financial data never leaves your iPhone unless you explicitly send it somewhere.**

---

## 1. Goals and non-goals

### Goals
1. Track income and spending across accounts, analyzed over **weeks, months, quarters, years, or a custom range**.
2. Use an **on-device LLM** for understanding transactions, with no cloud AI.
3. **Alert proactively** about:
   - spending you probably didn't notice (unrecognized, duplicate, fee, trial-to-paid, card-testing charges)
   - **monthly subscription review**: every month, list what you pay for and let you decide what to keep
   - **drastic increases**: category, merchant, or subscription price jumps
4. Be **easier and calmer than a spreadsheet**: little manual work, clear answers.

### Non-goals for Phase 1
- Investments and brokerage tracking, net worth over time (Phase 2)
- Multi-user or partner sharing (Phase 2+)
- Bill pay, money movement, or any write access to bank accounts (never)
- Cloud AI of any kind (never by default)

---

## 2. Monarch Money as reference: what to borrow and what to change

| Monarch feature | Our take for Phase 1 |
|---|---|
| Account aggregation (Plaid/MX/Finicity) | **Privacy-first ingestion** instead (see §4). Aggregators are optional and come later. |
| Dashboard (cash flow, spending, recent transactions) | ✅ Yes. Simpler, with one "what changed" card at the top. |
| Transactions list, search, rules, splits, tags | ✅ Search, edit, rules, tags. Splits are a *should*. |
| Auto-categorization | ✅ Rules + on-device ML + on-device LLM, learns from your corrections. |
| Cash flow reports and Sankey | ✅ Income vs spend by period. Sankey is a *could*. |
| Budgets (category and flex) | ✅ Simple monthly budgets plus "safe to spend". Flex budgeting is a *could*. |
| Recurring / subscriptions calendar | ✅ **Core feature.** Detection plus monthly review. |
| Goals / savings | Phase 2 |
| Net worth and investments | Phase 2 |
| AI Assistant (cloud LLM) | ✅ **On-device** "Ask" assistant. This is our main difference from Monarch. |
| Partner collaboration | Phase 2+ (would need end-to-end encrypted sync) |
| Web app | ❌ Not planned. On-device by design. |

**Positioning:** Monarch's insights, with no account, no server, and no data collection.

---

## 3. Security and privacy (priority #1)

### 3.1 Principles
1. **Local-first, local-only by default.** No backend owned by us. No user accounts. No telemetry.
2. **Zero third-party SDKs** in the binary: no analytics, crash reporters, or ad SDKs. Crash data only through Apple's opt-in MetricKit.
3. **Least privilege on the network.** Phase 1 runs fully offline unless the user turns on an optional connector. All network calls go through one auditable `NetworkGateway` with an allowlist of hosts.
4. **The LLM is sandboxed.** It reads data only through typed, read-only tools. It has no network access and cannot change data without the user confirming.
5. **Transparency.** An in-app "Privacy Ledger" screen shows every network request ever made, which should be empty for offline users. App Store privacy label: **"Data Not Collected."**

### 3.2 Concrete controls
| Area | Control |
|---|---|
| Data at rest | SQLite encrypted with **SQLCipher** (via GRDB). The 256-bit key is generated on first launch and stored in **Keychain** with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`, wrapped by a Secure Enclave key. Files also use iOS Data Protection `.complete`. |
| App access | **Face ID / Touch ID / passcode** lock (on by default), auto-lock after N minutes, re-auth for exports. |
| App switcher | Content blurred when the app goes to the background. |
| Notifications | **Redacted by default** on the lock screen (for example "A subscription review is ready", with no amounts or merchants). Full detail is opt-in. |
| Backups | Database excluded from iCloud device backup by default. Optional **encrypted export** (passphrase, AES-GCM, Argon2/PBKDF2 key derivation) to Files. Optional encrypted iCloud sync is Phase 2. |
| Imported files | Statement files are parsed and then securely deleted from the app sandbox. Original PDFs are kept only if the user chooses. |
| Credentials (if connectors are used) | Tokens only in Keychain, `ThisDeviceOnly`. Never logged. |
| Logging | `os.Logger` with `privacy: .private` for every value. No transaction data in logs. |
| Prompt-injection defense | Merchant strings and statement text are **untrusted input**. The LLM cannot call network, write, or export tools. Any write it proposes (for example a recategorize rule) needs a user tap. |
| Clipboard / sharing | No automatic copying. Share sheets for exports only. |
| Kill switch | "Erase all data" destroys the key, which makes the database unreadable instantly. Then files are deleted. |
| Supply chain | Few dependencies (GRDB/SQLCipher; optionally MLX Swift), pinned by version and hash, reviewed before upgrades. |

### 3.3 Threat model (short)
- **Lost or stolen phone:** iOS passcode + Data Protection + app lock + Keychain `ThisDeviceOnly`.
- **Malicious statement or merchant text:** treated as data, never as instructions. The LLM has read-only tools and outputs are validated against a schema.
- **Snooping over the shoulder or on the lock screen:** redacted notifications, blur in background, optional "hide amounts" mode.
- **Us, the developer:** we never receive any data. That is the strongest guarantee we can offer.
- **Aggregator (if enabled):** clearly disclosed. The user chooses which and can revoke. Off by default.

---

## 4. Getting bank data in, privately

This is the hardest product decision because it trades privacy against convenience. Proposal: offer **layered options**, private by default.

| Option | Privacy | Effort for user | Notes | Phase |
|---|---|---|---|---|
| **A. CSV / OFX / QFX / QIF import** | ⭐⭐⭐ fully local | Medium (monthly download) | Every bank supports at least one format. Auto-detect columns and save a "bank profile" per institution. | **1** |
| **B. PDF statement import** | ⭐⭐⭐ fully local | Medium | PDFKit + Vision OCR, with the on-device LLM pulling structured rows out of the text. Validated by checking that the statement balance reconciles. | **1** |
| **C. Apple FinanceKit** (Apple Card, Apple Cash, Savings; UK Wallet-connected banks) | ⭐⭐⭐ on-device API, no third party | Very low (automatic) | Needs a **managed entitlement from Apple** (apply early). Only covers Apple accounts in the US. | **1** (if entitlement granted) |
| **D. Apple Pay tap capture** via Shortcuts "Transaction" automation → our App Intent | ⭐⭐⭐ local | One-time setup | Near-real-time logging of Apple Pay purchases (merchant, amount, card). Good for "undetected spend" alerts. | **1** |
| **E. Share sheet / manual quick-add** | ⭐⭐⭐ | Low | Receipts, cash, Venmo screenshots (Vision OCR). | **1** |
| **F. SimpleFIN Bridge** (US) | ⭐⭐ third party sees data, no dev backend | Low | The user owns the token and the app fetches directly from the phone. | 1 (stretch) or 2 |
| **G. Plaid / Teller / MX** | ⭐ aggregator plus a minimal token relay | Lowest | Plaid needs a server-side secret, which means running a backend. Conflicts with "no server". | 2 (opt-in) |
| **H. India Account Aggregator / EU-UK Open Banking** | ⭐⭐ regulated | Low | Depends on the target region. Needs a license or partner. | 2+ |

**Recommendation for Phase 1:** A + B + D + E as the base, C if Apple grants the entitlement. Decide on F/G once the target region is confirmed (see open questions).

### Ingestion pipeline (all on device)
```
Source → Parser → Normalizer (dates, signs, currency) → Dedupe (hash + fuzzy match)
       → Merchant cleanup (rules → LLM) → Categorizer (rules → ML → LLM)
       → Transfer/refund matcher → Recurring detector → Insight engine → Notifications
```

---

## 5. Phase 1 feature list

Priority: **M** = must, **S** = should, **C** = could.

### 5.1 Onboarding and setup
- **M**: Privacy-first welcome (what stays on device, why there's no sign-up), set up biometric lock.
- **M**: Add accounts manually (name, type, currency, opening balance), then import.
- **M**: Guided import ("Which bank? Here's where to download a CSV"), using a bundled offline guide per major bank.
- **S**: Set up the Apple Pay Shortcut automation with step-by-step instructions.
- **S**: "Try with demo data" mode, so people can explore before trusting the app with real data.

### 5.2 Transactions
- **M**: Unified list across accounts, with search (merchant, amount, note), filters, and editing.
- **M**: Categories (sensible defaults, customizable, with parent/child).
- **M**: Rules ("if merchant contains X, category = Y"). The AI suggests rules after repeated corrections.
- **M**: Transfer detection (card payments, account-to-account moves are not counted as spending) and refund matching.
- **S**: Tags, notes, receipt photo attachment, split transactions.
- **S**: "Needs review" inbox for low-confidence categorizations, triaged with a swipe.

### 5.3 Analysis (the core)
- **M**: Period picker: **week / month / quarter / year / custom range**, plus a user-set start of month (for example payday on the 25th) and start of week.
- **M**: Cash flow: income vs spending vs net savings, savings rate, trend chart across periods.
- **M**: Spending breakdown by category and merchant, compared with the previous period and the average of the last N periods.
- **M**: Drill-down from any chart to its transactions.
- **S**: Income analysis: sources, regularity, paycheck detection.
- **S**: Fixed vs variable vs discretionary spending split.
- **C**: Sankey (income → categories), calendar heat map of spending days.

### 5.4 Subscriptions and recurring payments
- **M**: Automatic detection of recurring charges (weekly, monthly, quarterly, annual) with the expected next date and amount.
- **M**: Subscriptions screen: monthly total, **annualized total** ("$1,284/year"), sorted by cost.
- **M**: **Monthly Subscription Review notification** (user picks the day). Opens a swipe-card flow: *Keep / Cancel / Remind me later*.
  - "Cancel" → shows a how-to-cancel guide and deep link (bundled offline directory). For App Store subscriptions it uses StoreKit's manage-subscriptions sheet. The item is then tracked: "Did the charge stop? We'll check next cycle."
- **M**: Alerts for **price increases** and **new subscriptions** (including free trial → paid).
- **S**: Bill calendar for upcoming recurring charges in the next 30 days. Low-balance warning if a manual or known balance is available.
- **S**: "Subscriptions you may have forgotten": no interaction or tag in a long time, charged quarterly or annually, small amounts.

### 5.5 Alerts and anomaly detection
All alerts are **local notifications**, configurable, with redacted lock-screen text.

| Alert | Detection logic (deterministic first, AI to explain) |
|---|---|
| **Unrecognized / unexpected spend** | First-ever merchant, unusual time or location, foreign currency |
| **Duplicate charge** | Same merchant and amount within X hours/days, not a known recurring pattern |
| **Card-testing pattern** | Several very small charges ($0.01–$2) from unknown merchants |
| **Fees** | Overdraft, ATM, foreign transaction, late fee, interest (matched by keyword plus LLM) |
| **Trial → paid** | New recurring merchant whose first real charge follows a $0 or $1 charge |
| **Category spike** | Period spend > baseline (median of the last 3–6 periods) × threshold, *and* above an absolute minimum, so a jump from $5 to $15 does not trigger it |
| **Merchant spike** | Single transaction > p95 for that merchant, or > 3× its median |
| **Subscription price hike** | Recurring amount changed by more than ±5% |
| **Pace alert** | "You've spent 80% of your usual monthly dining by the 12th" |

- **M**: Thresholds controlled by sensitivity settings (Low / Medium / High), plus snooze or "this is normal" feedback that tunes future alerts.
- **M**: Insight feed inside the app that keeps every alert, so nothing is lost if a notification is dismissed.

### 5.6 Budgets (lightweight in Phase 1)
- **S**: Monthly per-category budgets. The AI suggests them from history ("based on your last 3 months").
- **S**: "Safe to spend this month" = expected income − fixed recurring − budgeted spending so far.

### 5.7 Reports and export
- **S**: Monthly recap screen (also a notification): top changes, wins, subscriptions, net savings.
- **S**: Export CSV or PDF report (local share sheet, after biometric re-auth).

### 5.8 Settings and privacy
- **M**: App lock, notification redaction, hide-amounts mode, Privacy Ledger, encrypted backup and restore, erase all data.

---

## 6. How AI makes this genuinely helpful

### 6.1 Rule: "Math is deterministic, language is AI"
The LLM **never calculates totals** and never invents numbers. All figures come from SQL and Swift. The LLM is used to **understand messy text**, **pick which tool to call**, and **explain results in plain language**. This keeps answers correct and makes them checkable: every AI answer links to the underlying transactions.

### 6.2 Model options
| Option | Pros | Cons |
|---|---|---|
| **Apple Foundation Models framework** (iOS 26+, on-device ~3B model) | Built in, no download, private, fast, battery-aware. Supports **guided generation** (`@Generable` structs) and **tool calling**. | Needs an Apple Intelligence device (iPhone 15 Pro or newer). Small context window (~4K tokens). Behavior can change between iOS versions. |
| **MLX Swift + open model** (e.g. Qwen / Llama / Gemma 1–4B, quantized) | Works on more devices, we control the model version, can fine-tune for transaction categorization | 1–3 GB download, more memory and heat, more for us to maintain |
| **llama.cpp (GGUF)** | Mature, flexible | Same trade-offs as MLX, C++ bridging |

**Recommendation:** use **Apple Foundation Models** first, behind a `LanguageModelProvider` protocol. Add MLX as an optional fallback in Phase 1.5 for devices without Apple Intelligence. **Every core feature also works with no LLM at all** (rules + classical ML), and AI improves the quality on top.

Small classical models (Core ML / Create ML text classifier) handle high-volume categorization quickly. The LLM handles the long tail and low-confidence cases.

### 6.3 AI features (Phase 1)
1. **Merchant name cleanup.** `SQ *BLUE BOTTLE 0423 OAKLAND CA` → **Blue Bottle Coffee** · Coffee shops. Results are cached per raw string, so each one is processed only once.
2. **Smart categorization with learning.** Categorize → confidence score → low-confidence items go to the "Needs review" inbox. When the user corrects something, the app offers: "Always categorize *Blue Bottle* as Coffee?" and creates a rule.
3. **Statement understanding.** Pulls structured transactions out of OCR'd PDF text with schema-constrained output, then checks that the balance reconciles.
4. **"Ask" your finances (natural language Q&A)** using tool calling:
   - "How much did I spend on food last month vs. my average?"
   - "What are all my subscriptions over $10?"
   - "Why was October so expensive?"
   - Tools: `querySpending(category, merchant, range)`, `comparePeriods(...)`, `listRecurring(...)`, `topChanges(range)`, `findTransactions(...)`. These are read-only, return compact aggregates, and fit in the small context window.
5. **Narrative period summaries.** "In October you spent $3,420 (+18% vs. your 6-month average). Most of the increase came from Travel ($640, one-time: flights) and Dining (+$180). Excluding travel you were on track." Numbers are filled in from computed data, and the LLM only writes the sentences around them.
6. **Explaining alerts.** Each anomaly comes with a one-line *why* and a suggested action ("This is your 3rd charge from *XYZ* this week; tap to review").
7. **Subscription review helper.** Groups subscriptions by overlap ("You pay for 3 streaming services: $46/mo"), flags annual renewals coming up in the next 30 days, and summarizes "if you cancel these 2, you save $312/yr."
8. **Budget suggestions** from history, explained in plain language.

### 6.4 AI ideas for later phases
- Goal coaching ("At this pace you'll reach $10k emergency fund by June").
- Forecasting next month's cash flow (deterministic forecast, explained by the LLM).
- Siri / App Intents: "Hey Siri, how much have I spent on groceries this month?" Answered on device.
- Receipt itemization from photos.
- Optional, explicit, per-question opt-in to a larger cloud model with **anonymized aggregates only**. This would stay off by default and might never ship. Shown only so it can be reviewed.

---

## 7. Making it easy to use
- **One-glance home:** "This month: $X spent, $Y left, 2 things need attention."
- **Inbox-zero for money:** one "Review" queue (low-confidence categorizations, alerts, subscription decisions) handled with swipe actions. About 2 minutes a week.
- **Smart defaults:** works well with zero configuration. Settings are progressive.
- **Import friction killers:** remember each bank's CSV format, drag-and-drop or share-sheet import from Files or Mail, a "last imported 23 days ago" nudge.
- **Explainable numbers:** every number can be tapped to see the transactions behind it.
- **Calm notifications:** batched digest by default (daily or weekly). Real-time only for high-severity items (duplicates, card testing, fees). Quiet hours.
- **Widgets (S):** month-to-date spend and next bill. They respect hide-amounts mode.
- **Accessibility:** Dynamic Type, VoiceOver labels on charts (Swift Charts audio graphs), color-blind-safe palette.

---

## 8. Proposed technical architecture

```
SwiftUI app (iOS 26+)
├── Features/          Dashboard · Transactions · Analysis · Subscriptions · Insights · Ask · Settings
├── Domain/            Models, period math, RecurringDetector, AnomalyEngine, BudgetEngine (pure Swift, unit-tested)
├── Data/              GRDB + SQLCipher store, migrations, repositories, FTS5 search
├── Ingestion/         CSV/OFX/QIF parsers, PDF+Vision parser, FinanceKit adapter, App Intent (Apple Pay)
├── Intelligence/      LanguageModelProvider (FoundationModels | MLX | None), tools, prompts, @Generable schemas, eval harness
├── Notifications/     Local scheduling, redaction, digests
├── Security/          KeyManager (Keychain + Secure Enclave), AppLock, NetworkGateway + Privacy Ledger
└── Background/        BGAppRefresh / BGProcessing for categorization, recurring detection, digest building
```

- **UI:** SwiftUI + Swift Charts.
- **Storage:** GRDB + SQLCipher rather than SwiftData, because it gives us real SQL for analytics, encryption with our own key, FTS5 search, and explicit migrations.
- **Concurrency:** Swift 6 strict concurrency. Heavy work runs in actors and background tasks.
- **Money:** `Decimal` with an ISO currency code. Never `Double`.
- **Background reality:** iOS doesn't guarantee background runs. Alerts are computed **on import / on app open / on BG refresh / on Apple Pay App Intent**, and scheduled digests use local notifications.
- **Testing:** a synthetic transaction generator (realistic merchants, subscriptions, anomalies), unit tests for every detector, and an **LLM eval set** (≥500 labeled real-world merchant strings) that tracks categorization accuracy across model and iOS versions.

---

## 9. Milestones (proposal)

| # | Milestone | Outcome |
|---|---|---|
| M0 | Decisions + design | Open questions answered, wireframes for 6 core screens, FinanceKit entitlement requested |
| M1 | Secure foundation | Encrypted DB, app lock, accounts, manual entry, CSV import, transactions list |
| M2 | Analysis | Period engine, cash flow, category/merchant breakdowns, comparisons, drill-down |
| M3 | Recurring + subscriptions | Detector, subscriptions screen, monthly review flow + notifications |
| M4 | Alerts | Anomaly engine, insight feed, notification settings and redaction |
| M5 | On-device AI | Merchant cleanup, categorization + learning, "Ask", narrative summaries, eval harness |
| M6 | More ingestion | OFX/QIF, PDF statements, Apple Pay App Intent, FinanceKit (if approved) |
| M7 | Polish + privacy audit | Accessibility, widgets, Privacy Ledger, security review, TestFlight beta |

---

## 10. Open questions for you
1. **Region and banks:** which country and which banks/cards? This decides FinanceKit, SimpleFIN, Plaid, or Account Aggregator.
2. **Ingestion vs privacy:** is "import statements manually, fully private" acceptable for Phase 1, or is automatic bank sync a must, even with a third-party aggregator?
3. **Devices:** OK to require **iOS 26** and lean on Apple Intelligence devices, with reduced AI on older phones?
4. **Sync:** single device only in Phase 1, or should iPad/second iPhone sync (end-to-end encrypted via CloudKit) come early?
5. **Users:** just you, or eventually a household or partner?
6. **Distribution:** personal use (TestFlight / dev install) or App Store? This affects entitlements, legal/disclaimers, and monetization.
7. **Currency:** single currency or multi-currency from day one?

---

## 11. Risks
| Risk | Mitigation |
|---|---|
| Manual import is too much friction, so people stop using the app | Apple Pay capture + FinanceKit for automatic data. Import nudges. Quick share-sheet import. |
| The small on-device model makes mistakes | Deterministic math, schema-constrained outputs, rules that learn from corrections, eval harness, no LLM-generated numbers |
| FinanceKit entitlement not granted / US-only | Not on the critical path, because the other import options don't need it |
| Background execution limits delay alerts | Compute on import/open/App Intent. Set expectations in the UI. |
| Recurring detection false positives | Confidence score + one-tap "Not a subscription" feedback |
| Apple Intelligence not available on the device or in the region | `None` provider fallback; MLX option |
