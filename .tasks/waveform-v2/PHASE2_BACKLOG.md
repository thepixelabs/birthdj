# Waveform v2 — Phase 2 Backlog

Owner: CTO. Decision framework: data from beta telemetry. The items below are seeded from Phase 1 known limitations and the epic's Phase 2 section. Re-prioritise after 4-6 weeks of beta data.

## Risk assessment from Phase 1 deviations

### P0 (must fix before beta) — RESOLVED in Phase 12
- ~~`PlaylistSession.from_dict` referenced `BlockArchetype` without importing it — would crash on every Resume.~~ FIXED.
- ~~`playlist_exported` analytics call missing `event_type` arg.~~ FIXED.
- ~~Genre weights lost on session resume.~~ FIXED (added to both `_serialize_session` and `from_dict`).
- ~~`EventTemplate.accent_color` field missing — Event Skin Change always tweened to violet.~~ FIXED with per-template defaults.

### P1 (should fix before beta) — none currently blocking
All P1 items from the Phase 11 known limitations list have either been resolved in Phase 12 or downgraded to P2 with a documented workaround.

### P2 (Phase 2 backlog)
1. Full keyboard navigation for block list (BlockCard not Tab-navigable)
2. Block Transition opacity simulation (CTk has no opacity — would require custom rendering)
3. Particle burst widget tree-walk fragility (refactor to direct widget reference)
4. Genre tag index expansion from ~90 to ~300 Spotify seed tags
5. Tooltip widget (CTk has none) for "max 6 genres" and similar UX hints

---

## Must-have for Phase 2

These are the highest-leverage carryovers from Phase 1 known limits:

1. **Full keyboard navigation pass.** BlockCard, GenreSlider, song preview card. WCAG AA is the floor; full keyboard parity is the ask.
2. **Session resume polish.** Confirm genre weights, vibe text, and exported_playlist_ids all round-trip cleanly. Add a "what got lost" toast if any field is dropped.
3. **Block Transition redesign.** CTk fade approximation is visually weak. Either accept it or render the active card on a separate Toplevel that we can fade properly.
4. **Custom block archetypes** beyond rename/recolor. Beta users will ask for this. Define the data model and the upload path.

## High-value from the epic's Phase 2 section

5. **Session history browser with full resume.** Today the dialog lists sessions; resume only re-creates blocks. Add: re-open exported playlist URL, re-export with new tweaks, fork into a new session.
6. **Tier 2 cover art (DALL-E / Imagen composite).** Stub exists behind `COVER_ART_TIER` flag. Wire real provider, A/B against Tier 1 in beta.
7. **Per-block cover art mode** — currently off by default to keep first export fast. Make it a one-toggle preference, surface render time so the user understands the trade.
8. **Veto reason chip analytics dashboards.** `reason_tag` is captured but not visible in PostHog. Build the dashboard before Phase 2 kickoff so we're not flying blind.
9. **Windows distribution pass.** macOS-first for beta; Phase 2 ships .exe, signing, install path differences.
10. **In-app master prompt editor.** Already user-editable on disk; promote to a hidden Advanced panel.
11. **Save / share custom event templates.** Local "Save as template" first; cloud sharing is Phase 3.

## Data-driven (prioritize off beta metrics)

These are the gates the beta data will open or close:

- **If `veto_depth` is high (>3 vetoes per kept song):** Gemini prompt-quality sprint. Iterate `master_prompt.md` and `veto_addendum.md` against a curated reference set.
- **If `preview_to_keep_rate` is <30%:** preview UX investigation. Likely culprits: preview audio cutting too short, album art too small, no waveform feedback. Run a usability session.
- **If `session_abandoned` spikes at event_setup:** template gallery redesign. Maybe we have too many templates, or the descriptions don't sell the differences.
- **If `time_to_first_export` >15 minutes:** onboarding flow. Add a "quickstart" path that pre-fills everything and lets the user export in three clicks.
- **If `generation_completed` latency >20s p50:** Gemini call optimisation, streaming chunk size, or model swap (Flash → Pro tradeoff).

## Strategic open questions to resolve before Phase 2 kickoff

These are the §11 items the CEO still owns:

1. Monetization model (free / paid / freemium with export limits)
2. Spotify preview_url fallback (legal review for YouTube scrape)
3. Gemini cost envelope per session at expected veto depth
4. Distribution: Windows signing cert investment timing
5. Existing v1 paying users — migration comms plan
6. Audio library reconfirmation (pygame vs python-vlc) based on beta crash reports
7. Custom archetype scope cap (how to ship this without opening infinite theming)
8. Veto history retention policy and any cloud upload — privacy policy gating

## Tauri + React migration: when?

The epic earmarks Tauri as the long-term path. Decision criterion: if (a) beta validates the hypothesis (`preview_to_keep_rate` >40% and session completion rate >60%) AND (b) we are CTk-bound on a feature beta users explicitly request (live waveform scrubbing, collaborative sessions, web parity), THEN start the migration in Phase 3. Until both are true, every CTk hour is cheaper than every Tauri hour.
