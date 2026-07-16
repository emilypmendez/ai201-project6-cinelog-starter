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

**How I verified:** Imports load cleanly; existing suite still passes. Comment 3 adds the first watchlist test (`FilmNotFoundError` path). A dedicated duplicate-add test — asserting the second add raises `AlreadyInWatchlistError` and that only one row exists — is a natural next test and is tracked as a candidate for the "second test" stretch goal; it is not yet in the suite.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` and added `test_add_to_watchlist_nonexistent_film_raises`, the watchlist equivalent of `test_add_to_collection_nonexistent_film_raises`. I mirrored the existing test module's structure rather than inventing my own: the same `app` / `sample_user` / `sample_film` fixtures (in-memory SQLite, app-context teardown) and the same assertion shape — call the service with a `film_id` that isn't in the database and assert `pytest.raises(FilmNotFoundError)`.

**Why:** The reviewer pointed at the collection test as the pattern to follow, so I kept the new file a faithful parallel — a maintainer who knows `test_collection.py` can read `test_watchlist.py` with no surprises. I used the same UUID-string fake id (`00000000-0000-0000-0000-000000000000`) as the collection test; this reads as "definitely not a real id" and stays correct after the Comment 6 int→UUID rebase, so the test won't need touching then. I brought `sample_film` along too so the file is ready for the duplicate-add and happy-path tests that will follow.

**How I verified:** `pytest tests/test_watchlist.py -v` → `1 passed`. The full suite continues to pass.

## Comment 4 — Default visibility
**My position:** Don't silently inherit *any* default. The add endpoint should make visibility an explicit choice by the caller; when it's left unspecified, the fallback should be **private (`public=False`)**, not public. This is a revision — my first instinct was to defend the existing `public=True`, and a devil's-advocate review talked me out of it (see "Engagement" below). Concretely, this decision recommends flipping the model default and is implemented cleanly by the visibility-toggle work (add an explicit `public` parameter to `add_to_watchlist`, prompt for it at add-time). The current code still defaults `public=True`; this note records the decision and the change it calls for.

**Reasoning — what user behavior I'm optimizing for:** User trust and least *irreversible* harm. The two outcomes are asymmetric. If we default to private and a user wants to share, they opt in — fully recoverable, zero harm. If we default to public and a user didn't realize it, their list is exposed the moment it's created; that exposure can be scraped or cached and **cannot be taken back**. When one side of a default is reversible and the other isn't, the reversible side should be the default. For a watchlist specifically, the sensitive content isn't hypothetical: a "want to watch" list can reveal *unacted-on, undisclosed* plans (fertility/pregnancy documentaries, coming-out films, addiction-recovery or leaving-an-abuser research). CineLog is a social discovery app, but "social by default" shouldn't mean "expose the most private signal in the product by default."

**Tradeoff acknowledged (the other option — public-by-default):** Public-by-default produces a richer discovery graph out of the box. Because defaults are sticky — most users never change them — a private default means the majority of watchlists start dark, and the friend-overlap / "let's watch this together" discovery that gives CineLog its network value is muted until users opt in. That is a real product cost and the strongest argument for the option I'm rejecting. I accept it because the cost is *recoverable* (opt-in restores discovery) whereas the cost of the reverse (irreversible exposure) is not, and because we can soften the discovery hit by surfacing the visibility choice prominently at add-time rather than hiding it in settings.

**Engagement with the reviewer's point (devil's-advocate revision):** I ran my original draft — "keep `public=True`; a watchlist is lower-sensitivity than viewing history" — past an adversarial review. It inverted my load-bearing premise: forward-looking *intent* can be **more** sensitive than *consumption*, because it exposes plans a person hasn't acted on or told anyone about. It also caught that my strongest proposed mitigation (an explicit `public` param on the endpoint) **doesn't exist in the code yet** — today 100% of entries are public with no creation-time path to private — so I was defending the default with a control that hadn't shipped. Both points are correct, so I revised the position instead of rationalizing the original. The maintainer asked us to be "intentional, not just inheriting a default"; the honest intentional answer turned out to be *don't inherit a permissive default at all.*

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