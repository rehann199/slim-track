# Engineering Challenges

Eight problems that stand out in the production code. Outcomes describe behaviour, not measured metrics; none are claimed.

---

## 1. Thresholds and rounding that disagree

**Problem**
BMI is displayed to one decimal but classified by cut-offs. If the category is derived from the rounded number, two weights that are a fraction of a kilogram apart can be classified inconsistently with the formula; if the number is rounded and then converted again, the value drifts.

**Context**
The same BMI appears on the dashboard, in the detail screen, in history rows and in exports. Users may enter weight in kilograms or pounds.

**Engineering approach**
- Compute BMI once from full-precision kilograms and centimetres.
- Compare that unrounded value against the cut-offs (18.5, 25.0 and 30.0, the commonly used adult cut-offs) using strict less-than comparisons, so a value exactly on a cut-off belongs to the higher band.
- Keep rounding in formatting helpers only, and apply it last.
- Derive the reference weight range from BMI 18.5 and 24.9 at the user's height, using the same formula.

**Trade-offs**
A user can see "25.0" next to a band that sits just below the 25.0 cut-off. That is a faithful consequence of the definition and is preferable to classifying on a rounded value. A short explanation in the UI would help, and boundary tests would lock the behaviour in; neither exists yet.

**Outcome**
Classification follows the definition of the cut-offs regardless of how the number is displayed.

---

## 2. Two unit systems, one truth

**Problem**
Weight can be shown in kilograms or pounds, height in centimetres or feet and inches, and the two choices are independent. Letting each screen convert and round on its own invites disagreement.

**Context**
Settings expose separate weight and height toggles. History, charts, goal progress, exports and the weekly summary input all read stored values.

**Engineering approach**
- Canonical storage in kilograms and centimetres, as double-precision values, including the goal weight.
- A single conversion utility with one factor per direction; feet-plus-inches combined before conversion.
- Input is quantised to one decimal in the user's display unit, converted once, and stored without further rounding.
- Range checks at the input in the display unit, and independent bounds in the cloud security rules.

**Trade-offs**
Pounds typed to one decimal do not map to a round kilogram value, so the stored kilograms look unround in a database view. That is intentional: the round-trip back to pounds is exact to the displayed decimal, which a pre-rounded kilogram value would not be (154 lb rounded to 69.9 kg reads back as 154.1 lb). Switching units during onboarding re-derives the value from a rounded display number, so a small drift can be introduced there, and feet-and-inches height entry is whole inches.

**Outcome**
In Settings, changing a unit toggle changes only presentation; stored values, BMI and categories stay identical.

---

## 3. Logging that never waits for the network

**Problem**
Users want instant, offline logging without an account, but also expect their data to survive a phone change.

**Context**
Firestore backup is optional, sign-in is optional, and a log entry is a tiny record per date.

**Engineering approach**
- SQLite is the working store; the write path updates it first and then updates in-memory state optimistically.
- A background cloud write with a timeout for signed-in users; a synced flag per row acts as the offline queue.
- A guarded flush on reconnect with failure tracking, where offline and timeout errors are expected and are not reported as crashes.
- On explicit sign-in: probe, push, pull and merge by date with the newest write winning, then replace local rows in one transaction. Cloud batch commits are chunked below the service limit.
- A pending-delete marker so an account deletion requested while signed out is completed after the next sign-in.

**Trade-offs**
Deletions are not tombstoned, so removing an entry on one device and later merging from another that still has it can restore the entry. Cross-device conflict handling is per date, not per field. Photos are not part of the sync model, and the synced backup is described as a backup, not as real-time multi-device sync.

**Outcome**
The app is fully usable as a guest, and sync problems never block the interface.

---

## 4. Keeping a paid provider key out of a mobile binary

**Problem**
The weekly summary needs a paid third-party language-model API. A key shipped in an app can be extracted.

**Context**
The feature is Pro only, and generated text costs money per call.

**Engineering approach**
- A small Cloudflare Workers proxy is the only caller of the provider.
- The app sends the user's identity token, a coarse application header and a short text built from the user's name, goal weight, unit and recent weight entries (recent weeks, with dates).
- The proxy verifies the token cryptographically with Web Crypto against the published signing keys, since the Admin SDK does not run on Workers, with a key cache that respects cache headers.
- It then applies a per-user rate limit, an entitlement check and a weekly usage quota, calls the provider with a capped output size, and records usage only after success.
- The app retries with backoff, refreshes the token once on an authentication failure, and falls back to the previous saved insight.

