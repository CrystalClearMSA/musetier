# MUSETIER — FULL ROADMAP TO RELEASE 1.0

---

## EA 1.1 — STABILITY TRACK
*Tightening everything shipped in 1.0–1.1. No new systems, only correctness and completeness.*

### 1.1.1 — Core Bug Fixes
- `_suppressBroadcast` boolean migrated to `_suppressDepth` re-entrant counter throughout all collab handlers
- Tier row reorder in edit mode pushes a `tier_reorder` entry to `undoStack` (currently a dead zone for Ctrl+Z)
- Undo `tier_reorder` restores the previous DOM order of all rows, not just the moved one
- `pendingRestore` hardened: entries that fail to find a matching item after all batches are consumed log a warning and clear cleanly rather than persisting as a dangling Map

### 1.1.2 — Bracket Keyboard & Persistence
- Matchup overlay: left/right arrow keys highlight the respective side, Enter/Space confirms pick
- Matchup overlay: `1` and `2` number keys as alternative pick shortcuts
- Bracket state serialised to sessionStorage on every pick — survives accidental page reload mid-tournament
- Bracket restore prompt on load if an in-progress session is detected in sessionStorage
- Bracket state cleared from sessionStorage on normal completion or manual exit

### 1.1.3 — Pool Controls
- Pool sort options: alphabetical A→Z, Z→A, reverse import order, random shuffle (seeded, reproducible)
- Sort state persists in localStorage alongside other settings
- Pool drag-to-reorder within a category (manual ordering saved to catStore)
- Pool item count badge updates immediately on sort/reorder without requiring save cycle

### 1.1.4 — Export Expansion
- Structured JSON export: full tier list as `{ tiers: [{ name, color, items: [{ alt, label }] }] }` — importable
- Bracket results JSON export: match log with winner, loser, round, timestamp per match
- Export modal gets tabs: Plain Text (existing), JSON Rankings, Bracket Results
- Clipboard fallback for environments where `navigator.clipboard` is unavailable

### 1.1.5 — MusicBrainz Hardening
- Request deduplication: identical release lookups within a session share a single in-flight Promise (extend existing `_mbThumbInFlight` pattern to cover all MB endpoints, not just thumb)
- Proper rate-limit backoff: 429 responses trigger exponential backoff queue rather than silent drop
- MB search results cache per session — searching the same term twice skips the network call
- CAA (Cover Art Archive) fallback chain: primary → release group → null, with per-level caching

### 1.1.6 — Mobile Audit Pass 2
- Bracket matchup overlay layout at <420px: stack vertically, full-width pick buttons, larger touch targets
- Grid pool strip on iOS: long-press drag verified, ghost preview clipped correctly within viewport
- Collab panel on small screens: room code input and join button stacked, not truncated
- Bottom sheet modals: max-height respects `dvh` not `vh` on all modal types (audit every `openOverlay` call)
- iOS Safari: verify no input zoom on any `font-size < 16px` input (audit all inline `oninput` fields)

### 1.1.7 — FX Refinement
- Rain FX mode: proper half-res render verified on retina — no blurring at scale
- Embers FX: spawn rate scales with window width so wide displays aren't sparse
- Stars FX: twinkle phase randomisation seeded on load so it doesn't reset on resize
- New FX mode: **Vinyl** — slow rotating record silhouette in background, subtle groove lines
- FX canvas resize debounced (currently fires on every pixel of window resize)

### 1.1.8 — Timing Hardening
- `onImagesLoaded` collab pool broadcast: 1500ms `setTimeout` replaced with a `dataUrl` settled Promise.all barrier — broadcasts only after all blob→dataUrl conversions complete
- All other `setTimeout` timing hacks audited and documented — each one either replaced with an event or marked with a comment explaining why a delay is genuinely necessary
- `save._timer` debounce reviewed: 80ms is probably fine but verify it doesn't cause missed saves on rapid operations like multi-item drag

