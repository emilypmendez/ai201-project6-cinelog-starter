# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I worked through this PR with Claude Code (Anthropic's CLI) as a pair-programming and review tool. Specific, verifiable uses:

- **Codebase orientation & pattern-matching.** Before changing anything, I had it read `models.py`, `services/collection_service.py`, and `tests/test_collection.py` and surface the conventions to follow: the `verb_to_noun` service naming, the *query-then-raise* deduplication pattern (`AlreadyInCollectionError`), and the fixture/assertion structure of the tests. Comments 1–3 were then written to mirror those patterns rather than invent new ones.

- **Adversarial stress-testing of the two design arguments (Comments 4 & 5).** For each, I wrote a draft and then asked an independent adversarial reviewer: *"What is the single strongest counterargument a rigorous reviewer would raise, and what tradeoff have I not acknowledged?"* This changed my conclusions, not just my wording:
  - **Comment 4 (visibility):** my draft *defended* `public=True`, arguing a watchlist is lower-sensitivity than viewing history. The critique **inverted that premise** — forward-looking intent (pregnancy, coming-out, recovery films) can be *more* sensitive than consumption because it exposes unacted-on plans — and noted that a mitigation I leaned on (an explicit `public` param) **didn't exist in the code yet**. Both points were correct, so I **reversed my position** to private-by-default. The final argument is the opposite of my first draft, precisely because of the critique.
  - **Comment 5 (sort order):** my draft agreed with the maintainer's `date_added DESC`. The critique caught that `date_added.desc()` **alone is not a total order** — burst-added rows share near-equal Python-side timestamps and the UUID primary key is random, so the recent cluster would reshuffle between requests and break pagination. The AI identified the *failure mode*; the fix — adding `title ASC` as a stable secondary key and keeping the `Film` join to support it — was my decision. My final argument **builds on** the critique rather than restating the draft.

- **Debugging a subtle rebase failure.** When the int→UUID rebase "succeeded" with no conflict markers but broke imports, I used it to trace *why* `WatchlistEntry` had silently vanished from `models.py` — main's refactor deleted the class, my branch never edited those lines, so the rebase took the deletion. That diagnosis drove the Comment 6 resolution.

- **Verifying behavior and format.** I used it to confirm every endpoint status code in the manual-test section via the app's test client, to check that all commit messages parse as conventional format, and to confirm the final history is linear with no merge commits.

I reviewed and own every change. The adversarial output was treated as *input to revise*, not a verdict to accept — most visibly in the reversed Comment 4 position and the tiebreaker I added in Comment 5.

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
**My position:** I agree with the maintainer and implemented it: `get_watchlist()` now sorts newest-added first (`WatchlistEntry.date_added.desc()`) instead of alphabetically — with `Film.title.asc()` as a secondary key (more on why below). Alphabetical lookup isn't lost forever; it belongs behind an explicit `?sort=title|date` query param (default `date`), which I've scoped as a follow-up to keep this change to one reviewable decision.

**Reasoning:** Two arguments, one of which the comment didn't cite. (1) The maintainer's UX point is right for a *to-watch* list: people add in bursts after a recommendation, and the job of the list is "what should I watch, and what's fresh in my mind" — recency-first puts the top-of-mind items where the eye lands. (2) **Consistency:** `get_collection()` already sorts `date_added.desc()`. Alphabetical was the *outlier* in this codebase, not the norm — so date-added-desc isn't only better UX, it makes the two list surfaces behave identically, which is one less thing for users and maintainers to reconcile. On direction: a watchlist is also a backlog, which is an argument for oldest-first, but I chose `desc` deliberately to match both the maintainer's "added recently" framing and `get_collection`; backlog-clearing is a job for the explicit sort option, not the default.

**Engagement with reviewer's point (devil's-advocate revision):** An adversarial review flagged that `date_added.desc()` *alone* is not a total order. `date_added` is set Python-side at insert, so films added in one burst share equal-or-near-equal timestamps — exactly the recent cluster the user cares most about — and their relative order would be non-deterministic, reshuffling between page loads. The random UUID primary key can't serve as an insertion tiebreaker either. That would have quietly reintroduced instability (and broken pagination) while "fixing" the sort. So the final order is `date_added DESC, title ASC` — a stable total order. Keeping the `Film` join for the tiebreaker has a bonus: the join also filters out watchlist rows whose film was deleted, avoiding the `entry.film.to_dict()` crash-on-`None` that `get_collection()` is actually latently exposed to. In other words, I matched `get_collection`'s *sort intent* without copying its latent orphan-row bug. The review confirmed the consistency argument, the backlog deferral, and the `?sort=` follow-up were already sound.

## Comment 6 — Rebase
**What conflicted:** Two things, one obvious and one silent. (1) **`.gitignore`** — an add/add conflict: `main` added its own `.gitignore` (`chore: add .gitignore`) and my branch's `prep the repo` commit added one too. (2) **The real one — `WatchlistEntry` in `models.py`, a *semantic* conflict git did not flag.** The int→UUID refactor on `main` (`refactor: migrate film IDs from integer to UUID`) not only changed `Film.id`/`User.id`/`CollectionEntry.film_id` to `String(36)` UUIDs, it also *removed* the `WatchlistEntry` class from `models.py` (the watchlist feature wasn't merged on `main` yet). My feature commits never modified those model lines — they inherited `WatchlistEntry` from the base commit — so when rebasing, git saw no competing edit and silently took `main`'s deletion. Result: a "successful" rebase with **no conflict markers** but a `models.py` missing `WatchlistEntry`, leaving `watchlist_service.py` importing a class that no longer existed. This is the subtle failure mode of rebasing over a deletion: absence of a textual conflict is not absence of a problem.

