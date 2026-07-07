# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- To be filled in at the end -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention (consistent with `add_to_collection()`). Updated the import and call site in `routes/watchlist/watchlist.py`.
**How I verified:** Used `Get-ChildItem -Recurse -Filter *.py | Select-String -Pattern "save_to_watchlist"` to confirm no remaining references to the old name. Ran `python -m pytest tests/ -v` — all 4 existing tests pass.

## Comment 2 — Deduplication
**What I did:** Added a pre-insert existence check in `add_to_watchlist()` mirroring `add_to_collection()`'s exact pattern — a `WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()` check that raises a new `AlreadyInWatchlistError` if a matching entry already exists. Also added a try/except block in the `add_film()` route in `routes/watchlist/watchlist.py`, matching the structure already used in `routes/collection.py`'s `add_film()` — catching `FilmNotFoundError` (404) and `AlreadyInWatchlistError` (409), with the successful `return` inside the `try` block.
**How I verified:** Ran `pytest tests/ -v` (all tests passing, including `test_add_to_watchlist_duplicate_raises`). Also manually verified with a one-off script that adds the same film to a user's watchlist twice in a row: the first call succeeds and the second correctly raises `AlreadyInWatchlistError` rather than creating a duplicate row.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` following the exact fixture and structure pattern from `tests/test_collection.py` (same `app`, `sample_user`, `sample_film` fixtures). Wrote `test_add_to_watchlist_nonexistent_film_raises` as the direct equivalent of `test_add_to_collection_nonexistent_film_raises`, as required by the comment. Also added `test_add_to_watchlist_creates_entry` and `test_add_to_watchlist_duplicate_raises` to cover the basic add path and the dedup logic from Comment 2, since that logic had no test coverage yet.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` (all passed), then the full suite `pytest tests/ -v` (no regressions in existing collection tests).

## Comment 4 — Default visibility
**My position:** Keep the default as `public=True`.
**Reasoning:** CineLog's value as a *community* film tracking app depends on watchlists being visible enough that discovery and social proof actually happen. Defaults dominate behavior — most users never touch a privacy setting they did not know existed — so if watchlists defaulted to private, the social graph the app depends on would stay empty by default, even though no one actively chose privacy. A social app that is quietly private by default fails at its core purpose regardless of how good the underlying feature is.
**Tradeoff acknowledged:** Some watchlist entries are more personal or aspirational than collection entries (things already watched, more "settled") — a guilty-pleasure pick or something tied to a private recommendation could get exposed before the user actively decided to share it. That is a real cost of defaulting to public. I think it is mitigated, not eliminated, by the fact that `public` is a per-entry field meant to be toggled, not a permanent lock — and by exposing a `public` parameter directly on `add_to_watchlist()` (see stretch feature), privacy-conscious callers can opt out at add-time instead of discovering the default after the fact.

## Comment 5 — Sort order
**My position:** Switch from alphabetical to date-added descending (newest first), matching @dev-lead's preference.
**Reasoning:** A watchlist is not a reference list you browse to look something up — it is a queue of "what am I going to watch next." What is freshest in a user's mind is usually what they just added, not whatever starts with the letter A. Surfacing the most recent add at the top also confirms the action worked and keeps a fresh recommendation top-of-mind, which is the behavior worth encouraging in a social app.
**Engagement with reviewer's point:** My first instinct was alphabetical, but @dev-lead is right that most users want to see what they added recently, and there is a consistency argument I had not considered until I read `get_collection()`: it already sorts by `date_added.desc()`. Having the watchlist sort differently (alphabetically) for what is functionally a sibling feature would be the kind of inconsistency a returning user notices — why does this list behave differently from that one? That internal-consistency point is what changed my mind, not just deference to the reviewer's preference.
**Bug found during verification:** Writing `test_get_watchlist_returns_newest_first` surfaced a pre-existing bug unrelated to sort order: `Film` had a `collection_entries` relationship (backref `film`) but no equivalent relationship for `WatchlistEntry`, so `entry.film` in `get_watchlist()` raised `AttributeError`. Fixed by adding `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to `Film`, mirroring the existing `collection_entries` relationship. Without this, `get_watchlist()` was broken regardless of sort order and had no test coverage catching it.
**How I verified:** Ran `pytest tests/ -v` — all 8 tests pass, including the new sort-order test confirming the newest-added film appears first.

## Comment 6 — Rebase
**What conflicted:** While `feature/watchlist` was open, `main` merged a refactor migrating `Film.id` from an auto-incrementing integer to a UUID string (`db.String(36)`), and updated `CollectionEntry.film_id` to match. `WatchlistEntry` did not exist on `main` at all, so it never received the same treatment — my branch's `WatchlistEntry.film_id` was still typed as `db.Integer`, which no longer matched `Film.id`'s new type.
**How I resolved it:** `git rebase origin/main` and `git merge origin/main` both completed without flagging a textual conflict in `models.py` — but silently produced an incorrect result each time, dropping the entire `WatchlistEntry` class definition while leaving a dangling `Film.watchlist_entries` relationship pointing at nothing. I verified this by checking for `WatchlistEntry` in `models.py` after each attempt rather than trusting a clean "Successfully rebased" message. After confirming the automatic resolution was unreliable, I manually rewrote `models.py` to restore the `WatchlistEntry` class with `film_id` correctly typed as `db.String(36), db.ForeignKey("film.id")`, matching the pattern already used for `CollectionEntry.film_id`.
**How I verified no conflict remains:** Ran `pytest tests/ -v` after the manual fix — all 8 tests pass, including `test_add_to_watchlist_creates_entry`, `test_add_to_watchlist_duplicate_raises`, and both nonexistent-film and sort-order tests, confirming the model change did not break dedup, creation, or ordering logic. Confirmed `git log --oneline` shows a single linear history with no merge commits.

## PR Description
<!-- To be filled in at the end -->
