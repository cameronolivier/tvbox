# RetroTV+ — Tech Setup

This file gets the repo from zero → dev-ready with shared contracts, Drizzle, Fastify API, Next.js admin, Android TV, and Xbox projects.

---

## 0) Prereqs

- Node 20+
- pnpm 9+ (`corepack enable` then `corepack prepare pnpm@latest --activate`)
- Docker Desktop (or podman; adjust compose)
- ffmpeg + ffprobe (system packages)
- Java 17 (Android), Android Studio (for `android-tv`)
- Visual Studio 2022 + UWP workload (for `xbox-uwp`)

---

## 1) Bootstrap the Monorepo

```bash
mkdir tv-linear && cd tv-linear
git init
pnpm init -y
````

**Directory layout**

```
tv-linear/
├─ apps/
│  ├─ server/
│  ├─ admin/
│  ├─ android-tv/      # created in Android Studio later
│  └─ xbox-uwp/        # created in Visual Studio later
├─ packages/
│  ├─ shared-contracts/
│  ├─ shared-utils/
│  └─ eslint-config/
├─ infra/
│  └─ docker/
└─ docs/
```

---

## 2) Workspace & Turbo

**`pnpm-workspace.yaml`**

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

**`turbo.json`**

```json
{
	"$schema": "https://turbo.build/schema.json",
	"pipeline": {
		"build": {
			"dependsOn": ["^build"],
			"outputs": ["dist/**", ".next/**", "build/**"]
		},
		"typecheck": {},
		"lint": {},
		"test": { "dependsOn": ["^build"], "outputs": ["coverage/**", ".vitest/**"] },
		"contract:gen": { "outputs": ["packages/shared-contracts/openapi.json"] },
		"dev": { "cache": false }
	}
}
```

**Root `package.json` (scripts only)**

```json
{
	"private": true,
	"scripts": {
		"dev": "turbo run dev --parallel",
		"build": "turbo run build",
		"typecheck": "turbo run typecheck",
		"lint": "turbo run lint",
		"test": "turbo run test",
		"contract:gen": "turbo run contract:gen",
		"prepare": "husky"
	},
	"devDependencies": {
		"turbo": "^2.1.0",
		"typescript": "^5.6.3",
		"eslint": "^9.13.0",
		"prettier": "^3.3.3",
		"husky": "^9.1.6",
		"lint-staged": "^15.2.10"
	}
}
```

**Root `.prettierrc` (your prefs)**

```json
{ "semi": false, "useTabs": true, "singleQuote": true }
```

**Root `.eslintrc.cjs` (baseline)**

```js
module.exports = {
	root: true,
	extends: ['eslint:recommended'],
	parserOptions: { ecmaVersion: 2023, sourceType: 'module' },
	ignorePatterns: ['dist', 'build', '.next', 'node_modules'],
}
```

**Root `tsconfig.base.json`**

```json
{
	"compilerOptions": {
		"target": "ES2022",
		"module": "ESNext",
		"moduleResolution": "Bundler",
		"strict": true,
		"skipLibCheck": true,
		"resolveJsonModule": true,
		"types": ["node"]
	},
	"exclude": ["**/dist", "**/build", "**/.next"]
}
```

**Husky + lint-staged**

```bash
pnpm dlx husky init
pnpm add -D lint-staged
```

`.husky/pre-commit`

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

pnpm typecheck && pnpm lint-staged
```

`package.json` (root) add:

```json
"lint-staged": {
	"**/*.{ts,tsx,js,json,md}": ["prettier --write"]
}
```

---

## 3) Drizzle + Postgres + Env

```bash
pnpm add -D drizzle-kit
pnpm add drizzle-orm pg
```

**Root `drizzle.config.ts`**

```ts
import 'dotenv/config'
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
	schema: './apps/server/src/db/schema.ts',
	out: './apps/server/drizzle',
	dialect: 'postgresql',
	dbCredentials: { url: process.env.DATABASE_URL! }
})
```

**`apps/server/.env.example`**

```env
DATABASE_URL=postgres://postgres:postgres@localhost:5432/retro
MEDIA_ROOT=/media
PORT=8080
```

---

