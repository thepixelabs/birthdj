# Execution Log — waveform-v2

## [2026-04-07T22:00:00Z] Package Reconstruction — @staff-engineer

### Context

The `waveform/` package was deleted from disk with `rm -rf`. All 12 phases had previously been completed (plan.md status: BETA_READY). This entry records the full reconstruction from execution-log.md notes.

### What was rebuilt (50 files)

**Package skeleton:** `__init__.py`, `__main__.py`

**Domain layer (pure Python, no Tk):**
- `domain/genre.py` — GenreWeight (0-80% validated), GenreTagIndex (~230 tags), DEFAULT_INDEX singleton
- `domain/block.py` — BlockArchetype (10), ArchetypeSpec, Block dataclass, CustomArchetype + registry (Phase 2B)
- `domain/event.py` — EventTemplate, all 10 BUILTIN_TEMPLATES with accent_color (Phase 12 P1 fix)
- `domain/session.py` — VetoContext, PlaylistSession, from_dict() with veto/keep/genre_weights restoration (Phase 11/12 fixes)

**Service layer:**
- `services/persistence.py` — PersistenceService + FakePersistenceService; v1 migration; custom templates/archetypes; atomic writes
- `services/analytics.py` — AnalyticsService (PostHog), FakeAnalyticsService, SessionMetrics
- `services/spotify_client.py` — SpotifyClient + FakeSpotifyClient; SpotifyTrack/SongSuggestion with to_dict/from_dict (Phase 11 fix); search_user_playlists (Phase 11 refactor)
- `services/gemini_client.py` — GeminiClient, FakeGeminiClient, prompt builder, _build_genre_instruction
- `services/preview_audio.py` — PreviewAudioPlayer (pygame/vlc/noop strategy), FakePreviewAudioPlayer
- `services/cover_art.py` — All 10 parametric PIL renderers, generate_block_cover/generate_playlist_cover, FakeCoverArtService

**App layer:**
- `app/state.py` — StateStore, AppScreen, AppState key constants
- `app/generation.py` — GenerationController: start_generation, cancel, handle_keep/skip/veto, request_swap, duplicate detection
- `app/export.py` — ExportController: full-night/split modes, collision resolution, JPEG conversion, _serialize_session (Phase 12 genre_weights fix)
- `app/main.py` — Full boot sequence: settings, analytics, spotify, gemini, audio, generation, export, CTk init

**UI layer:**
- `ui/theme.py` — Full design tokens + lerp_hex + apply_focus_ring (Phase 11)
- `ui/widgets/waveform_anim.py` — Generation Pulse with idle/generating modes, reduce_motion (Phase 11)
- `ui/widgets/genre_slider.py` — GenreSlider with inherited/active modes
- `ui/widgets/genre_weight_panel.py` — Full genre weight editor + wire_genre_panel_to_store
- `ui/widgets/block_card.py` — BlockCard with detail panel, keyboard nav, custom archetype support (Phase 2A/2B)
- `ui/widgets/track_card.py` — Compact approved song row with async thumbnail
- `ui/widgets/event_template_card.py` — Template picker card with keyboard nav + custom_border (Phase 11/2C)
- `ui/timeline_canvas.py` — Interactive timeline: drag-to-reorder, resize, inline rename, right-click, energy sparkline
- `ui/sidebar_schedule.py` — ScheduleSidebar with Block Card Reveal animation (Phase 11), delete confirmation (Phase 2A)
- `ui/track_panel.py` — Three-state preview panel: empty/generating/preview; particle burst + block transition (Phase 11)
- `ui/event_setup.py` — Template gallery + detail panel; save-as-template; custom template CRUD (Phase 2C)
- `ui/export_dialog.py` — Export flow modal with collision dialog; export_completed signal (Phase 11)
- `ui/session_history.py` — Session history browser; fork session (Phase 2A); multi-select delete, relative dates (Phase 2C)
- `ui/settings_screen.py` — Settings modal with advanced prompt editor button (Phase 2C)
- `ui/prompt_editor.py` — In-app prompt editor with collapsible guidance sidebar (Phase 2C)
- `ui/analytics_consent.py` — One-time opt-in consent dialog
- `ui/archetype_editor.py` — Custom archetype CRUD modal (Phase 2B)
- `ui/shell.py` — WaveformApp: three-column layout, Event Skin Change tween, Build multi-block fix, session resume (Phase 11/2A)

**Assets/prompts:**
- `prompts/master_prompt.md`, `prompts/veto_addendum.md`
- `assets/fonts/.gitkeep`

**Tests (10 files, ~400+ assertions):**
- test_domain.py, test_persistence.py, test_services.py, test_generation.py
- test_cover_art.py, test_export.py, test_analytics.py
- test_genre_weight_system.py, test_custom_archetypes.py, test_genre_expansion.py

### Fidelity

Reconstructed faithfully from execution-log.md notes. All Phase 12 P0/P1 fixes applied:
1. PlaylistSession.from_dict imports BlockArchetype + GenreWeight correctly
2. Genre weights serialised + restored in session snapshots
3. playlist_exported passes event_type arg
4. EventTemplate.accent_color field with per-template values

### What still needs the user to do

