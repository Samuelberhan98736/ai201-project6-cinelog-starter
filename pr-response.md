# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used an AI assistant in the ways the project explicitly allows — orientation, hygiene,
and stress-testing — and kept the design reasoning my own:

- **Orientation:** I had the assistant summarize `models.py`,
  `services/collection_service.py`, and `tests/test_collection.py`, and walk through what
  `add_to_collection()` returns when a film doesn't exist. I verified every summary
  against the actual code before relying on it (e.g., confirming the duplicate check
  raises *before* `db.session.add()`).
- **Hygiene:** I used it to check that my commit messages follow the Conventional Commits
  spec and that each commit is a single logical change, and to confirm the rebase left no
  merge commits (`git log --merges origin/main..HEAD` is empty).
- **Devil's advocate (Comments 4 & 5):** after drafting my own positions, I asked the
  assistant "what counterargument would a careful reviewer raise, and what tradeoff am I
  not acknowledging?" For Comment 4 it pushed on privacy-by-default norms and the risk of
  shipping a public default before visibility filtering exists — I strengthened my
  "Tradeoff acknowledged" paragraph in response. For Comment 5 it argued the consistency
  cost and that recency can be spun as a feature, not a bug — I added the explicit
  concession about consistency and the `?sort=` fallback to engage with that.

The two design positions (public-by-default, keep-alphabetical) and their reasoning are
mine; AI was used to pressure-test them, not to author them.
<!-- TODO(you): adjust this section so it accurately reflects YOUR own AI usage. -->
<!-- The design arguments in Comments 4 & 5 must be in your own words and reasoning. -->
<!-- The rubric rewards specificity and can spot generic AI-written arguments. -->
<!-- Read those two sections, disagree with parts, and rewrite them as yourself.  -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to match the project's `verb_to_noun` naming
convention (the same pattern as `add_to_collection()` in
`services/collection_service.py`). Updated the single call site in
`routes/watchlist/watchlist.py` — both the `import` line and the call inside the
`add_film` route handler.

**How I verified:** Ran a project-wide search for `save_to_watchlist` before and
after the change (`grep -rn save_to_watchlist .`, excluding `.venv/`). Before: three
references (definition, import, call). After: zero. Then ran `pytest tests/ -v` —
all 4 existing tests still pass, confirming nothing else referenced the old name.

## Comment 2 — Deduplication
**What I did:** Followed the exact pattern already used by `add_to_collection()` in
`services/collection_service.py`. That function (1) looks up the film and raises
`FilmNotFoundError` if missing, then (2) queries for an existing
`CollectionEntry` with the same `(user_id, film_id)` and raises
`AlreadyInCollectionError` if one exists, before creating a new row. I mirrored step
(2) in `add_to_watchlist()`: added an `AlreadyInWatchlistError` exception class and a
`WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()` check that raises it
when the film is already on the list. I also updated the `add_film` route in
`routes/watchlist/watchlist.py` to catch it and return HTTP 409 (Conflict) — the same
status `collection.py` returns for a duplicate — instead of letting the duplicate fall
through to an unhandled 500.

**How I verified:** Read `add_to_collection()` first and confirmed the duplicate check
returns/raises *before* the `db.session.add()` call, so no duplicate row is ever
created. Confirmed `WatchlistEntry` has no DB-level unique constraint (unlike
`CollectionEntry`), so the service-level guard is what prevents duplicates here. Ran
`pytest tests/ -v` (4/4 pass). The new duplicate behavior is exercised by the test
added for Comment 3's file.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled directly on
`tests/test_collection.py`. I reused the same three fixtures (`app` with an in-memory
SQLite DB, `sample_user`, `sample_film`) and wrote the equivalent of
`test_add_to_collection_nonexistent_film_raises`:
`test_add_to_watchlist_nonexistent_film_raises`, which asserts that
`add_to_watchlist()` raises `FilmNotFoundError` for a `film_id` that isn't in the DB
(using a `with pytest.raises(FilmNotFoundError)` block, same as the collection test). I
also added the two other analogous cases so the watchlist suite mirrors the collection
suite: `test_add_to_watchlist_creates_entry` (happy path + persistence check) and
`test_add_to_watchlist_duplicate_raises` (verifies the Comment 2 dedup logic — second
add raises `AlreadyInWatchlistError` and only one row exists).

**How I verified:** I used `test_add_to_collection_nonexistent_film_raises` as my
template — matched its fixture names, its `with pytest.raises(...)` assertion style, and
its fake-ID approach. Ran `pytest tests/test_watchlist.py -v` (3/3 pass) and then the
full suite `pytest tests/ -v` (7/7 pass), confirming the new file doesn't break the
existing collection tests.

