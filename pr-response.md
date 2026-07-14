# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**
- Renamed the service function `save_to_watchlist()` → `add_to_watchlist()` in `services/watchlist_service.py` (also updated its docstring summary line from "Save a film..." to "Add a film..." to match the new name). This is a rename-only change — no behavior was modified.
- Updated the one call site in `routes/watchlist/watchlist.py`: both the `from services.watchlist_service import ...` import (line 8) and the call inside `add_film()` (line 32).

**How I verified:**
- Ran a project-wide search (`grep -rn "save_to_watchlist" . --include="*.py"`) after the change — **no remaining references**.
- Confirmed the diff is rename-only (4 lines changed total: 2 in the service, 2 in the route; no logic changes).
- Ran the test suite: `python -m pytest tests/ -v` → **4 passed** (all existing `test_collection.py` tests still green).

## Comment 2 — Deduplication
**How `add_to_collection()` handles duplicates (the pattern I followed):**
- After confirming the film exists, it queries `CollectionEntry.query.filter_by(user_id=..., film_id=...).first()`. If a matching entry already exists, it raises `AlreadyInCollectionError` instead of committing a second row. The exception class is defined in the same service module.

**What I did (equivalent check added to `add_to_watchlist()`):**
- Added an `AlreadyInWatchlistError(Exception)` class to `services/watchlist_service.py` (mirroring where `collection_service.py` defines `AlreadyInCollectionError`).
- Inserted the same duplicate guard between the film-existence check and entry creation: query `WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()`; if it exists, raise `AlreadyInWatchlistError` rather than creating a second `WatchlistEntry`.
- Preserved all existing behavior: still checks the film exists, still raises `FilmNotFoundError` when it doesn't, still creates and returns a new `WatchlistEntry` when the film isn't already on the list. Updated the docstring `Raises:` section accordingly. No unrelated logic was touched.
- Note: `WatchlistEntry` has no DB-level unique constraint (unlike `CollectionEntry`), so this service-layer guard is what prevents the duplicate.

**How I verified:**
- Confirmed the structure matches `add_to_collection()` line-for-line (existence check → duplicate check → create).
- Ran the full suite: `python -m pytest tests/ -v` → **4 passed** (no regressions). A dedicated duplicate test for the watchlist is out of scope for these three comments (Comment 3 covers the missing-film case).

## Comment 3 — Missing test
**What test was added:**
- Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`.

**Collection test pattern it followed:**
- Modeled on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`. Reused the same `app` fixture (isolated in-memory SQLite app via `create_app({...})` with `db.create_all()` / `db.drop_all()`) and `sample_user` fixture, following the existing database-setup style.

**Nonexistent film ID used:**
- `999999` (a nonexistent integer id). The collection test uses a UUID string because that module is post-refactor, but on the `feature/watchlist` branch `Film.id` is still an integer (pre-UUID), so a nonexistent integer id is the semantically correct "missing film" value here. Deliberately not touching UUID handling (out of scope).

**How the test verifies the behavior:**
- Wraps the call in `pytest.raises(FilmNotFoundError)` and calls `add_to_watchlist(user_id=sample_user, film_id=999999)` against a database where no film exists — confirming the same `FilmNotFoundError` path used by the collection code (not a DB integrity error).

**Test commands that passed:**
- `python -m pytest tests/test_watchlist.py -v` → **1 passed**
- `python -m pytest tests/ -v` → **5 passed** (4 collection + 1 watchlist; no regressions)

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