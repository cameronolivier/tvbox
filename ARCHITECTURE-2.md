# RetroTV+ — System Architecture

---

## 1. High-Level Overview

**Architecture type:** polyglot monorepo (pnpm + Turborepo)  
**Stack:** TypeScript everywhere (Fastify + Drizzle + Next.js), Kotlin/Compose for TV, C#/UWP for Xbox.

apps/
server/        # Fastify + Drizzle + BullMQ
admin/         # Next.js + tRPC + Tailwind
android-tv/    # Kotlin + ExoPlayer
xbox-uwp/      # C# + UWP MediaPlayerElement
packages/
shared-contracts/  # Zod schemas → OpenAPI → clients
shared-utils/
infra/
docker/
k8s/

---

## 2. Architectural Principles

- **Determinism** – identical inputs ⇒ identical schedules  
- **Server = clock** – all clients use UTC offsets from `/now`  
- **Schema-first** – Zod schemas generate runtime & compile-time types  
- **TypeScript end-to-end** – one language across services  
- **Stateless clients** – playback logic lives on server  
- **Loose coupling** – shared contracts, independent deploys  

---

## 3. Tech Stack

| Layer | Technology | Purpose |
|-------|-------------|----------|
| API | Fastify + Drizzle (Postgres) | Media metadata, scheduling |
| DB | Postgres | Relational core |
| Validation | Zod | Shared runtime + type contracts |
| Queue | BullMQ | Background scans & schedule rebuilds |
| Admin UI | Next.js + tRPC + Tailwind | Channel & library mgmt |
| Android | Kotlin + ExoPlayer + Compose for TV | Viewer app |
| Xbox | C# + UWP MediaPlayerElement | Viewer app |
| Infra | Docker + Nginx | Deploy & serve media |

---

## 4. API Surface

| Method | Path | Description |
|--------|------|-------------|
| GET | `/channels` | list channels |
| POST | `/channels` | create/update |
| GET | `/channels/:slug/now` | current item + offset |
| GET | `/channels/:slug/epg?from&to` | EPG window |
| POST | `/channels/:slug/rebuild` | regenerate schedule |
| GET | `/library/folders` | list folders |
| POST | `/library/folders/:id/rescan` | trigger scan |
| GET | `/ads` | list ad inventory |
| POST | `/ads/import` | import Archive.org ads |

---

## 5. Database Schema (Drizzle)

```ts
import { pgTable, text, integer, boolean, timestamp, jsonb } from 'drizzle-orm/pg-core'

export const mediaAssets = pgTable('media_assets', {
  id: text('id').primaryKey(),
  folderId: text('folder_id').notNull(),
  path: text('path').notNull(),
  filename: text('filename').notNull(),
  durationMs: integer('duration_ms').notNull(),
  width: integer('width'),
  height: integer('height'),
  hash: text('hash').notNull(),
  addedAt: timestamp('added_at').defaultNow(),
  metadata: jsonb('metadata'),
})

export const channels = pgTable('channels', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  slug: text('slug').unique().notNull(),
  timezone: text('timezone').default('UTC'),
  isLive: boolean('is_live').default(true),
  policy: jsonb('policy').notNull(),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
})

export const scheduleItems = pgTable('schedule_items', {
  id: text('id').primaryKey(),
  channelId: text('channel_id').notNull(),
  assetId: text('asset_id'),
  type: text('type').notNull(), // asset | slate | ad
  startUtc: timestamp('start_utc').notNull(),
  endUtc: timestamp('end_utc').notNull(),
  title: text('title'),
  meta: jsonb('meta'),
})

export const adAssets = pgTable('ad_assets', {
  id: text('id').primaryKey(),
  title: text('title'),
  sourceUrl: text('source_url'),
  localPath: text('local_path'),
  license: text('license'),
  durationMs: integer('duration_ms'),
  eraTag: text('era_tag'),
})


⸻

6. Shared Zod Schemas

import { z } from 'zod'

export const ZScheduleItem = z.object({
  id: z.string(),
  channelId: z.string(),
  type: z.enum(['asset','slate','ad']),
  startUtc: z.string().datetime(),
  endUtc: z.string().datetime(),
  title: z.string().optional(),
  meta: z.record(z.unknown()).optional(),
})

export const ZNowResponse = z.object({
  item: ZScheduleItem,
  offsetMs: z.number().int().nonnegative(),
  assetUrl: z.string().url(),
  serverTimeUtc: z.string().datetime(),
})

export const ZAutoSchedulePolicy = z.object({
  ordering: z.enum(['shuffle','filenameAsc','recentFirst']),
  alignBlocksMinutes: z.number().optional(),
  allowRepeatWithinHours: z.number(),
  gapFill: z.enum(['loop','slate']),
  ads: z.object({
    enable: z.boolean(),
    insertEveryMinutes: z.number(),
    breakDurationSeconds: z.number(),
    eraBias: z.array(z.object({ era: z.enum(['80s','90s']), weight: z.number() })).optional(),
  }).optional(),
  blackouts: z.array(
    z.union([
      z.object({ kind: z.literal('daily'), startLocal: z.string(), stopLocal: z.string() }),
      z.object({ kind: z.literal('once'), startUtc: z.string(), stopUtc: z.string() }),
    ])
  ).optional(),
})


⸻

7. Testing Strategy

Levels

Type	Tool	Scope
Unit	Vitest	Scheduler logic, utilities
Integration	Supertest	API + DB + Zod validation
Contract	zod-to-openapi tests	Cross-client schema match
E2E	Playwright	Admin UI flows
Snapshot	Vitest + Mock clock	Deterministic EPG output

Critical Test Areas
	•	Schedule determinism – same seed ⇒ identical results
	•	Offset math – /now offset accurate ± 250 ms
	•	Boundary transitions – seamless next-item switch
	•	Blackout windows – slate inserted correctly
	•	Ad breaks – duration sum ≈ target block length
	•	Schema validation – reject invalid payloads early

⸻

8. Deployment

Environment	Tooling
Local	Docker Compose (API + Postgres + Next)
CI	GitHub Actions + Turbo cache
Prod	Fly.io / Railway containers
Storage	Local FS or S3 mount
Cron	BullMQ recurring jobs


⸻

9. Key Architectural Decisions

Decision	Justification
Drizzle ORM	type-safe SQL, light migrations
Zod contracts	single truth for validation & OpenAPI
Fastify	low-latency /now endpoint
Static media serving	avoids runtime concatenation
Monorepo	unified DX, atomic CI
Public-domain ads	legal clarity
Wall-clock scheduling	deterministic multi-client sync
Minimal client logic	reliability & cross-platform ease


⸻

10. Automation Targets
	•	pnpm test:unit → schedule, offset, blackout
	•	pnpm test:contract → schema drift guard
	•	pnpm test:e2e → channel CRUD happy path
	•	pnpm build:android / build:xbox in CI matrix
	•	Nightly job: rebuild schedules for next 7 days

⸻

11. Future Work
	•	HLS/DASH output pipeline
	•	AI bumper generator (text-to-video)
	•	Multi-viewer sync mode
	•	Analytics + watch stats
	•	Public API for community channel sharing

⸻

Prepared by: Senior Architect — RetroTV+ Core Engineering
Revision: v0.1