### 1.1.9 — Collab Stability
- Late join: peer who joins an active session receives current full tier state + pool state immediately on connect, not just future diffs
- Reconnect handling: if a peer drops and reconnects within the same room, state re-syncs automatically without requiring manual rejoin
- Mid-session peer count badge reflects actual connected count, not last-known count
- Collab disconnect during bracket: graceful degradation — solo play continues rather than hard error

### 1.1.10 — Accessibility Pass 1
- All tier zone drop targets reachable by keyboard: Tab to focus, Enter to drop currently "held" item
- Modal focus trap: Tab cycles within open modal, Shift+Tab reverses, no focus escape to background
- `aria-label` on all icon-only buttons (tier ctrl buttons, pool clear, edit mode toggle)
- Color contrast audit on all themes — flag any foreground/background pair below WCAG AA 4.5:1
- Reduced-motion media query: FX canvas and item-land animations respect `prefers-reduced-motion`

---

## EA 1.2 — TRANSPORT MIGRATION & MUSETIER.CC
*Tear out WebRTC entirely. Replace Trystero, MQTT, and Xirsys with Cloudflare Workers + Durable Objects. Register musetier.cc. Nothing changes for the user — everything changes under the hood.*

### 1.2.1 — Infrastructure Setup
- Register `musetier.cc` via Cloudflare Registrar — at-cost, no markup
- Cloudflare account configured: Workers, Durable Objects, and Pages all under one dashboard
- `wrangler.toml` bootstrapped: Worker entry point + Durable Object namespace (`ROOMS`) bound
- DNS configured: `musetier.cc` → app (GitHub Pages or Cloudflare Pages), `ws.musetier.cc` → Worker
- SSL handled automatically by Cloudflare — no manual certificate management ever

### 1.2.2 — Durable Objects Room Class
- `Room` Durable Object class written: holds connected WebSocket sessions + authoritative tier state in memory
- `WebSocketPair` pattern: Worker receives upgrade request, derives room ID from URL, forwards to correct Room object
- On connect: new client immediately receives full current state (`tier_state` + `pool_state`) — late join solved natively
- On message: state updated, broadcast fanned out to all other connected sessions
- On close/error: session removed from list cleanly, no stale reference leaks
- Room hibernation: Cloudflare sleeps idle objects automatically, `this.storage` persists across sleep cycles
- Room state written to `this.storage` on every meaningful change — survives cold wakes

### 1.2.3 — Collab Module Replacement
- `collab.js` internals swapped: Trystero send/receive replaced with native WebSocket send/receive
- `_suppressDepth` counter and all broadcast machinery retained — only the transport layer changes
- Room code generation: remains same UX (6-char alphanumeric), now maps directly to Durable Object ID
- Connection lifecycle: `WebSocket.onopen` → send join, `onmessage` → dispatch, `onclose` → reconnect attempt
- Reconnect logic: exponential backoff on unexpected disconnect, max 5 attempts before surfacing error to user
- `collabConnected` state and all UI bindings (peer count, room badge, connect/disconnect buttons) unchanged externally

### 1.2.4 — Teardown
- Trystero dependency removed entirely
- MQTT broker config removed
- Xirsys ICE server fetching and all three response-shape parsers removed
- TURN/STUN config removed
- All `peerjs`/`trystero` references purged from codebase
- `wrangler.toml` is now the only new config file required to run collab

### 1.2.5 — Regression & Verification
- 2-player collab: connect, sync tier state, sync pool state, disconnect, reconnect — verified
- 3-player Aux Battle: all three roles (Judge, Contestant A, Contestant B) verified over Workers transport
- Late join verified: peer joins mid-session and receives full current state without manual resync
- Cross-network verified: two clients on different networks (no LAN shortcut) — this was the Xirsys failure mode
- Mobile verified: iOS Safari and Android Chrome WebSocket connections stable
- Free tier headroom confirmed: normal EA usage comfortably within 100k requests/day and Durable Objects free allocation

