# Libros v2.0 — AI Book Recommender

**Status:** Brainstorming in progress (sections being reviewed one at a time)
**Started:** 2026-05-13
**Spec author:** Tomek + Claude
**Target file:** Inline in `index.html` (consistent with existing single-file architecture)

---

## Goal

Give Libros a way to recommend what to read next, using your full reading history (357 books across 2019–2026) as context. Two surfaces collapse into one: a chat where you can describe what you're in the mood for, with a quick-start path that gives suggestions immediately. Picks you like move to the wishlist with one click.

---

## Locked-in decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | **One chat surface with a quick-start button** | Less code than two separate surfaces. Quick-start is just a 1-click entry into the same chat, pre-prompted with "based on my library, what should I read next?". |
| 2 | **Anthropic Claude API**, key in `localStorage` as `libros_ai_key` | Same client-side key model as the existing GitHub PAT. Direct browser calls via Anthropic's `anthropic-dangerous-direct-browser-access: true` header. |
| 3 | **3 picks per recommendation**, each with author, title, and 1–2 sentence "why" | Enough variety to compare, not overwhelming. User can ask for more in a follow-up. |
| 4 | **Recommend immediately, offer to refine** | AI gives 3 picks on first turn; after each turn it offers refinement ("want me to lean lighter/heavier, fiction/non-fiction, PL/EN?"). Vague prompt → broad picks + one focused question. |
| 5 | **Fresh chat per session, no history** | Simplest, no extra storage, no sync. Library is always the AI's context — personalization persists across sessions, the back-and-forth doesn't. |
| 6 | **Floating action button entry point** | Glass orb bottom-right, mobile-native pattern, always within thumb reach on PWA. |
| 7 | **Claude Sonnet 4.6, streamed responses** | Best reasoning for nuanced book recs. Streaming makes the UI feel live. |
| 8 | **Tool-use structured output + full library + prompt caching** | Sonnet emits conversational text first, then a structured `recommend_books` tool call with `[{author, title, reasoning}]`. Schema-enforced output, no brittle JSON parsing. Prompt caching keeps cost negligible. |

---

## Cost model

| Scenario | Per query |
|---|---|
| First call in session (cache miss) | ~$0.02 |
| Cached calls (same library, same session) | ~$0.006 |
| Library mutated between calls (cache invalidate) | ~$0.02 (rebuilds cache) |
| 50 chats / month (typical) | ≈ $1 |

**Library context:** ~3,900 tokens for the 357-book list + year labels + JSON overhead. Plus ~500 tokens system prompt. Total input per turn ~4,500–5,000 tokens.

---

## Fluid context guarantee

The AI module reads `books` and `wishlist` from the same in-memory variables that the rest of the app mutates. The system prompt is **rebuilt on every turn** from current state — no snapshots, no caches at the app level. Add a book → open chat → Claude sees it immediately. Move a pick to wishlist → next turn it won't be re-suggested.

Prompt caching (Anthropic-side) is a cost optimization only; it invalidates automatically when the library prefix changes. From the user's perspective: the AI's knowledge is always current.

---

## Architecture overview

One new self-contained module inside `index.html` (consistent with existing single-file style). No new files, no build step. Six logical units:

```
┌─────────────────────────────────────────────────────────────┐
│  index.html (existing)                                       │
│                                                              │
│  ┌─────────────────┐    ┌──────────────────────────────┐   │
│  │  FAB button     │───▶│  Chat modal (existing modal  │   │
│  │  (always vis.)  │    │  pattern, like wish-modal)   │   │
│  └─────────────────┘    │                              │   │
│                         │  - Message list (bubbles)    │   │
│                         │  - 3 pick cards (when tool   │   │
│                         │    block arrives)            │   │
│                         │  - Input + send + quick-     │   │
│                         │    start chip                │   │
│                         └──────────────┬───────────────┘   │
│                                        │                    │
│  ┌────────────────────────┐  ┌─────────▼──────────────┐   │
│  │ AI key setup prompt    │  │ AI logic (inline JS):  │   │
│  │ → localStorage         │  │  - buildSystemPrompt() │   │
│  │ (libros_ai_key)        │  │  - streamRecommend()   │   │
│  └────────────────────────┘  │  - parseToolUse()      │   │
│                              │  - addPickToWishlist() │   │
│                              └────────┬───────────────┘   │
│                                       │                    │
└───────────────────────────────────────┼────────────────────┘
                                        │ HTTPS
                                        ▼
                       api.anthropic.com/v1/messages
                       (stream=true, tools=[recommend_books])
```

**Key principle:** the "Add to wishlist" action on a pick card calls the existing `saveWishlist()`, which already triggers auto-push to GitHub. **Zero new sync logic.**

---

## UI components

### Floating action button (FAB)

- Glass orb, ~44–48px circular, fixed bottom-right (24px from edges)
- Linear gradient (`#c4b5fd → #7b6bbd`) with inset highlight and soft outer shadow
- Sparkle (✨) icon centered
- Always visible, stays above book list while scrolling
- Tap → opens chat modal
- Hidden when the chat modal is open (no double-affordance)

