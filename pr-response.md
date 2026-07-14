# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used an AI assistant (Claude Code) as a pair-programming and review tool throughout this project. Specifically:

- **Codebase orientation:** to quickly map the repo — how `create_app()` wires the `films`, `collection`, and `watchlist` blueprints, where the service layer lives, and how the models relate — before making changes.
- **Understanding the collection/watchlist pattern:** to compare `add_to_collection()` against `add_to_watchlist()` so the watchlist changes (deduplication, sort order) mirrored the existing project conventions instead of inventing new ones.
- **Stress-testing design reasoning:** to play devil's advocate on the Comment 4 (default visibility) and Comment 5 (sort order) responses — surfacing the "sticky defaults" privacy counterargument and the "alphabetical is better for lookup" counterargument — so the written positions engage real trade-offs.
- **Commit-message review:** to check the existing commits against conventional-commit format and flag messages that were non-conventional or that bundled more than one logical change (e.g., the "rename-only" commit that also contained the dedup guard).
- **Rebase diagnosis:** to reason about why rebasing onto the UUID-migrated `main` produced no textual conflict yet still broke the branch, and what the correct UUID repair was.

I manually verified every AI suggestion against the actual code and the test suite: I read the diffs, ran `python -m pytest tests/ -v` after each change, and confirmed the git history/state myself. AI did not autonomously commit code or rewrite git history; all code edits were reviewed by me, and the test results quoted in this document are from real `pytest` runs, not AI-generated output.

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
- The final value is the UUID string `"00000000-0000-0000-0000-000000000000"`, matching `test_add_to_collection_nonexistent_film_raises`. (The test was originally written with the integer `999999` because the watchlist branch pre-dated the UUID migration; it was updated to a UUID during the Comment 6 rebase — see Comment 6 — so it now follows the post-refactor `Film.id` type.)

**How the test verifies the behavior:**
- Wraps the call in `pytest.raises(FilmNotFoundError)` and calls `add_to_watchlist(user_id=sample_user, film_id="00000000-0000-0000-0000-000000000000")` against a database where no film exists — confirming the same `FilmNotFoundError` path used by the collection code (not a DB integrity error).

**Test commands that passed:**
- `python -m pytest tests/test_watchlist.py -v` → **1 passed**
- `python -m pytest tests/ -v` → **5 passed** (4 collection + 1 watchlist; no regressions)

## Comment 4 — Default visibility
**My position:**
- Keep watchlist entries **public by default** for this project. This already matches the code: `WatchlistEntry.public` is defined in `models.py` as `db.Column(db.Boolean, default=True)`, so no code change is required for this decision.

**Reasoning:**
- CineLog is a social film-tracking app.
- Public watchlists support discovery, recommendations, and friend-to-friend browsing.
- A watchlist is less sensitive than private ratings or personal notes because it only signals *future* viewing interest, not a judgment or a completed activity.
- Keeping the default public is consistent with the social nature of the app and avoids hiding the feature's core social value behind an opt-in.

**Tradeoff acknowledged:**
- Some users may treat their watchlist as private taste data, and watch *intent* can be identity-revealing (signalling interests a user has not chosen to disclose).
- Because defaults are sticky — most users never change them — "public by default" effectively makes the majority of watchlists public regardless of individual preference. This is the strongest form of the privacy objection and is the main reason a production build should not rely on the default alone.
- Private-by-default would better protect users who do not expect their watch intentions to be visible.
- A production version should make visibility explicit in the UI or add a per-entry visibility toggle rather than relying only on a default — the `public` column already exists to support that, so the model does not lock us into either policy.

## Comment 5 — Sort order
**My position:**
- Accept the reviewer's preference and sort watchlists by **date added, newest first**.

**Reasoning:**
- A watchlist functions like a queue or reminder list.
- The most recently added films are usually the most relevant to the user's current intent.
- Alphabetical order is useful for lookup, but less useful for answering "what did I recently decide I want to watch?"
- Date-added order better matches user behavior for a watchlist, and it makes the watchlist consistent with `get_collection()`, which already sorts by `date_added` descending.

**Engagement with reviewer's point:**
- The reviewer's point is persuasive because watchlists are time-sensitive.
- I originally used alphabetical order because it is predictable and stable.
- I changed the implementation to date-added order because it better reflects how users interact with a watchlist.