### 1.2.6 — Collab UX Improvements (Now Possible)
- Connection status indicator shows actual WebSocket ready state (CONNECTING / OPEN / CLOSED) — not inferred from Trystero events
- Room persistence: room stays alive on server for 24h after last disconnect — rejoin same code next day and state is still there
- Peer list now authoritative: server tracks who is connected, client list reflects reality not last-known state
- Disconnect reason surfaced to user: "connection closed by server", "network error", "room expired" — not generic failure toast

---

## EA 1.3 — THE GRID GROWS UP
*Grid was shipped functional but shallow. This track makes it a first-class feature.*

### 1.3.1 — Grid Undo
- Full undo stack for grid: cell place, cell clear, cell swap, batch fill
- Grid undo is separate from tier undo stack — Ctrl+Z context-aware based on active tab
- Grid undo max depth configurable in settings (default 50)
- Undo toast shows what was undone ("Cell (3,4) cleared ↩")

### 1.3.2 — Grid Save Slots
- 5 named grid save slots, separate from tier save slots
- Each slot stores: cell contents, grid dimensions, cell size, label position, background colour, corner radius
- Slot thumbnail preview: small canvas render of the grid layout
- Slots persist in localStorage under a separate key so tier and grid saves never collide

### 1.3.3 — Grid Cell Controls
- Per-cell right-click context menu: Clear, Move to Pool, Replace, Edit Label
- Cell hover tooltip shows full item name when cell is too small to display it
- Cell label display options: None, Bottom, Side, Overlay (bottom-left corner, semi-transparent)
- Cell border toggle: show/hide grid lines independently of background colour

### 1.3.4 — Grid Export
- Export at configurable resolution: 1x (screen), 2x, 3x pixel density
- Export includes optional title bar above grid (text input, font follows current theme font)
- Export crops to content bounds — no giant empty whitespace for partially filled grids
- Copy to clipboard option alongside download

### 1.3.5 — Grid Templates
- Named grid templates: save current layout (dimensions + settings, no items) as a reusable template
- Built-in templates: 3x3, 4x4, 5x5, 3x10, 5x10
- Template picker in grid settings cog
- Templates exported/imported as part of full app JSON export

### 1.3.6 — Grid Categories
- Pool strip filter by category: dropdown above strip to show only one category's items
- Strip "show placed" toggle: hide items already placed in the grid (currently they just dim)
- Strip search: same filter-pool-style text input above the grid pool strip
- Strip order follows active pool sort setting

### 1.3.7 — Grid Multi-Select
- Shift+click in pool strip to multi-select items
- Drag multi-selected group onto a grid cell: fills from that cell across in row order
- Delete/Backspace removes selected strip items (mirrors tier pool behaviour)
- Select-all button for pool strip

### 1.3.8 — Grid Misc Polish
- Grid label font inherits current theme font (currently hardcoded)
- Grid label font size slider (8px–18px)
- Cell size keyboard shortcut: `[` and `]` to decrease/increase cell size while on grid tab
- Grid settings cog remembers last open state (expanded/collapsed) across sessions

---

## EA 1.4 — BRACKET EXPANSION
*Tournament mode becomes a full feature rather than a novelty.*

### 1.4.1 — Bracket Persistence & History
- Full bracket state (all rounds, all results) persists to localStorage on completion
- Bracket history panel: list of past completed brackets with date, size, winner
- Click a past bracket to view it in read-only replay mode
- Delete individual history entries or clear all

### 1.4.2 — Third Place & Consolation
- Third-place match toggle in bracket settings (off by default)
- Consolation bracket option: losers drop into a separate bracket rather than being eliminated
- Final standings display: 1st, 2nd, 3rd (4th if consolation) with item art
- Standings exportable as image or JSON

### 1.4.3 — Double Elimination
- Double-elimination mode: losers bracket runs parallel to winners bracket
- UI: two bracket trees rendered side by side, winners bracket left, losers bracket right
- Grand final: winners bracket winner vs losers bracket winner, with optional bracket reset
- Seeding logic respected across both brackets

