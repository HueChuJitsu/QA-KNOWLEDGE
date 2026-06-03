# QA Training — Jitsu

QA training documentation and lessons, set in the context of the **Jitsu** last-mile delivery platform.

## Structure

- [docs/](docs/) — the QA onboarding & reference documentation set (read these first)
- [lessons/](lessons/) — topic-based QA lessons (added over time)

## Documentation set

| Document | What it covers |
|---|---|
| [jitsu-context.en.md](docs/jitsu-context.en.md) | **Read first.** Domain foundation: shipment lifecycle, routing, booking, delivery, apps & platform tech. |
| [jitsu-qa-welcome-kit.md](docs/jitsu-qa-welcome-kit.md) | Day-1 starter kit: schedule, people, Slack channels, golden rules, contacts. |
| [jitsu-qa-onboarding-plan-2weeks.md](docs/jitsu-qa-onboarding-plan-2weeks.md) | Day 1 → Day 14 learning roadmap. |
| [jitsu-qa-training-document.md](docs/jitsu-qa-training-document.md) | Main reference — business, systems, processes, full E2E testing guide. |
| [jitsu-qa-database-cheat-sheet.md](docs/jitsu-qa-database-cheat-sheet.md) | Common PostgreSQL & MongoDB queries for daily QA work. |
| [jitsu-qa-postman-guide.md](docs/jitsu-qa-postman-guide.md) | Install, configure, and use the QA Master Postman collection. |
| [jitsu-qa-webapp-release-process-cheatsheet.md](docs/jitsu-qa-webapp-release-process-cheatsheet.md) | Quick reference for the isolated-release-branch SDLC. |
| [jitsu-qa-webapp-release-process-full-guide.md](docs/jitsu-qa-webapp-release-process-full-guide.md) | Full QA workflow per environment + edge cases. |
| [jitsu-qa-probation-goals.md](docs/jitsu-qa-probation-goals.md) | 2-month probation goals for Junior & Senior QA. |

## Where to start

1. Read [docs/jitsu-context.en.md](docs/jitsu-context.en.md) to learn the domain: shipment lifecycle, routing, booking, delivery, apps.
2. Follow the [2-week onboarding plan](docs/jitsu-qa-onboarding-plan-2weeks.md) day by day.
3. Use the [training document](docs/jitsu-qa-training-document.md) as your main reference, and the lesson index below.

## Lesson index (proposed)

| # | Lesson | QA focus |
|---|---|---|
| 01 | Domain & terminology | Understand shipment, route, zone, IC/DSP/3P, OTD, SLA |
| 02 | Shipment lifecycle | Test the inbound → sort → outbound → delivery flow |
| 03 | Routing & booking | Routing edge cases, advance/direct booking, sprinkling |
| 04 | Delivery & POD | Geofencing, POD validation, fraud detection |
| 05 | Apps & API | Web/mobile apps, Public/Driver API, offline-first |

> Lesson files in `lessons/` will be added over time.