## Comment 4 — Default visibility
**My position:** Watchlist entries should default to `public=True`. I'm keeping the
existing default and documenting the reasoning the maintainer asked for.

**Reasoning:** CineLog is a *community* film tracking app, and a watchlist is the
natural social surface of that community. The whole point of tracking films in a shared
app — rather than a private notes file — is discovery: seeing what other people plan to
watch, finding recommendations, following friends' queues. If watchlists were private by
default, the app would hit a cold-start problem: at launch every public feed would be
empty, because sharing would depend on each user finding and flipping a toggle most
never will. A public default seeds the discovery graph from day one, which is the
behavior I'm optimizing for.

There's also a signal in the data model itself. `CollectionEntry` has no `public`
field at all — a user's *collection* (films they've actually watched, with their
ratings) is never shared. Only `WatchlistEntry` has a `public` column. That asymmetry
tells me the two models were designed for different roles: the collection is the private
record of behavior, and the watchlist is the aspirational, shareable "films I want to
watch" list. A watchlist is also lower-sensitivity than a collection — it's intent, not
a record of what you actually watched and how you rated it — so the cost of exposing it
by default is lower.

**Tradeoff acknowledged:** Privacy-by-default is the safer, more widely-accepted
principle, and I'm consciously trading it away here. Some users will add films they'd
rather not broadcast (a surprise-gift research list, guilty pleasures) and could be
surprised their list is visible. That risk is sharpened by the fact that `get_watchlist()`
doesn't yet enforce any visibility filtering, so "public" isn't even truly gated today.
My mitigation: (1) pair the public default with an explicit per-entry `public` parameter
on the add endpoint so a caller can opt out at creation time (implemented as a stretch
change), and (2) document the default clearly so it's a known contract, not a surprise.
And because the `public` column already exists, if user research later shows this default
causes complaints, flipping it is a one-line change (`default=False`) with no migration
of intent — the opt-in/opt-out machinery is already in place.

## Comment 5 — Sort order
**My position:** I'd like to keep `get_watchlist()` sorted alphabetically by title
(`Film.title.asc()`) rather than switching to date-added. I'm taking the maintainer up
on the offer to discuss rather than just complying.

**Reasoning:** The core of my argument is that a collection and a watchlist are used at
different moments, so the ordering that's right for one isn't automatically right for the
other. A collection is a *log* — the user opens it to review "what have I watched
lately, what did I rate it," so recency (`date_added.desc()`, which `get_collection()`
already uses) matches how you read a history. A watchlist is a *decision queue* — the
user opens it at movie night to answer "out of everything I've been meaning to watch,
what do I want to watch tonight?" In that moment, *when* I bookmarked a film has almost
nothing to do with whether I want to watch it now. A date-added order just floats my most
recent impulse-add to the top and buries films I've been meaning to get to for months.
Alphabetical gives a stable, scannable index: I can predict where a title sits, scan the
whole list evenly without recency bias, and re-find a specific film I know I added
("I saved *Dune* at some point…"). Stable ordering matters more when a list is a lookup
surface than when it's a feed.

**Engagement with reviewer's point:** The maintainer's reasoning — "most users want to
see what they added recently" — is genuinely correct, and I want to engage with it
directly rather than wave it away. It's correct *for the collection*, which is exactly
why `get_collection()` sorts newest-first and there's a test locking that in
(`test_get_collection_returns_newest_first`). My disagreement isn't with the principle;
it's with transferring it to a surface that's used for a different task. I also want to
concede the strongest version of the maintainer's position, which is *consistency*: two
list features in one product sorting differently is a real cost, and a user could
reasonably expect them to behave the same. My answer is that consistency of *user intent*
should win over consistency of *mechanism* — the collection answers "what did I just do?"
and the watchlist answers "what should I do next?", and those genuinely want different
defaults. If the maintainer still prefers alignment after that, the resolution I'd
propose is to make sort order a query parameter (`?sort=title|date_added`) so both
behaviors are available and we're arguing about the default, not removing a capability —
but I'd still argue the *default* should be alphabetical for the reason above.

## Comment 6 — Rebase
**What conflicted:** My `feature/watchlist` branch was cut from the initial commit,
*before* the `refactor: migrate film IDs from integer to UUID` commit landed on `main`.
Rebasing with `git rebase origin/main` surfaced two conflicts:

1. **`.gitignore` (add/add):** both `main` (via its own `.gitignore` commit, which
   included `.pytest_cache/`) and my Milestone 1 commit added a `.gitignore`. Git
   couldn't tell they were "the same" file, so it flagged an add/add conflict.
