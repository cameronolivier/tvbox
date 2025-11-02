# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

**RetroTV+** is a classic TV experience recreation system for modern media libraries. The system schedules continuous, non-interactive 24/7 playback across multiple themed "channels" that users can tune into on Android TV, Xbox, or web clients.

**Current Status:** Planning/Design phase - no implementation code yet. Only architecture and requirements documentation exists.

## Core Architectural Concepts

### Key Principles
- **Determinism**: Identical inputs (seed) → identical schedules → reproducible EPGs
- **Wall-clock truth**: Server UTC time is the single source of truth for "now"
- **Stateless clients**: Clients only query `/now` endpoint; no local timeline logic
- **Schema-driven contracts**: Shared Zod schemas generate runtime + compile-time validation across all apps

### Tech Stack (Planned)
- **Monorepo**: pnpm + Turborepo
- **Server**: TypeScript + Fastify + Drizzle ORM (Postgres)
- **Admin UI**: Next.js (App Router) + tRPC + Tailwind + shadcn/ui
- **Clients**: Android (Kotlin + ExoPlayer), Xbox (C# + UWP)
- **Shared Contracts**: Zod + zod-to-openapi
- **Queue**: BullMQ for background scans and schedule rebuilds
- **Storage**: Local FS or mounted NAS for media files

### Repository Structure (Planned)
```
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
```

## API Design

### Critical Endpoints
- `GET /channels/:slug/now` - Returns current item + playback offset (must respond < 200ms)
- `GET /channels/:slug/epg?from&to` - Get schedule window
- `POST /channels/:slug/rebuild` - Force schedule regeneration
- `POST /library/folders/:id/rescan` - Trigger media scan

### Response Requirements
- All timestamps must be UTC ISO 8601 strings
- `/now` endpoint includes `offsetMs` for precise playback positioning
- Zod validation on all request/response payloads

## Database Schema Concepts

Four primary tables:
1. **media_assets** - ffprobe metadata (path, duration, hash, dimensions)
2. **channels** - channel config + scheduling policy (JSONB)
3. **schedule_items** - EPG entries (type: asset | slate | ad)
4. **ad_assets** - vintage ad inventory with era tags

Policy JSONB includes:
- `ordering`: shuffle | filenameAsc | recentFirst
- `allowRepeatWithinHours`: anti-repeat window
- `ads.insertEveryMinutes`: ad break cadence
- `blackouts`: daily or one-time off-air windows (test pattern)

## Scheduling Logic

The **scheduler** is the most complex component:
- Must be **deterministic** (same seed → same EPG output)
- Generates full 24-hour schedules
- Prevents repeats within configured hours
- Inserts slate during blackout windows
- Inserts ads at specified intervals with era-based weighting
- Gap-fill strategy: loop or slate

Critical offset calculation: Given UTC now, find current schedule item and return milliseconds elapsed within that item.

## Testing Strategy

When implementing tests, prioritize:

1. **Schedule determinism** - Property-based tests ensuring identical seeds produce identical EPGs
2. **Offset math** - `/now` offset accuracy ± 250ms
3. **Boundary transitions** - Seamless next-item switches (test near end-of-file scenarios)
4. **Blackout windows** - Slate correctly inserted during off-air periods
5. **Ad breaks** - Duration sum matches target block length
6. **Schema validation** - Zod rejects invalid payloads early

### Testing Tools
- **Unit**: Vitest for scheduler logic, utilities
- **Integration**: Supertest for API + DB + Zod validation
- **Contract**: zod-to-openapi tests for cross-client schema match
- **E2E**: Playwright for admin UI flows
- **Snapshot**: Mock clock tests for deterministic EPG output

## Development Commands (When Implemented)

Once code exists, expected commands will be:
```bash
# Install dependencies
pnpm install

# Database
pnpm db:migrate
pnpm db:seed

# Development
pnpm dev              # Start all apps
pnpm dev:server       # Fastify API only
pnpm dev:admin        # Next.js UI only

# Testing
pnpm test:unit        # Schedule, offset, blackout tests
pnpm test:contract    # Schema drift guard
pnpm test:e2e         # Channel CRUD happy path
pnpm test:integration # API + DB validation

# Build
pnpm build
pnpm build:android    # Android TV client
pnpm build:xbox       # Xbox UWP client

# Type checking
pnpm typecheck

# Linting
pnpm lint
```

## Implementation Guidance

### When Starting Implementation
1. Set up monorepo structure first (pnpm-workspace.yaml, turbo.json)
2. Create shared-contracts package with Zod schemas before any apps
3. Set up Drizzle schema and migrations early
4. Build scheduler as pure functions (easier to test)
5. Mock clock in tests from the start (Date.now() abstraction)

### Scheduler Implementation
- Accept a seed parameter for deterministic shuffling
- Use cryptographic-quality PRNG (seeded) for ordering
- Calculate total duration before building schedule
- Pre-compute all transitions to avoid runtime gaps
- Return immutable schedule objects

### Client Implementation
- Clients should poll `/now` every 30-60 seconds
- Handle time drift: if offsetMs suggests we're behind, seek forward
- Never implement seek/pause/rewind controls
- Display channel name and current title only

### Media Scanning
- Use `ffprobe` to extract duration, codec, dimensions
- Hash files to detect duplicates
- Store original path + metadata in `media_assets`
- Queue scans via BullMQ, not inline

## Non-Goals (V1)
- Multi-user accounts or cloud sync
- Live transcoding or adaptive bitrate
- Monetization, analytics, or DRM
- HLS/DASH output (future enhancement)

## Success Criteria
- `/now` endpoint responds in < 200ms
- ≥ 99.9% deterministic EPG generation under identical seeds
- Zero playback gaps across file transitions
- CI runs all critical schedule tests in < 2 min

## Additional Resources
- See `PRD.md` for full product requirements
- See `ARCHITECTURE` for detailed system design
- Ad sources: Archive.org public-domain 80s/90s commercials