### Chat modal

Reuses the existing `wish-modal` overlay pattern: fade-in overlay, centered glass card, click-outside-to-close, ESC-to-close. Dark theme matches the rest of the app.

**Structure (top to bottom):**

1. **Header bar** — small ✨ sparkle icon + "Ask the librarian" title on the left; close (×) button on the right; thin border-bottom separator.
2. **Message stream** — vertical column, scrollable:
   - **AI bubbles** — left-aligned, glass background (`rgba(255,255,255,0.07)`), 1px inner border, blur+saturate backdrop, 16/16/16/6 radius, inset top highlight. Hover lifts -1px and brightens.
   - **User bubbles** — right-aligned, purple gradient (`rgba(196,181,253,0.35) → rgba(123,107,189,0.35)`), inset highlight, 16/16/6/16 radius. Same hover lift.
   - **Pick cards** — flex row with a 42×60 cover thumbnail on the left and info column on the right (author label uppercase, title bold, 1–2 sentence "why", primary action button). Cards lift -2px on hover with a subtle purple shadow.
   - **Pick action button states**:
     - Default: `+ Add to wishlist` — purple-tinted glass pill
     - After click: `✓ Added` — green-tinted glass pill, non-interactive
   - **Refine-question drawer** (appears after picks, attached to the input): when the AI emits a follow-up question, it renders as a glass "drawer" that physically attaches to the top of the text input area — rounded top corners, square bottom corners that meet the input's square top. Drawer contains:
     - A small "tab handle" stub on the top edge (visual indicator that this is a layered element)
     - An uppercase purple label ("I have a follow-up") preceded by a small pulsing dot
     - The question text in a slightly larger font than chat bubbles
     - Tappable refinement chips ("Polish only", "More fiction", "Heavier")
     - Background: subtle purple gradient (`rgba(196,181,253,0.18) → rgba(123,107,189,0.12)`) with `rgba(196,181,253,0.45)` border
     - Shadow blooms upward (`box-shadow: 0 -6px 24px rgba(123,107,189,0.15)`) to lift it off the chat behind
   - **Why a drawer, not a bubble**: questions must be hardest to miss — the drawer attaches to where the answer goes (the input), uses a pulsing dot, and breaks the "everything is a bubble" pattern. Only one drawer exists at a time (it replaces on the next AI turn) — the previous question disappears once you've moved past it, which mirrors how you'd actually engage with it.
3. **Input area** — pinned bottom: rounded glass capsule with text field + circular gradient send button.
4. **Empty state** (just opened, no messages yet) — centered icon (📚) + tagline ("I've read your 357 books. Tell me what you're in the mood for…") + a single primary "✨ Recommend me something" chip that fires the quick-start prompt. The text input is still available below for typed prompts.

### Book covers — fetched from Open Library

- After the AI's tool call returns 3 picks, for each pick we async-fetch covers from Open Library (free, no API key, no quota issues — Google Books anonymous quota proved too tight during testing)
- **Two-step lookup:**
  1. Search: `GET https://openlibrary.org/search.json?q=<title>+<author>&limit=1` → returns `docs[0].cover_i` (integer cover ID)
  2. Construct URL: `https://covers.openlibrary.org/b/id/<cover_i>-M.jpg` (M = medium, ~180px wide; suitable for our 42×60 slot)
- Render the image inside the existing 42×60 cover slot
- **Fallback if no result or fetch fails:** keep the gradient-with-title-text shown by default — looks intentional, never appears broken
- Cards render immediately with the gradient; cover image swaps in when the fetch resolves (~100–400ms typical)
- One small cache (`libros_cover_cache` in localStorage, keyed by `${author}|${title}`, storing either the cover URL or a `null` sentinel for "we tried and there was no result") so repeat asks for the same book are instant on subsequent turns and we don't re-query the API for known misses

### Liquid-glass treatment (matches existing app)

Every interactive surface (bubbles, pick cards, refine chips, FAB) gets:
- `backdrop-filter: blur(20px) saturate(140%)` for real glass
- `inset 0 1px 0 rgba(255,255,255,X)` for the bright top edge
- Soft outer drop shadow (purple tint on the heavier elements)
- Hover transition: `translateY(-1px or -2px)` + background brighten + shadow bloom — matches existing book-card hover

---

## Data flow (pending — Section 3)

_To be filled after approval._

---

## API + prompt structure (pending — Section 4)

_To be filled after approval._

---

## Error handling (pending — Section 5)

_To be filled after approval._

---

## Testing (pending — Section 6)

_To be filled after approval._

---

## Open questions / followups

- None yet — will accumulate here as they come up.

---

## Brainstorming session log

- 2026-05-13 — Initial scope, all 8 decisions locked. Architecture diagram approved.
- 2026-05-14 — Section 2 (UI) v2 reviewed: liquid bubbles + cover art approved. Question-vs-pick differentiation revisited with three alternative styles (section labels / banner with avatar / drawer above input). User selected **drawer above input** (Option C). Cover source switched from Google Books to Open Library after Google's anonymous quota proved unreliable during testing. Section 2 now locked.