### 1.4.4 — Group Stage
- Optional group stage before knockout: items divided into groups of 4, round-robin within group
- Group stage results feed into knockout seeding (group winners, runners-up)
- Group stage configurable size (groups of 3, 4, or 5)
- Group stage UI: compact round-robin table per group with running tallies

### 1.4.5 — Bracket Results → Tiers
- Post-bracket: "Push to Tiers" button generates tier rows from bracket results
- Winner → S tier, finalist → A tier, semifinalists → B tier, quarterfinalists → C tier (configurable mapping)
- Existing tier contents preserved — pushed items append or merge based on user choice
- Undo covers the full push operation as a single `bracket_push` entry

### 1.4.6 — Spectator Mode (Bracket)
- Read-only collab join for bracket sessions — spectators see live matchup updates but cannot vote
- Spectator count shown in bracket header
- Spectator chat sidebar (same chat system as Aux Battle)
- Spectator join via same room code system, differentiated by role on join

### 1.4.7 — Mystery Mode Polish
- Mystery mode: item art revealed after pick is confirmed, not before (currently reveals on hover)
- Mystery mode: optional "reveal all" button at end of round
- Mystery mode: title/alt text also hidden until reveal (currently only art is hidden)
- Mystery mode setting persists across bracket sessions

### 1.4.8 — Bracket UI Polish
- Bracket scroll: large brackets (64+ items) scroll horizontally within the bracket frame, not the whole page
- Bracket zoom: pinch-to-zoom on mobile, Ctrl+scroll on desktop
- Bracket node size: small/medium/large option in settings (affects how much art vs text is shown)
- Bracket connector lines animated on round advance (draw from left to right)

---

## EA 1.5 — SHARING & SOCIAL
*Make tier lists shareable without requiring the recipient to run a collab session.*

### 1.5.1 — Shareable URL
- Tier list state encoded as compressed base64 in URL hash (item alts + tier assignments only, no art)
- URL fits in ~2000 chars for typical lists (32 items across 6 tiers)
- Share button in toolbar generates and copies URL
- Opening a share URL prompts: "Load this tier list? It will not replace your current list unless you confirm."

### 1.5.2 — Read-Only View
- `/view` URL mode: renders tier list in read-only display, no drag, no edit, no controls
- Hides toolbar, pool, edit controls — clean presentation layout
- Responsive: works on mobile as a pure viewer
- Meta tags populated from tier list name for link preview (og:title, og:image if exportable)

### 1.5.3 — Tier List Image Card
- "Share as Card" export: styled HTML card rendered to canvas, downloaded as PNG
- Card includes: tier rows with art, tier labels, theme colours, optional title and date
- Card aspect ratio options: square (for social), wide (for Twitter/X), tall (for stories)
- Font and colour pulled from current theme automatically

### 1.5.4 — Aux Battle Discovery (Optional)
- Optional public room listing: rooms can be marked "open to join" by host
- Public room list page (separate route or modal) — shows room name, host, current player count
- Rooms expire from listing after 10 minutes of no activity
- Opt-in only — default is private, same as current behaviour

### 1.5.5 — Share History
- Recent shares logged locally: URL shares, image exports, text exports with timestamps
- Share history panel in export modal
- Re-share from history without regenerating
- Clear history option

---

## EA 1.6 — THE REFACTOR
*No new features. Single-file architecture replaced with a proper module structure. External shape of the app does not change.*

### 1.6.0 — Module Extraction (Phase 1: State & Persistence)
- Extract: `state.js` — all global let/const (catStore, undoStack, activeCategory, currentTheme, etc.)
- Extract: `persist.js` — save(), load(), save slots, localStorage keys
- Extract: `collab.js` — all Trystero/MQTT logic, `_suppressDepth`, broadcast throttle, handlers
- Build tool introduced: Vite, single-file HTML output bundle preserved for GitHub Pages deploy
- No functional changes — pure extraction with identical behaviour verified by manual regression