**Code changed:**
- `services/watchlist_service.py`, `get_watchlist()`: replaced `.join(Film).order_by(Film.title.asc())` with `.order_by(WatchlistEntry.date_added.desc())`. The real timestamp field on the model is `WatchlistEntry.date_added` (a `DateTime` column). The `.join(Film)` was only needed to sort on `Film.title`, so it was removed. This now mirrors `get_collection()`'s ordering exactly. (Note: `get_watchlist()` still references `entry.film` when building each result dict; that access relies on a relationship the model does not currently define — a pre-existing gap flagged under Comment 6, independent of this sort-order change.)

**How tests verified the behavior:**
- `python -m pytest tests/test_watchlist.py -v` → **1 passed**
- `python -m pytest tests/ -v` → **5 passed** (no regressions).
- Honesty note: there is currently **no dedicated watchlist sort-order test** (the equivalent of `test_get_collection_returns_newest_first`). The suite confirms the change did not break anything, and the new ordering reuses the same `date_added.desc()` pattern that *is* directly tested for the collection. A dedicated `test_get_watchlist_returns_newest_first` would be a good follow-up but was out of scope for this milestone.

## Comment 6 — Rebase
**What conflicted:**
- The refactor on `main` (`refactor: migrate film IDs from integer to UUID`) rewrote `models.py`: `Film.id` and `CollectionEntry.film_id` changed from `db.Integer` to `db.String(36)` (UUID), and that same commit removed the `WatchlistEntry` model.
- Because of how this repo is structured, the rebase itself produced **no textual merge conflict**: the `WatchlistEntry` model and integer `film_id` lived in the shared base commit, and no watchlist commit re-touched `models.py`, so `feature/watchlist` and `main` modified disjoint files. `git rebase origin/main` therefore replayed all commits cleanly.
- The real (semantic) conflict surfaced *after* the clean rebase: the rebased `models.py` was `main`'s UUID version with **no `WatchlistEntry`**, so `services/watchlist_service.py` and `tests/test_watchlist.py` (which `import WatchlistEntry`) failed with `ImportError` and the suite could not even collect.

