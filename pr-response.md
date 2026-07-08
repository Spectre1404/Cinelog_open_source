# PR Response Doc — CineLog Watchlist Feature

## AI Usage
Used an AI assistant for codebase orientation and to stress-test the Comment 4
and Comment 5 design arguments. All code, decisions, and final wording are my own.

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to follow the project's `verb_to_noun` service
naming convention, consistent with `add_to_collection()`. Updated the single
call site in `routes/watchlist/watchlist.py` — both the `import` line and the
invocation inside the `/add` handler.

**How I verified:**
Ran a project-wide search (`grep -rn "save_to_watchlist" --include="*.py"`) to
find all references; it returned no remaining matches after the rename.
Confirmed the app still imports via `create_app()`, and `pytest tests/ -v`
passed with no failures.

## Comment 2 — Deduplication
**What I did:**
Modeled the fix on `add_to_collection()` in `services/collection_service.py`.
Added an `AlreadyInWatchlistError` exception; before inserting,
`add_to_watchlist()` now queries for an existing `WatchlistEntry` with the same
`(user_id, film_id)` and raises if one is found. Added a
`UniqueConstraint("user_id", "film_id", name="unique_user_film_watchlist")` on
the `WatchlistEntry` model for a database-level guarantee, mirroring the
collection model's `unique_user_film_collection`. The route maps the exception
to HTTP 409 (and `FilmNotFoundError` to 404).