### 1.6.1 — Module Extraction (Phase 2: Features)
- Extract: `tier.js` — addTier, deleteTier, tier drag, edit mode, tier label/colour
- Extract: `pool.js` — filterPool, updatePoolCount, catBar, category management
- Extract: `items.js` — createItem, setupDraggable, bitmapCache, urlRegistry, blob management
- Extract: `undo.js` — undoStack, undoPush helper, all undo handlers

### 1.6.2 — Module Extraction (Phase 3: Features cont.)
- Extract: `bracket.js` — all bracket logic, seeding, matchup, results, history
- Extract: `grid.js` — all grid logic, pool strip, cell management, export
- Extract: `library.js` — library panel, MusicBrainz search, Last.fm search, item generation
- Extract: `fx.js` — fxCanvas, all FX modes, resize handler

### 1.6.3 — Module Extraction (Phase 4: UI)
- Extract: `ui.js` — toast, overlay system, modal open/close, tour/onboarding
- Extract: `export.js` — all export functions (image, JSON, text, card)
- Extract: `theme.js` — applyTheme, theme definitions, FX-per-theme mapping
- Extract: `settings.js` — all settings panel logic

### 1.6.4 — Cleanup & Hardening Post-Refactor
- All `setTimeout` timing hacks that survived the refactor documented or replaced
- Dead code removed (grep for unreferenced functions post-extraction)
- ESLint pass: no `var`, no implicit globals, consistent formatting
- Bundle size check: output HTML should not exceed 110% of pre-refactor size (verify no accidental duplication)

### 1.6.5 — Test Harness Setup
- Basic smoke tests: load app, add tier, add item, drag to tier, undo, save, reload
- Collab integration test: two headless instances, connect, broadcast tier state, verify sync
- Bracket test: 8-item bracket, complete all rounds, verify winner
- Tests run in CI on every commit going forward

---

## EA 1.7 — LIBRARY & CATALOGUE
*Pool population becomes powerful enough that manual image import is optional for music use cases.*

### 1.7.1 — Last.fm Integration
- Last.fm scrobble history import via API key + username input
- Pulls top albums (7-day, 1-month, 3-month, 6-month, all-time) — user selects period
- Cover art fetched from MusicBrainz CAA using MBIDs from Last.fm data
- Imported items carry Last.fm playcount as metadata, optionally shown in label

### 1.7.2 — Discogs Collection Import
- Discogs username input → fetch public collection via Discogs API
- Filter by format: LP, EP, Single, Cassette, CD (checkboxes)
- Filter by rating (only import 4★+ items etc.)
- Cover art from Discogs thumbnail with MusicBrainz CAA fallback

### 1.7.3 — Spotify Playlist Import
- Paste a Spotify playlist URL → parse playlist ID → fetch via Spotify embed API (no OAuth)
- Extract track/album list from playlist metadata
- Cover art from Spotify CDN URLs (no auth required for public playlists)
- Duplicate detection: skip items already in pool by matching title+artist

### 1.7.4 — MusicBrainz Artist Browser
- Artist search → artist page showing all release groups (albums, EPs, singles, compilations)
- Release groups filterable by type and date range
- Batch add: select multiple release groups → add all to pool in one action
- Artist page breadcrumb: navigate from label → artist → release group

### 1.7.5 — Metadata Layer
- Pool items can carry structured metadata: artist, title, year, genre tags, playcount, rating
- Metadata displayed on hover as a tooltip (configurable in settings)
- Metadata sourced automatically from MusicBrainz/Last.fm/Discogs on import
- Manual metadata edit: right-click item → Edit Metadata → form with artist/title/year/tags fields

### 1.7.6 — Pool Sort by Metadata
- Sort pool by: year (asc/desc), artist name, playcount (if Last.fm sourced), user rating
- Sort persists per category
- Multi-sort: primary + secondary sort key
- Sort indicator shown in pool header