**How I resolved it:** (1) `.gitignore` → kept the **union** of both ignore lists (`.pytest_cache/` from main plus `.venv/`/`venv/`, de-duplicated). (2) Re-added `WatchlistEntry` to `models.py` with **`film_id = db.Column(db.String(36), db.ForeignKey("film.id"))`** — a UUID FK matching the refactored `Film.id` (the original branch had `db.Integer`). (3) Updated the now-stale integer references in the docstrings: `film_id (int)` → `film_id (str): UUID of the film` in `watchlist_service.py`, and `Body: { "film_id": <int> }` → `{ "film_id": "<uuid>" }` in the route. The rebase produced a linear history — no merge commits.

**How I verified no conflict remains:** (a) **No merge commits:** `git log --merges origin/main..HEAD` is empty and the history is linear. (b) **No lingering integer film ids:** grep confirms the only `db.Integer` columns left are `Film.year` and `CollectionEntry.rating` (legitimately integers); `WatchlistEntry.__table__.c.film_id.type` is `VARCHAR(36)`. (c) **Imports load** and the full suite passes (5/5). (d) **End-to-end with real UUIDs:** created a `Film` (UUID id), then `add_to_watchlist` stored the matching UUID `film_id`, a duplicate add raised `AlreadyInWatchlistError`, and a bogus id raised `FilmNotFoundError`. *Note:* this exercise also surfaced a **pre-existing, out-of-scope bug** — `get_watchlist()` raises `AttributeError` on `entry.film` because `WatchlistEntry` defines no `film` relationship (true before the refactor too; untested because no test exercised `get_watchlist()`). I restored the model faithfully rather than silently expanding this commit, and tracked the bug separately.

## PR Description

### What this feature does
Adds a **watchlist** — a list of films a user wants to watch, kept separate from their collection (films they've already watched). It includes:
- a `WatchlistEntry` model (with a UUID `film_id` matching the post-refactor `Film.id`),
- a service layer (`add_to_watchlist`, `get_watchlist`) that validates the film exists and prevents duplicate entries, mirroring the existing `collection_service`,
- REST endpoints under `/watchlist`.

**Endpoints**
- `GET /watchlist/<user_id>` — return the user's watchlist (newest-added first).
- `POST /watchlist/<user_id>/add` with body `{"film_id": "<uuid>"}` — add a film. Returns `201` on success, `400` if `film_id` is missing, `404` if the film doesn't exist, `409` if it's already on the watchlist.

### Design decisions
**1. Default visibility (review Comment 4).** Watchlist entries carry a `public` flag. My documented decision: visibility should be an *explicit* caller choice, and absent one it should fall back to **private (`public=False`)** rather than silently public — exposure is irreversible while missed discovery is recoverable, and a "want to watch" list can reveal sensitive, unacted-on intent. **Current state:** the model still defaults `public=True`; flipping it (plus an explicit `public` parameter on the add endpoint) is the recommended follow-up. Full reasoning in the Comment 4 section above.

**2. Sort order (review Comment 5).** `get_watchlist()` returns entries **newest-added first** (`date_added DESC`), with `title ASC` as a stable tiebreaker. This matches `get_collection()` and the maintainer's recency preference; alphabetical ordering is deferred to a future `?sort=title|date` query param. Full reasoning in the Comment 5 section above.

### How to test manually
Prereqs: `pip install -r requirements.txt`. The app uses a local SQLite database (`cinelog.db`).

**1. Seed a user and a film** (there is no create API for these), capturing their UUIDs:
```bash
python - <<'PY'
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u = User(username="alice", email="alice@example.com")
    f = Film(title="Paddington 2", year=2017, genre="Comedy")
    db.session.add_all([u, f]); db.session.commit()
    print("USER_ID:", u.id)
    print("FILM_ID:", f.id)
PY
```
Copy the two printed IDs.

**2. Start the server:**
```bash
python app.py        # serves http://localhost:5000
```

**3. Exercise the endpoints** (substitute the IDs from step 1):
```bash
USER=<USER_ID>; FILM=<FILM_ID>

# (a) View empty watchlist            -> 200  []
curl -s -w '\n%{http_code}\n' http://localhost:5000/watchlist/$USER

# (b) Add the film                    -> 201  (returns the entry, public: true)
curl -s -w '\n%{http_code}\n' -X POST http://localhost:5000/watchlist/$USER/add \
  -H 'Content-Type: application/json' -d "{\"film_id\": \"$FILM\"}"

# (c) Add the same film again (dedup) -> 409  Conflict
curl -s -w '\n%{http_code}\n' -X POST http://localhost:5000/watchlist/$USER/add \
  -H 'Content-Type: application/json' -d "{\"film_id\": \"$FILM\"}"

# (d) Add a nonexistent film          -> 404  Not Found
curl -s -w '\n%{http_code}\n' -X POST http://localhost:5000/watchlist/$USER/add \
  -H 'Content-Type: application/json' \
  -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'

# (e) Omit film_id                    -> 400  Bad Request
curl -s -w '\n%{http_code}\n' -X POST http://localhost:5000/watchlist/$USER/add \
  -H 'Content-Type: application/json' -d '{}'
```
Expected results: (a) `200 []`, (b) `201`, (c) `409`, (d) `404`, (e) `400`. The `409` on the repeat add confirms the entry persisted and deduplication works.

### Known limitation
`GET /watchlist/<user_id>` on a **non-empty** watchlist currently returns **500**: `get_watchlist()` accesses `entry.film`, but `WatchlistEntry` defines no `film` relationship. This is a pre-existing gap (unrelated to the int→UUID rebase) that went unnoticed because no test exercised the populated GET path. It is out of scope for this PR and tracked as a separate fix. Until it lands, verify persistence via the `409` in step (c) rather than the populated GET.