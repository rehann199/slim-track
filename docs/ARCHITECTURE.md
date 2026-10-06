# Architecture

All diagrams below are original and conceptual. Component names describe responsibilities; they are not the names of production classes, files, collections or routes.

## 1. System overview

<p align="center">
  <img src="../assets/diagrams/system-overview.svg" alt="System overview: Flutter app, on-device SQLite, Firebase services, serverless proxy on Cloudflare Workers, third-party language-model API, store billing and ad mediation" width="900" />
</p>

The device is the working store. Cloud services are either optional (backup for signed-in users) or feature-specific (the Pro weekly summary and purchase verification behind one small serverless proxy).

## 2. Layers inside the app

```mermaid
flowchart TD
    subgraph Presentation
        SCR[Feature screens<br/>dashboard, log, history, chart, goal,<br/>BMI detail, insights, photos, settings]
        NAV[go_router<br/>tabbed shell + pushed routes]
        DS[Design system<br/>theme, motion, feedback, widgets]
    end
    subgraph State
        PROV[Riverpod providers<br/>entries, derived statistics, profile,<br/>auth, pro status, locale, theme, connectivity]
    end
    subgraph Domain
        UTL[Pure utilities<br/>unit conversion, BMI, dates, export]
    end
    subgraph Data
        REPO[Repositories]
        DB[(SQLite)]
        SYNC[Cloud sync service]
    end
    subgraph Services
        NET[Proxy clients<br/>summary, purchase verification]
        NOTI[Notifications]
        ADS[Ad gate + remote ad config]
        ANA[Analytics layer]
        BILL[Billing]
    end
    SCR --> NAV
    SCR --> DS
    SCR --> PROV
    PROV --> UTL
    PROV --> REPO
    REPO --> DB
    REPO --> SYNC
    PROV --> NET
    PROV --> BILL
    SCR --> NOTI
    SCR --> ADS
    SCR --> ANA
```

| Layer | Responsibility |
|---|---|
| **Presentation** | Feature-first screens, a tabbed shell with pushed detail routes, a shared design system (colours, motion tokens, staggered entrances, pressable feedback, tap sound and haptics, a guided tour) |
| **State** | Riverpod notifiers and streams hold entries, profile, auth and Pro state; derived providers compute statistics (start, current, lowest, highest, change, tracking span) so every screen reads the same numbers |
| **Domain utilities** | Stateless functions for unit conversion, BMI and category, reference range, date handling and export formatting. They hold no state and have no framework dependency beyond localisation labels |
| **Data** | Repositories wrap SQLite and, when signed in, the cloud sync service. SQLite is written first |
| **Services** | Clients for the serverless proxy, local notifications, the startup ad gate, the analytics layer and billing |

## 3. Calculation pipeline

See the diagram in the [technical case study](TECHNICAL_CASE_STUDY.md#2-numerical-correctness-the-headline). The key invariant is that stored values are kilograms and centimetres at full precision, and rounding is applied only by formatting helpers at render time.

## 4. Logging a weight

```mermaid
sequenceDiagram
    participant U as User
    participant S as Log sheet
    participant N as Entries notifier
    participant R as Repository
    participant D as SQLite
    participant C as Cloud sync
    U->>S: enter weight in display unit
    S->>S: range check, one decimal
    S->>N: save (converted to kg, full precision)
    N->>R: add entry
    R->>D: insert, replace on same date
    N-->>N: update in-memory list, newest first
    opt signed in
        R-->>C: background write with timeout
        C-->>D: mark synced on success
    end
    N-->>S: derived statistics and BMI update everywhere
```

If the cloud write fails or times out, the row simply stays unsynced and is retried by a flush on reconnect.

## 5. Sign-in merge

```mermaid
flowchart TD
    A[Explicit sign-in] --> B{Cloud has data?}
    B -- no --> C[Push local entries, profile and weekly insights]
    B -- yes --> D[Push local entries, profile and insights up]
    D --> E[Pull cloud data down]
    E --> F[Merge by date, newest write wins]
    F --> G[Replace local rows in one transaction]
    C --> H[Done]
    G --> H
```

The merge is intentionally tied to an explicit sign-in rather than cold start, because it replaces local rows.

## 6. Weekly summary request

```mermaid
sequenceDiagram
    participant A as App
    participant W as Proxy (Workers)
    participant K as Key-value store
    participant L as Language-model API
    A->>W: name, goal, unit, recent weights + identity token
    W->>W: verify token (signature, issuer, audience, expiry)
    W->>K: rate limit check
    W->>K: read entitlement (fail closed)
    W->>K: check weekly usage
    W->>L: generate short summary
    L-->>W: text
    W->>K: record usage (only on success)
    W-->>A: text + remaining-usage metadata
    A->>A: save to local database
```

## 7. Purchase verification

```mermaid
sequenceDiagram
    participant A as App
    participant ST as Store billing
    participant W as Proxy (Workers)
    participant K as Key-value store
    A->>ST: purchase or restore
    ST-->>A: signed purchase data
    A->>A: set cached Pro flag (UI only)
    A->>W: signed purchase data + identity token
    W->>W: verify signatures offline against platform trust anchors
    W->>K: store time-boxed entitlement for the user
    Note over A,W: later summary requests read the entitlement
```

## 8. Startup

Startup initialises local services and Firebase, loads remote ad configuration with a fail-closed default, seeds the cached Pro flag before the first frame so Pro users never see upgrade prompts or ad slots, then runs a launch gate. On iOS the tracking prompt comes first and analytics collection is enabled only afterwards. Each phase is timed with a performance trace.

## 9. Responsibilities at a glance

| Concern | Where it lives | Why |
|---|---|---|
| Numbers shown to the user | Domain utilities plus derived providers | One source for every screen |
| Entitlement decisions for paid backend features | Serverless proxy | A client flag cannot protect a server resource |
| Provider credentials | Serverless proxy only | Nothing secret ships in the app |
| Offline correctness | SQLite first | Logging must never wait on the network |
| Ad on/off control | Remote configuration, fail closed | Misconfiguration or SDK failure must not block or surprise users |
| Analytics hygiene | Typed, bucketed event layer | Health-adjacent values do not leave the device through analytics |

## 10. Deliberate simplicity

The app uses repositories and a handful of singleton services beside Riverpod providers rather than a heavier layered framework. For a product of this size the benefit of fewer indirections outweighs the cost of a few cross-layer calls, and the pure calculation utilities remain trivially testable.
