
# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**
- Renamed save_to_watchlist() to add_to_watchlist(). Then updated all call sites pertaining to add_to_watchlist().
**Checked all files who had save_to_watchlist():**

## Comment 2 — Deduplication
**What I did:** Added `AlreadyInWatchlistError`; `add_to_watchlist()` now checks for an existing entry before inserting and raises it on duplicates. Route catches it and returns 409.
**How I verified:** Called the add endpoint twice with the same film_id — first returns 201, second returns 409 instead of a duplicate row.

## Comment 3 — Missing test
**What I did:** Added `test_add_to_watchlist_nonexistent_film_raises`, mirroring `test_add_to_collection_nonexistent_film_raises`.
**How I verified:** `python -m pytest tests/test_watchlist.py -v` — passed.

## Comment 4 — Default visibility
**My position:**
- A users watchlistEntry should always stay private at default and public if changed in setttings.
**Reasoning:**
- This gives the user the time needed to build there watchlistEntry from scratch and later on if desiered to share with others. 
**Tradeoff acknowledged:**
- Users may not no at all if they are able to share there watchlist or not, which dosen't fully execute the purpose of the watchlist. 

## Comment 5 — Sort order
**My position:**
- Keep `get_watchlist()` sorted alphabetically by title, not by date added.
**Reasoning:**
- A watchlist is a to-watch list users scan to find a title, unlike the collection which is a watched history where recency matters.
**Engagement with reviewer's point:**
- Fair to flag the inconsistency, but the two endpoints serve different purposes, so a different sort order is intentional here.

## Comment 6 — Rebase
**What conflicted:**
- `.gitignore` — add/add conflict (both branches added one; main's had an extra `.pytest_cache/` line).
- A silent semantic conflict in `models.py`: main refactored film IDs from int→UUID and removed `WatchlistEntry`, while my branch still relied on the pre-refactor `WatchlistEntry` (with an `Integer` `film_id`). No textual conflict marker, but the model was dropped after rebase and `film_id` no longer matched `Film.id`.
**How I resolved it:**
- `.gitignore`: took the union of both versions.
- Re-added `WatchlistEntry` to `models.py` with `film_id` as `db.String(36)` to match main's UUID `Film.id`, added the missing `film` relationship so `get_watchlist()` works, and updated stale `film_id (int)` docstrings to UUID.
**How I verified no conflict remains:**
- `python -m pytest tests/ -v` — all 5 pass.
- End-to-end via test client: add → 201, duplicate → 409, nonexistent → 404, GET → 200 sorted alphabetically with UUID film IDs.

## PR Description

**What it does:**
- Adds a watchlist feature: users can save films they want to watch later, separate from their watched Collection.
- `POST /watchlist/<user_id>/add` saves a film to the watchlist; `GET /watchlist/<user_id>` returns it, sorted alphabetically by title.

**Design decisions:**
- Duplicate adds are rejected with 409 (`AlreadyInWatchlistError`) instead of creating a duplicate row.
- Entries default to `public=True`. Comment 4 raises whether private-by-default would be safer — shipped as public for now, open for follow-up.
- `get_watchlist()` sorts alphabetically by title rather than by date added (unlike `get_collection()`) since a watchlist is scanned to find a title, not reviewed as a history — see Comment 5.

**How to test manually:**
1. `python app.py`
2. `POST /watchlist/<user_id>/add` with `{"film_id": "<id>"}` — expect `201` and the new entry.
3. Repeat the same request — expect `409`.
4. `POST` with a nonexistent `film_id` — expect `404`.
5. `GET /watchlist/<user_id>` — expect the film list sorted alphabetically by title, each with `date_added` and `public`.