**Trade-offs**
The coarse header is a low-value filter only; the token and entitlement are the real checks. Usage counters are best-effort by design. The proxy has no automated tests.

**Outcome**
No provider credential lives in the app, and a failed call never consumes usage.

---

## 5. Paid access without a billing vendor

**Problem**
A client-side "is Pro" flag cannot protect a server resource, because a client-side flag cannot enforce a server-side limit.

**Context**
Subscriptions are sold through the two platform stores. There is no third-party subscription-management service.

**Engineering approach**
- The app forwards the store's signed purchase data after a purchase, restore or launch check.
- The proxy verifies it offline. On iOS it checks the signature algorithm, requires the certificate chain to end at a pinned platform root, verifies each link, requires the vendor's signing marker on the leaf, verifies the transaction signature, and checks bundle, product and revocation. On Android it verifies the RSA signature over the exact purchase payload against the store's public key and checks package, state and product.
- It writes a time-boxed entitlement record that the request path reads in a single lookup, and fails closed when none exists.

**Trade-offs**
On Android the signed payload carries no expiry, so expiry is derived from the purchase time and plan period; a direct server-side check against the store's subscription API would be a natural next step.

**Outcome**
Backend access is decided from cryptographic evidence rather than a client flag.

---

## 6. Daily reminders without exact alarms

**Problem**
A reminder must fire at a user-chosen time, in the user's language, without a privileged exact-alarm permission and despite aggressive Android battery management.

**Context**
iOS and Android have different scheduling models, and the language can change after the reminder is scheduled.

**Engineering approach**
- Android: a self-rescheduling WorkManager task whose initial delay is the time until the next occurrence; each run schedules the next. The chain is re-armed at every launch, and text is resolved at fire time from the saved locale.
- iOS: repeating local notifications with daily time matching, rescheduled when the language changes because the text is fixed at schedule time.
- The permission is requested after an in-app soft prompt, with a back-off schedule and a "do not ask again" option.
- A per-day guard prevents a stale Android launch intent from reopening the log sheet.

**Trade-offs**
WorkManager timing is approximate by design, so a reminder may land a little after the chosen minute. That is an accepted cost of avoiding the exact-alarm permission.

**Outcome**
Reminder behaviour is designed around each platform's constraints and permission etiquette.

---

## 7. Ads that cannot block or surprise the user

**Problem**
The free tier shows ads, but an SDK failure, a slow network or a configuration mistake must not block the app, and on iOS the tracking prompt must come before ad-related data collection.

**Context**
Ad behaviour is controlled remotely so it can be changed without a release.

**Engineering approach**
- Fail-closed gating: an ad shows only after a real read of remote configuration, a global switch and a per-placement switch, and never for Pro users.
- A startup gate races a minimum splash time against a foreground-time hard cap, so it always resolves.
- Analytics collection is disabled natively on iOS until the tracking prompt is answered.
- Cooldown, hard-cap and retry policies for full-screen formats.

**Trade-offs**
A strict fail-closed policy means a user with a slow first config read sees no ads, a cost accepted for safety. The ad plugin is a local fork that adds preloading, which carries a maintenance cost.

**Outcome**
Ads can be switched off live, and the launch gate always completes.

---

## 8. Analytics for health-adjacent data

**Problem**
Product analytics on weight data risks sending sensitive values to a third party.

**Context**
The product needs funnels (onboarding, paywall, ads, sync health) but not the numbers users log.

**Engineering approach**
- A typed event vocabulary behind a checker that enforces platform naming and size limits.
- Values are enums or buckets (counts, distances, durations); raw weight, name, height and goal are not sent.
- The user identifier is a salted one-way hash; payloads, URLs and tokens are never logged.
- Sessions, screen timings and funnel steps are computed on the client. The analytics layer is the best-tested part of the app.

**Trade-offs**
Bucketing limits what can be learned about individual progress. That is the intent.

**Outcome**
Rich funnels without raw health values leaving the device through analytics.