**How I verified:**
Compared the logic line-by-line against `add_to_collection`, which returns the
new entry on success and raises `AlreadyInCollectionError` on a duplicate.
`test_add_to_watchlist_duplicate_raises` confirms the second add raises and that
exactly one row exists afterward. A Flask test-client check of
`POST /watchlist/<user_id>/add` returned 201 on first add and 409 on the repeat.
`pytest tests/ -v` passed.

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py`, following the fixtures (`app`, `sample_user`,
`sample_film`) and assertion structure of `tests/test_collection.py`. Wrote
`test_add_to_watchlist_nonexistent_film_raises` — the equivalent of
`test_add_to_collection_nonexistent_film_raises` — which asserts that adding a
nonexistent `film_id` raises `FilmNotFoundError` rather than a database error.
Also added tests for a successful add, deduplication, and sort order.

**How I verified:**
`pytest tests/test_watchlist.py -v` — all pass. `pytest tests/ -v` — 8 passed
(4 collection + 4 watchlist).

## Comment 4 — Default visibility
**My position:**
New watchlists should default to **private** (`public=False`).

**Reasoning:**
A watchlist and a collection carry different signals. A collection is a record
of films a user has already watched and chosen to log — a curated, outward-facing
identity ("here's my taste"). A watchlist is a record of *intent*: films a user
hasn't watched yet. Intent is more personal and more easily misread — it can
reveal moods, curiosities, or "guilty pleasure" titles a user never meant to
broadcast. Optimizing for user trust, a new user who saves a film should not
unknowingly publish that decision. Defaulting to private follows the principle
of least surprise and privacy-by-design: sharing becomes a deliberate,
reversible opt-in rather than an inherited default.

The asymmetry of harm is the deciding factor. If we default to private and a
user wants visibility, they flip one flag and lose nothing. If we default to
public and a user didn't want it, the disclosure has already happened and can't
be undone. When the two failure modes are unequal, defaulting to the recoverable
one is the safer product decision.

**Tradeoff acknowledged:**
Public-by-default optimizes for social discovery — following friends' lists,
recommendations, and the network effects that can make CineLog stickier. A
private default means fewer lists are shareable out of the box, which dampens
that social loop. If CineLog's core thesis were social discovery, public-default
would be the stronger choice. I weighed that against the irreversibility of
accidental disclosure and chose privacy; users who want discovery can opt in.

I also want to be explicit about a current limitation (surfaced during review):
`get_watchlist()` / `GET /watchlist/<user_id>` returns every entry with no
visibility filter, and the endpoint is unauthenticated. So today the `public`
flag records *intent* but is not *enforced* — enforcement requires an
authentication layer that can distinguish the list owner from a visitor (which
is out of scope for this PR, and which the collection endpoint also lacks).
Keeping the default private means the stored data is already correct for when
that auth layer lands, rather than having to retroactively privatize lists.

## Comment 5 — Sort order
**My position:**
I agree with the maintainer and implemented **date added, descending
(newest first)** as the default.

**Reasoning:**
A watchlist is functionally a backlog or queue. The reason a film lands on it is
usually a fresh trigger — a trailer, a friend's recommendation, a review just
read. That recency of intent is exactly what makes the most recent addition the
most likely next watch, so surfacing newest-first matches the feature's primary
job: "what should I watch next?" It also keeps the mental model consistent with
`get_collection()`, which already sorts date-added descending; two list views
that behave the same are easier to reason about than two that sort differently.

**Engagement with reviewer's point:**
The maintainer's reasoning — "most users want to see what they added recently" —
is the right frame, and I want to engage with the alternative rather than just
defer. Alphabetical order's real strength is *lookup*: finding a known title in
a long list. But on a watchlist the dominant action is deciding what to watch,
not locating a specific film; for the lookup case, filtering/search scales far
better than an alphabetical sort (which degrades as the list grows and forces
the recent, most-relevant additions to arbitrary positions). I also considered a
third option — sorting by film metadata (release year, or average rating) — but
that presumes a discovery/ranking use-case, whereas "saved for later" is
fundamentally about the user's own recent intent, so recency wins. If usage data
later showed users primarily scanning watchlists alphabetically to find titles,
I'd revisit and likely add sort options rather than change the default.

## Comment 6 — Rebase
**What conflicted:**
Git reported **no textual merge conflict** when rebasing `feature/watchlist`
onto `main` — but the rebase produced a semantically broken result: the
`WatchlistEntry` model disappeared from `models.py`, and the app could no longer
import it. Root cause: `WatchlistEntry` was defined only in the shared base
commit (`014ae54`); the watchlist feature commits never modified `models.py`
(they added the service, route, and blueprint registration). Meanwhile the
integer→UUID refactor on `main` had *removed* `WatchlistEntry` (watchlist isn't a
feature on `main`). Replaying feature commits that don't touch `models.py` on top
of a `main` whose `models.py` no longer contains the model leaves it gone — a
silent, conflict-free loss rather than a `<<<<<<<` marker.

**How I resolved it:**
Interpreting the comment's "update your watchlist code to use UUIDs" as the
goal, I re-added the `WatchlistEntry` model on top of `main` with `film_id` as
`db.String(36)` (a UUID foreign key to `Film.id`, matching the refactored
`Film.id`), added the `Film.watchlist_entries` relationship so
`WatchlistEntry.film` resolves (mirroring the collection relationship), and
updated the service and route docstrings from integer to UUID. This is commit
`982ba12`.

**How I verified no conflict remains:**
- `git status` is clean; no rebase in progress and no conflict markers
  (`grep -rn "<<<<<<<"` finds none).
- The app imports: `create_app()` succeeds.
- `pytest tests/ -v` passes 8/8, including watchlist add/dedup/nonexistent-film
  /sort tests that exercise the UUID `film_id` end to end.
- A Flask test-client run of the endpoints returns 201 / 409 / 404 / 400 as
  expected against real UUID film IDs.
- History is linear with no merge commits: `git log --merges main..HEAD` is
  empty. (I also kept a `backup/pre-rebase-watchlist` branch as a safety net.)

## Stretch Features

### Stretch 1 — `remove_from_watchlist()`
**What I did:**
Added `remove_from_watchlist(user_id, film_id)` in
`services/watchlist_service.py`, following the collection feature's remove
pattern: it looks up the `(user_id, film_id)` entry, raises a new
`NotInWatchlistError` if absent, otherwise deletes it and returns `True`. Added a
`DELETE /watchlist/<user_id>/remove` route returning 200 on success and 404 when
the film isn't on the watchlist.

**How I verified:**
`test_remove_from_watchlist_deletes_entry` (add then remove leaves zero rows) and
`test_remove_from_watchlist_not_present_raises` (raises `NotInWatchlistError`).

### Stretch 2 — Second test (my choice of edge case)
**What I did:**
Added `test_same_film_on_two_users_watchlists`: two different users each add the
same film, and both adds succeed with one entry per user.

**Why I chose it:**
Deduplication is only correct if it is scoped *per user*. A naive implementation
keyed on `film_id` alone would wrongly reject the second user once the first has
saved the film. This test guards that the uniqueness rule is on
`(user_id, film_id)` — a subtle correctness property that none of the review
comments exercised.

### Stretch 3 — Visibility toggle
**What I did:**
Added an optional `public` parameter to `add_to_watchlist()` and to the
`POST /watchlist/<user_id>/add` endpoint (`{"film_id": ..., "public": true}`),
so callers can set visibility explicitly. It defaults to `False`, preserving the
private-by-default decision (Comment 4) when the field is omitted.

**How I verified:**
`test_add_to_watchlist_respects_public_flag` confirms `public=True` overrides the
default. Full suite: 12 passing.

## Commit History
`git log --oneline` (feature/watchlist, rebased on main — no merge commits):

```
docs: document stretch features in pr-response.md
test: add remove, visibility, and cross-user dedup tests
feat: allow setting watchlist visibility on add
feat: add remove_from_watchlist service and endpoint
docs: add pr-response.md documenting review responses
feat: default new watchlists to private
test: add watchlist service tests
feat: sort watchlist by date added, newest first
feat: deduplicate watchlist entries
refactor: rename save_to_watchlist to add_to_watchlist
refactor: migrate watchlist to UUID film IDs after rebase onto main
fix: update film retrieval method to use db.session.get in collection and watchlist services
feat: add watchlist service and add-to-watchlist endpoint
```

## PR Description

### What this feature does
Adds a per-user **watchlist** — films a user wants to watch later — alongside
the existing collection feature. Endpoints:
- `GET /watchlist/<user_id>` — return the user's watchlist, newest first.
- `POST /watchlist/<user_id>/add` — add a film with body
  `{"film_id": "<uuid>", "public": false}` (`public` optional, default false).
  Returns **201** on success, **409** if the film is already on the watchlist,
  **404** if the film doesn't exist, **400** if `film_id` is missing.
- `DELETE /watchlist/<user_id>/remove` — remove a film with body
  `{"film_id": "<uuid>"}`. Returns **200** on success, **404** if the film isn't
  on the watchlist. *(stretch)*

### Design decisions
1. **Default visibility: private (`public=False`).** New watchlists are private;
   sharing is an explicit opt-in. Full rationale and tradeoff in the *Comment 4*
   section above. Known limitation: the flag records intent but is not yet
   enforced — `GET /watchlist/<user_id>` is unauthenticated and returns all
   entries, so enforcement awaits an auth layer (the collection endpoint has the
   same limitation).
2. **Sort order: date added, newest first.** Matches how users treat a backlog
   and stays consistent with the collection view. Rationale in the *Comment 5*
   section above.

Known limitation (from review): `add_to_watchlist` validates `film_id` but not
`user_id`; a nonexistent user causes a foreign-key error. This mirrors
`add_to_collection` and is left consistent with the reference implementation.

### How to test manually
1. Set up and run:
   ```
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   python app.py   # http://127.0.0.1:5000 — no frontend, 404 at / is expected
   ```
2. Seed a user and a film (there is no create endpoint), noting their UUIDs:
   ```
   python - <<'PY'
   from app import create_app, db
   from models import User, Film
   app = create_app()
   with app.app_context():
       u = User(username="demo", email="demo@example.com")
       f = Film(title="Heat", year=1995)
       db.session.add_all([u, f]); db.session.commit()
       print("USER_ID=", u.id); print("FILM_ID=", f.id)
   PY
   ```
3. Add to watchlist (expect 201):
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
        -H "Content-Type: application/json" -d '{"film_id":"<FILM_ID>"}'
   ```
4. Add the same film again → **409** (deduplication).
5. Add a nonexistent film id → **404**; omit `film_id` → **400**.
6. Add with explicit visibility (stretch) → **201**, response shows `"public": true`:
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
        -H "Content-Type: application/json" \
        -d '{"film_id":"<FILM_ID>","public":true}'
   ```
   (Remove first if the film is already present — see step 8.)
7. View the watchlist (newest first, each entry has a `public` flag):
   ```
   curl http://127.0.0.1:5000/watchlist/<USER_ID>
   ```
8. Remove from watchlist (stretch) → **200**; repeat → **404**:
   ```
   curl -X DELETE http://127.0.0.1:5000/watchlist/<USER_ID>/remove \
        -H "Content-Type: application/json" -d '{"film_id":"<FILM_ID>"}'
   ```
9. Run the test suite: `pytest tests/ -v` → 12 passing.