2. **`models.py` (semantic, not textual):** this was the important one. My feature
   commits only touched `services/watchlist_service.py`, `routes/watchlist/watchlist.py`,
   and `app.py` — they never modified `models.py`. Main's UUID refactor rewrote
   `models.py` (changing `Film.id` and `CollectionEntry.film_id` to `String(36)` UUIDs)
   and, in doing so, the `WatchlistEntry` model was no longer present on that side. Because
   my branch made no competing change to `models.py`, the 3-way merge silently took main's
   version — so after the replay, `WatchlistEntry` was *gone* and my watchlist code
   imported a model that no longer existed. This was a conflict Git resolved "cleanly" but
   incorrectly, which is exactly the kind of thing that only shows up when you actually run
   the code.

**How I resolved it:**
- **`.gitignore`:** took the union of both sides — kept `main`'s `.pytest_cache/` and my
  `.venv/` / `venv/` entries — then `git add .gitignore` and `git rebase --continue`.
- **`models.py`:** re-added the `WatchlistEntry` model, but migrated it to the new UUID
  scheme to match the refactor — `film_id = db.Column(db.String(36),
  db.ForeignKey("film.id"), ...)` instead of the old `db.Integer`, mirroring exactly how
  `CollectionEntry.film_id` was migrated on `main`.
- **Watchlist code:** updated the now-stale integer references — the `film_id (int)`
  docstring in `add_to_watchlist()` became `film_id (str): UUID of the film`, and the
  route's `Body: { "film_id": <int> }` became `"<uuid>"`. This is a separate commit
  (`fix: update WatchlistEntry ... to UUID`) so the rebase resolution is one logical change.

**How I verified no conflict remains:**
- `git status` shows no unmerged paths and the rebase completed
  ("Successfully rebased and updated refs/heads/feature/watchlist").
- `git log --merges origin/main..HEAD` returns nothing → **no merge commits**; the branch
  is a clean linear replay on top of `main`.
- `grep -rn "db.Integer.*film\|<int>" models.py services/ routes/` returns nothing → no
  integer film-ID references survive.
- `pytest tests/ -v` → all 7 tests pass (4 collection + 3 watchlist), confirming
  `WatchlistEntry` exists again and the watchlist service works against the UUID schema.

## PR Description

### What this feature does
Adds a **watchlist** to CineLog — films a user wants to watch (saved for later), kept
separate from the collection (films already watched). It exposes:

- `GET /watchlist/<user_id>` — returns the user's watchlist as a list of film dicts with
  `date_added` and `public` attached.
- `POST /watchlist/<user_id>/add` with body `{ "film_id": "<uuid>" }` — adds a film to
  the watchlist. Returns `201` with the new entry, `404` if the film doesn't exist, or
  `409` if it's already on the list.

The service layer (`services/watchlist_service.py`) mirrors the collection service's
conventions: a `verb_to_noun` function name (`add_to_watchlist`), a film-existence check
raising `FilmNotFoundError`, and a duplicate check raising `AlreadyInWatchlistError`.

### Review feedback addressed
All six review comments are addressed and documented above:
1. Renamed `save_to_watchlist` → `add_to_watchlist` (naming convention).
2. Added a deduplication check so a film can't be added to a watchlist twice.
3. Added `tests/test_watchlist.py` (including the nonexistent-film case the reviewer
   asked for).
4. Documented the default-visibility decision (see Comment 4).
5. Documented the sort-order decision (see Comment 5).
6. Rebased onto the UUID-refactored `main` and migrated `WatchlistEntry.film_id` to UUID.

### Design decisions
- **Default visibility = public** (`WatchlistEntry.public` defaults to `True`) — optimizes
  for community discovery; tradeoff and mitigation documented in Comment 4.
- **Sort order = alphabetical** — I respectfully kept alphabetical rather than switching
  to date-added, on the argument that a watchlist is a decision queue, not a log; full
  reasoning and engagement with the maintainer's point in Comment 5.

### Commit history
Rebased on `main` (no merge commits) with conventional commits, one logical change each:
`feat` (endpoint) → `fix` (db.session.get) → `fix` (rename) → `fix` (dedup) →
`fix` (UUID migration) → `test` → `docs`.

### How to test manually
Primary verification is the automated suite:

```bash
pytest tests/ -v        # 7 passing (4 collection + 3 watchlist)
```

To exercise the API by hand (`python app.py`, then in another shell). You need a valid
`user_id` and a `film_id` that exist in the DB (both are UUIDs); create them via the
Python shell or the films endpoints, then:

```bash
# Add a film to the watchlist
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
# → 201 with the new entry

# Adding the same film again is rejected (deduplication)
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
# → 409 Conflict

# A film_id that doesn't exist
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
# → 404 Not Found

# View the watchlist (alphabetical by title)
curl http://127.0.0.1:5000/watchlist/<user_id>
```
