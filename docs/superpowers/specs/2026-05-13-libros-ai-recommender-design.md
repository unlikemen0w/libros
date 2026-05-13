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

## UI components (mockup pending — Section 2)

_To be filled after browser mockup review._

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

- 2026-05-13 — Initial scope, all 8 decisions locked. Architecture diagram approved. Continuing with UI/data-flow sections.