**How I resolved it:**
- Rebased `feature/watchlist` onto the updated `origin/main` with `git rebase origin/main` (no `-i`, no merge commit).
- Repaired the branch to follow the UUID film-ID pattern used by the refactored collection code:
  - `models.py`: re-added the `WatchlistEntry` model with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), ...)` — a UUID, matching `CollectionEntry`. Integer film IDs were **not** reintroduced.
  - `services/watchlist_service.py`: updated the `add_to_watchlist` docstring from `film_id (int)` to `film_id (str): UUID` (the logic is ID-type-agnostic).
  - `tests/test_watchlist.py`: changed the nonexistent film ID from the integer `999999` to the UUID string `"00000000-0000-0000-0000-000000000000"`, matching `test_add_to_collection_nonexistent_film_raises`.
- The untracked local `.gitignore` (a subset of main's) was removed before rebasing so it would not block the checkout; `origin/main` provides the tracked `.gitignore`.

**How I verified no conflict remains:**
- Rebase completed successfully (`Successfully rebased and updated refs/heads/feature/watchlist`).
- `git status` is clean (nothing to commit, working tree clean).
- `git log --merges origin/main..HEAD` returns nothing → no merge commits; history is linear on top of `origin/main`.
- `python -m pytest tests/ -v` → **5 passed** (previously the suite errored on import until the model was re-added).
- Watchlist code and tests now use UUID-compatible film IDs consistent with the collection code.
- Known pre-existing issue (out of scope, not introduced by the rebase): `get_watchlist()` reads `entry.film`, but neither the original nor the restored `WatchlistEntry` defines a `film` relationship/backref, so that path is untested and would fail at runtime. Flagged for a follow-up; not changed here to keep the rebase scope tight.

## PR Description

### 1. Feature overview
This PR adds a **watchlist** feature to CineLog — films a user saves to watch later — alongside the existing collection feature.

- **Model:** `WatchlistEntry` in `models.py` — `id` (UUID), `user_id` (UUID FK → `user.id`), `film_id` (UUID FK → `film.id`), `date_added` (DateTime), `public` (Boolean, defaults to `True`).
- **Service:** `services/watchlist_service.py` — `add_to_watchlist(user_id, film_id)` and `get_watchlist(user_id)`; raises `AlreadyInWatchlistError` (defined in this module) and `FilmNotFoundError` (reused from `services/collection_service.py`).
- **Routes:** `routes/watchlist/watchlist.py`, registered under the `/watchlist` prefix in `create_app()`:
  - `GET /watchlist/<user_id>` → `view_watchlist()` — returns the user's watchlist (newest first).
  - `POST /watchlist/<user_id>/add` → `add_film()` — body `{ "film_id": "<uuid>" }`, returns `201` with the created entry.

### 2. Code review comments addressed
1. **Rename** — `save_to_watchlist()` → `add_to_watchlist()`, matching the `add_to_collection()` naming convention; the call site in `routes/watchlist/watchlist.py` was updated.
2. **Deduplication** — `add_to_watchlist()` raises `AlreadyInWatchlistError` if a `(user_id, film_id)` entry already exists, mirroring the `AlreadyInCollectionError` guard in `add_to_collection()`.
3. **Missing test** — added `tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises`, mirroring `test_add_to_collection_nonexistent_film_raises`; asserts `FilmNotFoundError` for a nonexistent film UUID.
4. **Default visibility** — see §3.
5. **Sort order** — see §4.
6. **Rebase / UUID** — see §5.

### 3. Design decision: default visibility
Watchlist entries remain **public by default** (`WatchlistEntry.public` defaults to `True`). CineLog is a social app; public watchlists support discovery and friend-to-friend browsing, and a watchlist signals only *future* viewing intent. Trade-off acknowledged: defaults are sticky (most users never change them), so a production build should expose an explicit visibility toggle rather than relying on the default — the `public` column already exists to support that. (Full reasoning under "Comment 4" above.)

### 4. Design decision: sort order
`get_watchlist()` now sorts by **`date_added` descending (newest first)** instead of alphabetically by title, accepting the reviewer's point that a watchlist is a time-ordered queue where recent additions are most relevant. This also makes the watchlist consistent with `get_collection()`. (Full reasoning under "Comment 5" above.)

### 5. Rebase / UUID update summary
The branch was rebased onto `origin/main`, which had migrated film IDs from integer to UUID (`refactor: migrate film IDs from integer to UUID`). The rebase replayed cleanly — **no textual conflicts and no merge commits** — but `main`'s `models.py` no longer contained `WatchlistEntry`, which broke the watchlist imports. `WatchlistEntry` was re-added with a **UUID `film_id` (`db.String(36)`, FK → `film.id`)** to match `CollectionEntry`; the watchlist test's nonexistent film ID and the `add_to_watchlist` docstring were updated to UUIDs. No integer film IDs remain. (Full detail under "Comment 6" above.)

### 6. Testing instructions
```bash
pip install -r requirements.txt
python -m pytest tests/ -v
```
All tests pass (currently 5: 4 in `tests/test_collection.py`, 1 in `tests/test_watchlist.py`).

### 7. Manual API testing
Start the app:
```bash
python app.py   # serves http://127.0.0.1:5000 (debug)
```

The film catalog is read-only (`routes/films.py`) and there is no user-creation endpoint, so create a user and grab a film UUID from a shell first:
```python
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u = User(username="demo", email="demo@example.com")
    f = Film(title="Paddington 2", year=2017, genre="Comedy")
    db.session.add_all([u, f])
    db.session.commit()
    print("user_id:", u.id)
    print("film_id:", f.id)
```

Then exercise the endpoints (substitute the printed IDs):
```bash
# Add a film to the watchlist -> expect 201 with the new entry
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_id>"}'

# List available films (to find film UUIDs)
curl http://127.0.0.1:5000/films/
```

### 8. Known limitations / follow-ups (not addressed in this PR)
- **Watchlist route error handling:** unlike the collection route (which maps `FilmNotFoundError` → 404 and `AlreadyInCollectionError` → 409), `add_film()` does not yet catch `FilmNotFoundError` / `AlreadyInWatchlistError`, so those currently surface as HTTP 500. A follow-up should wrap the service call the way `routes/collection.py` does.
- **`get_watchlist()` relationship:** it reads `entry.film`, but `WatchlistEntry` defines no `film` relationship and `Film` has no watchlist backref, so `GET /watchlist/<user_id>` errors on a non-empty watchlist. A follow-up should add the relationship (mirroring `Film.collection_entries`).
- **No dedicated watchlist sort-order test** yet (the equivalent of `test_get_collection_returns_newest_first`).

### Commit history
![Git log showing conventional commits](git-log-feature-watchlist.png)
