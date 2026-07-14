# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention (compare `add_to_collection()`). Updated both call sites in `routes/watchlist/watchlist.py` — the import on line 8 and the call on line 32. Also updated the docstring's leading verb from "Save" to "Add" so the documentation stays consistent with the new name.

**Why:** Consistency in the service API surface matters more than any individual name. `collection_service` already establishes `add_to_collection()` / `remove_from_collection()`, so `add_to_watchlist()` lets a reader predict the watchlist API from the collection API. The old `save_` prefix was the only outlier.

**How I verified:** Project-wide `grep -rn "save_to_watchlist"` returns no remaining references; the only occurrences of `add_to_watchlist` are the definition and its two call sites. Full test suite still passes (4/4).

## Comment 2 — Deduplication
**What I did:** Added deduplication to `add_to_watchlist()` mirroring the collection service exactly. Introduced an `AlreadyInWatchlistError` exception in `watchlist_service.py` (parallel to `AlreadyInCollectionError` in `collection_service.py`). After the film-exists check, the function now queries for an existing `WatchlistEntry` with the same `(user_id, film_id)` and raises `AlreadyInWatchlistError` instead of inserting a second row. I also updated the watchlist route to translate that exception into a `409 Conflict` and `FilmNotFoundError` into a `404`, matching the collection route's error handling (the watchlist route previously had none, so both errors would have surfaced as a 500).

**Why:** I deliberately reused the collection service's approach — a service-layer query-then-raise — rather than relying solely on a database `UniqueConstraint`. Two reasons: (1) it produces a clean, typed domain error the route can map to a meaningful HTTP status, instead of an opaque `IntegrityError`; (2) it keeps the watchlist and collection services symmetric, so the dedup behavior is discoverable to anyone who already knows the collection code. Note: unlike `CollectionEntry`, the `WatchlistEntry` model does not currently carry a `UniqueConstraint`, so this service check is the sole guard. A DB-level constraint would be a defensible belt-and-suspenders follow-up, but it belongs in the model refactor rather than this PR.

**How I verified:** Imports load cleanly; existing suite still passes (4/4). A dedicated duplicate-add test is added under Comment 3 (see `tests/test_watchlist.py`) which asserts the second add raises `AlreadyInWatchlistError` and that only one row exists.

## Comment 3 — Missing test
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->