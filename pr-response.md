# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** 
  - I changed all occurrences of `save_to_watchlist` to `add_to_watchlist` in `services/watchlist_service.py` and `routes/watchlist/watchlist.py`
**How I verified:** 
  - I checked my code editor's find-all-references to ensure I hadn't missed any occurrences

## Comment 2 — Deduplication
**What I did:** 
  - Followed the same pattern as `add_to_collection()` in `services/collection_service.py`. 
  - Added an `AlreadyOnWatchlistError` exception to `services/watchlist_service.py`, and in `add_to_watchlist()` added a check that queries for an existing `WatchlistEntry` with the same `user_id`/`film_id` before creating a new one, raising `AlreadyOnWatchlistError` if found. 
  - Updated `routes/watchlist/watchlist.py` to catch this exception and return a 409 response, mirroring how `routes/collection.py` handles `AlreadyInCollectionError`.
**How I verified:** 
  - Ran a manual script against an in-memory SQLite app instance (`create_app` with `SQLALCHEMY_DATABASE_URI=sqlite:///:memory:`): added a film to a user's watchlist once (succeeded), then called `add_to_watchlist()` again with the same `user_id`/`film_id` and confirmed it raised `AlreadyOnWatchlistError` instead of creating a second row. 
  - Queried `WatchlistEntry` afterward and confirmed the count was still 1.

## Comment 3 — Missing test
**What I did:**
  - Created `tests/test_watchlist.py`, following the same `app`/`sample_user`/`sample_film` fixture structure as `tests/test_collection.py`.
  - Added `test_add_to_watchlist_nonexistent_film_raises`, mirroring `test_add_to_collection_nonexistent_film_raises`: calls `add_to_watchlist()` with a fake UUID `film_id` and asserts it raises `FilmNotFoundError`.
**How I verified:**
  - Ran `pytest tests/test_watchlist.py -v` — the test passed.

## Comment 4 — Default visibility
**My position:**
  - I'm keeping `public=True` as the default for `WatchlistEntry`.
**Reasoning:**
  - CineLog's core value is social: seeing what other people are watching and want to watch. A watchlist is one of the highest-signal, lowest-effort pieces of content a user produces — it costs them nothing to create (unlike a review or rating) but tells friends and followers a lot about their taste and what to watch next.
  - I'm optimizing for discovery and organic engagement, not for cautious data minimization. If the default were `public=False`, the vast majority of users would never find or flip a visibility toggle (this is well-documented default-effect behavior in social products), and the watchlist would end up functioning as a private to-do list for almost everyone — which defeats the reason we're building "public" as a concept at all. If we wanted a purely private feature we wouldn't need a `public` column in the first place.
  - This mirrors the model used by comparable film-logging products (e.g., Letterboxd), where activity — including watchlists — is public by default and privacy is an opt-in, deliberate action taken by users who specifically want it.
**Tradeoff acknowledged:**
  - The real cost here is unintentional exposure: a user may add a film to their watchlist without realizing it's visible to others, and a watchlist can reveal things about a person's tastes (or things they're curious about but haven't publicly "endorsed" the way a finished/rated `CollectionEntry` implies) before they've decided whether they're comfortable sharing that.
  - I'm not treating this as zero-risk — it's a real privacy tradeoff, just one I think is worth taking for this specific field. Note that `CollectionEntry` (already-watched films) has no `public` flag at all in the current model, so this decision is scoped to watchlists only; if we later add visibility to collections, that's a separate, similarly intentional decision and shouldn't inherit this default automatically.
  - To reduce the risk of "accidental" public sharing without walking back the default, I'd suggest a fast follow-up (separate PR): a one-time UI callout when a user adds their first watchlist item, telling them it's public and linking to the toggle. That keeps the discovery-first default while giving users an explicit, low-friction moment to opt out.

## Comment 5 — Sort order
**My position:**
  - Agreed — I switched `get_watchlist()` to sort by `date_added` descending (newest first), matching how `get_collection()` already sorts. Updated `services/watchlist_service.py` accordingly.