### 1.7.7 — Duplicate Detection
- On import (any source), check pool for items with same title+artist — flag as potential duplicate
- Duplicate resolution modal: keep both, keep existing, replace with new, merge metadata
- Option to deduplicate entire pool on demand: scan all items, surface duplicates grouped

### 1.7.8 — Library Bulk Operations
- Multi-select in library search results (Shift+click, Ctrl+click)
- Bulk add to pool from multi-select
- Bulk add to specific category
- Bulk remove from library (if library has saved items — a saved favourites list)

### 1.7.9 — Catalogue Templates
- Bundled lists: RYM Top 1000, Rolling Stone 500, Mercury Prize nominees, Pitchfork AOTY by year
- Templates stored as lightweight JSON (artist+title+year, no images — images fetched on add)
- Template browser: search, filter by era/genre, preview item count
- Community template import: paste a URL pointing to a valid template JSON

---

## EA 1.8 — COLLAB MATURITY
*Real-time collaboration becomes reliable and featureful enough for serious use.*

### 1.8.1 — Collab Chat Persistence
- Chat history persists for session duration in memory (cleared on room leave)
- Chat scroll position preserved when switching tabs and returning to collab panel
- Chat timestamps per message (HH:MM)
- Chat message character limit enforced client-side (256 chars)

### 1.8.2 — Conflict Resolution
- If two peers move the same item simultaneously (detected by matching item alt within a 500ms window), show a resolution modal: "Both you and [peer] moved [item] — keep your position or theirs?"
- Non-conflicting simultaneous moves apply silently as before
- Conflict log in collab panel (collapsible): lists resolved conflicts with outcomes

### 1.8.3 — Per-User Colour Coding
- Each peer assigned a distinct accent colour on join
- Items moved by a peer briefly flash that peer's colour (2s)
- Peer colour shown next to their name in collab panel and chat
- Host colour is always the current theme accent; peers get contrasting secondaries

### 1.8.4 — Session Replay
- Full event log recorded during collab session: item moves, tier changes, chat, timestamps
- Post-session "Replay" button: watch back the sequence as a timed animation
- Replay speed controls: 0.5x, 1x, 2x, 4x
- Replay exportable as a JSON event log

### 1.8.5 — Collab Permissions
- Host can set room to read-only for all peers (spectator-only mode)
- Host can grant/revoke edit permission per peer
- Permission changes reflected immediately — in-flight drags cancelled gracefully if permission revoked
- Permission state visible in collab panel for all participants

### 1.8.6 — Collab State Versioning
- Each broadcast carries a monotonic sequence number per sender
- Receiver buffers out-of-order messages and applies in sequence
- Duplicate message detection: ignore replayed messages with seen sequence numbers
- Sequence gap detection: request re-sync if gap exceeds threshold (e.g. 5 missed messages)

### 1.8.7 — Aux Battle Polish
- Aux Battle pick phase: album art previewed larger on hover (not just thumbnail)
- Pick timer now configurable per session (30s, 60s, 90s, unlimited)
- Pick history panel: see all past picks in current session chronologically
- Post-session summary: all picks displayed in a shareable layout

---

## EA 1.9 — AUDIO
*Music previews within the app, where licensing and API availability allow.*

### 1.9.1 — MusicBrainz Preview Integration
- 30-second preview on pool item hover via MusicBrainz recording links where available
- Preview plays only on explicit click of a play button (not autoplay on hover)
- One preview active at a time — starting a new preview pauses the current
- Preview progress bar and stop button rendered inside item tile on active playback

### 1.9.2 — Matchup Preview
- Bracket matchup: optional audio preview for each side before picking
- Preview plays sequentially: left side plays 15s → right side plays 15s → pick prompt
- Skip button to bypass preview and go straight to pick
- Preview mode toggleable in bracket settings

### 1.9.3 — Pool Audio Sort
- If items have preview URLs (from MB or other source), sort pool by: BPM (if available from metadata), year, duration
- BPM pulled from AcousticBrainz data where available (MB integration)
- Duration pulled from MB recording data
- Audio-sourced metadata displayed in hover tooltip alongside other metadata