## 4) Docker (DB + API + static media)

**`infra/docker/docker-compose.yml`**

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: retro
    ports: ["5432:5432"]
    volumes: [ "db:/var/lib/postgresql/data" ]

  api:
    build: ../../apps/server
    env_file:
      - ../../apps/server/.env
    volumes:
      - /mnt/media:/media:ro
    ports: ["8080:8080"]
    depends_on: [ db ]

volumes:
  db: {}
```

Spin up DB locally during dev:

```bash
docker compose -f infra/docker/docker-compose.yml up -d db
```

---

## 5) Server App (Fastify + Drizzle)

```bash
mkdir -p apps/server/src/{db,routes,lib}
pnpm add fastify fastify-type-provider-zod zod
pnpm add @fastify/static @fastify/cors
pnpm add -D vitest supertest tsx
```

**`apps/server/src/db/schema.ts`**

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
	metadata: jsonb('metadata')
})

export const channels = pgTable('channels', {
	id: text('id').primaryKey(),
	name: text('name').notNull(),
	slug: text('slug').unique().notNull(),
	timezone: text('timezone').default('Africa/Johannesburg'),
	isLive: boolean('is_live').default(true),
	policy: jsonb('policy').notNull(),
	createdAt: timestamp('created_at').defaultNow(),
	updatedAt: timestamp('updated_at').defaultNow()
})

export const scheduleItems = pgTable('schedule_items', {
	id: text('id').primaryKey(),
	channelId: text('channel_id').notNull(),
	assetId: text('asset_id'),
	type: text('type').notNull(),
	startUtc: timestamp('start_utc').notNull(),
	endUtc: timestamp('end_utc').notNull(),
	title: text('title'),
	meta: jsonb('meta')
})

export const adAssets = pgTable('ad_assets', {
	id: text('id').primaryKey(),
	title: text('title'),
	sourceUrl: text('source_url'),
	localPath: text('local_path'),
	license: text('license'),
	durationMs: integer('duration_ms'),
	eraTag: text('era_tag')
})
```

**`apps/server/src/db/client.ts`**

```ts
import { drizzle } from 'drizzle-orm/node-postgres'
import { Pool } from 'pg'

const pool = new Pool({ connectionString: process.env.DATABASE_URL })
export const db = drizzle({ client: pool })
```

**`apps/server/src/index.ts`**

```ts
import Fastify from 'fastify'
import cors from '@fastify/cors'
import fstatic from '@fastify/static'
import { ZodTypeProvider } from 'fastify-type-provider-zod'
import { join } from 'node:path'
import { nowRoute } from './routes/now'

const app = Fastify().withTypeProvider<ZodTypeProvider>()
await app.register(cors, { origin: true })
await app.register(fstatic, { root: join(process.env.MEDIA_ROOT ?? '/media') })

app.get('/health', async () => ({ ok: true }))

await nowRoute(app)

const port = Number(process.env.PORT ?? 8080)
app.listen({ port, host: '0.0.0.0' })
```

**`apps/server/src/routes/now.ts`**

```ts
import { z } from 'zod'
import type { FastifyInstance } from 'fastify'
import { ZNowResponse } from '@tv/shared-contracts'

export async function nowRoute(app: FastifyInstance) {
	app.get('/channels/:slug/now', {
		schema: {
			params: z.object({ slug: z.string() }),
			response: { 200: ZNowResponse }
		}
	}, async (req, reply) => {
		const { slug } = req.params as { slug: string }
		// TODO load scheduleItem from DB
		const now = new Date()
		return reply.send({
			item: {
				id: 'stub',
				channelId: 'ch1',
				type: 'slate',
				startUtc: now.toISOString(),
				endUtc: new Date(now.getTime() + 60_000).toISOString(),
				title: 'Station Slate'
			},
			offsetMs: 0,
			assetUrl: 'http://localhost:8080/smpte_60s.mp4',
			serverTimeUtc: now.toISOString()
		})
	})
}
```

**`apps/server/package.json`**

