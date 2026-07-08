# CineLog

A community film-tracking REST API. Users log films they've watched into a
**collection** (with ratings) and save films they want to watch to a
**watchlist**. Built with Flask and SQLAlchemy over SQLite.

> CineLog is a JSON API with no frontend. Opening the root URL in a browser
> returns `404` — that's expected. Interact with it via `curl`, an HTTP client,
> or the test suite.

---

## Setup

```bash
python -m venv .venv
source .venv/bin/activate         # Windows: .venv\Scripts\activate.bat
pip install -r requirements.txt
python app.py
```

The app starts on `http://localhost:5000` and creates a local SQLite database
(`cinelog.db`) on first run.

---

## Project Structure

```
cinelog/
├── app.py                          # Flask app factory + blueprint registration
├── models.py                       # SQLAlchemy models (User, Film, CollectionEntry, WatchlistEntry)
├── services/
│   ├── collection_service.py       # Business logic for collections
│   └── watchlist_service.py        # Business logic for watchlists
├── routes/
│   ├── films.py                    # Film browsing endpoints
│   ├── collection.py               # Collection endpoints
│   └── watchlist/watchlist.py      # Watchlist endpoints
├── tests/
│   ├── test_collection.py          # Collection service tests
│   └── test_watchlist.py           # Watchlist service tests
├── CONTRIBUTING.md                 # Commit conventions and PR guidelines
├── pr-response.md                  # Code-review responses and design rationale
└── requirements.txt
```

---

## API Overview

### Films
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/films/` | List all films (supports `?genre=` and `?year=` filters) |
| GET | `/films/<film_id>` | Get a single film by UUID |

### Collection — films already watched
| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| GET | `/collection/<user_id>` | — | Get a user's collection (newest first) |
| POST | `/collection/<user_id>/add` | `{"film_id": "<uuid>", "rating": 4}` | Add a film (`rating` optional) |
| DELETE | `/collection/<user_id>/remove` | `{"film_id": "<uuid>"}` | Remove a film |

### Watchlist — films to watch later
| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| GET | `/watchlist/<user_id>` | — | Get a user's watchlist (newest first) |
| POST | `/watchlist/<user_id>/add` | `{"film_id": "<uuid>", "public": false}` | Add a film (`public` optional, default `false`) |
| DELETE | `/watchlist/<user_id>/remove` | `{"film_id": "<uuid>"}` | Remove a film |

**Response codes (add / remove):** `201` created · `200` removed · `400`
missing `film_id` · `404` film or entry not found · `409` already present.

---

## Data Models

- **Film** — a film in the catalog. IDs are UUIDs.
- **User** — a registered user. IDs are UUIDs.
- **CollectionEntry** — links a user to a film they've watched; stores `rating`
  and `date_added`. Unique per `(user, film)`.
- **WatchlistEntry** — links a user to a film they want to watch; stores
  `date_added` and a `public` visibility flag. Unique per `(user, film)`.

---

## Design Decisions

- **Watchlists are private by default** (`public=False`). Sharing is an explicit
  opt-in — pass `"public": true` when adding. The flag records intent; it is not
  yet enforced on the unauthenticated read endpoint.
- **Watchlists sort by date added, newest first**, matching the collection view
  and how users work through a backlog.

See [`pr-response.md`](pr-response.md) for the full rationale.

---

## Conventions

- Service functions follow a `verb_to_noun` pattern (`add_to_watchlist`,
  `remove_from_watchlist`, `get_watchlist`).
- Commits follow Conventional Commits with a linear, rebased history.

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for details.

---

## Running Tests

```bash
pytest tests/ -v
```

There is no create endpoint for users or films, so seed some data for manual
testing:

```bash
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

Then exercise the API, e.g.:

```bash
curl -X POST http://localhost:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" -d '{"film_id":"<FILM_ID>"}'
curl http://localhost:5000/watchlist/<USER_ID>
```