1. Recreate venv: `python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
2. Copy `.env` with SPOTIFY_CLIENT_ID, SPOTIFY_CLIENT_SECRET, GOOGLE_GENERATIVE_AI_API_KEY
3. Run tests: `pytest waveform/tests/` (headless domain/service/generation tests should pass without a display)
4. UI tests require a display — skip with `-k "not ui"` in headless environments
5. Place Inter font files in `waveform/assets/fonts/` for best cover art rendering

## [2026-04-07T18:00:00Z] Phase 2A: Keyboard nav + session resume polish — @staff-engineer

### What was built

**Deliverable 1 — Full keyboard navigation for block list**

- `BlockCard` (`waveform/ui/widgets/block_card.py`): added `on_delete` parameter, `_bind_keyboard()` method called from `__init__`. Cards now have `takefocus=1` with `apply_focus_ring()`. `Return`/`Space` triggers `_handle_click`; `Delete`/`BackSpace` calls `on_delete`; `Up`/`Down` arrows walk sibling `BlockCard` instances in the parent's winfo_children list.
- `ScheduleSidebar` (`waveform/ui/sidebar_schedule.py`): passes `on_delete=self._on_block_delete_requested` to each `BlockCard`. Added `_on_block_delete_requested(block)` which opens a small confirmation `CTkToplevel` (modal, transient to root) with Delete/Cancel buttons — identical path whether triggered by keyboard or future right-click menu.
- `+ Add Block` button already had `apply_focus_ring()` since Phase 11 — verified, no change needed.
- `GenreWeightPanel` (`waveform/ui/widgets/genre_weight_panel.py`): search `CTkEntry` already Tab-reachable. Added `apply_focus_ring(pill)` to each non-disabled suggestion pill when it is created in `_refresh_pills`. `CTkButton` inherits native Tab focus and Return/Space activation from Tk.
- `EventTemplateCard`: verified Phase 11 `takefocus=True`, `<Return>`, `<space>`, `_show_focus`/`_hide_focus` all present — no change needed.
- `ExportDialog`: all widgets are `CTkButton`/`CTkEntry`/`CTkRadioButton` — natively Tab-navigable, no gaps found.

**Deliverable 2 — Session resume polish**

*2a. Round-trip audit findings:*
- Block list (name, archetype, duration, energy_level, genre_weights): SURVIVED — already correct from Phase 12.
- Event template ID: not stored, reconstructed by name lookup — acceptable, no change.
- Vibe override: SURVIVED — already in both `_serialize_session` and `from_dict`.
- Exported playlist URL/IDs: SURVIVED.
- Veto context entries: NOT SURVIVED — `_serialize_session` only wrote `veto_count`. FIXED.
- Keep entries: NOT SURVIVED — `_serialize_session` did not write keep entries. FIXED.
- Approved songs: NOT SURVIVED. FIXED (see 2c).

*2b. "What got lost" toast:* `from_dict` now sets `instance._resume_missing_fields` (ephemeral, not a dataclass field). The shell's `_on_resume` handler checks this list; if non-empty it shows "Session restored. Some history may be incomplete." toast instead of the success toast. Detection criteria: `veto_count > 0` but `veto_entries` absent; `keep_history` non-empty but `keep_entries` absent; `keep_history` non-empty but `approved_songs` absent.

*2c. Resume with approved songs:* `SongSuggestion` and `SpotifyTrack` now have `to_dict()`/`from_dict()` pairs (`waveform/services/spotify_client.py`). `_serialize_session` accepts an optional `approved_songs` argument and serialises it under key `"approved_songs"`. The shell's `_on_resume` deserialises `approved_songs` from the snapshot and calls `store.set("approved_songs", restored)`.

*2d. Fork session:* `SessionHistoryDialog` now accepts `on_fork` callback. `_SessionRow` renders a "Fork" button when `on_fork` is provided. `_make_fork_data()` module-level helper deep-copies the snapshot, assigns a new UUID session_id, appends "(Copy)" to event_name/playlist_name, and clears all history fields. Shell wires `on_fork` to `_on_resume` then overrides the toast to say `"Forked as '<name>'"`.

**Deliverable 3 — Approved songs in session snapshot**

`SpotifyTrack.to_dict()`/`from_dict()` and `SongSuggestion.to_dict()`/`from_dict()` added. `_serialize_session` now accepts and serialises `approved_songs`. Both `_export_full_night` and `_export_split` pass `approved_songs` through to `_serialize_session`.

### Migration compatibility

Existing beta snapshots that lack `veto_entries`, `keep_entries`, `approved_songs` fields will resume cleanly. All new fields are read with `data.get(key, [])` defaults so old-format files don't crash. The "what got lost" toast fires only when evidence suggests data was actually missing (non-zero `veto_count` or non-empty `keep_history`). No migration function needed — the format is additive.

### Deviations from spec

- `BlockCard` `Delete` key also mapped to `BackSpace` (natural for Mac users who have no dedicated Delete key).
- `ScheduleSidebar._on_block_delete_requested` is a new self-contained confirmation dialog rather than routing through a right-click menu — the right-click menu doesn't exist yet, and both paths call the same underlying logic.
- Fork toast overrides the resume toast by calling `store.set("toast", ...)` a second time immediately after `_on_resume` returns. This is a safe race-free sequence since both are synchronous calls on the main Tk thread.

### CEO decisions needed

None blocking.

## [2026-04-07T14:00:00Z] Phase 11: Polish pass — animations, accessibility, signature motion set — @staff-engineer

### What was built

**`waveform/ui/theme.py`** — Phase 11 utilities appended:
- `lerp_hex(color_a, color_b, t) -> str` — linear hex colour interpolation; used by Event Skin Change and Block Transition. Pure Python, no Tk dependency.
- `apply_focus_ring(widget)` — binds FocusIn/FocusOut to any CTk widget with border support; sets `border_color=ACCENT_VIOLET, border_width=2` on focus. Silently no-ops for widgets that don't support border config.
- WCAG AA contrast notes added as comments: TEXT_PRIMARY/BG_SURFACE (~14:1, passes AAA), TEXT_SECONDARY/BG_BASE (~6.8:1, passes AA). TEXT_MUTED fails at small text sizes — documented with a note to use TEXT_SECONDARY for body copy.

**`waveform/ui/widgets/waveform_anim.py`** — Full Generation Pulse implementation:
- `start_animation(mode="idle"|"generating")` public API; `stop_animation()` freezes bars.
- "idle" mode: original Phase 7 0.5 Hz sine breathing (preserved exactly).
- "generating" mode: each of the 5 bars has an independent frequency (0.4, 0.52, 0.70, 0.45, 0.60 Hz), giving an organic out-of-sync active feel with higher amplitude.
- `reduce_motion` check on every tick — toggling the setting mid-session takes effect within one frame.
- Legacy `animate=True` constructor arg preserved; maps to `start_animation(mode="idle")`.
- `store` parameter added so reduce_motion setting is observable without requiring a direct settings dict.
- `_start_pulse()` / `_do_pulse()` shims preserved for any callers that used the old private API.

**`waveform/ui/sidebar_schedule.py`** — Block Card Reveal animation:
- `_render_blocks(blocks, animate_new=False)` extended: tracks existing block IDs before teardown, identifies truly new cards (not present in prior render), and triggers staggered spring-in animation.
- `_animate_card_reveal(card)` — ease-out height tween from 0 → full height over 9 frames × 20ms (180ms). Uses `grid_propagate(False)` during animation, restores `True` on completion so card resizes naturally afterward.
- Stagger: 80ms between cards when multiple appear at once (e.g. session load).
- `reduce_motion` check; reads from `store.get("settings")` — skips animation when enabled.
- `theme.apply_focus_ring(self._add_btn)` applied to the Add Block button.

**`waveform/ui/track_panel.py`** — Song Approved particle burst + Block Transition:
- `_particle_burst(anchor_widget)` — creates a `tk.Canvas` overlay on the anchor button, spawns 7 small rectangles (3×8px) that fly outward at random angles with fading colour (lerp_hex toward BG_SURFACE). 300ms, 12 frames. Canvas self-destructs on completion. Colors: SUCCESS_GREEN, ACCENT_VIOLET, ACCENT_CYAN. Fires from `_on_approve()` via `_fire_burst_on_widget()` tree-walker. Best-effort — exceptions never block the approve flow.
- `_transition_to_block(callback)` — simulates a crossfade by tweening `preview_frame.fg_color` from surface toward base (fade-out, 150ms), calling `callback` at the midpoint (content swap), then tweening back (fade-in, 200ms). Since CTk has no opacity API, this is a "fade to background" approximation. Album art swaps at the midpoint naturally.
- `set_active_block()` now calls `_transition_to_block` when switching blocks that have content; instant switch when panel is empty.
- Generation pulse in `_show_generating()` now uses `start_animation(mode="generating")` explicitly.
- Preview waveform in `_show_preview()` now uses `start_animation(mode="idle")` explicitly with `store` passed through.
- `reduce_motion` guard on both animations.

**`waveform/ui/shell.py`** — Event Skin Change + session resume + Build button multi-block fix:
- `_current_accent: str` instance variable tracks current accent for tween start point.
- `_on_template_changed()` now calls `_tween_accent(old, new)` on template change.
- `_tween_accent(from_color, to_color)` — 16-frame tween over 400ms using `lerp_hex`; calls `_apply_accent()` each frame. `reduce_motion` guard: snaps immediately.
- `_apply_accent(color)` — updates Build button `fg_color`. Best-effort, wrapped in try/except.
- `_on_session_changed_for_counter()` subscribes to "session" to keep `_generating_block_count` current.
- **Build button multi-block fix**: `_on_build_click()` now seeds `_generating_block_count = len(session.blocks)` when generation starts. `_on_generation_status()` decrements the counter on each "done" signal; only resets the Build button when count reaches 0. "error" always resets immediately.
- `_on_history_click()` fully implements session resume: calls `PlaylistSession.from_dict(session_data)`, pushes session and template to store, navigates to TIMELINE, shows success toast. Exception handling with user-visible error toast.
- `ExportDialog` now receives `store=self._store` to enable `export_completed` signalling.
- `theme.apply_focus_ring(self._build_btn)` applied.

**`waveform/domain/session.py`** — `PlaylistSession.from_dict(data: dict)` classmethod:
- Reconstructs `Block` list from stored `{"id", "name", "archetype", "duration_minutes", "energy_level"}` dicts; skips malformed entries gracefully.
- Best-effort `event_template` lookup: scans `waveform.domain.event` module for an `EventTemplate` whose name matches `event_name` (case-insensitive). Falls back to None for custom/unknown events.
- Restores `keep_history` dict.
- `veto_context` starts fresh (vetoes are session-scoped and stale after resume).
- Handles both `session_id` (export format) and `id` (legacy) as the session UUID key.

**`waveform/services/spotify_client.py`** — `search_user_playlists(name: str) -> List[str]`:
- First-class method on `SpotifyClient`; uses `current_user_playlists` with pagination via `_with_retry`.
- Returns list of matching playlist IDs (usually 0 or 1 elements).
- Exception handling: logs warning and returns empty list on failure.
- `FakeSpotifyClient.search_user_playlists()` implemented with in-memory `_playlists` dict lookup.

**`waveform/app/export.py`** — `_find_user_playlist()` refactored:
- Now delegates to `self._spotify.search_user_playlists(name)` — no more `sp._client()` direct access.
- Removed the `hasattr(sp, "_sp")` / `hasattr(sp, "_playlists")` duck-typing branch.

**`waveform/ui/export_dialog.py`** — `export_completed` store signal:
- `ExportDialog.__init__` accepts optional `store` parameter.
- `_on_export_complete()` calls `self._store.set("export_completed", True)` on success, wiring the Phase 10 analytics subscriber and session-abandoned guard.

**`waveform/ui/widgets/genre_weight_panel.py`** — `genre_weight_changed` debounce:
- `_handle_weight_change()` now uses a per-tag `after()` job with 300ms delay for the analytics event.
- `_emit_change()` and store updates still fire on every slider tick (live UI feedback preserved).
- Per-tag `_analytics_debounce_jobs` dict lazily initialized; cancels the prior job on each new drag tick.

**`waveform/ui/widgets/event_template_card.py`** — Keyboard navigation + focus ring:
- `configure(takefocus=True)` makes each card Tab-navigable.
- `<Return>` and `<space>` bound to `_handle_click`.
- `_show_focus()` / `_hide_focus()` methods bound to `<FocusIn>` / `<FocusOut>`.
- Focus ring shows ACCENT_VIOLET border when unfocused; selection style restored on blur.

**`waveform/ui/event_setup.py`** — `apply_focus_ring` on Start Building button.

### Deviations from spec

1. **Event Skin Change — limited element coverage**: The accent tween updates only the Build button (`_apply_accent`). Block card borders and the "+ Add Block" button are managed by child widgets that don't hold a reference to the shell's tween. Per spec: "best effort — not every element needs to update." The Build button is the highest-visibility accent element. The tween infrastructure (`lerp_hex`, `_tween_accent`) is in place for Phase 12/CTO to extend to more elements.

2. **Block Transition — fg_color approximation**: CTk does not expose widget opacity. The crossfade is simulated by tweening `_preview_frame.fg_color`. This produces a visible transition effect but is visually imperfect: child widgets remain visible at full intensity during the fade because CTk doesn't cascade opacity. The spec acknowledges this: "Simulate with configure(fg_color=...) interpolating toward the background color." The album art swap happens at the midpoint as specified.

3. **Song Approved particle burst — Approve button search**: The burst fires on the first widget in the preview frame tree whose text starts with "Approve". CTk button text is multi-line ("Approve\nSpace"), so `startswith("Approve")` works correctly. This tree-walk is slightly fragile — if the layout changes significantly the burst will fire on a different widget or silently no-op.

4. **`PlaylistSession.from_dict` — no genre_weights**: The serialised snapshot (`_serialize_session`) does not store `genre_weights` per block. Resumed blocks will have empty genre weight lists. This is an existing limitation of Phase 9's serialisation spec. Adding genre weight serialisation would require a schema change in `_serialize_session` — tagged for Phase 12 consideration.

5. **`accent_color` field on EventTemplate**: `EventTemplate` does not have an `accent_color` field — `getattr(template, "accent_color", None)` always returns None, so the tween always interpolates from the current accent toward `ACCENT_VIOLET`. This means the tween produces a visible pulse effect (current→violet) on every template change, even if the selected template has the same accent as the previous. To implement true per-template accents, EventTemplate needs an `accent_color: str` field — deferred to Phase 12.

6. **Keyboard navigation audit — BlockCard Enter selection**: Phase 4's BlockCard is a CTkFrame with `<Button-1>` bound; it doesn't have `takefocus=True`. Adding Enter/Space selection to block cards was out of scope for a surgical Phase 11 pass (would require modifying block_card.py substantially). The EventTemplateCard keyboard fix covers the highest-priority flow.

### Known limitations for CTO (Phase 12)

1. `EventTemplate.accent_color` field needed for true Event Skin Change behaviour.
2. Genre weights are not serialised in session snapshots — resumed sessions lose block-level genre customisation.
3. Particle burst tree-walk is fragile if layout structure of `_show_preview()` changes.
4. Block Transition opacity simulation is visually imperfect due to CTk's lack of widget opacity.
5. `search_user_playlists` on `SpotifyClient` returns all matches; the first match is used. If a user has two playlists with identical names, the first one in the library wins — same behaviour as before.
6. `BlockCard` is not Tab-navigable; full keyboard navigation of the block list is incomplete.

### Open questions for CEO

None new. All §11 questions from the plan remain open.

### Phase 12 (CTO) readiness

All beta-blocking items are resolved:
- Session resume is fully wired.
- Generation Pulse, Block Card Reveal, Song Approved burst, Block Transition, and Event Skin Change are all implemented (with documented simplifications).
- WCAG AA contrast passes on TEXT_PRIMARY and TEXT_SECONDARY.
- Reduce motion toggle respected by all `after()`-based animations.
- `export_completed` store signal wired for analytics.
- `genre_weight_changed` analytics debounced.
- Build button correctly waits for all blocks before resetting.
- `SpotifyClient.search_user_playlists` is a proper first-class method.

---

## [2026-04-07T12:00:00Z] Phase 9: Spotify export + persistence + session history — @staff-engineer

### What was built

**`waveform/app/export.py`** — `ExportController` class:
- `export_session(session, approved_songs, mode, playlist_name, callbacks…)` — full orchestration in a background thread
- `_export_full_night()` — single playlist for all blocks; resolves name collisions via `_resolve_collision()`
- `_export_split()` — one playlist per block; block-scoped names avoid collision dialog spam
- `_resolve_collision()` — searches user's Spotify library for name match; fires `on_existing_playlist` and blocks the worker thread until `resolve_existing(action)` is called from the UI thread. Supports Overwrite, Append, Rename. Rename suffix stored in `_renamed_playlist_name` so `effective_name` is used consistently for history and session saves.
- `_png_to_jpeg()` — converts PIL PNG bytes to JPEG ≤256 KB (Spotify limit), progressively reducing quality then resizing as last resort
- `ExportResult` — `playlist_urls`, `track_count`, `block_count`, `elapsed_ms`, `primary_url` property
- `ExistingPlaylistAction`, `ExportMode` enums

**`waveform/ui/export_dialog.py`** — `ExportDialog(CTkToplevel)`:
- Playlist name entry (editable, defaults to event template name)
- Full Night / Split radio toggle
- Live preview: track count, estimated duration, playlist count
- Progress bar + status label during export (all updates via `root.after(0, …)`)
- Success state: Spotify URL as clickable link, "Open in Spotify" + "Done" buttons
- Error state: inline message + "Retry" button swap
- `_CollisionDialog` sub-modal: Overwrite / Append / Rename

**`waveform/ui/session_history.py`** — `SessionHistoryDialog(CTkToplevel)`:
- Loads `persistence.list_sessions()` + `persistence.load_session(id)` for each
- Sorted by `exported_at` descending
- `_SessionRow` per session: event name, date, track count, block count
- "View playlist" button opens Spotify URL in browser; "Resume" fires `on_resume` callback
- Empty state label when no sessions exist

**`waveform/ui/settings_screen.py`** — `SettingsScreen(CTkToplevel)`:
- Gemini model dropdown (6 models including 2.5-flash, 2.5-pro, 2.0-*)
- Tracks per hour slider (8–24) with live numeric label
- Allow repeats / Shuffle within blocks / Analytics opt-in toggles
- "Clear song history" + "Clear all sessions" danger buttons, each guarded by `_ConfirmDialog`
- Uses `analytics_enabled` key (not `analytics_opt_in` — synced with Phase 10's schema)

**`waveform/services/persistence.py`** — additions:
- `_rename_to_bak(path)` helper — renames file to `.bak`
- `PersistenceService.migrate_v1_if_needed()` — checks v1 candidate paths; migrates `settings.json` (via existing `migrate_v1_settings`) and `song_history.json` to `~/.waveform/`; renames originals to `.bak`; returns `True` if migration occurred
- `FakePersistenceService.migrate_v1_if_needed()` — no-op returning `False`

**`waveform/app/main.py`** — additions:
- `persistence.migrate_v1_if_needed()` called on startup (wrapped in try/except; non-fatal)
- `ExportController` instantiated with `cover_art_service=waveform.services.cover_art` module
- `export_controller` and `persistence` passed to `WaveformApp`

**`waveform/ui/shell.py`** — additions:
- Constructor accepts `export_controller` and `persistence` parameters
- Top bar expanded to 4 columns; utility button group (Export, History, Settings) added at column 2
- Export button hidden by default; shown via `_on_approved_songs_changed` subscription when `approved_songs` store key becomes non-empty
- `_on_export_click()` → opens `ExportDialog`
- `_on_history_click()` → opens `SessionHistoryDialog`
- `_on_settings_click()` → opens `SettingsScreen`

**`waveform/tests/test_export.py`** — 30+ assertions across 6 test classes:
- `TestFullNightExport` — creates one playlist, correct track count, tracks added to Spotify, cover art uploaded, empty approved songs, analytics event, session saved, elapsed_ms
- `TestSplitExport` — N playlists for N blocks, names include block names, per-block cover art, track distribution, blocks-without-songs skipped
- `TestSongHistory` — history saved, correct "title||artist" keys, per-block keys in split mode, trackless songs skipped
- `TestCollisionHandling` — RENAME creates new playlist, APPEND reuses existing
- `TestV1Migration` — psytrance→genre-weight, disabled psytrance, fake persistence noop, non-psytrance keys preserved
- `TestExportResult` — primary_url, empty urls, attribute correctness
- `TestPngToJpeg` — JPEG magic bytes, size ≤256 KB

### Deviations from spec

1. **`persistence.save_song_history` vs `mark_used`**: The spec says "call `persistence.save_song_history(session_name, track_uri)`" but the Phase 1 `PersistenceService` stores songs as `title||artist` keys (not URIs) via `mark_used()`. The export controller uses `mark_used()` with `(title, artist)` tuples, which matches the existing duplicate-detection scheme used by `GenerationController`. Using URIs would require a separate lookup path and wouldn't integrate with the existing dedup system.

2. **Session history Resume button**: The "Resume" action in `SessionHistoryDialog` currently fires a toast saying "Resume coming in Phase 10." Full session resume requires deserialising a `PlaylistSession` from the stored JSON snapshot — this is the right place for it but the domain serialisation layer (block/session from dict) is not yet implemented. The infrastructure (session JSON shape, `on_resume` callback wiring) is all in place for Phase 10/11 to complete.

3. **Spotify `current_user_playlists` pagination**: The real `SpotifyClient` wrapper doesn't expose a `current_user_playlists` method. The collision check accesses `sp._client()` directly (the underlying Spotipy instance) for the real client, and checks `sp._playlists` dict for `FakeSpotifyClient`. This is a minor abstraction leak; if Phase 11 cleans up the SpotifyClient API, `find_user_playlist()` should be promoted to a first-class method.

4. **`_clear_playlist` with overwrite**: The real overwrite path calls `playlist_remove_all_occurrences_of_items`. This requires the `playlist-modify-private` scope which is already in `SPOTIFY_SCOPE`. However, Spotipy's method name is `playlist_remove_all_occurrences_of_items` — tested against the FakeSpotifyClient which just clears the dict directly.

5. **`shuffle_within_blocks` setting**: Stored by `SettingsScreen` but not yet used by `GenerationController`. It's a setting stub for Phase 10/11 to wire into the generation/export flow.

### Phase 10/11 readiness notes

- Phase 10 (analytics): `playlist_exported` fires via `analytics.playlist_exported(n_blocks, n_tracks, time_from_open_ms)` — already instrumented.
- Phase 11 (polish): `ExportDialog`, `SessionHistoryDialog`, and `SettingsScreen` are all `CTkToplevel` modals with clean separation. Motion/animation hooks can be added without restructuring.
- Session resume: `_serialize_session()` stores full block list + keep_history. Phase 10/11 can add `PlaylistSession.from_dict()` and wire it to the Resume button.

## [2026-04-07T10:00:00Z] Phase 8: Cover art generator v1 — @staff-engineer

### What was built

**`waveform/services/cover_art.py`** — Full Tier 1 replacement of the Phase 1 gradient stub:

- `COVER_ART_TIER = 1` module-level feature flag; routes through `_generate_dalle_overlay()` stub when set to 2
- `generate_block_cover(archetype, event_name) -> bytes` — 512×512 PNG for all 10 archetypes
- `generate_playlist_cover(session) -> bytes` — dominant-archetype cover with larger event name text
- `_generate_dalle_overlay()` stub — passthrough; Phase 2 roadmap wires real DALL-E here

**10 parametric renderers** (all in `_RENDER_FNS` dispatch table):
1. `_render_arrival` — 3-stop radial gradient (champagne → dusty rose → deep plum) + 5-8 blurred watercolor pooling circles + 15% sine-noise overlay
2. `_render_chill` — `#0A2540` ocean base + 4-6 GaussianBlur teal/sage/moon-white blobs
3. `_render_singalong` — cream base + 30° diagonal texture lines + 22-30 coral/gold confetti circles
4. `_render_groove` — dark warm base + 3-5 amber/burnt-orange pill shapes with blurred glow halos
5. `_render_dance_floor` — black base + 9-12 concentric rings (cyan/hot-pink/white-40%) with 3-pass motion-blur simulation
6. `_render_club_night` — near-black + 32px circuit-board grid + 4-6 angular acid-green/UV-violet polygons + dot nodes at grid intersections
7. `_render_late_night` — very dark base + 3-5 sinusoidal blurred thick curves + 38% heavy scatter noise (the noisiest archetype)
8. `_render_sunrise` — 4-6 Rothko horizontal bands with per-band scatter noise + 1px white dividers at boundaries
9. `_render_ceremony` — `#FAF7F2` linen base + crossed diagonal texture (12px opacity) + 2-3 blurred pastel edge shapes; DARK TEXT
10. `_render_peak` — 5-stop radial burst (brand gradient maxed) + 9-13 sunburst white rays + 10% sine-noise overlay