```json
{
	"name": "@tv/server",
	"private": true,
	"type": "module",
	"scripts": {
		"dev": "tsx watch src/index.ts",
		"build": "tsc -p tsconfig.json",
		"typecheck": "tsc -p tsconfig.json --noEmit",
		"lint": "eslint .",
		"test": "vitest",
		"db:push": "drizzle-kit push",
		"db:generate": "drizzle-kit generate"
	},
	"dependencies": {
		"@fastify/cors": "^10.0.0",
		"@fastify/static": "^7.0.0",
		"drizzle-orm": "^0.36.0",
		"fastify": "^5.0.0",
		"fastify-type-provider-zod": "^4.0.0",
		"pg": "^8.12.0",
		"zod": "^3.23.8",
		"dotenv": "^16.4.5"
	},
	"devDependencies": {
		"tsx": "^4.16.0",
		"vitest": "^2.1.3"
	}
}
```

**`apps/server/tsconfig.json`**

```json
{ "extends": "../../tsconfig.base.json", "compilerOptions": { "outDir": "dist" }, "include": ["src"] }
```

Run migrations:

```bash
cp apps/server/.env.example apps/server/.env
pnpm --filter @tv/server db:push
```

---

## 6) Shared Contracts (Zod → OpenAPI)

```bash
mkdir -p packages/shared-contracts/src
pnpm add -w zod @asteasolutions/zod-to-openapi
```

**`packages/shared-contracts/src/contracts.ts`**

```ts
import { z } from 'zod'

export const ZScheduleItem = z.object({
	id: z.string(),
	channelId: z.string(),
	type: z.enum(['asset','slate','ad']),
	startUtc: z.string().datetime(),
	endUtc: z.string().datetime(),
	title: z.string().optional(),
	meta: z.record(z.unknown()).optional()
})

export const ZNowResponse = z.object({
	item: ZScheduleItem,
	offsetMs: z.number().int().nonnegative(),
	assetUrl: z.string().url(),
	serverTimeUtc: z.string().datetime()
})

export type NowResponse = z.infer<typeof ZNowResponse>
```

**`packages/shared-contracts/scripts/build.ts`**

```ts
import { OpenAPIRegistry, OpenApiGeneratorV3 } from '@asteasolutions/zod-to-openapi'
import { ZNowResponse, ZScheduleItem } from '../src/contracts.js'
import { writeFileSync, mkdirSync } from 'node:fs'
import { dirname } from 'node:path'
const reg = new OpenAPIRegistry()
reg.register('ScheduleItem', ZScheduleItem)
reg.register('NowResponse', ZNowResponse)

const gen = new OpenApiGeneratorV3(reg.definitions)
const doc = gen.generateDocument({
	openapi: '3.0.3',
	info: { title: 'Linear TV API', version: '1.0.0' },
	paths: {
		'/channels/{slug}/now': {
			get: {
				parameters: [{ name: 'slug', in: 'path', required: true, schema: { type: 'string' } }],
				responses: { '200': { description: 'ok', content: { 'application/json': { schema: { $ref: '#/components/schemas/NowResponse' } } } } }
			}
		}
	}
})
mkdirSync(dirname('packages/shared-contracts/openapi.json'), { recursive: true })
writeFileSync('packages/shared-contracts/openapi.json', JSON.stringify(doc, null, 2))
```

**`packages/shared-contracts/package.json`**

```json
{
	"name": "@tv/shared-contracts",
	"private": true,
	"type": "module",
	"exports": { ".": "./src/contracts.ts" },
	"scripts": {
		"build": "node scripts/build.ts",
		"contract:gen": "node scripts/build.ts",
		"typecheck": "tsc -p tsconfig.json"
	},
	"devDependencies": {
		"@asteasolutions/zod-to-openapi": "^7.1.1",
		"typescript": "^5.6.3"
	},
	"dependencies": { "zod": "^3.23.8" }
}
```

**`packages/shared-contracts/tsconfig.json`**

```json
{ "extends": "../../tsconfig.base.json", "include": ["src", "scripts"] }
```

---

## 7) Admin App (Next.js)

```bash
pnpm dlx create-next-app@latest apps/admin --ts --eslint --tailwind --app --src-dir --no-experimental-app
pnpm --filter apps/admin add @tanstack/react-query zod
pnpm --filter apps/admin add -D @types/node
```

