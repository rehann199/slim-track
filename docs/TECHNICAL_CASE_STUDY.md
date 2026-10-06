# Technical Case Study

A write-up of the engineering behind Slim Track: Weight Loss & BMI. Production source is private and no code is reproduced.

## 1. Scope

| Item | Detail |
|---|---|
| Product | Weight and BMI tracker for Android and iOS |
| App | Flutter / Dart, about 41,500 lines across 173 hand-written files (app source only, generated localisation code excluded) |
| Serverless proxy | TypeScript on Cloudflare Workers, about 1,250 lines across six source files |
| State and navigation | Riverpod, go_router with a tabbed shell |
| Persistence | SQLite (schema-versioned) plus shared preferences; optional Firestore backup |
| Services | Firebase Auth, Firestore, Analytics, Crashlytics, Performance, Messaging; in-app purchase on both stores; an ad-mediation SDK |

The product is deliberately small: log a weight, derive BMI and progress, chart it, remind the user. The engineering interest is in doing that small amount of arithmetic carefully and wrapping it in careful local-first persistence and sync.

## 2. Numerical correctness (the headline)

<p align="center">
  <img src="../assets/diagrams/calculation-flow.svg" alt="Calculation flow from input through normalisation, BMI and category to presentation, with persistence on the side" width="900" />
</p>

### 2.1 Why this is trickier than it looks

- Two unit systems feed one computation. A pound value and a kilogram value must produce identical results once normalised.
- BMI is shown to one decimal but compared against cut-offs. A value that displays as 25.0 may be 24.98 or 25.02.
- Weights are converted, stored, converted back and charted. Any intermediate rounding accumulates as visible drift.
- Several screens (dashboard, history rows, charts, goal, exports, the weekly summary input) derive numbers independently. If each did its own conversion and rounding they would disagree.

### 2.2 Canonical storage

Every weight is stored in kilograms and every height in centimetres, as double-precision values. The goal weight is stored in kilograms too. The profile's unit settings (kg or lb, cm or ft/in) are display preferences only. Conversions happen at two edges: when a typed value enters the app, and when a stored value is rendered.

Weight conversion uses one fixed factor in each direction. Height conversion uses the exact inch-to-centimetre definition: feet and inches are combined into inches, then converted. Going from centimetres to feet and inches floors the feet and keeps the remaining inches fractional; only the displayed inches are rounded.

### 2.3 Input handling

The log sheet accepts one decimal in the user's display unit, supports a stepper, and checks the value against a wide plausible range in the display unit: out-of-range numbers are clamped to the nearest limit with a warning haptic, and non-numeric text keeps the previous value. The accepted display value is then converted to kilograms and stored without further rounding. Onboarding and settings apply similar range checks to weight and height, and the cloud security rules independently bound weight, height and goal as defence in depth. The date picker rejects future dates, and the database allows one entry per date: logging again on the same date replaces the earlier entry.

### 2.4 BMI and categories

BMI is the standard metric formula, weight in kilograms divided by the square of height in metres, computed on the stored kilogram and centimetre values. Metric and imperial users therefore go through the same formula and get the same result for the same underlying weight and height. The value is computed from the latest entry by date and the single profile height, and the BMI value itself is not clamped; only the gauge position on the dashboard scale is clamped to its visual range.

The app classifies BMI into four standard adult categories using the commonly used cut-offs of 18.5, 25.0 and 30.0. These are public standards. The About BMI screen lists public health sources and states that the content is for general information and not medical advice. The labels are an informational classification; they are not a diagnosis, and the app does not subdivide the upper category, adjust for population, or handle children (the About screen notes that children need age-specific charts).

The comparison is made on the unrounded value. A weight of 72.2 kg at 170 cm gives 24.983, which displays as 25.0 but is classified in the band below the 25.0 cut-off; 72.3 kg gives 25.017, which also displays as 25.0 but falls in the next band. Classifying the rounded value would silently move the first case. The display can look inconsistent for a moment, but it is the classification that stays faithful to the definition.

### 2.5 Worked example