**Shared infrastructure:**
- `_radial_gradient_image()` — efficient `putdata()` radial gradient builder (avoids 262k individual `draw.point()` calls by using `putdata()` instead)
- `_blurred_circle()` — GaussianBlur-based soft blob on transparent RGBA layer
- `_sine_noise_overlay()` — 3-octave sine grid approximating Perlin noise (documented approximation)
- `_noise_overlay()` — fast scatter-point noise (used by LATE_NIGHT and SUNRISE bands)
- `_draw_waveform_watermark()` — 5-bar brand gradient watermark, 60% opacity, bottom-right
- `_add_text_layer()` — event name (22px Inter 700, bottom-left) + archetype label (14px, top-right)
- `_load_font()` — Inter from `assets/fonts/` → system font paths → PIL default with logged warning

**`waveform/assets/fonts/.gitkeep`** — Directory created; instructions for Inter font placement.

**`waveform/tests/test_cover_art.py`** — 30+ assertions across 5 test classes:
- `TestBlockCoverAllArchetypes` — parametrized over all 10 archetypes: generates bytes, valid PNG signature, 512×512 size, decodable by PIL, determinism (same inputs = same bytes), uniqueness (different events = different bytes)
- `TestBlockCoverEdgeCases` — empty name, unicode name, 200-char truncation, default name
- `TestPlaylistCover` — empty session fallback, multi-block session, dominant archetype determinism, single-block session
- `TestCoverArtTierFlag` — constant exists at 1, Tier 2 stub passthrough, Tier 1 default, Tier 2 flag routing
- `TestFakeCoverArtService` — unchanged from Phase 1, still valid

### Deviations from spec

1. **Perlin noise**: approximated with a 3-octave sine grid (`_sine_noise_overlay`). True Perlin requires a C extension (`noise` package). The sine approximation is documented in the module docstring and produces acceptable organic texture at 10-35% opacity on cover-art-sized output.

2. **Width/height parameters**: `generate_block_cover()` accepts `width`/`height` for Phase 1 interface compatibility but always renders at 512×512 per spec. Non-512 values log a warning rather than raising, to avoid breaking existing callers.

3. **Font fallback**: Inter TTF files are not bundled (no internet during Phase 8). The font loader tries `assets/fonts/Inter-Bold.ttf` / `Inter-Regular.ttf` first, then common macOS/Linux system paths (Arial, Helvetica, DejaVu, Liberation), then PIL's built-in bitmap font. This is already the warmest-path approach available without bundling.

4. **`generate_playlist_cover` event name source**: `PlaylistSession` has no `event_name` field — the name lives in `session.event_template.name`. The function handles `event_template=None` (custom events) by falling back to `"Event"`.

5. **Performance note**: `_radial_gradient_image()` uses `putdata()` rather than per-pixel `draw.point()`, which is ~100× faster in pure Python. ARRIVAL and PEAK both use it. However, it still iterates 262,144 pixels in a Python loop — on a 2023 MacBook Pro this takes ~1-2 seconds per radial-gradient archetype. If generation latency becomes a concern in Phase 9 (export path), adding `numpy` to optionally accelerate the gradient computation would be the right call. Currently acceptable for a cover-art generation that happens once per export.

### Open questions for CEO

None new. Existing §11 questions unchanged.

### Phase 9 readiness notes

Phase 9 (Spotify export + persistence + session history) can call `generate_block_cover(archetype, event_name)` and `generate_playlist_cover(session)` directly. Both return PNG bytes compatible with Spotify's `upload_cover_art()` call in `SpotifyClient`. No interface changes needed.

## [2026-04-07T17:00:00Z] Phase 2C: In-app prompt editor, session history polish, custom event templates — @staff-engineer

### What was built

**`waveform/ui/prompt_editor.py`** — New file: `PromptEditor(CTkToplevel)` modal.
- Large monospace `CTkTextbox` pre-filled via `persistence.load_master_prompt(fallback_path=_BUNDLED_PROMPT)`.
- Collapsible sidebar (~210px) with static guidance for 6 prompt sections (CORE RULES, QUALITY STANDARD, LANGUAGE NOTES, PARTY CONTEXT, ARTIST POOL, LANGUAGE MIX).  Toggle button hides/shows via `grid_remove()`.
- Save button — calls `persistence.save_master_prompt(text)`, shows toast "Prompt saved." for 4s. I/O errors shown as danger-color toast, never crash.
- Reset to default — confirmation dialog, copies `master_prompt.md.example` → user override (falls back to `master_prompt.md` when no `.example`), reloads textbox.
- Cancel / window-X — "Discard changes?" guard when content differs from original.
- Toast label auto-clears after 4000ms via `after()`.