**Reasoning:**
  - I went in with a mild preference for alphabetical (it makes it easy to scan for a specific title and check "did I already add this?"), but that use case is secondary to the primary one: someone opening their watchlist is almost always asking "what did I want to watch?", and the freshest additions are the most likely answer — they reflect a recommendation you just got or a trailer you just saw, i.e. current intent. Alphabetical order buries that signal under whatever title happens to start with "A".
  - There's also a consistency argument that tipped this from "reasonable either way" to "clearly correct": `get_collection()` already sorts by `date_added.desc()`. Two list-viewing endpoints in the same app with different, unstated sort conventions is the kind of inconsistency that's confusing in the UI and easy to regress later. Matching the existing pattern removes a decision another engineer would otherwise have to re-litigate.
**Engagement with reviewer's point:**
  - Your framing ("most users want to see what they added recently") is exactly the case that changed my mind — I was thinking about the watchlist as a reference list to check against, but it's really more like a queue, and queues are naturally ordered by recency. If we ever hear from users that they want to jump to a specific title in a long watchlist, I'd treat that as a search/filter problem rather than a reason to change the default sort — that keeps "recently added first" as the one clear default while still solving the lookup case you'd otherwise get from alphabetical order.
  - Implemented and verified: added `test_get_watchlist_returns_newest_first` to `tests/test_watchlist.py` (same pattern as `test_get_collection_returns_newest_first`), and ran the full suite (`pytest tests/ -v`) — all 6 tests pass. Along the way, this test caught a pre-existing bug: `Film` had no `watchlist_entries` relationship/backref, so `entry.film` in `get_watchlist()` would have raised `AttributeError` for anyone who actually called it. Added the missing relationship in `models.py` to fix it.

## Comment 6 — Rebase
**What conflicted:**
  - Ran `git fetch origin` then `git rebase origin/main` on `feature/watchlist`. There was one textual conflict: `.gitignore` was "both added" — main's version (from the `chore/add-gitignore` merge) additionally ignores `.pytest_cache/`, which my branch's version didn't have. Resolved by keeping main's version (the superset) and staging it.
  - The real issue wasn't a textual conflict at all, so git didn't flag it: none of my feature-branch commits ever touched `models.py`, so during the rebase it was simply inherited from main untouched. But main's `07ca580 refactor: migrate film IDs from integer to UUID` commit had rewritten `models.py` from scratch and, since `WatchlistEntry` didn't exist on main yet at that point, it dropped the `WatchlistEntry` class entirely along with the int→UUID column changes. After rebasing, `from models import Film, WatchlistEntry` in `services/watchlist_service.py` started raising `ImportError: cannot import name 'WatchlistEntry'`.
**How I resolved it:**
  - Re-added the `WatchlistEntry` model to `models.py`, matching the post-refactor `CollectionEntry` pattern: `id` as a UUID string primary key, and `film_id` as `db.String(36)` with a `ForeignKey("film.id")` instead of the old `db.Integer` column.
  - Updated the leftover integer-ID references in comments/docstrings: `services/watchlist_service.py`'s `add_to_watchlist()` docstring (`film_id (int): ... pre-refactor` → `film_id (str): UUID of the film.`) and `routes/watchlist/watchlist.py`'s request-body example (`{ "film_id": <int> }` → `{ "film_id": "<uuid>" }`).
  - No other code needed changes — `add_to_watchlist()`, `get_watchlist()`, and the dedup/sort logic from Comments 2 and 5 all operate on `film_id` as an opaque value and didn't assume it was an integer.
**How I verified no conflict remains:**
  - `git log --oneline --merges main..HEAD` returns nothing, confirming the rebase produced a clean linear history with no merge commits.
  - Ran `pytest tests/ -v` — all 6 tests pass, including `test_add_to_watchlist_nonexistent_film_raises` and `test_get_watchlist_returns_newest_first`, both of which exercise `WatchlistEntry` end-to-end with real UUID values generated by the model.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->