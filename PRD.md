📄 Product Requirements Document (PRD)

Product Name

RetroTV+ — AI-assisted “linear” media player that turns user-loaded videos into continuous TV-like channels.

⸻

Overview

RetroTV+ provides a nostalgic television experience: continuous playback, no seek/pause, and scheduled programming curated via a web interface.
The user (admin) defines channels, uploads content, sets scheduling rules, and optionally inserts off-air periods or vintage ads. Clients (Android TV, Xbox, Web) simply tune in and watch what’s “on now.”

⸻

Goals

Goal	Description
🕓 Continuous linear playback	Create deterministic 24/7 programming schedules from a media library
📺 Multi-channel support	Multiple channels with unique scheduling policies (Kids, Movies, Music, etc.)
🧭 Wall-clock deterministic playback	All clients play the same segment at the same time
🧱 Offline-safe	No live transcoding required; files can be pre-encoded
🛠️ Simple admin UX	Lightweight “Plex-lite” interface for media ingestion and scheduling
💤 Off-air periods	Play a test pattern or slate between defined hours
📺 Vintage ads	Insert PD ads from Archive.org between shows
🌍 Cross-platform	Android TV + Xbox (UWP) clients; web admin
✅ Deterministic testing	Given the same seed and policy, schedules are reproducible


⸻

Non-Goals (V1)
	•	No multi-user streaming (each client acts independently)
	•	No live transcoding / adaptive bitrate (may add later)
	•	No user personalization (no login required for viewers)
	•	No DRM

⸻

Core Features

Feature	Description
Library scanner	Watches local folders, uses ffprobe to extract duration/codecs
Scheduler	Deterministically generates EPG per channel
Channel lineup	Weighted selection of folders for each channel
Ad block generator	Pulls PD vintage commercials from Archive.org
Blackouts / Off-air	Test pattern between configurable times
REST API	/channels/:slug/now, /channels/:slug/epg, /library/*
Web Admin	Channel creation, scheduling preview, ad import
Clients	Android TV (ExoPlayer) + Xbox (UWP MediaPlayerElement)
Contracts	Shared via Zod schemas → OpenAPI for Android/Xbox codegen
Testing	Integration tests for schedule generation, time offsets, and playback boundaries