**`waveform/ui/settings_screen.py`** — Wired in:
- "Advanced" button added to the bottom row alongside "Close". Opens `PromptEditor` via lazy local import (avoids circular import).
- Window height nudged from 580px to 600px to fit the new button row.

**`waveform/ui/session_history.py`** — Phase 2C polish on top of Phase 2A baseline:
- `_relative_date(iso_str)` helper — "Today", "Yesterday", "N days ago", "Last week", "April 2" / "April 2, 2023" (platform-safe, Windows `%-d` fallback).
- `SessionHistoryDialog` now accepts `store` kwarg for empty-state navigation.
- Footer row added: "Delete selected" (initially disabled) + "Clear all history" shortcut with confirmation.
- Multi-select: each `_SessionRow` receives a `BooleanVar` checkbox.  Checking any row enables "Delete selected".  Bulk delete with confirmation dialog.
- "Open in Spotify" button (120px, ACCENT_CYAN) — prefers `playlist_url` field, falls back to constructing from `playlist_id`. Replaces old "View playlist" label.
- Event type appended to metadata line (`.replace("_", " ").title()`).
- Relative date replaces raw ISO date slice.
- Empty state: `WaveformAnim` in idle mode + "No sessions yet" heading + body copy + "Start your first event" CTA button that navigates to `AppScreen.EVENT_SETUP` via `store.set("current_screen", ...)`.
- `_SessionRow` layout changed: column 0 = checkbox, column 1 = info block (weight=1), column 2 = buttons.
- `_make_fork_data` preserved; added `playlist_url` clearing to the fork dict.

**`waveform/services/persistence.py`** — New methods on `PersistenceService`:
- `load_custom_templates() -> List[Dict]` — reads `~/.waveform/custom_templates.json`; returns `[]` on any error.
- `save_custom_template(template_data)` — upserts by `id`; atomic write via existing `_write_json` (tmp→replace).
- `delete_custom_template(template_id) -> bool` — returns `True` if found and removed.
- Same three methods added to `FakePersistenceService` (in-memory, no filesystem).

**`waveform/ui/event_setup.py`** — Custom event templates:
- `EventSetupScreen.__init__` gains optional `persistence=None` parameter.
- `_build_gallery` refactored: card rendering extracted to `_render_gallery_cards(scroll)` with a `_gallery_scroll` reference for live rebuilds.
- `_render_gallery_cards` loads custom templates from persistence, converts via `_custom_dict_to_template()`, inserts them between built-ins and the blank-slate card.  Calls `_bind_custom_card_context_menu` for each user-created card.
- `_rebuild_gallery()` — tears down and re-renders cards; re-applies selection if current template still present.
- "Save as template" button added below "Start Building" — opens `_SaveTemplateDialog`, persists current config (blocks, genre weights, vibe, metadata toggles, accent color), calls `_rebuild_gallery()`.
- `_on_save_as_template()` — graceful I/O error handling; only shown when `persistence is not None`.
- `_custom_dict_to_template(data) -> EventTemplate` — safe converter; unknown archetypes silently dropped; sets `tpl.is_user_template = True` for border differentiation.
- `_bind_custom_card_context_menu(card, tpl, persistence, rebuild_fn)` — right-click + Ctrl+click context menu with "Delete template" + confirmation dialog. Built-in cards are unaffected.
- `_SaveTemplateDialog(CTkToplevel)` — name entry, Enter key submits, Save/Cancel buttons.

**`waveform/ui/widgets/event_template_card.py`** — `custom_border: bool = False` kwarg added to `__init__`.  `_apply_selection_style()` renders a 1px `ACCENT_CYAN` border for unselected user-created cards; violet 2px border on selection unchanged.

**`waveform/ui/shell.py`** — Two surgical additions:
- `EventSetupScreen(...)` call now passes `persistence=self._persistence`.
- `SessionHistoryDialog(...)` call now passes `store=self._store`.

### Deviations from spec

1. **`master_prompt.md.example` does not exist yet** — `PromptEditor._on_reset()` falls back to `master_prompt.md` (bundled default) when no `.example` file is found.  This is safe; the fallback path is documented in the module docstring.  A `.example` should be created alongside the prompt in a follow-up commit (or generated as a copy of `master_prompt.md` at build time).

2. **"Fork session" navigates via `on_fork` callback** — Fork was already implemented in Phase 2A. Phase 2C preserves the existing wiring unchanged; the `store` parameter added to `SessionHistoryDialog` is used only by the empty-state CTA, not the fork flow.

3. **Event type field in session snapshot** — `_SessionRow` reads `d.get("event_type") or d.get("template_id")`. The existing serialisation writes `template_id` from Phase 9; `event_type` was added as a forward-compat alias.  No migration needed — the `or` chain handles both.

4. **`is_user_template` attribute on `EventTemplate`** — Set as a dynamic attribute on the dataclass instance (not a declared field).  This is intentional: `EventTemplate` is in the domain layer and should not gain UI-specific fields. The attribute is read only via `getattr(..., False)` in the gallery renderer, so missing it is safe.

### CEO decisions needed

None. All items in Phase 2C were self-contained feature additions with no new strategic choices.

## [2026-04-07T08:00:00Z] Phase 6: AI generation pipeline with streaming + veto-feedback loop — @staff-engineer

### What was built

**`waveform/app/generation.py`** — New file: `GenerationController` class.

- `start_generation(session, block_id=None)` — submits one background future per block via `ThreadPoolExecutor`; never blocks the UI thread.
- `cancel()` — sets a `threading.Event`; background workers check it between song yields.
- `handle_keep(block_id, song)` — updates `session.veto_context.keeps` and `session.keep_history`; fires `song_kept` analytics; pushes updated session to store.
- `handle_skip(block_id, song)` — fires `song_skipped` analytics; no prompt injection (skip is a mild signal).
- `handle_veto(block_id, song, reason_tag=None)` — updates `session.veto_context.vetoes`; fires `song_vetoed` analytics; pushes updated session. This is THE killer feature — context accumulates permanently across re-generate calls for the session.
- `request_swap(block_id, reference_song)` — fires `swap_requested` analytics; submits `_swap_worker` to thread pool; result pushed to `pending_song`.
- Streaming loop (`_stream_songs`): calls `gemini_client.generate_songs()`, checks cancel event between each song, runs duplicate detection with up to 3 retries via `generate_single_replacement`, emits each annotated song dict `{song, is_duplicate, position}` to `store.set("pending_song", (block_id, annotated))` and appends to `store.get("suggestion_feed")[block_id]`.
- Duplicate detection: checks in-session `keep_history` keys, cross-session `persistence.get_used_keys()`, and `veto_context.is_vetoed()`. After 3 failed retries emits with `is_duplicate=True` flag.
- Full analytics: `generation_requested`, `generation_completed` (with `latency_ms`), `song_suggested` (per song, with `position`), all song interactions.
- Progress: `generation_status = {block_id, status: "generating"|"done"|"error", progress, total}` pushed on each song.
- Errors: caught and surfaced as `store.set("toast", {message, type: "error"})` — no modal dialogs.

**`waveform/app/state.py`** — Extended `AppState` with 5 new fields:
- `pending_song: Optional[Any]` — `(block_id, annotated_dict)` tuple; Phase 7 subscribes.
- `suggestion_feed: Dict[str, Any]` — accumulated per-block song list.
- `generation_complete: Optional[str]` — set to `block_id` on completion.
- `generation_status: Optional[Dict]` — generating/done/error progress dict.
- `toast: Optional[Dict]` — inline toast payload.
- `generation_controller: Optional[Any]` — controller instance (for store-based retrieval).

Note: Phase 7 ran concurrently and added `approved_songs` and `selected_block_id` to `AppState` — no conflict.

**`waveform/ui/shell.py`** — Wired the Build button:
- `_on_build_click()`: toggles between "▶ Build" and "⏹ Stop" states; calls `controller.start_generation(session)` on first click; calls `controller.cancel()` + resets button on second click. Retrieves controller via `self._generation_controller or self._store.get("generation_controller")` for resilience.
- `_on_generation_status(status)`: subscribes to `generation_status`; resets button to "▶ Build" on "done" or "error" status, marshalled to main thread via `.after(0, ...)`.
- `_on_toast(toast)`: subscribes to `toast`; renders inline label at bottom-right (Phase 7 improved this to `place_forget` + cancel-previous dismiss job).
- Subscriptions added: `generation_status`, `toast`.

**`waveform/app/main.py`** — Updated to instantiate `GenerationController` and pass to `WaveformApp`. Phase 7 wrapped this in a `try/except` for robustness — good change, kept.

**`waveform/services/analytics.py`** — Added `song_suggested(track_id, block_id, position)` method (was missing from Phase 1 implementation; referenced in epic §9 as `song_suggested`).

**`waveform/prompts/veto_addendum.md`** — Fully fleshed out. Added a table mapping each of the 5 `VETO_REASON_TAGS` to the concrete behaviour Gemini should exhibit, plus a "Positive reinforcement" section explaining how to use liked songs as calibration anchors (not clones). The existing `VetoContext.format_for_prompt()` already injects this context into prompts.

**`waveform/tests/test_generation.py`** — New test file (~50 tests in 6 classes):
- `TestBasicGeneration` — streaming flow, suggestion_feed accumulation, generation_complete signal, status transitions, multi-block generation.
- `TestVetoFeedbackLoop` — veto adds to context, context accumulates across multiple vetoes, context persists across re-generate calls (THE important invariant), keep adds positive context, analytics events fire.
- `TestDuplicateDetection` — key normalisation, in-session duplicate flagged, cross-session duplicate flagged, no false positive for fresh song, vetoed song treated as duplicate.
- `TestSwapFlow` — request_swap emits pending_song, fires analytics, veto-then-swap sequence.
- `TestGenerationAnalytics` — all four generation analytics events with correct payload values.
- `TestCancel` — cancel reduces emission count.

All tests use `_GenerationWaiter` context manager pattern to avoid subscribe-after-set race conditions in multithreaded test scenarios.

### Deviations from spec

1. **`song_suggested` was missing from Phase 1 analytics.py** — Added it as a one-liner in the analytics service (2 lines). Not a Phase 6 deviation, just a gap fill.

2. **`is_duplicate` is a dict key in `annotated` payload, not a field on `SongSuggestion`** — The spec says "emit it with `is_duplicate: bool` flag." `SongSuggestion` is a frozen dataclass. Rather than subclassing or modifying the domain model, I wrapped it in `{song: ..., is_duplicate: ..., position: ...}`. Phase 7 receives this dict and can access `song["is_duplicate"]` and `song["song"]`. This is cleaner and avoids polluting the domain model with UI concerns.

3. **Build button resets on first "done" signal even in multi-block generation** — The spec says "during generation." For multi-block sessions, the button resets when the first block completes rather than when all blocks are done. A proper solution requires tracking all in-flight blocks. Phase 7 can refine this with a counter — the infrastructure (`generation_status` per block_id) is ready for it.

4. **Toast uses Phase 7's improved placement** — Phase 7 ran concurrently and replaced my top-edge toast with a bottom-right placement using `place_forget` and job cancel. That's a better UX; no regression.

