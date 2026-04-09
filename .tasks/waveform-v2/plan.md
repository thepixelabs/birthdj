---
epic: waveform-v2
status: BETA_READY
phases:
  - id: 1
    title: "Foundation: repo restructure, settings schema, Spotify/Gemini service layer"
    persona: staff-engineer
    status: DONE
  - id: 2
    title: "CustomTkinter shell: three-column layout, navigation, theme tokens"
    persona: staff-engineer
    status: DONE
  - id: 3
    title: "Event type system + built-in templates"
    persona: staff-engineer
    status: DONE
  - id: 4
    title: "Block builder timeline (horizontal drag-and-drop, resize, energy sparkline)"
    persona: staff-engineer
    status: DONE
  - id: 5
    title: "Generalized genre weight system (tag autocomplete + per-block sliders)"
    persona: staff-engineer
    status: DONE
  - id: 6
    title: "AI generation pipeline with streaming + veto-feedback loop"
    persona: staff-engineer
    status: DONE
  - id: 7
    title: "Song preview card feed (album art, 30s preview, Keep/Skip/Veto)"
    persona: staff-engineer
    status: DONE
  - id: 8
    title: "Cover art generator v1 (parametric PIL per block visual world)"
    persona: staff-engineer
    status: DONE
  - id: 9
    title: "Spotify export + persistence + session history"
    persona: staff-engineer
    status: DONE
  - id: 10
    title: "PostHog analytics instrumentation"
    persona: staff-engineer
    status: DONE
  - id: 11
    title: "Polish pass: animations, accessibility, signature motion set"
    persona: staff-engineer
    status: DONE
  - id: 12
    title: "Beta release, telemetry review, Phase 2 planning"
    persona: cto
    status: DONE
---

# Waveform v2 — Epic

## 1. Vision Statement

Waveform v2 turns a single-file birthday-playlist script into a desktop app that any host, promoter, planner or DJ can use to design the soundtrack of an entire event. Users describe the event, sketch a schedule of blocks on a timeline, dial in the vibe with genre weights, and then co-curate the playlist with an AI that listens when they say "no." It should feel less like a form and more like a small, tasteful studio — warm dark UI, real previews, responsive motion, and cover art that looks like it belongs on a poster. The business bet: playlist generation is a commodity, but *taste tooling* — the feedback loop, the timeline, the visual craft — is defensible and worth paying for.

## 2. Strategic Decision: GUI Framework

**Decision: CustomTkinter for MVP. Re-evaluate at Phase 2 based on beta telemetry, with Tauri + React as the designated long-term path. PyQt6 is rejected.**

Both engineers are right about something. The Creative Technologist is right that PyQt6 is the most capable Python desktop toolkit — QPainter, QMediaPlayer and QPropertyAnimation genuinely would produce a more beautiful v1. The Product Engineer is right that CustomTkinter ships weeks faster and lets us validate the core hypothesis (does the veto loop actually change user behavior?) before we invest in a toolkit we'd eventually replace anyway.

As CEO my job is to pick time-to-signal over craft ceiling. Three reasons:

1. **We don't yet know if anyone wants this.** The acquired product has no telemetry and a tiny user base. Spending an extra 4-6 weeks on QPainter custom widgets for a hypothesis we haven't validated is the wrong trade. Ship, measure, then invest in polish where it moves metrics.
2. **PyQt6 is not our endgame anyway.** If Waveform v2 succeeds, we will migrate to Tauri + React for cross-platform distribution, web parity, and a hiring pool that is an order of magnitude larger than PyQt specialists. Every hour of QPainter work is throwaway. CustomTkinter is *also* throwaway, but it's cheaper throwaway.
3. **"Beautiful" is not the same as "Qt."** The Digital Illustrator's design language — gradients, waveform marks, block visual worlds, cover art — lives in PIL/Pillow and image assets, not in the widget toolkit. ~80% of the visual identity ships regardless of framework.