Generic numbers, not user data.

| Step | Value |
|---|---|
| Entered | 154 lb, height 5 ft 7 in |
| Weight normalised | 154 / 2.20462 = 69.853 kg (kept at full precision) |
| Height normalised | (5 x 12 + 7) x 2.54 = 170.18 cm |
| BMI | 69.853 / (1.7018)^2 = 24.12 (unrounded) |
| Displayed | 24.1 |
| Band | the 18.5 to below 25.0 band |
| Cost of rounding early | 69.853 rounded to 69.9 kg reads back as 154.1 lb, not 154.0 |

### 2.6 Reference range

The reference range for a height is computed from BMI 18.5 and 24.9 at that height, using the same formula. It is shown on the BMI detail screen as a range for the user's height, rounded to whole units. The repository describes it as a calculation reference only.

### 2.7 Statistics, progress and rate

- **Start and current weight** are the earliest and latest entries by date. Lowest, highest and total change are derived from the same entry list.
- **Goal progress** infers direction from whether the goal is below or above the start weight, divides the distance covered by the distance needed, and clamps the result to 0 to 100 percent. The goal counts as reached when the current weight is at or beyond the goal in the inferred direction, which triggers a one-time celebration screen per visit.
- **Weekly rate** is the average slope over the whole logged history: last weight minus first weight, divided by the days between the first and last entry dates, scaled to a week. It needs entries on at least two different days. It is deliberately simple and is not a recent-window or smoothed rate.
- **Projected date** extrapolates that rate in a straight line from the current weight to the goal. It appears only when the extrapolation moves toward the goal, and otherwise reads as unavailable. It is an estimate from the user's own history, not a prediction, and the app attaches no judgement about pace.
- **Trend line (Pro)** is an ordinary least-squares linear fit of weight against entry order across the visible entries, drawn as a dashed line and requiring at least three points. The chart line itself is curved for appearance; that smoothing is visual only and does not change values.
- **Chart ranges** select entries strictly newer than a cut-off computed from the current time.
- **Rounding policy** is display-only: weights and BMI to one decimal, the weekly rate to two, reference-range bounds to whole units. Formatting uses the platform's fixed-point formatting with no custom rounding mode.

### 2.8 Known limits of the numerics (trade-offs)

- The trend line regresses against entry index rather than calendar date, so very uneven logging intervals distort the slope relative to time. A date-based fit would be the natural refinement.
- The weekly rate is a whole-history average. For a long history it reacts slowly to recent changes.
- History rows and exports compute BMI from the current profile height, not the height at the time of the entry.
- If no profile height exists, screens fall back to a default height rather than hiding BMI.
- The weekly-summary change chip shows magnitude with colour for direction and keeps a fixed "lost" label, so it does not adapt its wording to a gain.
- Switching units during onboarding re-derives the value from a rounded display number, so a small drift can be introduced there; feet-and-inches height entry is whole inches.
- **There is no dedicated automated test suite for BMI, units, goal or trend maths.** The functions are pure and small, so adding table-driven tests for conversions, boundary BMI values (18.5, 25.0, 30.0 and their neighbours) and the regression would be inexpensive.

## 3. Local-first persistence and optional backup

SQLite is the working store with three tables: a single-row profile, weight entries (unique on date, with a synced flag) and generated weekly insights. A first-open race is coalesced behind a shared in-flight future, and bulk writes use transactions.

- **Write path.** Saving updates SQLite, updates in-memory state optimistically (kept sorted newest first so "latest" is always the first element), and, if the user is signed in, pushes to Firestore in the background with a timeout. Success marks the row synced; failure leaves it unsynced for a later flush.
- **Reconnect.** A flush re-pushes unsynced rows, is guarded against re-entrancy, tracks consecutive failures, and treats offline, timeout and cancellation as expected rather than as crashes.
- **Sign-in merge.** On explicit sign-in the app probes the cloud. If data exists it pushes local data, pulls the cloud set, and merges by date with the newest write winning, then replaces local rows in a single transaction. If the cloud is empty it simply pushes. The merge is never run at cold start because it replaces local data.
- **Photos** are referenced by local path and are not part of the sync model.
- **Deletion.** Account deletion revokes the Apple token where applicable, wipes cloud and local data, and removes the Firebase user, with a re-authentication fallback. A deletion requested while signed out is queued and completed after the next sign-in.