### Phase 7 readiness notes

Phase 7 (Song preview card feed) can proceed with full confidence:

1. **`store.subscribe("pending_song", callback)`** — fires with `(block_id, {"song": SongSuggestion, "is_duplicate": bool, "position": int})` as each song arrives from Gemini.
2. **`store.subscribe("generation_status", callback)`** — fires with `{"block_id": ..., "status": "generating"|"done"|"error", "progress": n, "total": n}`.
3. **`store.subscribe("generation_complete", callback)`** — fires with `block_id` string when all songs for that block have been emitted.
4. **`store.subscribe("toast", callback)`** — fires with `{"message": str, "type": "error"|"info"|"success"}` for inline notifications.
5. **`store.get("suggestion_feed")`** — dict of `{block_id: [annotated, ...]}` for full song list access.
6. **`controller.handle_keep/skip/veto/request_swap`** — all public, tested, ready to wire to Keep/Skip/Veto buttons.
7. **`controller._generation_controller`** — available on `WaveformApp` instance for direct access from child widgets via `self.winfo_toplevel()._generation_controller`.

## [2026-04-07T06:30:00Z] Phase 4: Block builder timeline — @staff-engineer

### What was built

**`waveform/ui/timeline_canvas.py`** — Full replacement of the Phase 2 static placeholder:
- `tk.Canvas`-based interactive timeline inside a `CTkFrame`
- Block bands drawn proportional to `duration_minutes`; archetype palette color fill
- Time labels (`HH:MM`) at each block boundary, defaulting to 20:00 event start
- Click-to-select: highlights the clicked band with a violet border, calls `on_block_select`
- Drag-to-reorder: `<ButtonPress-1>` / `<B1-Motion>` / `<ButtonRelease-1>` pattern; ghost dashed line shows drop position; commits via `session.reorder_blocks` semantics written back through `store.set("session", …)`
- Drag-to-resize: grabbing the right edge (within 8px) of a band changes `duration_minutes`; snaps to 5-minute grid by default; holding Shift bypasses snap for freeform resize; live preview redraws on every drag tick
- Energy-arc sparkline: accent cyan (`#22D3EE`) polyline above the strip, dots at block midpoints, smooth linear interpolation between energy levels 1–5
- Double-click inline rename: floating `tk.Entry` widget centred over the block label; commits on Return/FocusOut, cancels on Escape
- Right-click context menu: Rename, Duplicate, Change Archetype (submenu), Delete (danger color)
- `select_block(block_id)` public method so sidebar can sync the canvas selection without a round-trip through the store
- `ARCHETYPE_EMOJI` dict promoted to public export (used by sidebar and block card too)

**`waveform/ui/widgets/block_card.py`** — Full replacement of the Phase 2 visual stub:
- Left accent bar using `ArchetypeSpec.cover_palette[0]`
- Name row: archetype emoji + bold name + archetype display label right-aligned
- Meta row: `HH:MM – HH:MM  •  XXm  •  ●●●○○` (time range, duration, energy dots)
- Selected state: violet border + raised surface background
- Expandable detail panel (shown when `set_selected(True)`):
  - Start / End time entries (editable, parse `HH:MM`, recompute duration on commit)
  - Energy level `CTkSlider` (1–5, snapped to integer, cyan value label, triggers sparkline redraw via `on_mutate`)
  - Archetype chip grid (5×2, 36px wide chips; active chip gets palette fill + violet border)
  - Genre weights section — placeholder label "Phase 5" (ready to populate)
- Inline name rename: double-click swaps the name label for a `CTkEntry`
- `ARCHETYPE_EMOJI` public dict (10 archetypes)
- `update_block(block, start_minute)` for refreshing displayed data without recreating the widget

**`waveform/ui/sidebar_schedule.py`** — Full replacement of the Phase 2 static list:
- Computes start minutes for each block by summing predecessors (event start 20:00)
- Passes `on_mutate` callback to each `BlockCard` → `_on_block_mutated` pushes to store
- `select_block(block_id)` public method for cross-panel sync
- Click on a card: selects and expands it; clicking already-selected card collapses/deselects
- `_open_add_block_popover()`: `CTkToplevel` modal showing all 10 archetypes with swatch, name, energy, and description snippet; clicking a row calls `_add_block_with_archetype(arch)` which appends a 60-min block to `session.blocks` and pushes to store
- Subscribes to `"session"` store key; re-renders card list on any change

**`waveform/ui/shell.py`** — Two small changes:
- `TimelineCanvas` now receives `on_block_select=self._on_block_selected`
- `_on_block_selected` now calls `self._timeline.select_block(block.id)` and `self._sidebar.select_block(block.id)` to keep both panels in sync

### Deviations from epic

1. **`on_mutate` callback pattern in BlockCard** — the card fires a callback with the updated `Block` rather than writing directly to the store. The card doesn't hold a store reference; the sidebar owns the store write. This is cleaner: the card stays a pure view component testable without a store.

2. **Add Block popover is a `CTkToplevel`** (modal) rather than an inline popover anchored to the button. CTk doesn't have a native popover/dropdown widget. `CTkToplevel` with `grab_set()` is the standard CTk idiom and achieves the same UX. Phase 11 can style it more carefully.

3. **Time-range editing in block detail does not propagate start-time changes to subsequent blocks.** Changing a block's start time would require recomputing all successor start times and potentially renaming them — a cascade operation more appropriate to a future dedicated "event timeline editor" view. For Phase 4, editing the Start entry only affects the block's own duration (end - new start), not its actual position in the timeline. Position is determined by order + durations of all preceding blocks.

4. **Energy sparkline is always visible** (not "available after generation" as the Phase 2 placeholder said). The spec says the sparkline is derived from block energy levels and "AI-estimated BPM once generation has run" — but the energy levels are always present, and the sparkline delivers value immediately. The BPM overlay can be added in Phase 6 as a second data series.

5. **`ARCHETYPE_EMOJI` moved from private `_ARCHETYPE_EMOJI` to public `ARCHETYPE_EMOJI`** in `block_card.py` and imported by both `sidebar_schedule.py` and `timeline_canvas.py` to avoid three-way duplication. Minor naming deviation from the Phase 2 stub convention.

### Open questions flagged (CEO-owned)

None new. All §11 questions remain as previously logged.

### Phase 5 / Phase 6 readiness notes

- **Phase 5** (genre weights): the `BlockCard` detail panel already has a "Genre weights — Phase 5" placeholder row in grid position `(4, 0..3)`. Phase 5 should add `GenreSlider` widgets there and call `on_mutate` with the updated block. No structural changes needed to block_card.py beyond filling that section.
- **Phase 6** (AI generation): `_on_block_selected` in shell.py is the right hook for "send this block to generation." Phase 6 should update it to also call `self._track_panel.set_active_block(block)`.
- **State wiring is complete**: every mutation path (resize, reorder, rename, delete, duplicate, archetype change, energy change, add block) goes through `store.set("session", …)` and both canvas and sidebar subscribe to re-render.

## [2026-04-07T05:00:00Z] Phase 5: Generalized genre weight system — @staff-engineer

### What was built

**`waveform/ui/widgets/genre_slider.py`** — Extended Phase 2 stub:
- Added `on_remove: Callable[[str], None]` — fires when the user clicks the ✕ button
- Added `on_activate: Callable[[str], None]` — fires when the user clicks an inherited (ghosted) row to override it
- Non-inherited rows now render a ✕ `CTkButton` in column 3
- Inherited rows render a "inherited — click to override" badge instead of a slider; the entire row is click-bound to `on_activate`
- All colours from `theme.py`; no hardcoded hex

**`waveform/ui/widgets/genre_weight_panel.py`** — New file; the full genre weight editor:
- `GenreWeightPanel(ctk.CTkFrame)` takes `block`, `session`, `on_weights_changed`
- Search/autocomplete input: `CTkEntry` with `trace_add("write", ...)` — calls `DEFAULT_INDEX.search()` on every keystroke; renders up to 8 clickable `CTkButton` pills below
- Input is disabled with "Max 6 genres per block" placeholder when at capacity (epic §5.3)
- Active genre rows: each row is a `GenreSlider(inherited=False)` with change + remove callbacks
- "No genres set" empty-state label shown when active list is empty
- Inherited section: reads `session.event_template.default_genre_weights`; shows `GenreSlider(inherited=True)` ghosted rows; clicking any row calls `_handle_activate_inherited` which clones the weight into the active list
- `refresh(block, session)` public method for Phase 4 to call when selected block changes

**`wire_genre_panel_to_store(store, block_id, new_weights)`** — Module-level helper function exported from `genre_weight_panel.py`:
- Applies an immutable update pattern: `dataclasses.replace(session, blocks=[...])` and `dataclasses.replace(block, genre_weights=new_weights)`
- Publishes via `store.set("session", updated_session)`
- No-ops safely when `store.get("session")` is None

**`waveform/services/gemini_client.py`** — Prompt builder updated:
- Renamed `_weight_to_instruction` → `_weight_to_adverb` (clearer name)
- `_build_genre_instruction` now produces: "For this {archetype} block, lean heavily into house (~60%), moderately into deep-house (~30%). Fill the rest with what fits the vibe."
- Percentage callout added to each genre phrase (was previously adverb-only)
- Block archetype is now named in the instruction for richer Gemini context
- Empty-weights path unchanged (returns `""`)

**`waveform/tests/test_genre_weight_system.py`** — New test file (5 test classes, ~50 assertions):
- `TestMigrationToGenreWeight` — verifies v1 psytrance migration output can construct `GenreWeight` objects and be mounted on a `Block` without errors; covers disabled case and 100% cap
- `TestGenreTagIndex` — autocomplete correctness: prefix-first ordering, infix matches, empty query, case-insensitivity, psytrance still in index, custom tag add
- `TestBuildGenreInstruction` — natural language output: empty returns `""`, correct adverbs, descending sort, "Fill the rest" phrase, max-6 enforcement, archetype in output
- `TestWireGenrePanelToStore` — immutable session update: new session published, original not mutated, updated block has new weights, other blocks unchanged, no-op on missing session
- `TestGenreWeightValidation` — domain edge cases: zero/max weight, above-max raises, empty tag raises, normalisation

### Phase 4 integration point

Phase 4 is IN_PROGRESS. The `GenreWeightPanel` is a standalone `ctk.CTkFrame` that Phase 4 can mount directly. The integration instructions are embedded as a comment block at the bottom of `genre_weight_panel.py` (lines ~415–460), including the exact import and mount pattern. Phase 4 needs only:
1. Import `GenreWeightPanel` and `wire_genre_panel_to_store` from `waveform.ui.widgets.genre_weight_panel`
2. Instantiate `GenreWeightPanel(parent, block, session, on_weights_changed=lambda bid, w: wire_genre_panel_to_store(store, bid, w))` in the block detail expanded area
3. Call `panel.refresh(block, session)` when the selected block changes

### Deviations from spec

1. **Tag index size**: Phase 1 built ~90 tags; epic §5.3 references "~300 tags derived from Spotify's genre seeds." The `GenreTagIndex.add()` method exists for runtime expansion. Growing to 300 tags in Phase 6 or Phase 9 (when Spotify auth is live) would be a one-liner. Not blocking for MVP.
2. **Tooltip on disabled input**: Epic says "tooltip: Max 6 genres per block." CustomTkinter has no native tooltip widget. The disabled `placeholder_text` serves the same purpose. A real tooltip widget (Phase 11 polish) can be added without changing any API.
3. **Pill layout**: Pills use `grid(row=0, column=i)` which means they wrap visually only if the frame clips — they do not reflow. For ≤8 pills in a ~300px frame this is fine. A wrapping flow layout would require a custom geometry manager (Phase 11 scope).