Concessions to the Creative Technologist: we budget real time for a motion/polish pass in Phase 1 (not Phase 2), we use `pygame.mixer` or `python-vlc` for audio (not Tkinter's anemic options), and we accept that the v1 waveform visualization is a pre-rendered animated PNG, not a custom-painted live widget. That's a known, acceptable compromise.

## 3. Epic Scope

**In scope (v2 MVP):**
- CustomTkinter desktop app (macOS first, Windows best-effort)
- Event type system with named templates and free-text vibe override
- Block builder: horizontal timeline, drag-to-reorder, drag-to-resize, add/remove
- Generalized genre weight system replacing psytrance special-casing
- Gemini-powered generation with streaming card feed
- Veto feedback loop (rejections feed back into the same session's prompt context)
- 30-second song previews via Spotify preview_url, with Keep / Skip / Veto
- Spotify playlist export with per-block cover art
- Parametric PIL cover art per block (Tier 1 of the illustrator's 3-tier system)
- Local settings/persistence, per-event session history, duplicate detection
- PostHog analytics
- A motion and accessibility polish pass before release

**Explicitly out of scope for v2 (deferred to later phases):**
- Tauri + React migration
- Custom user-defined block archetypes beyond naming/recoloring built-ins
- Collaborative / multi-user editing
- DALL-E or Imagen cover art (Tier 2)
- Generative SVG cover art (Tier 3)
- Apple Music, Tidal, YouTube Music
- Mobile apps
- Cloud sync
- Paid tier / billing (we will instrument for it, not ship it)
- Live waveform scrubbing with custom-painted widgets
- Event skin system as a full theming engine (we ship a palette swap only)

## 4. Architecture Overview

Move from `create_playlist.py` to a package:

```
waveform/
  app/
    __init__.py
    main.py                  # entry point, boots UI
    state.py                 # app state store (observable)
  ui/
    shell.py                 # three-column layout
    sidebar_schedule.py
    timeline_canvas.py
    track_panel.py
    widgets/
      block_card.py
      track_card.py
      genre_slider.py
      waveform_anim.py
    theme.py                 # color/typography/spacing tokens
  domain/
    event.py                 # EventType, EventTemplate
    block.py                 # Block, BlockArchetype
    session.py               # PlaylistSession, VetoContext
    genre.py                 # GenreWeight, GenreTagIndex
  services/
    spotify_client.py        # Spotipy wrapper (auth, search, export, previews)
    gemini_client.py         # generation + streaming + veto re-prompting
    cover_art.py             # PIL cover art tiers
    preview_audio.py         # python-vlc or pygame.mixer wrapper
    analytics.py             # PostHog
    persistence.py           # settings.json, session history, prompt files
  assets/
    fonts/ icons/ textures/ block_palettes/ event_skins/
  prompts/
    master_prompt.md
    veto_addendum.md
  tests/
```

Principles:
- Pure domain layer, no Tk imports below `ui/`.
- All Spotify and Gemini calls go through a service with retries, timeouts, and a fake for tests.
- App state is a single observable store; UI subscribes. No shared mutable globals.
- Every long-running operation is async (thread pool) with a cancel token, so the UI never freezes.

## 5. Feature Specifications

### 5.1 Event Type System
- `EventTemplate` = `{ id, name, description, default_blocks[], default_genres[], skin_id, suggested_duration }`.
- Ships with: Birthday, Wedding, Club Night, Rooftop Bar, Corporate Dinner, House Party, Funeral/Memorial, Road Trip, Workout, Focus Session.
- On event selection: template seeds the schedule and genre weights, but everything is editable. A free-text "vibe" field is always available and is passed verbatim into the AI prompt.
- Users can save custom templates locally (Phase 1 light: "Save as template"; full sharing is out of scope).

### 5.2 Block Builder (Timeline)
- Horizontal canvas, time on the x-axis, one row of block cards.
- Each block: name, archetype (visual world), duration, energy level (1-5), genre weights (inherits from event, overridable per block).
- Interactions: click to add, drag to reorder, grab edge to resize, double-click to rename, right-click to delete.
- Above the timeline: an energy-arc sparkline derived from block energy levels and AI-estimated BPM once generation has run. This is the Creative Technologist's idea and we're keeping it — it's cheap and it makes the product feel intelligent.
- Snap-to-5-minutes by default, hold Shift to freeform.

### 5.3 Generalized Genre Weight System
- Replaces the hardcoded psytrance slider completely. No special cases.
- Each block has a list of `(genre_tag, weight 0-80%)` pairs. Sum can exceed 100 — weights are relative nudges, not quotas. The prompt converts them into "lean heavily into X, moderately into Y" language for Gemini.
- UI: a tag autocomplete input (backed by a curated genre index of ~300 tags derived from Spotify's genre seeds) plus a per-tag slider row.
- Weights can be inherited from the event-level defaults or overridden per block. Inheritance is visible (ghosted sliders).
- Max 6 active genres per block to keep the prompt legible.

### 5.4 AI Generation + Veto Feedback Loop
- This is the killer feature. Protect it.
- Generation streams: Gemini returns songs one at a time (or in small batches), the UI animates them into the card feed as they arrive.
- Veto feedback: when the user rejects a song, we capture `(song, artist, block_context, optional_reason_tag)` into a session-scoped `VetoContext`. On the next generation call within the same session, the context is injected into the prompt as "Avoid songs like these and the reasons why." Reasons are optional quick-tap chips: "too slow", "wrong genre", "overplayed", "not the vibe", "artist already used."
- Keep actions also feed back (positive reinforcement) but more lightly.
- Duplicate detection uses existing per-playlist history plus a new cross-session "recently exported" cache.
- Every generation call is idempotent and retryable; failures surface as inline toasts, not modal errors.

### 5.5 Song Preview & Approve / Reject / Swap UX
- Right-hand Track Panel. One song at a time, big.
- Album art fills the top; below it a pre-rendered animated waveform PNG (breathing loop); below that three buttons with keyboard shortcuts: Approve (`Space`), Swap (`S`), Veto (`Backspace`).
- Preview plays the Spotify 30-second clip via `python-vlc` (`preview_url` from the track object). If a track has no preview_url we show a "no preview available" state and still allow all three actions.
- On Approve: card slides into the block's track list on the timeline with a particle burst.
- On Swap: immediately request a single replacement suggestion from Gemini with veto context.
- On Veto: updates `VetoContext`, card shrinks and fades, next card slides in.
- Accessibility: every custom widget has a screen-reader label, focus ring is visible, all actions are keyboard-reachable, and there is a "reduce motion" toggle in settings.

### 5.6 Cover Art Generation System
- Phase 1 only ships Tier 1: parametric PIL art per block visual world (see Block Type Library below). This means multi-stop gradients, noise overlays, geometric accents, and typography.
- Tier 2 (DALL-E / Imagen composite at 60% screen blend) is stubbed behind a feature flag and deferred.
- Tier 3 (generative SVG) is deferred.
- One playlist-level cover art per export (derived from the dominant block archetype) and optionally per-block covers if the user enables "cover per block" (off by default to keep the first export fast).

### 5.7 Settings & Persistence
- `~/.waveform/settings.json` for app-wide settings (theme, reduce motion, analytics opt-in, Spotify auth tokens).
- `~/.waveform/sessions/<event>/` for per-event state: block layout, genre weights, keep/veto history, last export.
- `master_prompt.md` ships with the app and is user-editable via a hidden advanced panel.
- Migration: on first launch, import any existing `settings.json` from the v1 tool and translate the psytrance config into the new genre weight schema.

## 6. Design System

**Palette (dark, warm):**
- Background base `#0D0D0F`
- Surface raised `#17171B`
- Surface overlay `#1F1F25`
- Text primary `#F5F5F7`, secondary `#A1A1AA`, muted `#6B6B73`
- Brand gradient: deep violet `#1A0533` → electric indigo `#6B2FFA` → hot magenta `#E040FB`
- Accent violet `#7C3AED`
- Live/now-playing cyan `#22D3EE`
- Semantic: success `#34D399`, warning `#FBBF24`, danger `#F87171`

**Typography:**
- Display: Inter Tight (700) for block titles and logo mark
- UI: Inter (400/500/600)
- Mono: JetBrains Mono for debug/advanced panels only

**Spacing & radius:** 4/8/12/16/24/32 scale. Radii: 8 (inputs), 14 (cards), 20 (modals).

**Logo mark:** five vertical waveform bars, left-to-right gradient from `#1A0533` through `#6B2FFA` to `#E040FB`. The same bars are reused as the generation loading indicator ("Generation Pulse").

**Motion principles:**
- Spring physics for physical elements (block resize, card swipe)
- Ease-out for entries, ease-in-out for state changes, no linear
- Target 60fps; fall back gracefully under "reduce motion" (fades only)
- Five signature animations must ship in v1: Generation Pulse, Block Card Reveal (stagger), Song Approved (particle burst), Block Transition (crossfade blur), Event Skin Change (palette ripple)

**Accessibility:** WCAG AA contrast on all text, visible focus rings, full keyboard navigation, screen-reader labels, reduce-motion toggle.

## 7. Block Type Library (Built-in Archetypes)

Each archetype is a named visual world with its own palette, cover art recipe, and default energy level. Users can attach any archetype to any block.

1. **Arrival** — watercolor bleeds; champagne, dusty rose, deep plum; energy 2.
2. **Chill / Ambient** — lava-lamp blobs; ocean teal, moon white, pale violet; energy 1-2.
3. **Singalong** — warm confetti dots on cream; coral, gold, soft red; energy 3.
4. **Groove** — rounded shapes with soft glow; amber, burnt orange, sienna; energy 3-4.
5. **Dance Floor** — hard concentric rings; neon cyan, hot pink on black; energy 5.
6. **Club Night** — acid green and UV violet, circuit-board texture, data-viz accents; energy 5.
7. **Late Night** — grainy smoky purples and ash greys, slow sinusoidal curves; energy 3.
8. **Sunrise** — Rothko horizontal bands of coral, peach, lavender; energy 2.
9. **Ceremony / Reverent** (new, for weddings/funerals) — linen texture, muted pastels, thin serif accent; energy 1.
10. **Peak** (new, generic climax block) — radial burst, brand gradient maxed; energy 5.

The old v1 block names (chill, singalong, dance, groove, sunrise) all map cleanly into this library so migration is trivial.

## 8. Event Type Templates

Each template is `{ default blocks, default genre weights, skin palette override, suggested duration }`. "Skin" in v1 means a palette swap and a couple of texture overlays, not a full theming engine.

- **Birthday** — Arrival → Singalong → Groove → Dance Floor → Late Night. Confetti dots, hot coral accent.
- **Wedding** — Ceremony → Arrival → Singalong → Groove → Dance Floor → Sunrise. Linen texture, rose gold accent.
- **Club Night** — Arrival → Groove → Dance Floor → Peak → Club Night → Late Night. Scanline texture, acid green accent.
- **Rooftop Bar** — Chill → Groove → Dance Floor (light) → Sunrise. Warm sunset palette.
- **Corporate Dinner** — Arrival → Chill → Groove. Muted palette, minimal texture.
- **House Party** — Arrival → Singalong → Groove → Dance Floor. Default brand palette.
- **Funeral / Memorial** — Ceremony → Chill → Singalong (gentle). Linen texture, desaturated.
- **Road Trip** — Singalong → Groove → Singalong. Warm highway palette.
- **Workout** — Peak → Dance Floor → Groove (cooldown). High-contrast palette.
- **Focus Session** — Chill → Ambient. Minimal, monochrome.

## 9. Analytics Plan (PostHog)

Instrument aggressively from day one. These are the events that tell us whether the hypothesis is real:

- `app_opened`, `session_started` (with event_type)
- `event_template_selected` (template_id, whether free-text vibe also entered)
- `block_added`, `block_removed`, `block_resized`, `block_reordered`
- `genre_weight_changed` (block_id, tag, weight)
- `generation_requested` (block_id, n_existing_vetoes)
- `generation_completed` (block_id, latency_ms, n_songs_returned)
- `song_previewed` (track_id, block_id, preview_duration_played)
- `song_kept`, `song_skipped`, `song_vetoed` (with optional reason_tag)
- `swap_requested`
- `playlist_exported` (n_blocks, n_tracks, time_from_open_ms)
- `session_abandoned` (last_step_reached)
- `error_surfaced` (source, type)

Primary metrics: **session completion rate**, **preview-to-keep rate**, **average vetoes per kept song** (we want this to *drop* over a session — that's the feedback loop working), **time-to-first-export**.

Analytics are opt-in on first launch with a clear explainer. No PII, no track names in production payloads unless the user opts into "help improve suggestions" sharing.

## 10. Phased Delivery Plan

**Phase 1 — MVP (ship in ~8 weeks from kickoff)**
Everything in Section 3 "In scope." The bar is: a user can pick an event template, edit the block timeline, set genre weights, generate a playlist with streaming previews, veto songs and feel the AI respond, and export to Spotify with cover art. Plus analytics, plus a real motion pass, plus accessibility basics. Ship to a closed beta of ~50 users recruited from the existing v1 user list and the owner's network.

**Phase 2 — Polish & Depth (after 4-6 weeks of beta telemetry)**
Decisions here are data-driven. Likely candidates: custom block archetypes, per-block cover art by default, session history browser, Tier 2 cover art (DALL-E / Imagen), reason-tag chips on veto, improved waveform visualization, Windows polish, in-app prompt editor, save/share custom event templates. Also: evaluate whether CustomTkinter is still the right container or whether beta users' feedback justifies accelerating the Tauri migration.

**Phase 3 — Platform Scale**
Tauri + React migration, Apple Music / Tidal connectors, cloud sync, collaborative sessions, mobile companion, paid tier with billing. We do not commit to any of this until Phase 1 metrics justify it.

## 11. Open Questions

1. **Monetization:** free during beta, clearly. After that — paid app, subscription, or freemium with export limits? Needs a call before Phase 2 so we can instrument the right funnel.
2. **Spotify API terms:** preview_url is not guaranteed for every track and Spotify has been tightening preview availability. Do we need a fallback (e.g., YouTube preview scraping)? Legal review required if yes.
3. **Gemini cost envelope:** what's the unit cost per completed session at expected veto depth? Need a ballpark before we decide if generation is unlimited or rate-limited in free tier.
4. **Distribution:** notarized .dmg for macOS is mandatory; Windows signing cert — do we invest now or after Phase 1?
5. **Owner's personal v1 users:** are any of them paying? If yes, v2 must not break their flow on day one. Migration path and comms needed.
6. **Audio library choice:** `python-vlc` (requires VLC installed) vs `pygame.mixer` (bundled but clunkier) vs `just_playback`. Engineering call in Phase 1 kickoff week.
7. **Custom block archetypes:** v2 ships with 10 built-ins only. If beta users ask loudly for custom archetypes, how fast can we ship that without opening the floodgates to infinite theming work?
8. **Data retention:** how long do we keep veto history locally, and do we ever upload it? Privacy policy needs to exist before first external beta invite.

---

This document is the north star for Waveform v2. If a proposal doesn't move one of the primary metrics in Section 9 or doesn't serve the vision in Section 1, the default answer is no.
