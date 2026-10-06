# Privacy and Source

## What this repository is

A portfolio showcase for a production application. It describes the engineering of Slim Track: Weight Loss & BMI; it is not a source distribution. It describes software engineering, not health outcomes, and nothing in it is medical, health or nutrition advice. BMI is a general screening statistic, not a diagnosis.

## What is published

- Written descriptions of architecture, calculations, design decisions and trade-offs
- Original diagrams created for this showcase
- Real screenshots from the Android build, cropped to the app content (system status bar and bottom strip removed), captured with a placeholder profile and sample data
- The app icon, resized
- Generic worked numbers used only to illustrate arithmetic; they are not user data

## What is intentionally not published

- Production source code, in whole or in part
- Tuning values, exact limits, quotas and thresholds specific to the product's backend
- Build and signing configuration, certificates and keystores
- Firebase and Google service configuration, API keys and tokens
- Ad network identifiers, placement identifiers and account details
- Subscription product identifiers and store listing internals
- The name of the third-party language-model provider, the model and its prompts
- Internal endpoints, database and collection names, and security-rule contents
- Version control history of the production repositories
- User data of any kind

Line, file and test counts are static counts; no test results, performance figures, user numbers or ratings are claimed. Diagram labels are conceptual and do not correspond to production identifiers.

## About the app's data handling

This repository does not restate or replace the app's own privacy policy, which is the authoritative source and is available from the app and its store listings. At the engineering level, the following design properties are:

- The app works as a guest with no account; weight data stays on the device unless the user signs in for the optional cloud backup.
- Photos are referenced by local path and are not part of the sync model.
- Analytics events use counts, durations and buckets rather than raw weight, name, height or goal, and use a salted one-way hash as identifier.
- The weekly summary sends the user's name, goal weight, weight unit and recent logged weights (dates and weights) through the app's own serverless proxy to a third-party language-model API; no provider credential is stored in the app.
- The profile (name, gender, height, goal, units, reminder setting) and the weekly insights are part of the optional cloud backup.
- Local data is not additionally encrypted by the app beyond operating-system defaults, and the app offers no PIN or biometric lock.

## Images

- **Screenshots** are real captures from the Android build, with a placeholder profile and sample data. They show no account details or notifications. Screenshots were re-encoded to WebP with metadata removed.
- **Diagrams** are original vector illustrations.
- **App icon** is the application's own artwork, resized.

## Access

Source access for hiring or due-diligence purposes can be discussed at the owner's discretion.
