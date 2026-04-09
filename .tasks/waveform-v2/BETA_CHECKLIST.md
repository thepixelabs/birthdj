# Waveform v2 — Beta Readiness Checklist

Owner: CTO. Bar to ship to 50 closed-beta users.

## Smoke / launch

- [ ] `pip install -r requirements.txt` succeeds in a clean venv (Python 3.11+)
- [ ] `python -m waveform` launches without exceptions on macOS 13+
- [ ] Window opens at intended layout (sidebar / timeline / track panel visible)
- [ ] Closing the window terminates the process cleanly (no zombie threads, pygame mixer released)
- [ ] `~/.waveform/` is created on first launch; v1 settings/song-history are migrated and `.bak`'d

## Auth & external services

- [ ] Spotify OAuth flow completes end-to-end and tokens land in `~/.waveform/settings.json`
- [ ] Spotify token refresh works on second launch (no re-auth required)
- [ ] Gemini API key from settings produces a non-empty generation
- [ ] Network failure on Gemini surfaces an inline toast, not a modal crash
- [ ] Network failure on Spotify export surfaces an inline error in the export dialog with a Retry button

## Core happy path

- [ ] Event Setup screen lists all 10 templates and the Custom card
- [ ] Selecting a template seeds blocks, genre weights, and accent color
- [ ] Block builder: add, remove, drag-reorder, drag-resize, double-click rename, energy slider all work
- [ ] Genre weight panel: autocomplete, sliders, max-6 enforcement, inherited rows clickable
- [ ] Build button generates streaming songs into the track panel
- [ ] Approve / Skip / Veto keyboard shortcuts (Space / S / Backspace) work
- [ ] Veto context demonstrably feeds back into the next generation call
- [ ] 30-second preview audio plays via pygame.mixer; absence of preview_url is handled gracefully
- [ ] Cover art renders for all 10 archetypes (no exceptions)
- [ ] Export dialog: Full Night and Split modes both produce playlists in Spotify
- [ ] Cover art uploads successfully to Spotify (≤256 KB JPEG conversion)
- [ ] Existing-playlist collision dialog (Overwrite / Append / Rename) works
- [ ] Session is saved to `~/.waveform/sessions/` after export
- [ ] Session history dialog lists prior sessions and Resume restores blocks + genre weights

## Persistence & history

- [ ] Per-playlist song history dedup works across sessions (no duplicate songs in successive generations)
- [ ] Cross-session "recently exported" cache prevents repeats
- [ ] Block-level genre weights survive a session resume (Phase 12 fix)
- [ ] Settings screen accessible from top bar; all toggles persist across restarts

## Analytics & privacy

- [ ] Analytics opt-in prompt appears on first launch
- [ ] Opting out actually disables PostHog capture (verify no network calls)
- [ ] No track names or PII in production payloads when "share for improvement" is off
- [ ] `playlist_exported` includes `event_type`, `n_blocks`, `n_tracks`, `time_from_open_ms` (Phase 12 fix)
- [ ] `session_abandoned` fires on close when no export occurred

## Polish bar

- [ ] Generation Pulse animation visible during generation
- [ ] Block Card Reveal stagger fires on session load
- [ ] Song Approved particle burst fires on Approve
- [ ] Event Skin Change tween produces a visible per-template accent shift (Phase 12 fix: real per-template colors)
- [ ] Reduce Motion toggle in settings disables all `after()`-based animations within one frame
- [ ] Visible focus ring on Tab navigation through Event Template cards, Build button, Add Block button
- [ ] WCAG AA text contrast on TEXT_PRIMARY / TEXT_SECONDARY against backgrounds

## Distribution

- [ ] `requirements.txt` pins minimum versions for: spotipy, google-genai, Pillow, customtkinter, pygame, posthog, python-dotenv, questionary
- [ ] README documents how to obtain Spotify and Gemini API keys
- [ ] Privacy policy exists and is linked from the analytics opt-in dialog
- [ ] Notarized .dmg build instructions documented (signing cert may be deferred)
- [ ] A short Loom-style video walks a new user from launch to first export

## Known acceptable limitations entering beta (P2)

- BlockCard not Tab-navigable (full keyboard nav incomplete)
- Block Transition fade approximation only (CTk has no opacity API)
- Particle burst widget tree-walk is fragile to track-panel layout changes
- Tier 2 cover art (DALL-E/Imagen) deferred
- Per-block cover art off by default
- Veto reason tags captured but not surfaced in PostHog dashboards
- Windows polish deferred (macOS-first beta)