### 1.9.4 — Ambient Theme Audio
- Optional ambient audio per FX theme (off by default, opt-in in settings)
- Rain theme: light rain/thunder ambience
- Embers theme: soft crackling fire
- Stars theme: subtle synth pad
- Volume control independent of system volume, persists in settings

---

## EA 1.10 — PERSONALIZATION
*Aesthetic and layout customisation goes deep.*

### 1.10.1 — Custom Theme Builder
- Theme editor modal: full colour picker for every CSS variable (background, panel, accent, accent2, text, border, etc.)
- Live preview updates the whole app as you adjust
- Save as named custom theme
- Custom themes stored in localStorage, included in app JSON export

### 1.10.2 — Theme Import/Export
- Export single theme as JSON
- Import theme JSON — validate all required variables present before applying
- Share theme as URL (same base64 encoding as tier list share)
- Community theme gallery: paste a URL pointing to a valid theme JSON

### 1.10.3 — Custom FX Editor
- FX editor panel: particle type (rain, embers, stars, vinyl, custom), spawn rate, speed, size, colour, opacity
- Custom FX saved as named presets
- FX preview in a small canvas within the editor panel
- FX presets exportable/importable as JSON

### 1.10.4 — Per-Tier Font Override
- Right-click tier label → Label Settings → Font picker for that row only
- Per-tier font size override
- Per-tier label text alignment (left, center, right)
- Per-tier overrides stored in tier data and persist through save/load

### 1.10.5 — Item Label Customisation
- Item label display mode: filename (default), custom string, artist only, title only, year, none
- Custom label editable per item via right-click → Edit Label
- Global label display mode toggle in settings affects all items without custom overrides
- Label font size slider independent of tier label size

### 1.10.6 — Layout Options
- Pool position: right sidebar (current) or bottom panel (toggle)
- Compact mode: reduce tier row height for dense lists
- Pool item size: separate slider from tier tile size (currently both use `--tile`)
- Tier list width: constrained (current) or full-width toggle

---

## EA 1.11 — PLATFORM
*Distribution and offline capability.*

### 1.11.1 — PWA Manifest
- `manifest.json`: name, icons (192, 512), theme_color, background_color, display: standalone
- Service worker: cache-first for the app shell, network-first for MusicBrainz/Last.fm API calls
- Offline mode: full tier list + bracket + grid usable without network (only external API calls fail gracefully)
- Install prompt handled correctly on Chrome/Android and Safari/iOS

### 1.11.2 — PWA Update Flow
- Service worker update detection: when new version deployed, show unobtrusive "Update available" banner
- One-click update + reload
- Changelog shown in update banner (first 3 entries from current patch notes)
- Update flow does not clear localStorage or disrupt in-progress sessions

### 1.11.3 — Desktop Wrapper (Tauri)
- Tauri app wrapping the single-file HTML build
- File system access via Tauri API: open local image files directly without drag-drop
- Native window title updates to current "session name" if set
- Auto-updater integrated with Tauri's built-in update mechanism
- Windows, macOS, Linux builds produced from CI

### 1.11.4 — Full App JSON Export/Import
- Export entire app state: all tier lists (main + all save slots), pool, grid, bracket history, custom themes, custom FX presets, settings
- Single JSON file, versioned schema (`"schemaVersion": "1.10"`)
- Import with conflict resolution: merge vs replace options per section
- Import validates schema version and warns on mismatch

### 1.11.5 — iOS & Android Home Screen
- iOS: correct splash screens for all device sizes via `apple-touch-startup-image` meta tags
- iOS: status bar styling matches current theme colour
- Android: TWA (Trusted Web Activity) manifest entries for proper home screen integration
- Both: app name shows as "Musetier" not URL on home screen

---

## EA 1.12 — POLISH & PERFORMANCE
*Before RC: profile, optimise, and remove rough edges accumulated across 1.1–1.10.*

