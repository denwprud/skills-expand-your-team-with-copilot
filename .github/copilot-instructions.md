# Copilot Instructions — Mergington High School Activities App

## Project Overview

This is **Mergington High School's extracurricular activities management web app**. It lets students browse activities and teachers sign students up (or remove them). The school is a public high school in Mergington, Florida with the motto "Branch out and grow", serving grades 9–12.

---

## Tech Stack

| Layer     | Technology                                      |
|-----------|-------------------------------------------------|
| Backend   | Python 3.13, FastAPI, Uvicorn (ASGI)            |
| Database  | MongoDB 7 (local, port 27017)                   |
| Frontend  | Vanilla HTML + CSS + JavaScript (no frameworks) |
| Dev env   | GitHub Codespaces / devcontainer (jammy)        |

**Constraints:** Only use HTML, CSS, JavaScript, and Python. No additional languages, frameworks, or services. Do not create CLI tools or additional apps/services.

---

## Repository Structure

```
/
├── .devcontainer/          # Codespace setup (installs MongoDB, pip deps)
│   ├── devcontainer.json   # Python 3.13 image, port 8000 forwarded
│   ├── postCreate.sh       # pip install + MongoDB install/start
│   └── postStart.sh        # Starts MongoDB on container restart
├── .github/
│   ├── copilot-instructions.md   # This file
│   ├── copilot-instructions-ext.md  # Extended school/staff context
│   └── steps/              # Exercise step instructions (not app code)
├── docs/
│   └── how-to-develop.md   # Developer setup guide
└── src/
    ├── app.py              # FastAPI app entrypoint, mounts static + routers
    ├── requirements.txt    # fastapi, uvicorn, pymongo, argon2-cffi
    ├── backend/
    │   ├── database.py     # MongoDB client, seed data, Argon2 password hashing
    │   └── routers/
    │       ├── activities.py  # GET /activities, POST signup/unregister
    │       └── auth.py        # POST /auth/login, GET /auth/check-session
    └── static/
        ├── index.html      # Single-page frontend
        ├── app.js          # All frontend logic (fetch, filters, modals)
        └── styles.css      # All styles
```

---

## Development Environment Setup

The devcontainer handles everything automatically. On first open:
1. `postCreate.sh` runs `pip install -r src/requirements.txt` and installs + starts MongoDB.
2. `postStart.sh` restarts MongoDB each time the container starts.

**To run the app manually:**
```bash
python -m uvicorn src.app:app --host 0.0.0.0 --port 8000 --reload
```

Or use VS Code's "Launch Mergington WebApp" debug config (`.vscode/launch.json`).

- App UI: http://localhost:8000
- Interactive API docs: http://localhost:8000/docs

---

## API Endpoints

| Method | Path                                    | Auth required | Description                          |
|--------|-----------------------------------------|---------------|--------------------------------------|
| GET    | `/activities`                           | No            | List activities (filterable by day/time) |
| GET    | `/activities/days`                      | No            | List all days that have activities   |
| POST   | `/activities/{name}/signup?email=&teacher_username=` | Yes (teacher) | Sign a student up |
| POST   | `/activities/{name}/unregister?email=&teacher_username=` | Yes (teacher) | Remove a student |
| POST   | `/auth/login`                           | No            | Teacher login (returns user info)    |
| GET    | `/auth/check-session`                   | No            | Validate a session by username       |

"Auth required" means the `teacher_username` query param must match a valid teacher in the DB.

---

## Database

MongoDB runs locally (`mongodb://localhost:27017/`), database `mergington_high`, collections:
- `activities` — keyed by activity name (`_id`), stores description, schedule, max_participants, participants list
- `teachers` — keyed by username (`_id`), stores display_name, hashed password, role

Seeded on first run by `database.init_database()` (only if collections are empty).

**Seeded teacher accounts:**

| username    | display_name        | role    | password |
|-------------|---------------------|---------|----------|
| mrodriguez  | Ms. Rodriguez       | teacher | art123   |
| mchen       | Mr. Chen            | teacher | chess456 |
| principal   | Principal Martinez  | admin   | admin789 |

---

## ⚠️ Known Bugs and Workarounds

### 1. Password hashing mismatch (login always fails with seeded data)

**Bug:** `src/backend/database.py` stores seeded passwords hashed with **Argon2** (`argon2-cffi`), but `src/backend/routers/auth.py` verifies passwords using **SHA-256** (`hashlib.sha256`). These hashes are incompatible, so login with any seeded account will return `401 Invalid username or password`.

**Workaround / Fix:** Standardize on one hashing scheme. The recommended fix is to update `auth.py` to use Argon2 for verification (matching how passwords are stored):

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher()

@router.post("/login")
def login(username: str, password: str):
    teacher = teachers_collection.find_one({"_id": username})
    if not teacher:
        raise HTTPException(status_code=401, detail="Invalid username or password")
    try:
        ph.verify(teacher["password"], password)
    except VerifyMismatchError:
        raise HTTPException(status_code=401, detail="Invalid username or password")
    return {"username": teacher["username"], "display_name": teacher["display_name"], "role": teacher["role"]}
```

### 2. Stale documentation in `docs/how-to-develop.md`

The doc says "All data is stored in memory, which means data will be reset when the server restarts." This is **outdated** — data is persisted in MongoDB. Update that note if editing the docs.

---

## Architecture Conventions

- **One FastAPI app** in `src/app.py` — do not create additional services.
- **Add new backend routes** as new router files in `src/backend/routers/` and include them in `src/app.py`.
- **Frontend is a single page** (`index.html` + `app.js` + `styles.css`). Keep it simple; no JS frameworks.
- **Directory structure must remain easy to navigate** — avoid long single-file applications.
- Always update `docs/how-to-develop.md` or the main README when changing how the app is run or used.
- If the README grows too long, organize additional docs into the `docs/` directory.

---

## Code Quality & Security

- **Unit test numerical logic** (grades, scores, counts) if any is added.
- **Never hardcode credentials** in source code. Prompt for them at runtime or store locally.
- Personal data (student emails) is stored — keep privacy in mind.
- Keep code simple and readable; the staff is non-technical and may maintain it.
- Activity categories used in the frontend: `sports`, `arts`, `academic`, `community`, `technology`.

---

## School Context (from `copilot-instructions-ext.md`)

- School year: August–May, 3 trimesters + optional summer cycle.
- Users: students (browse/view) and teachers (sign up/remove students via teacher login).
- Staff are non-technical — avoid jargon in any UI text or documentation.
- School colors are **white and lime green** (not blue — see open bug issue about school pride).