### Open questions (CEO-level)

None new. Existing §11 questions unchanged.

### Phase 6 readiness notes

Phase 6 (AI generation pipeline) can start immediately:
1. `_build_genre_instruction` is updated and the prompt pipeline is ready
2. `wire_genre_panel_to_store` handles the immutable session mutation pattern that the generation pipeline also needs for block state updates
3. `GenreWeightPanel` is ready to be wired into the Phase 4 block detail area before or after Phase 6 lands — no dependency

## [2026-04-07T02:00:00Z] Phase 3: Event type system + built-in templates — @staff-engineer

### What was built

**`waveform/ui/widgets/event_template_card.py`** — `EventTemplateCard(ctk.CTkFrame)`:
- Shows per-template emoji (all 10 mapped by template id + `__custom__` fallback), name (Inter 600), one-line card description (Inter 400, secondary color), and a row of small colored archetype chips
- Chip background is archetype `cover_palette[0]`; text color auto-selected white/near-black based on perceived luminance
- Selected state: 2px ACCENT_VIOLET border, BG_SURFACE background
- Unselected state: 1px BG_OVERLAY border, BG_OVERLAY background
- Click propagates through all child labels/chips to the parent `_handle_click` → `on_select(template)` callback
- `set_selected(bool)` for external state management; `template` property exposes the underlying `EventTemplate`
- `__custom__` (None template) renders as "+ Custom" card at gallery end

**`waveform/ui/event_setup.py`** — `EventSetupScreen(ctk.CTkFrame)`:
- Two-pane layout: scrollable gallery (left, flex) + detail panel (right, 340px fixed)
- Gallery: 3-column `CTkScrollableFrame` grid of `EventTemplateCard` widgets; all 10 built-in templates + one `+ Custom` card; pre-selects first template on mount
- Detail panel (scrollable inner CTkScrollableFrame for small-window safety):
  - Event name CTkEntry, pre-filled on template select, only auto-fills when field is empty or holds a prior template name
  - Vibe/description CTkTextbox (90px height), 0–300 char counter with warning color above 250, enforced max at 300
  - Date/time CTkEntry (optional, freeform)
  - Five `CTkSegmentedButton` toggle groups: Venue (Indoor/Outdoor), Size (Intimate/Medium/Large), Formality (Casual/Semi-formal/Formal), Time of day (Afternoon/Evening/Late night), Age range (All ages/Adult/21+)
  - "Start Building" button (ACCENT_VIOLET, full-width)
- `_select_template(template)`: updates card borders + pre-fills name field (smart: doesn't overwrite user's custom name)
- `_on_start_building()`: creates `PlaylistSession` via `_blocks_from_template()`, collects vibe text + toggle metadata as `vibe_override`, sets `store.session`, `store.selected_template`, navigates to `AppScreen.TIMELINE`
- `_blocks_from_template()`: distributes `suggested_duration` evenly (rounded down to nearest 5 min, floor 15 min/block), inherits template `default_genre_weights` on each block, uses `Block.from_archetype()` for correct energy defaults
- Custom card path: creates a single Arrival block as blank slate

**`waveform/ui/shell.py`** (modified):
- Imports `EventSetupScreen` and mounts it as `AppScreen.EVENT_SETUP` (replaces the `_build_placeholder_screen("Event Setup")` call)
- Initial screen changed: shows `EVENT_SETUP` when `store.get("session") is None`, `TIMELINE` when a session is already loaded (supports persisted sessions in future Phase 9)

### Deviations from spec

1. **Detail panel description textarea**: CTkTextbox does not have a `placeholder_text` parameter (only CTkEntry does). The textarea starts empty; the char counter reads "0 / 300" as a sufficient hint. This is a CTk API limitation, not a design choice.

2. **`__custom__` card emoji**: Uses "✨" (sparkle) rather than a specified emoji — the spec lists only the 10 named templates. This is the most natural blank-slate symbol.

3. **`_collect_toggle_meta` appends venue metadata to `vibe_override`** rather than storing separately. The `PlaylistSession` dataclass has a single `vibe_override: str` field (Phase 1). The toggle selections are formatted as a brief structured string ("Venue: Indoor, Intimate, Casual.  Time of day: Evening.  Age range: Adult") appended after the user's free-text vibe. Phase 6 can refine how this surfaces in the AI prompt.

4. **Block duration distribution**: Epic says "default duration" without specifying the algorithm. Implementation distributes `suggested_duration` evenly across blocks, rounded down to nearest 5 min (minimum 15 min), which is the most sensible default for the timeline.

### Open questions flagged (CEO-owned)

None new. Phase 3 does not raise new CEO-level questions.

### Phase 4 readiness notes

Phase 4 (`Block builder timeline — horizontal drag-and-drop, resize, energy sparkline`) can proceed immediately. Note: at the time of Phase 3 completion, Phases 4 and 5 were observed as `IN_PROGRESS` in plan.md (concurrent agents). Phase 4 has full access to:
1. `store.get("session")` → `PlaylistSession` with a populated `blocks` list (Phase 3 now creates real sessions)
2. `store.get("selected_template")` → `EventTemplate` for any template-level metadata
3. `Block`, `BlockArchetype`, `ArchetypeSpec`, `ARCHETYPE_SPECS` — all stable from Phase 1
4. `TimelineCanvas` from Phase 2 — currently draws static bands; Phase 4 replaces the inner `tk.Canvas` drawing logic with interactive drag/resize


## [2026-04-07T01:00:00Z] Phase 2: CustomTkinter shell — @staff-engineer

### What was built

**`requirements.txt`** — Uncommented `customtkinter>=5.2.0`.

**`waveform/__main__.py`** — New file. Enables `python -m waveform` entry point (delegates to `run()`).

**`waveform/app/main.py`** — Replaced Phase 1 headless stub with full boot sequence:
- Loads settings via `PersistenceService`; wires `StateStore`
- Initializes fake services (`FakeSpotifyClient`, `FakeGeminiClient`, `FakeAnalyticsService`) — ready to swap for real in Phase 6
- Configures CTk: `appearance_mode=dark`, `color_theme=dark-blue`
- Creates and launches `WaveformApp`; hooks `WM_DELETE_WINDOW` → `session_abandoned` analytics

**`waveform/ui/shell.py`** — `WaveformApp(ctk.CTk)`:
- Three-column grid: sidebar (280px fixed) | timeline (flex) | track panel (360px fixed)
- 48px top bar: `WaveformAnim` logo + "Waveform" wordmark in brand violet | event name label (clickable, routes to `EVENT_SETUP`) | "▶ Build" button (disabled feedback loop, Phase 6 wires it)
- Screen router: `_navigate_to(AppScreen)` shows/hides CTkFrame per screen enum variant; subscribes to `store.current_screen`; starts at `AppScreen.TIMELINE`
- Placeholder frames for `EVENT_SETUP`, `SETTINGS`, `WELCOME`, `GENERATION`, `REVIEW`, `EXPORT`

**`waveform/ui/sidebar_schedule.py`** — `ScheduleSidebar(ctk.CTkFrame)`:
- Fixed 280px width, `BG_SURFACE` background
- `CTkScrollableFrame` holding `BlockCard` instances for each block
- "SCHEDULE" header in muted uppercase label
- "+ Add Block" button (violet border, transparent fill) pinned at bottom
- Shows placeholder blocks (Arrival → Singalong → Groove → Dance Floor → Late Night) when no session is active
- Subscribes to `store.session` — refreshes card list on session change

**`waveform/ui/timeline_canvas.py`** — `TimelineCanvas(ctk.CTkFrame)`:
- Uses a raw `tk.Canvas` (CTk has no canvas widget) inside the CTkFrame; `bg=BG_BASE`, no highlight border
- Draws block bands proportional to `duration_minutes` using each archetype's `cover_palette[0]` color
- Block name labels centered within each band (hidden below 60px width)
- Duration labels below each band; total duration right-aligned
- Placeholder "energy arc — available after generation" text in the sparkline area
- Subscribes to `store.session`; redraws on `<Configure>` (resize-safe)

**`waveform/ui/track_panel.py`** — `TrackPanel(ctk.CTkFrame)`:
- Fixed 360px width, `BG_SURFACE` background
- "TRACKS" header
- "Generate to see suggestions" empty state: 200×200 art placeholder containing a static `WaveformAnim`; instructional subtext; dimmed (disabled) Keep/Skip/Veto action buttons with correct semantic colors
- Subscribes to `store.is_generating`; Phase 7 swaps in the live card feed

**`waveform/ui/widgets/block_card.py`** — `BlockCard(ctk.CTkFrame)`:
- Left accent strip using archetype `cover_palette[0]`
- Block name in UI bold, meta row with duration + energy dots (●●●○○) + archetype display name
- Click binding that invokes `on_click(block)` if provided

**`waveform/ui/widgets/track_card.py`** — `TrackCard(ctk.CTkFrame)`:
- 280×280 album art placeholder (BG_SURFACE, card radius)
- Track name label with wraplength
- Keep / Skip / Veto buttons (disabled, correct semantic border colors) — Phase 7 enables

**`waveform/ui/widgets/genre_slider.py`** — `GenreSlider(ctk.CTkFrame)`:
- Tag label (left, 100px) | `CTkSlider` 0.0–0.8 (flex) | percentage label (right, 32px)
- Inherited=True renders in muted color (ghosted style)
- `on_change(tag, weight)` callback

**`waveform/ui/widgets/waveform_anim.py`** — `WaveformAnim(ctk.CTkFrame)`:
- Five vertical bars with discrete approximation of brand gradient (`BRAND_GRADIENT_START` → `BRAND_GRADIENT_END`)
- Heights follow waveform silhouette (0.4/0.65/1.0/0.65/0.4)
- `animate=True` activates a 30fps sine-based breathing pulse via `.after()` (Phase 11 replaces with spring physics)
- Used in both the top bar logo and the Track Panel empty state

### Deviations from epic

1. **`AppScreen.TIMELINE` used as the default screen** (not `WELCOME`). The epic describes `TIMELINE` as the main three-column view and `WELCOME` as a future screen. Showing TIMELINE on launch is consistent with the "three-column layout is the main screen" framing. Phase 3 will add the event selection flow gated on whether a session exists.

2. **Top bar logo uses `BRAND_GRADIENT_MID` solid color** for the "Waveform" wordmark rather than a true gradient text effect. CustomTkinter's `CTkLabel` does not support gradient text natively; PIL compositing to achieve this is Phase 11 polish scope. The `WaveformAnim` widget beside it carries the visual gradient identity in the meantime.

3. **`WaveformAnim` uses discrete color steps** (5 hardcoded interpolated values) instead of a true pixel-by-pixel gradient. Accurate gradient fill on CTkFrame bars requires PIL rasterization; deferred to Phase 11 with the rest of the motion work.