Trade-offs: deletions are not tombstoned, so removing an entry on one device and later signing in on another that still holds it can bring the row back through the by-date union merge; weekly insights are saved locally and also included in the optional cloud backup, merged per week with the newest generation winning, and they are generated text that is not recomputed on another device; and picked photos are not confirmed to be copied into durable app storage, so photo durability is not claimed.

## 4. The weekly summary and the serverless proxy

The weekly summary is a Pro feature that turns a short text summary of recent weights into a few sentences of generated prose via a third-party language-model API. The text can include a general suggestion; it is generated text and not professional advice. The vendor, model and limits are intentionally not described here.

Request path: the app (signed-in, Pro, with enough recent data) sends the user's name, goal weight, unit and the last two weeks of dated weights as text, with a Firebase ID token, to a Cloudflare Worker. The Worker:

1. Checks a coarse application header (a low-value filter).
2. Verifies the ID token itself with Web Crypto against the published signing keys, checking signature, issuer, audience, expiry and subject. The Admin SDK does not run on Workers, so verification is hand-written.
3. Applies a per-user rate limit held in a small key-value store. The rate limiter is deliberately simple.
4. Reads a time-boxed entitlement record and fails closed when none exists.
5. Checks the weekly usage allowance, calls the language-model API and records usage only after a successful call, so failed calls never consume quota.

The app retries timeouts and server errors with backoff, refreshes the token once on an authentication error, maps typed failures to localised messages, and keeps the previous insight on screen as a fallback. Regeneration is limited to re-expressing an existing insight in a different language, which bounds cost.

## 5. Subscriptions and entitlement

Purchases are handled client-side through the platform billing APIs, with price and trial eligibility resolved from the store at runtime. The app keeps a cached local Pro flag for UI decisions, re-checked by restoring purchases at launch. That flag does not protect the paid backend feature: the Worker does, by verifying the store's signed purchase data offline.

- **iOS.** The signed transaction is parsed, its signature algorithm checked, its certificate chain required to terminate at a pinned platform root, each link's validity and signature checked, the leaf required to carry the vendor's signing marker, and the payload checked for bundle, product and revocation.
- **Android.** The RSA signature over the exact purchase payload is verified with the store's public key, then package, purchase state and product are checked. Because the signed payload carries no expiry, expiry is derived from purchase time, the plan period and a short grace period.
- The result is stored as a time-boxed entitlement record that the request path reads with a single lookup.

Trade-off: on Android the signed payload carries no expiry, so expiry is derived from the purchase time and plan period; a direct server-side check against the store's subscription API would be a natural next step.

## 6. Reminders, ads and analytics

These supporting systems are covered as problem, approach and trade-off in [Engineering challenges](ENGINEERING_CHALLENGES.md). In short: Android reminders use a self-rescheduling WorkManager chain re-armed at every launch, iOS uses repeating local notifications rescheduled on language change; ads are free-tier only and fail closed on remote configuration, with a startup gate that always resolves and, on iOS, collection held off until the tracking prompt is answered; analytics events are typed and use enums or buckets rather than raw weight, name, height or goal, with a salted one-way hash as identifier. Startup phases, database open, sync and exports are timed with performance traces.

## 7. Localisation

The UI is localised through ARB files for 20 selectable languages, including right-to-left languages. Right-to-left layout quality has not been separately reviewed, and the in-app privacy policy screen is English only.

## 8. Testing

The project has 18 test files and roughly 178 test declarations, counted statically. They cover the analytics event checks and session logic, screen tracking, gating by consent, bucket helpers, connectivity tracking and ad-lifecycle logging. A default widget test remains a placeholder. There are no tests for calculations, persistence, sync, billing, reminders, export or the proxy. The test count is a static count and no result is claimed here.