Add a simple page that calls `/channels/:slug/now` and renders “On Now”. Wire your own tRPC or fetch, your call.

**Dev**

```bash
pnpm --filter @tv/server dev
pnpm --filter apps/admin dev
```

---

## 8) Android & Xbox Projects

* **Android**: create `apps/android-tv` in Android Studio (Empty Activity, Kotlin).
  Add dependencies: ExoPlayer, Compose for TV. Point your Retrofit/OkHttp client at `openapi.json` (add codegen later).
* **Xbox**: create `apps/xbox-uwp` (Blank App, C# + UWP).
  Use `MediaPlayerElement` and a tiny `HttpClient` calling `/now`.

Keep these in monorepo; Turbo won’t build them unless you wire tasks:

```json
// root turbo.json (optional tasks)
"android:build": {},
"xbox:build": {}
```

---

## 9) Testing

**Root `vitest.config.ts`**

```ts
import { defineConfig } from 'vitest/config'
export default defineConfig({
	test: {
		globals: true,
		environment: 'node',
		reporters: ['default'],
		coverage: { reporter: ['text', 'lcov'] }
	}
})
```

Example server test **`apps/server/src/routes/now.test.ts`**

```ts
import { describe, it, expect } from 'vitest'

describe('now', () => {
	it('returns a NowResponse stub', async () => {
		expect(true).toBe(true)
	})
})
```

Run:

```bash
pnpm test
```

---

## 10) ffmpeg Slate Generator

**`scripts/gen_slates.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

OUT_DIR="${1:-/media/slates}"
mkdir -p "$OUT_DIR"

gen () {
	local SECS="$1"
	local OUT="$OUT_DIR/smpte_${SECS}s.mp4"
	ffmpeg -y \
		-f lavfi -i smptehdbars=size=1920x1080:rate=30000/1001 \
		-f lavfi -i "sine=frequency=1000:sample_rate=48000:duration=${SECS}" \
		-c:v libx264 -preset veryfast -crf 18 -pix_fmt yuv420p \
		-c:a aac -b:a 192k \
		-shortest "$OUT"
}

gen 30
gen 60
gen 300
echo "Slates written to $OUT_DIR"
```

---

## 11) Seed Script (optional)

**`apps/server/src/seed.ts`**

```ts
import { db } from './db/client'
import { randomUUID } from 'node:crypto'

async function main() {
	const channelId = randomUUID()
	await db.execute(`insert into channels (id,name,slug,policy) values ($1,$2,$3,$4)`, [
		channelId, 'Retro Cartoons', 'retro-cartoons',
		JSON.stringify({ ordering: 'shuffle', allowRepeatWithinHours: 24, gapFill: 'slate' })
	])
	console.log('seeded channel retro-cartoons')
}
main()
```

Run:

```bash
tsx apps/server/src/seed.ts
```

---

## 12) GitHub Actions (CI Skeleton)

**`.github/workflows/ci.yml`**

```yaml
name: ci
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: corepack enable
      - run: corepack prepare pnpm@latest --activate
      - run: pnpm i --frozen-lockfile
      - run: pnpm contract:gen
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build
```

---

## 13) Daily Jobs

Use BullMQ in `apps/server` to:

* scan folders (ffprobe)
* rebuild EPG for next 7 days
* refresh ad inventory (Archive.org importer)

Wire a `pnpm task-runner` or cron that calls an endpoint:

```bash
curl -X POST http://localhost:8080/admin/rebuild?days=7
```

---

## 14) Dev Commands

```bash
# 1) DB only
docker compose -f infra/docker/docker-compose.yml up -d db

# 2) Contracts
pnpm contract:gen

# 3) Server
pnpm --filter @tv/server dev

# 4) Admin
pnpm --filter apps/admin dev

# Optional: build all
pnpm build
```

---

## 15) Gotchas

* Ensure `MEDIA_ROOT` points at a real directory with readable files
* If serving media over Nginx, set `add_header Accept-Ranges bytes`
* South Africa TZ has no DST — still store UTC in DB and convert for UI
* Don’t ingest YouTube unless you have explicit rights; prefer Archive.org PD assets