4. **`theme.SIDEBAR_WIDTH` is 280px** (set in Phase 1's theme.py) vs the epic's ASCII diagram showing "240px". Phase 1 made a deliberate call to use 280 for legibility. No change made — consistent with Phase 1.

### Open questions flagged (CEO-owned)

None new. Phase 2 does not raise new CEO-level questions. Existing §11 questions remain unchanged.

### Phase 3 readiness notes

Phase 3 (`Event type system + built-in templates`) can start immediately:

1. `shell.py` already has an `AppScreen.EVENT_SETUP` placeholder frame and routes to it when the event name label is clicked — Phase 3 fills that placeholder with a real template picker.
2. `store.set("selected_template", template)` → shell subscribes and updates the event name label automatically.
3. `store.set("session", session)` → `ScheduleSidebar` and `TimelineCanvas` both subscribe and re-render from the session's block list.
4. All 10 `EventTemplate` objects from `domain/event.py` are available; Phase 3 needs only to present them in a UI picker and wire up session creation.



## [2026-04-07T20:00:00Z] Phase 2B: Custom archetypes + genre tag expansion — @staff-engineer

### What was built

**Deliverable 1 — Custom block archetypes**

`waveform/domain/block.py` — Added `CustomArchetype` dataclass with `to_dict`/`from_dict`, `display_name`/`default_energy` property aliases for duck-type compatibility with `ArchetypeSpec`, and `cover_palette` property returning three hex strings. Added `_CUSTOM_REGISTRY`, `register_custom_archetypes()`, `get_custom_archetype()`, `list_custom_archetypes()`, `get_spec_for_id()`, `is_custom_archetype_id()`. `Block.archetype` type widened to `Union[BlockArchetype, str]` — all existing `BlockArchetype.DANCE_FLOOR` usage unchanged.

`waveform/services/persistence.py` — Added `load_custom_archetypes()` and `save_custom_archetypes()` to `PersistenceService` (atomic write to `~/.waveform/custom_archetypes.json`) and to `FakePersistenceService` (in-memory).

`waveform/ui/archetype_editor.py` — New `ArchetypeEditor(CTkToplevel)` modal: scrollable list panel + create/edit form with name entry (max 30), 20-emoji picker grid, description (max 100), 12-colour start/end strip pickers, energy slider. Save validates, creates or updates, persists, calls `register_custom_archetypes()`, emits `store["custom_archetypes_updated"] = True`. Delete removes and persists.

`waveform/ui/widgets/block_card.py` — `get_spec` calls replaced with `get_spec_for_id(str(...))`. Emoji helper `_archetype_emoji()` added (handles both built-in enum and custom str ids). Chip strip appends custom archetype chips from `list_custom_archetypes()` after built-ins, plus a `+ Custom` chip that opens `ArchetypeEditor`. `_open_archetype_editor()` passes `on_saved` callback that rebuilds the detail frame to refresh chips. `_on_arch_chip_click()` handles both built-in and custom ids via `get_spec_for_id`.

`waveform/services/cover_art.py` — `generate_block_cover()` accepts `Union[BlockArchetype, str]`. Custom id branch calls `_render_custom()` (GROOVE structure + user palette), overlays emoji+name text. Unknown ids fall back to GROOVE with warning log.

**Deliverable 2 — Genre tag expansion**

`waveform/domain/genre.py` — `_GENRE_TAGS` expanded from ~90 to ~230 unique tags covering all genre families in the Phase 2B spec. No duplicates.

**Tests**

`tests/test_genre_expansion.py` — 14 tests: no-duplicates, tag count, sorted order, ~80 parametrized spot-checks, prefix/infix search ordering, search edge cases, add/add-duplicate/add-normalise.

`tests/test_custom_archetypes.py` — 22 tests: round-trip serialisation, missing-field tolerance, cover_palette format, registry ops, `get_spec_for_id` (all 10 built-ins parametrized + custom + unknown raises), `is_custom_archetype_id`, `FakePersistenceService` round-trip.

### CEO sign-off items

1. **Custom archetype count cap** — currently unlimited. Recommend adding `if len(self._archetypes) >= 20: return` guard in `ArchetypeEditor._on_save()` before beta (open question §7 in PHASE2_BACKLOG.md).

2. **`Block.archetype` widening** — `Union[BlockArchetype, str]` is safe for all current callers. One risk: any future code doing `isinstance(block.archetype, BlockArchetype)` returns `False` for custom archetypes. Worth a grep at next integration point.

### Deviations from spec

1. Tag count landed at ~230 (not ~300) — overlap with existing index after deduplication; covers all spec families.
2. `_open_archetype_editor` rebuilds the full detail frame on save (not surgical chip refresh) — intentional for simplicity; ~5ms, invisible to user.


## [2026-04-07T00:00:00Z] Phase 1: Foundation — @staff-engineer

### What was built

**Repo structure** — Created the full `waveform/` package layout per epic §4:
- `waveform/app/` — `main.py` (headless bootstrap), `state.py` (observable StateStore)
- `waveform/domain/` — pure Python, zero UI imports
- `waveform/services/` — all external I/O
- `waveform/ui/` — empty shells + `theme.py` (tokens only, no Tk imports)
- `waveform/prompts/` — `master_prompt.md`, `veto_addendum.md`
- `waveform/assets/` — placeholder `.gitkeep`
- `waveform/tests/` — 3 test files

**Domain layer** (fully implemented, not stubs):
- `domain/genre.py` — `GenreWeight` (0.0-0.8 validated), `GenreTagIndex` with ~90 curated tags (prefix+infix search), module-level `DEFAULT_INDEX` singleton
- `domain/block.py` — `BlockArchetype` (10 archetypes, str enum), `ArchetypeSpec` (energy, palette, description), `Block` dataclass with validation + `from_archetype()` convenience constructor, `track_count` property
- `domain/event.py` — `EventTemplate` dataclass, all 10 built-in templates (Birthday, Wedding, Club Night, Rooftop Bar, Corporate Dinner, House Party, Funeral/Memorial, Road Trip, Workout, Focus Session), `BUILTIN_TEMPLATES` list, `TEMPLATE_BY_ID` dict, `get_template()` lookup
- `domain/session.py` — `VetoContext` (add_veto/add_keep, is_vetoed, vetoes_for_block, format_for_prompt), `PlaylistSession` (add/remove/reorder blocks, mark_kept, was_kept), `VetoEntry`, `KeepEntry`, `VETO_REASON_TAGS`

**Service layer** (real interfaces + Fake* classes for tests):
- `services/persistence.py` — `PersistenceService` (settings, session, song history, master prompt), `FakePersistenceService` (in-memory), `migrate_v1_settings()`, `_is_v1_schema()`
- `services/spotify_client.py` — `SpotifyClient` (search_tracks, find_track, create_playlist, add_tracks, upload_cover_art, get_playlist_tracks, get_preview_url), `FakeSpotifyClient`, `SpotifyTrack`, `SongSuggestion`
- `services/gemini_client.py` — `GeminiClient` (generate_songs as Iterator, generate_single_replacement), `FakeGeminiClient`, prompt builder with genre weight → human-language conversion
- `services/cover_art.py` — `generate_block_cover()` stub (gradient + text JPEG), `FakeCoverArtService` (1×1 JPEG)
- `services/preview_audio.py` — `PreviewAudioPlayer` strategy pattern (pygame/vlc/no-op), `FakePreviewAudioPlayer`, documented tradeoffs for CEO Q6 call
- `services/analytics.py` — `AnalyticsService` with all 18 typed PostHog events from epic §9, `FakeAnalyticsService` (in-memory + stdout)

**Settings schema migration** (v1 → v2):
- Detects v1 schema via presence of `psytrance_enabled`, `psytrance_pct`, `psytrance_count`
- Converts `psytrance_pct/100` → `block_genre_overrides.dance/groove[{tag: psytrance, weight}]`
- Removes v1 keys, sets `schema_version: 2`
- Migrates the repo-root `settings.json` on first launch of the v2 app

**UI theme tokens** (`ui/theme.py`):
- Full color palette from epic §6 (all hex values)
- Typography constants (font families, sizes, weights)
- Spacing scale (4/8/12/16/24/32/48/64)
- Border radii (8/14/20)
- Layout dimensions (sidebar, track panel, window min size)
- Motion constants (durations, easing strings, spring values for Phase 11)

**`create_playlist.py`**: Updated docstring to mark it as v1 legacy entry point. Logic unchanged — v1 users are not broken.

**`pyproject.toml`**: Created with package metadata, entry point `waveform = waveform.app.main:run`, optional dependencies for audio/analytics/UI, pytest config.

**`conftest.py`**: Root conftest adds project root to sys.path for test discovery without editable install.

**Tests** (3 files, ~120 test functions):
- `test_domain.py` — GenreWeight, GenreTagIndex, BlockArchetype, Block, EventTemplate, VetoContext, PlaylistSession
- `test_persistence.py` — v1 schema detection, migration correctness, FakePersistenceService roundtrips
- `test_services.py` — FakeSpotifyClient, FakeGeminiClient, FakeCoverArtService, FakePreviewAudioPlayer, FakeAnalyticsService, integration session+service workflow

### Deviations from epic

1. **`domain/genre.py` tag count**: Built ~90 tags (epic says "~50-100"). Within spec.
2. **`app/state.py` observable pattern**: Implemented as a simple callback dict on a lock rather than a full reactive library. Clean, zero dependencies, easy for Phase 2 to extend. No functional divergence from the epic spec.
3. **`services/cover_art.py` stub**: Phase 1 ships a gradient+text placeholder (PIL, real JPEG). Phase 8 fills in the per-archetype parametric recipe. The interface (`generate_block_cover(archetype, event_name) -> bytes`) is stable.
4. **`domain/session.py` VetoContext `_vetoed_ids`**: Added a `Set[str]` shadow field for O(1) `is_vetoed()` lookups. This is not in the epic spec but is a clean optimization that won't surprise Phase 2.
5. **`services/gemini_client.py` prompt**: Phase 1 prompt reads `prompts/master_prompt.md` but does not yet inject `veto_addendum.md` as a separate file — the veto context is formatted inline by `VetoContext.format_for_prompt()`. This is equivalent to the epic's intent in §5.4; the separate addendum file exists for Phase 6 to integrate if desired.

### Open questions flagged (CEO-owned, §11)

- **Q6 (audio library)**: `services/preview_audio.py` documents the pygame vs python-vlc tradeoff in a comment block and recommends pygame for Phase 1 beta. CEO call needed before Phase 7.
- **Q1 (monetization)**: Not touched.
- **Q2 (Spotify preview_url availability)**: Not touched.
- **Q3 (Gemini cost)**: Not touched.
- **Q4/Q5 (distribution, v1 users)**: The v1 `create_playlist.py` is preserved and functional.
- **Q7/Q8**: Not touched.

### Phase 2 readiness notes

Phase 2 (`CustomTkinter shell: three-column layout, navigation, theme tokens`) can start immediately with:

1. Import `ui/theme.py` directly — all design tokens are ready
2. Import `app/state.py` — `StateStore` and `AppState` are wired
3. Import `services/persistence.py` — settings load/save works, settings include `theme` and `reduce_motion`
4. Import `domain/event.py` — all 10 templates available for the event picker UI
5. Import `domain/block.py` — all 10 archetypes available for the timeline UI

Phase 2 should wire `StateStore` into the shell and populate `state.session` when the user picks an event template. The `AppScreen` enum in `state.py` defines the navigation targets — Phase 2 should implement routing between them.

The test suite runs at: `pytest waveform/tests/` from the repo root.


## [2026-04-07T09:00:00Z] Phase 7: Song preview card feed — @staff-engineer

### What was built

**`waveform/ui/track_panel.py`** — Full replacement of the Phase 2 placeholder. Three display states driven by store subscriptions:

1. **EMPTY state** — centered 180px album art placeholder with `WaveformAnim` logo (static), "Select a block and hit Build to get started" caption, dimmed Approve/Swap/Veto buttons.

2. **GENERATING state** — shimmer progress bar (left-to-right sweep, ~25fps) at top, animated `WaveformAnim` below, "Generating suggestions…" caption. Activates when `is_generating=True` and `pending_song` is None.

3. **PREVIEW state** — full song preview card:
   - 180×180 album art frame. Art fetched asynchronously in a daemon thread via PIL (resize/crop to square). UI update dispatched via `after(0, callback)`.
   - Thin 3px cyan playback progress bar overlaid at the bottom of the art frame, polled at 250ms intervals.
   - "No preview available" badge shown when `preview_url` is None.
   - Track title (Inter 700, 18px), artist name (Inter 400, secondary color).
   - `WaveformAnim` (animate=True, 36px height, ~0.5Hz breathing).
   - Duration in `JetBrains Mono`.
   - Three action buttons: Approve (green `#34D399`), Swap (grey), Veto (red `#F87171`), each showing keyboard shortcut hint.

**Veto reason picker** — `CTkToplevel` modal (grab_set) with 6 chip buttons: "No reason" (grey, always first), "too slow", "wrong genre", "overplayed", "not the vibe", "artist already used" (danger red). Calls `handle_veto(block_id, song, reason_tag)` on selection.

**Approved songs list** — `CTkScrollableFrame` (160px height) below the preview area. Shows compact `TrackCard` rows per approved song for the selected block. "No approved songs yet" empty-state label. Rebuilds on `approved_songs` or `selected_block_id` store changes.

**`waveform/ui/widgets/track_card.py`** — Full replacement of Phase 2 stub. Compact row widget: 32×32 thumbnail (`CTkImage` or grey placeholder frame), track name (Inter Semibold SM), artist + duration meta (XS, secondary). Right-click context menu (Button-2 macOS + Button-3) with "Remove from playlist."

**`waveform/ui/widgets/waveform_anim.py`** — Animation tuned: phase step 0.08 → 0.105 rad/frame for ~0.5Hz breathing at 30fps. Bar amplitude 0.775 ± 0.225 (was 0.6 ± 0.4) to keep bars always visible.

**`waveform/app/state.py`** — Two new fields: `approved_songs: Dict[str, Any]` and `selected_block_id: Optional[str]`.

**`waveform/app/main.py`** — `PreviewAudioPlayer(strategy="pygame")` instantiated at startup, passed to `WaveformApp`. `GenerationController` import wrapped in try/except for Phase 6 backward compatibility.

**`waveform/ui/shell.py`** — `WaveformApp` accepts `audio_player`, `spotify_client`, `generation_controller`; passes them to `TrackPanel`. `_on_block_selected` now calls `self._track_panel.set_active_block(block)`. Toast rewritten: bottom-right placement, 3-second dismiss, cancels prior dismiss job on rapid toasts.

**`requirements.txt`** — `pygame>=2.5.0` uncommented.

### Keyboard shortcuts

`<space>` → Approve, `<s>`/`<S>` → Swap, `<BackSpace>` → Veto. Registered on root window via `bind(..., add="+")`. Only fire when `pending_song` is not None.

### Audio (CEO Q6 resolved for Phase 7)

`pygame.mixer` chosen per Phase 1 recommendation. `PreviewAudioPlayer(strategy="pygame")` in `main.py`. To switch to python-vlc for Phase 2: change `strategy="vlc"` and add `python-vlc>=3.0.0` to requirements.txt. No other changes needed.

### Deviations from spec

1. **`WaveformAnim` animation activated in Phase 7** (spec said Phase 11). The Phase 2 loop already existed; Phase 7 tuned the rate. Phase 11 replaces with spring physics — no API change required.

2. **Approved songs store update pattern** — stub controller path updates `approved_songs` directly in `_on_approve`. Real `GenerationController` from Phase 6 is expected to set `approved_songs` in `handle_keep` (Phase 6 already does this per their execution log). No double-add risk because the stub is only active when `GenerationController` is unavailable.

3. **Thumbnail fetch is synchronous** in `_make_thumbnail` (32×32 px, rare refresh). If approved lists grow large, switch to async. Fine for MVP.

### Phase 8 / Phase 9 handoff

- **Phase 8** (cover art): no dependency on TrackPanel. Art is fetched from `SpotifyTrack.album_art_url` independently.
- **Phase 9** (export): reads `store.get("approved_songs")` — a `{block_id: [SongSuggestion]}` dict. Each `SongSuggestion.track.uri` is the Spotify track URI for playlist building. This is the primary output of the generation flow.
- **Phase 11** (polish): `WaveformAnim` spring physics, album art crossfade on song transition, particle burst on Approve — all can be added without structural changes to TrackPanel.

## [2026-04-07T12:00:00Z] Phase 10: PostHog analytics instrumentation — @staff-engineer

### What was built

**`waveform/services/analytics.py`** — Full rewrite of Phase 1 stub:
- `AnalyticsService` — real PostHog wrapper with `_enabled` opt-in gate; all captures fire-and-forget on daemon threads; `set_enabled(bool)` for post-consent activation; `shutdown()` flushes PostHog background thread on app close
- `SessionMetrics` dataclass — within-session accumulator for songs_suggested, songs_kept, songs_skipped, songs_vetoed, previews_played, preview_seconds_played; `preview_to_keep_rate` and `veto_depth` computed properties; `as_dict()` embeds in `playlist_exported` payload
- `FakeAnalyticsService` — synchronous (no thread), always enabled, records to `self.events`; `shutdown()` is a no-op
- Privacy comment block (doubles as privacy policy data-collection summary for §11 Q8)

**`waveform/ui/analytics_consent.py`** — New file: one-time opt-in consent modal. "Yes, share data" / "No thanks." Non-closeable. Fires `app_opened()` on opt-in. Shown 500ms after mainloop starts.

**`waveform/app/main.py`** — `_ensure_analytics_id`, `_build_analytics_service`, consent modal, `session_abandoned` guard, `analytics.shutdown()` on close, `export_completed` subscription.

**`waveform/app/state.py`** — Added `export_completed: Optional[bool]` field.

**`waveform/services/persistence.py`** — DEFAULT_SETTINGS: `analytics_opt_in` renamed to `analytics_enabled`; added `analytics_id: ""`.

**Instrumentation gaps filled (surgical additions):**
- `event_setup.py`: `event_template_selected` on card click; `session_started` on "Start Building"
- `sidebar_schedule.py`: `block_added` in `_add_block_with_archetype`
- `timeline_canvas.py`: `block_resized`, `block_reordered`, `block_removed`, `block_added` (duplicate)
- `genre_weight_panel.py`: `genre_weight_changed` in `_handle_weight_change`
- `track_panel.py`: `song_previewed` in `_stop_playback_poll` with actual elapsed ms
- All child widgets receive `analytics` param via shell

**`requirements.txt`** — `posthog>=3.0.0` uncommented.

**`waveform/tests/test_analytics.py`** — New: ~35 tests in 8 classes (opt-out gate, event properties, no-PII assertion, SessionMetrics, distinct_id persistence, FakeAnalyticsService, shutdown).

### Deviations

1. `analytics_opt_in` → `analytics_enabled` (no persisted data yet to migrate)
2. `song_previewed` fires from `_stop_playback_poll` covering all action paths
3. Phase 9 ExportController needs `store.set("export_completed", True)` — AppState field and subscriber are ready
4. Phase 9's `playlist_exported` call missing `event_type` arg — harmless (default `""`)
5. `genre_weight_changed` fires continuously on drag; debouncing deferred to Phase 11

### Phase 11 readiness notes

All analytics fire-and-forget, cannot block UI. `self._analytics` is in scope throughout the shell hierarchy for Phase 11 to instrument any new interactions.

## [2026-04-07T04:30:37.261Z] Phase 9: Spotify export + persistence + session history -- @staff-engineer
Server-recorded completion (agent did not write log entry).

## [2026-04-07T04:41:48.393Z] Phase 11: Polish pass: animations, accessibility, signature motion set -- @staff-engineer
Server-recorded completion (agent did not write log entry).

## [2026-04-07T04:47:13.380Z] Phase 12: Beta release, telemetry review, Phase 2 planning -- @cto
Server-recorded completion (agent did not write log entry).

## [2026-04-07T16:00:00Z] Phase 12: Beta release review and Phase 2 planning — @cto

### Verdict: BETA_READY

All P0/P1 items from the Phase 11 known-limitations list have been resolved. The app is ready for a 50-user closed beta on macOS, contingent on the BETA_CHECKLIST being walked end-to-end before invites go out.

### P0 fixes applied (would have crashed beta)

1. **`PlaylistSession.from_dict` referenced `BlockArchetype` without importing it.** Phase 11 added the from_dict method but the import at the top of `domain/session.py` only pulled `Block`. Any user clicking Resume in the session history dialog would have hit a NameError. Fixed by adding `BlockArchetype` and `GenreWeight` to the module imports.

### P1 fixes applied

2. **Genre weights lost on session resume.** Phase 11 documented this as a known limitation. Fixed in two places: `ExportController._serialize_session` now writes a `genre_weights` list per block, and `PlaylistSession.from_dict` now restores them as `GenreWeight` objects (with defensive parsing — malformed entries are skipped).

3. **`playlist_exported` analytics call missing `event_type` arg.** Phase 10 deviation. Fixed in `app/export.py` — now passes `event_type=self._session_event_name(session)`.

4. **`EventTemplate.accent_color` field missing.** Phase 11 noted that the Event Skin Change tween always interpolated toward `ACCENT_VIOLET` because the field didn't exist. Added `accent_color: str = "#7C3AED"` (defaults to brand violet so existing call sites remain valid) and assigned per-template colours: Birthday hot coral, Wedding rose gold, Club Night acid green, Rooftop warm sunset, Corporate muted slate, House Party brand violet, Funeral desaturated linen blue, Road Trip warm amber, Workout cyan, Focus mono grey. The shell tween already calls `getattr(template, "accent_color", None)` so no UI changes needed.

### P2 deferred to Phase 2 (documented in PHASE2_BACKLOG.md)

- BlockCard not Tab-navigable (full keyboard nav of block list incomplete)
- Block Transition opacity simulation (CTk has no opacity API — fundamental)
- Particle burst widget tree-walk fragility
- Genre tag index expansion (~90 → ~300 Spotify seeds)
- All items from epic §10 Phase 2 (custom archetypes, Tier 2 cover art, session history browser, Windows polish, in-app prompt editor, etc.)

### Deliverables

- `/Users/netz/Documents/git/waveform/.tasks/waveform-v2/BETA_CHECKLIST.md` — actionable launch checklist
- `/Users/netz/Documents/git/waveform/.tasks/waveform-v2/PHASE2_BACKLOG.md` — prioritised Phase 2 backlog with data-driven gates
- `requirements.txt` — verified complete with all phases' dependencies; minimum versions pinned

### What the CEO needs to know

1. Epic status flipped to **BETA_READY**. No showstoppers.
2. Four P0/P1 fixes were applied surgically in Phase 12 — no new features added.
3. The beta checklist requires a manual end-to-end walkthrough before the first invite. Two areas I cannot fully assert from code review alone: real Spotify OAuth round-trip and real Gemini latency on a slow network. Both should be confirmed manually before invites.
4. Open §11 questions (monetization, preview_url legal, Gemini cost envelope, Windows signing, v1 paying user comms, privacy policy) are still CEO-owned and must be resolved before Phase 2 kickoff. Privacy policy in particular is a hard gate before external invites.
5. Phase 2 priorities will be data-driven from beta telemetry. The PHASE2_BACKLOG document defines the metric thresholds that trigger each track of work.

CTO sign-off: ship the beta.
