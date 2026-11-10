# Infinite Motion Xpress Inc.

We build software for Philippine local government units (LGUs), healthcare and urban mobility.

## Xicar: ride-hailing, food and mart

Xicar handles ride-hailing for two services, **Xiclo** (two-wheels) and **Xicar** (four-wheels), plus food and mart delivery.

| Repo | What it is | Stack |
|---|---|---|
| [Xicar-core](https://github.com/Infinite-Motion-Xpress-Inc/Xicar-core) | Backend. Replaces the older Laravel API. Runs the ride-hailing booking engine, a Pusher-protocol socket, background workers, and the food-mart GraphQL/REST service. Money is handled by a separate wallet service, and every money call carries an idempotency key. | Rust, Postgres, Redis, FCM, M360 SMS |
| [Xicar-app](https://github.com/Infinite-Motion-Xpress-Inc/Xicar-app) | Mobile monorepo with three apps: **Customer**, **Driver** and **Attendant**. | Flutter, mise |
| [Xicar-console](https://github.com/Infinite-Motion-Xpress-Inc/Xicar-console) | Admin dashboard for operations: customers, drivers, fare settings, cancellation fees, refunds. Has no database of its own. It proxies to core's admin API, and admins sign in with Cloudflare Access. | Cloudflare Workers, Hono, React, Vite |

**Status:** Xiclo and Xicar rides work and are tested. Food and mart runs end to end against the backend but has not yet been tested with the apps on a device.

## WorkForce360: HRIS for LGUs and private organizations

WorkForce360 is a multi-tenant HRIS that covers attendance, payroll, leave and directives. Each organization runs its own tenant, either on an edge box on site or hosted on our VPS. A central server handles licensing and routes users to their tenant.

| Repo | What it is | Stack |
|---|---|---|
| [Central-Server](https://github.com/Infinite-Motion-Xpress-Inc/Central-Server) | Runs the platform: license authority, tenant broker (maps an org code to its Cloudflare Tunnel URL), edge hardware provisioning, admin governance, audit logs, PSGC reference data, and demo-request intake. | Django REST Framework, Next.js 16 |
| [LGUProject-admin](https://github.com/Infinite-Motion-Xpress-Inc/LGUProject-admin) | The per-tenant backend and admin dashboard. `tenant-api` serves tRPC, REST and WebSockets and handles Philippine statutory payroll (BIR TRAIN, GSIS, SSS, PhilHealth, Pag-IBIG) and face liveness checks. `tenant-dashboard` covers org charts, DTR, batch payroll and approvals. Also holds the edge-box and hosted deployment kits. | NestJS, Prisma, PostgreSQL, React 19 |
| [HRISApplication](https://github.com/Infinite-Motion-Xpress-Inc/HRISApplication) | Employee mobile app, offline-first. Clock-in requires a liveness check and GPS, and is blocked on rooted, jailbroken or mock-location devices. Also shows urgent directives employees must acknowledge, and a task inbox with photo and GPS proof. | React Native (Expo), SQLite |
| [Product-website](https://github.com/Infinite-Motion-Xpress-Inc/Product-website) | Marketing site with an ROI calculator, an AI demo, and a "Book a demo" form that sends leads to the Central Server. | Astro, React, Tailwind v4, Cloudflare Pages |

## CityKiosk: city services portal

[CityKiosk](https://github.com/Infinite-Motion-Xpress-Inc/CityKiosk) is a public services platform for the Iloilo City government. It has two parts:

- **Citizen mobile app:** a directory of government service requirements (welfare, civil registry, permits, tax, clearances), telehealth video calls with barangay doctors, a clinic directory, and push notifications.
- **Barangay kiosk:** a touchscreen web terminal with offline caching, a document checklist generator (print it or scan a QR code to move it to a phone), a telehealth booth for walk-ins, and queue tickets.

Stack: React Native (Expo), web kiosk, admin web, backend, LiveKit/WebRTC.

## OneHealth PH: healthcare

[OneHealth](https://github.com/Infinite-Motion-Xpress-Inc/OneHealth) gives patients and practitioners their own workspaces. Features include clinic discovery and booking, health passports, messaging, clinical documents, medicine tracking and clinic inventory, plus an admin console for review and support.

Stack: Expo / React Native, Next.js, Node API, Prisma, PostgreSQL, Turborepo.

## Miscellaneous utilities

| Repo | What it is |
|---|---|
| [git-discord-bot](https://github.com/Infinite-Motion-Xpress-Inc/git-discord-bot) | Internal tooling, not tied to any product. Syncs GitHub issues with Discord threads both ways and provides `/issue`, `/repos` and `/link` commands. Sends alerts for workflow failures, releases and usage (Brevo credits, AWS Rekognition cost) to Discord or Telegram. Runs on Render with Neon Postgres, and has a dashboard on Vercel. |
| [.github](https://github.com/Infinite-Motion-Xpress-Inc/.github) | Org profile and shared community files. |

## Organization standards

### CI runners

Every job declares the kind of machine it needs, using a label. Heavy platform builds go to runners reserved for them, and everything else goes to the general pool.

| Label | Runners | Use it for |
|---|---|---|
| `light` | Utarou PC Shared Runner (Linux x64, 5 runners). Utarou Fedora Shared Runner is the backup. | General work: lint, typecheck, unit tests, web builds, Docker image builds, deploy scripts. |
| `build-android` | TotoBakodz Runner Group (x64, 5 runners) | Android only: Gradle, `flutter build apk/appbundle`, `eas build --local --platform android`. |
| `build-ios` | Cassandra Runner (macOS ARM64) | iOS only: Xcode, CocoaPods, `flutter build ipa`, `eas build --local --platform ios`. |
| `<repo>-<env>`, e.g. `xicar-core-stable` | One runner on the target server, registered with no default labels | Deploy steps that must run on that server. |
| `ubuntu-latest`, `macos-latest` | GitHub-hosted | Fallback when our runners are down, and any job that runs code from a fork. |

#### Rules

1. **Route jobs through variables, never hardcode a runner.** Read the label from an org variable and fall back to GitHub-hosted, so changing one variable reroutes every repo:

   ```yaml
   lint:          { runs-on: "${{ vars.CI_RUNNER_LIGHT || 'ubuntu-latest' }}" }
   build-android: { runs-on: "${{ vars.ANDROID_RUNNER || 'ubuntu-latest' }}" }  # ANDROID_RUNNER=build-android
   build-ios:     { runs-on: "${{ vars.IOS_RUNNER || 'macos-latest' }}" }        # IOS_RUNNER=build-ios
   ```

2. **Specialized runners are reserved.** A `build-android` or `build-ios` runner carries only its own label, never `light`, so a lint job can never hold up a release build. General runners never carry a build label.
3. **There is no automatic fallback or priority.** GitHub Actions queues a job until a runner with *all* of its labels is free, and it never falls back to another runner. Priority comes only from rules 1 and 2. If a pool goes down, set its variable to `ubuntu-latest` / `macos-latest` until the pool is back.
4. **Split by platform.** A mobile release is separate `build-android` and `build-ios` jobs, never one job that builds both. Shared steps (tests, versioning) run first on `light`.
5. **Throttle heavy builds.** Builds that share a machine set `strategy.max-parallel` or a `concurrency` group. Point Gradle/EAS temp and output directories at the workspace, not `/tmp`, and stop Gradle daemons when the job ends.
6. **Fork code never runs on self-hosted runners.** Jobs triggered by a fork pull request use GitHub-hosted runners or are skipped.
7. **Each job logs into the container registry itself.** Self-hosted runners on one machine share `~/.docker`, so one job's login must not be reused by another.
