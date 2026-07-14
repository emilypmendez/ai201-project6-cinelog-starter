# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention (compare `add_to_collection()`). Updated both call sites in `routes/watchlist/watchlist.py` — the import on line 8 and the call on line 32. Also updated the docstring's leading verb from "Save" to "Add" so the documentation stays consistent with the new name.

**Why:** Consistency in the service API surface matters more than any individual name. `collection_service` already establishes `add_to_collection()` / `remove_from_collection()`, so `add_to_watchlist()` lets a reader predict the watchlist API from the collection API. The old `save_` prefix was the only outlier.

**How I verified:** Project-wide `grep -rn "save_to_watchlist"` returns no remaining references; the only occurrences of `add_to_watchlist` are the definition and its two call sites. Full test suite still passes (4/4).

## Comment 2 — Deduplication
**What I did:**
**How I verified:**

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