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
**Comment:** Please add a test for the case where film_id doesn't exist in the database. Look at the existing tests in test_collection.py — the pattern is there.
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**Comment:** I notice watchlists default to public=True. We don't have a documented decision on default visibility for user lists. Before I can approve this, I need you to add a note to your PR description explaining your reasoning. I want to make sure we're being intentional here, not just inheriting a default.
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**Comment:** I'd prefer watchlists to default to "date added" order rather than alphabetical. Most users want to see what they added recently. I'm open to discussion if you see it differently — but let's make a decision and document it.
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**Comment:** A refactor merged to main that changed film IDs from integers to UUIDs. Your watchlist code still references integer IDs. Please rebase on main and update accordingly.
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->