### 1.12.1 — Memory Profiling
- Profile a large session: 250+ items, 10 tiers, 3-hour collab
- Identify and fix any blob URL leaks surviving edge-case remove paths
- BitmapCache eviction: cap at 500 entries, evict LRU when exceeded
- Object URL registry audit: verify every `createObjectURL` has a corresponding `revokeObjectURL` path

### 1.12.2 — Render Performance
- Pool render: virtualise display when item count exceeds 200 (only render visible items)
- Tier zone render: same virtualisation at 100+ items per tier
- Bracket render: large brackets (128-item) profiled and node creation batched
- Grid render: cell redraw batched with `requestAnimationFrame` — no layout thrash on bulk fill

### 1.12.3 — Bundle Size Audit
- Audit all loaded Google Fonts: remove any font families with zero usage in any theme
- CSS audit: remove dead rules accumulated from feature additions across 1.x
- Inline SVG audit: deduplicate repeated SVG strings (mask-image SVGs appear multiple times)
- Target: final bundle under 15,000 lines / 600KB uncompressed

### 1.12.4 — Cross-Browser Matrix
- Chrome/Chromium (latest): full regression pass
- Firefox (latest): drag-and-drop, WebRTC, CSS grid, CSS custom properties
- Safari 16+ (macOS): all of the above, plus touch handling
- Safari iOS 16+: full mobile feature set
- Known Firefox WebRTC quirks with Trystero/MQTT documented and worked around

### 1.12.5 — Error Boundary Pass
- Every `localStorage` read/write wrapped and recovers gracefully on `QuotaExceededError`
- Every MusicBrainz/Last.fm/Discogs/Spotify API call handles: network error, 429, 500, malformed JSON
- Every `createImageBitmap` call has a fallback for unsupported formats
- All `catch(_) {}` silent swallows replaced with at minimum a `console.warn` (audited)

### 1.12.6 — Accessibility Pass 2
- Full keyboard-navigable tier list: move items between tiers without mouse
- Screen reader pass: `role`, `aria-live` on toast region, `aria-label` on all interactive elements
- Focus visible outlines on all interactive elements (not suppressed by `outline: none` without replacement)
- High contrast mode: verify app is usable with forced-colors / high contrast OS mode

---

## RC 1.0 — RELEASE CANDIDATES
*Feature freeze. Only bug fixes and polish.*

### RC 1.0.1 — First RC
- Feature freeze declared
- Full regression test pass across all features: tier, bracket, grid, library, collab, export, settings
- All known P1 (crash/data loss) bugs fixed
- EA badge removed from version string
- Version badge updated to RC 1.0

### RC 1.0.2 — Bug Bash
- Community bug bash period (if applicable)
- All bugs filed against RC 1.0.1 triaged: P1 fixed, P2 scheduled for 1.0.x, P3 for 1.1
- Collab stress test: 3-person session, 100+ items, sustained 30 minutes
- Bracket stress test: 128-item double-elimination, full completion

### RC 1.0.3 — Documentation
- In-app tour updated for all features added since 1.1
- Support modal copy updated: accurate feature list, correct keyboard shortcuts
- What's New modal converted to full changelog (all EA versions summarised)
- README.md and/or documentation site updated

### RC 1.0.4 — Final Sign-Off
- Changelog for 1.0 written (human-readable, not just a diff)
- Schema version locked at `1.0`
- All TODO/FIXME comments in source resolved or converted to tracked issues
- Version badge set to `1.0`

---

## RELEASE 1.0

- EA designation removed everywhere
- `1.0` in version badge, page title, manifest
- GitHub release tagged
- Stable branch cut
- 1.1 development branch opened

---

*Total estimated patches: ~86 discrete releases across the EA track.*
*Refactor breakpoint: EA 1.6 (after 1.5 ships, before 1.6 feature development begins).*
*Recommended MVP for public visibility: EA 1.4 (bracket history + sharing basics in place).*
