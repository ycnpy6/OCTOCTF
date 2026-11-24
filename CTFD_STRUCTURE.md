# CTFd: Project Structure and Architecture Guide

This guide explains how this CTFd repository (version **3.8.9**, see `CTFd/__init__.py`) is organized and how its parts fit together. Read it before you build or customize a CTF.

---

## 1. What CTFd is

CTFd is a **Capture The Flag platform** written in Python. It is built on the **Flask** web framework.

- **Admins** create challenges, flags, hints, files, pages and configuration.
- **Players**, either individual users or teams, register, solve challenges and submit flags. Correct flags earn points.
- **The scoreboard** ranks players or teams by points, with ties broken by solve time.

| Layer         | Technology                                                                        |
| ------------- | --------------------------------------------------------------------------------- |
| Web framework | Flask with blueprints, plus Flask-RESTX for the REST API                          |
| ORM / DB      | SQLAlchemy and Flask-Migrate (Alembic). Supports SQLite, MySQL/MariaDB, PostgreSQL |
| Serialization | Marshmallow (`CTFd/schemas`)                                                      |
| Cache         | Flask-Caching on Redis, or the filesystem when Redis is not configured            |
| Templates     | Jinja2 in a **sandboxed** environment, organized as themes                        |
| Frontend      | Vite-built JS/SCSS (Alpine.js and Bootstrap) under `CTFd/themes/*`                |
| Server        | Gunicorn with gevent in production, `serve.py` for development                    |
| i18n          | Flask-Babel (`CTFd/translations`, `messages.pot`)                                 |

---

## 2. Top-level layout

```
CTFd/                      <- repository root
├── CTFd/                  <- the Python application package (main code)
├── migrations/            <- Alembic database migrations (versions/)
├── tests/                 <- pytest test suite
├── conf/nginx/            <- nginx reverse-proxy config used by docker-compose
├── scripts/               <- helper scripts (e.g. install_docker.sh)
├── Dockerfile             <- production image
├── docker-compose.yml     <- ctfd + nginx + mariadb + redis stack
├── docker-entrypoint.sh   <- waits for DB, runs migrations, starts gunicorn
├── serve.py               <- development server (python serve.py --port 4000)
├── wsgi.py                <- WSGI entry point for gunicorn
├── manage.py              <- Flask CLI wrapper (shell, db commands, etc.)
├── populate.py            <- fills the DB with fake users/challenges for testing
├── export.py / import.py  <- back up / restore a whole CTF as a zip
├── ping.py                <- DB connectivity check
├── pyproject.toml, uv.lock, requirements.txt  <- Python dependencies
├── Makefile               <- lint, format, test, serve shortcuts
└── README.md, CHANGELOG.md, CONTRIBUTING.md, SECURITY.md
```

---

## 3. The application package (`CTFd/CTFd/`)

```
CTFd/
├── __init__.py        <- create_app() factory, custom Flask class, theme loader
├── config.ini         <- main configuration file (server, redis, email, uploads...)
├── config.py          <- reads config.ini + env vars into the Config class
│
├── views.py           <- core routes: /setup, /, custom pages, /files, /themes, /healthcheck
├── auth.py            <- /login, /register, /logout, /confirm, /reset_password, OAuth (MLC)
├── challenges.py      <- /challenges page (UI shell; data comes from the API)
├── scoreboard.py      <- /scoreboard page
├── users.py           <- /users, /user (profiles)
├── teams.py           <- /teams, /team, join/create team
├── share.py           <- social sharing of solves
├── git.py             <- GitHub/GitLab integration helpers
├── errors.py          <- 403/404/500 handlers
│
├── admin/             <- the /admin panel (blueprint)
├── api/               <- REST API at /api/v1 (Flask-RESTX)
├── models/            <- SQLAlchemy models (all DB tables)
├── schemas/           <- Marshmallow schemas (validation + JSON output)
├── forms/             <- WTForms definitions used by templates
├── plugins/           <- plugin system + built-in challenge/flag types
├── themes/            <- frontend themes: core, admin, core-deprecated
├── utils/             <- shared helpers (config, user, security, scoring...)
├── cache/             <- caching helpers and cache-clearing functions
├── events/            <- Server-Sent Events stream (/events) for live notifications
├── cli/               <- `flask` CLI commands (export, import, get/set config...)
├── constants/         <- enums and constants (themes, user modes, visibility...)
├── exceptions/        <- custom exceptions
├── logs/              <- log output (registrations, logins, submissions)
├── uploads/           <- uploaded challenge files (filesystem provider)
├── translations/      <- compiled translation files
└── fonts/             <- fonts used when generating images
```

### 3.1 Application startup: `create_app()` in `CTFd/__init__.py`

`create_app()` is called by `serve.py`, `wsgi.py`, `manage.py` and the tests. It runs these steps in order:

1. Creates `CTFdFlask`, a Flask subclass that adds a sandboxed Jinja environment, cached sessions and a request class that is aware of subdirectory deployments.
2. Loads configuration from `CTFd.config.Config`.
3. Initializes the **cache** (Redis or filesystem).
4. Installs the **ThemeLoader**, which resolves templates from `themes/<ctf_theme>/templates`, or from `themes/admin/templates` for `admin/*`.
5. Initializes the **database**. SQLite tables are created directly. Other databases run Alembic `upgrade()`.
6. Initializes Marshmallow, Babel and `ProxyFix` when `REVERSE_PROXY` is set.
7. Calls `init_request_processors`, `init_template_filters` and `init_template_globals` from `utils/initialization`.
8. **Registers blueprints**: `views`, `teams`, `users`, `challenges`, `scoreboard`, `auth`, `api`, `events`, `social` and `admin`.
9. Calls `init_logs`, `init_events`, **`init_plugins`** and `init_cli`.

### 3.2 Configuration (`config.ini` and `config.py`)

CTFd has **two kinds of configuration**:

- **Server config** lives in `CTFd/config.ini`. Each value can be overridden by an **environment variable** with the same name. Sections:
  - `[server]`: `SECRET_KEY`, `DATABASE_URL` (defaults to SQLite at `CTFd/ctfd.db`), `REDIS_URL`
  - `[security]`: session cookie settings, `TRUSTED_HOSTS`, `TRUSTED_PROXIES`
  - `[ratelimits]`: login, register, reset-password and similar limits
  - `[email]`: SMTP or Mailgun
  - `[logs]`, `[uploads]`: filesystem or S3 storage
  - `[optional]`: `REVERSE_PROXY`, `SWAGGER_UI`, `SAFE_MODE` (disables plugins), `HTML_SANITIZATION`, and others
  - `[oauth]`, `[git]`, `[management]`: `PRESET_ADMIN_*` to create an admin automatically
- **CTF config** (runtime settings) is stored in the `config` database table (`Configs` model) and edited from **Admin → Config**. Examples: `ctf_name`, `user_mode` (`users` or `teams`), `start`/`end`/`freeze` times, `ctf_theme`, visibility settings, `max_attempts_behavior`, and registration code. Code reads these with `CTFd.utils.get_config(key)` and writes them with `set_config(key, value)`.

---

## 4. Data model (`CTFd/models/__init__.py`)

Many models use SQLAlchemy **polymorphism**: a `type` column picks the subclass.

### Core competition entities

| Model                                      | Table                                   | Purpose                                                                                                                                                                                                                                   |
| ------------------------------------------ | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Challenges`                               | `challenges`                            | A challenge: `name`, `description`, `category`, `value`, `type` (`standard` or `dynamic`), `state` (`visible`/`hidden`/`locked`), `max_attempts`, `requirements` (prerequisites JSON), `logic` (`any`/`all`/`team`), decay fields (`initial`, `minimum`, `decay`, `function`), `next_id`, `scheduled_at`, `module_id` |
| `Flags`                                    | `flags`                                 | Correct answers for a challenge. `type` is `static` or `regex`, `data` can be `case_insensitive`                                                                                                                                          |
| `Hints`                                    | `hints`                                 | Hints with an optional point `cost` and prerequisites                                                                                                                                                                                     |
| `Tags`, `Topics`, `ChallengeTopics`        |                                         | Labels shown to players (tags) or admins (topics)                                                                                                                                                                                         |
| `Files` → `ChallengeFiles`, `PageFiles`, `SolutionFiles` | `files`                   | Uploaded attachments                                                                                                                                                                                                                      |
| `Solutions`                                | `solutions`                             | Official write-ups that can be unlocked                                                                                                                                                                                                   |
| `Ratings`                                  | `ratings`                               | Player ratings and reviews of challenges                                                                                                                                                                                                  |

### Players

| Model                                                | Purpose                                                                                 |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `Users` → `Admins`                                   | Accounts. `type` is `user` or `admin`. Also holds `hidden`, `banned`, `verified`, `team_id`, `bracket_id` |
| `Teams`                                              | Teams with a `captain_id` and members. Used only when `user_mode = teams`               |
| `Brackets`                                           | Divisions such as "Students" or "Pro", each with its own scoreboard filter              |
| `Fields`/`FieldEntries` (User/Team variants)         | Custom registration fields defined by admins                                            |
| `Audiences`, `AudienceMembers`                       | Groups of users or teams                                                                |
| `Modules`, `ModuleAudienceAccess`                    | Groups of challenges, visible only to the audiences allowed to see them                 |

### Activity and scoring

| Model                                                                | Purpose                                                                                                                                      |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `Submissions` → `Solves`, `Fails`, `Partials`, `Discards`, `Ratelimiteds` | Every flag attempt, stored with `provided`, `ip` and `date`. `type` records the outcome (`correct`, `incorrect`, `partial`, and so on)   |
| `Awards`                                                             | Manual point adjustments. Hint costs are stored as negative awards                                                                           |
| `Unlocks` → `HintUnlocks`, `SolutionUnlocks`                         | Records which hints or solutions an account has unlocked                                                                                     |
| `Tracking`                                                           | IP addresses seen per user                                                                                                                   |

### Content and system

`Pages` holds custom pages (the home page is the page with route `index`). The other system models are `Notifications`, `Configs`, `Tokens`/`UserTokens` (API access tokens) and `Comments` (admin notes on challenges, users, teams and pages).

> **"Account" concept:** the account is a **user** in users mode and a **team** in teams mode. `user.account_id` and `CTFd.utils.modes.get_model()` hide this difference, so solves and awards are recorded against the account.

---

## 5. REST API (`CTFd/api/`)

The REST API is mounted at **`/api/v1`** and defined in `CTFd/api/__init__.py`. Each namespace has its own file in `CTFd/api/v1/`:

```
/challenges   /tags        /topics      /awards       /hints       /flags
/submissions  /scoreboard  /teams       /users        /statistics  /files
/notifications /configs    /pages       /unlocks      /tokens      /comments
/shares       /brackets    /audiences   /modules      /exports     /solutions
```

- The player UI calls these endpoints from JavaScript. For example, the challenges page loads `GET /api/v1/challenges` and submits through `POST /api/v1/challenges/attempt`.
- Requests authenticate with a session cookie and CSRF nonce, or with an **API token** (`Authorization: Token <token>`).
- Set `SWAGGER_UI = true` in `config.ini` to get interactive API docs.
- `api/v1/helpers/` contains request validation and model filtering. `api/v1/schemas/` contains response schemas used for the documentation.

---

## 6. Life of a flag submission

This is the most important flow in CTFd. It lives in `CTFd/api/v1/challenges.py` (`/attempt`) and `CTFd/plugins/challenges/`.

```
Player submits flag
   │
   ▼
POST /api/v1/challenges/attempt  {challenge_id, submission}
   │  checks: logged in, CTF running (ctftime / paused), challenge visible,
   │          prerequisites met, not already solved
   │  rate limiting: per-account lock + Redis counter (kpm) and
   │          max_attempts (lockout or timeout behavior)
   ▼
chal_class = get_chal_class(challenge.type)      # "standard" / "dynamic" / plugin type
status, message = chal_class.attempt(challenge, request)
   │  → logic.py: challenge_attempt_any / _all / _team
   │  → for each Flag: get_flag_class(flag.type).compare(flag, provided)
   ▼
correct  → chal_class.solve(...)   → creates a Solves row (+ dynamic value recalculated)
partial  → chal_class.partial(...) → creates a Partials row ("all" logic)
incorrect→ chal_class.fail(...)    → creates a Fails row
ratelimited → chal_class.ratelimited(...) → Ratelimiteds row
   │
   ▼
Cache cleared (scoreboard / standings) → JSON response to the browser
```

**Flag types** (`CTFd/plugins/flags/__init__.py`):

- `static` compares the whole string in constant time, with optional case-insensitivity.
- `regex` uses `re.match`, and the match must cover the entire submission.

**Challenge logic** (`CTFd/plugins/challenges/logic.py`):

- `any`: one correct flag solves the challenge.
- `all`: the player must submit every flag. Each partial capture is saved as a `Partials` row.
- `team`: in teams mode, every team member must submit a flag.

---

## 7. Scoring and the scoreboard

- `CTFd/utils/scores/` computes standings. Score = sum of solve values + sum of awards. The account with the earlier last solve ranks higher on a tie. Hidden and banned accounts are excluded.
- If `freeze` is set, solves after the freeze time do not appear to players.
- **Dynamic value challenges** (`CTFd/plugins/dynamic_challenges/`) lose points as more accounts solve them. The `function` setting picks `linear` or `logarithmic` decay (see `decay.py`), which runs from `initial` down to `minimum` with a `decay` rate. Each solve recalculates the value for **everyone**.
- Standings are cached heavily. After you edit data manually, call the functions in `CTFd/cache/__init__.py` such as `clear_standings()` and `clear_challenges()`.

---

## 8. Plugin system (`CTFd/plugins/`)

Every **folder** inside `CTFd/plugins/` is a plugin. During `init_plugins(app)`, CTFd imports each plugin folder and calls its `load(app)` function. Set `SAFE_MODE` to skip loading plugins.

Built-in plugins:

| Plugin                | What it provides                                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------------------ |
| `challenges/`         | `BaseChallenge`, `CTFdStandardChallenge`, the `CHALLENGE_CLASSES` registry, attempt logic, and the JS/HTML assets for the create/update/view modals |
| `dynamic_challenges/` | The `dynamic` challenge type with decaying points (has its own DB migrations)                          |
| `flags/`              | `BaseFlag`, `static` and `regex` flags, and the `FLAG_CLASSES` registry                                |
| `migrations.py`       | Helpers so plugins can ship their own Alembic migrations                                               |

Useful plugin API functions in `CTFd/plugins/__init__.py`:

- `register_plugin_assets_directory(app, base_path)`: serve a plugin's static files.
- `register_plugin_script` / `register_plugin_stylesheet`: inject JS or CSS into every page. Admin variants also exist.
- `register_admin_plugin_menu_bar` / `register_user_page_menu_bar`: add navigation links.
- `override_template(name, html)`: replace a theme template.
- `bypass_csrf_protection(f)`: disable CSRF checks on one route.
- To add a new challenge type, subclass `BaseChallenge` and register it with `CHALLENGE_CLASSES["mytype"] = MyChallenge`.
- To add a new flag type, subclass `BaseFlag` and register it with `FLAG_CLASSES["mytype"] = MyFlag`.

Minimal plugin skeleton:

```python
# CTFd/plugins/my_plugin/__init__.py
from flask import Blueprint

def load(app):
    bp = Blueprint("my_plugin", __name__)

    @bp.route("/hello")
    def hello():
        return "Hello from my plugin"

    app.register_blueprint(bp)
```

---

## 9. Themes and frontend (`CTFd/themes/`)

| Folder             | Purpose                                                            |
| ------------------ | ------------------------------------------------------------------ |
| `core/`            | Default player-facing theme                                        |
| `admin/`           | The admin panel UI, always used for `/admin`                       |
| `core-deprecated/` | Older theme kept for backward compatibility                        |

Each theme contains:

```
<theme>/
├── templates/      Jinja2 HTML (base.html, challenges.html, scoreboard.html, login.html, ...)
├── assets/         source JS (Alpine.js components), SCSS, images, sounds
├── static/         compiled output from Vite (what is actually served)
├── package.json    yarn dependencies
└── vite.config.js  build config
```

- The active player theme is the `ctf_theme` setting in **Admin → Config → Theme**.
- Theme files are served from `/themes/<theme>/static/...`.
- To build a theme, run `yarn install && yarn build` inside the theme folder.
- Templates render in a **sandboxed** Jinja environment, which limits what they can access in Python.
- To make a custom theme, copy `core/` to a new folder, edit it, then select it in the admin config.

---

## 10. Admin panel (`CTFd/admin/`)

The admin panel is a blueprint mounted at `/admin`. Every route uses the `@admins_only` decorator. Each section has its own file:

| File                  | Section                                            |
| --------------------- | -------------------------------------------------- |
| `__init__.py`         | Admin home, config, reset, import/export, plugins  |
| `challenges.py`       | Challenge list, create, edit                       |
| `users.py` / `teams.py` | Account management                               |
| `scoreboard.py`       | Admin scoreboard                                   |
| `submissions.py`      | Submission log (correct, incorrect, partial...)    |
| `statistics.py`       | Stats dashboards                                   |
| `notifications.py`    | Send notifications to players                      |
| `pages.py`            | Edit custom pages                                  |
| `audiences.py` / `modules.py` | Groups of accounts and groups of challenges |

Most admin pages render a template and then load or save data through the `/api/v1` endpoints.

---

## 11. Utilities (`CTFd/utils/`)

| Package                                | Purpose                                                                                                                               |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `config/`, `__init__.py`               | `get_config` / `set_config`, `is_teams_mode()`, visibility helpers                                                                    |
| `user/`                                | `get_current_user()`, `get_current_team()`, `is_admin()`, `authed()`, `get_ip()`                                                      |
| `decorators/`                          | `@authed_only`, `@admins_only`, `@require_team`, `@during_ctf_time_only`, `@require_verified_emails`, `@ratelimit`                    |
| `dates/`                               | `ctftime()`, `ctf_started()`, `ctf_ended()`, `ctf_paused()`                                                                           |
| `challenges/`                          | Which challenges are visible or solved, and prerequisite checks                                                                      |
| `scores/`, `scoreboard/`               | Standings computation                                                                                                                 |
| `modes/`                               | Handles the users-mode vs teams-mode difference (`get_model()`)                                                                       |
| `security/`                            | CSRF, password hashing, signing (`itsdangerous`), HTML sanitization                                                                   |
| `uploads/`                             | Filesystem or S3 upload providers                                                                                                     |
| `email/`                               | SMTP or Mailgun providers and email templates                                                                                         |
| `exports/`                             | CTF export and import to zip                                                                                                          |
| `events/`, `notifications/`            | Real-time notification pub/sub through SSE and Redis                                                                                  |
| `initialization/`                      | Request hooks (`before_request`, tracking, setup redirect), template globals and filters, CLI setup                                   |
| `sessions/`                            | Server-side session storage in the cache                                                                                              |
| `validators/`, `countries/`, `humanize/`, `formatters/`, `crypto/`, `csv/` | Miscellaneous helpers                                                                     |

---

## 12. Request lifecycle

1. The request arrives at gunicorn, then Flask (`CTFdFlask`).
2. `before_request` hooks in `utils/initialization` run:
   - If the CTF is not set up yet, redirect to `/setup`.
   - Track the user's IP (`Tracking`).
   - Validate the session and the CSRF nonce on POST requests.
   - Check bans and verified-email requirements.
3. The blueprint route runs. Decorators such as `@authed_only` and `@during_ctf_time_only` guard access.
4. The route either renders a Jinja template from the active theme (HTML pages) or returns JSON (API).
5. `after_request` hooks run, setting security headers and similar.

---

## 13. Running CTFd

### Development (local)

```bash
pip install -r requirements.txt   # or: uv sync
python serve.py                   # http://127.0.0.1:4000  (uses SQLite + filesystem cache)
```

On first launch, `/setup` asks you to set the CTF name, mode (users or teams), admin account, start and end times, and theme.

### Docker (production-like)

```bash
docker compose up -d              # http://localhost (nginx) -> ctfd:8000
```

This starts **ctfd** (gunicorn), **nginx**, **mariadb** and **redis**. Persistent data is stored under `.data/`.

### Useful commands

| Command                                         | What it does                                 |
| ----------------------------------------------- | -------------------------------------------- |
| `make serve`                                    | Same as `python serve.py`                    |
| `make test`                                     | Runs the pytest suite in parallel            |
| `make lint` / `make format`                     | ruff + prettier + theme linters              |
| `python manage.py shell`                        | Python shell with the app context            |
| `python populate.py`                            | Generates fake data for testing              |
| `python export.py` / `python import.py <zip>`   | Back up or restore a CTF                     |
| `flask db upgrade` (via `manage.py`)            | Apply database migrations                    |

---

## 14. Tests (`tests/`)

The test suite uses **pytest**. Folders mirror the app: `admin/`, `api/`, `challenges/`, `teams/`, `users/`, `models/`, `utils/`, `oauth/`, `cache/`, `brackets/` and others.

`tests/helpers.py` provides fixtures and builders: `create_ctfd()`, `destroy_ctfd()`, `register_user()`, `login_as_user()`, `gen_challenge()`, `gen_flag()`, `gen_solve()`, and more. Tests use an in-memory SQLite database by default.

---

## 15. Where to start for your own CTF

| Goal                                     | Where to work                                                                        |
| ---------------------------------------- | ------------------------------------------------------------------------------------ |
| Add challenges, flags, hints, files      | **Admin panel** (no code) or `POST /api/v1/challenges` and `/flags` with an admin token |
| Change look and branding                 | Copy `CTFd/themes/core` → new theme, build with Vite, select it in Admin → Config     |
| New challenge type (e.g. per-user docker instances, custom checker) | New plugin in `CTFd/plugins/<name>/` subclassing `BaseChallenge` |
| New flag-check logic                     | New `BaseFlag` subclass registered in `FLAG_CLASSES`                                  |
| Change server settings                   | `CTFd/config.ini` or environment variables                                            |
| Change competition rules (time, mode, visibility) | Admin → Config (stored in the `config` DB table)                             |
| Deploy                                   | `docker compose up -d`, with a real `SECRET_KEY` and DB passwords set                 |
| Database schema changes                  | Edit `CTFd/models`, then add an Alembic migration in `migrations/versions/`           |

---

## 16. Quick reference: key files

| File                                     | Why it matters                                          |
| ---------------------------------------- | ------------------------------------------------------- |
| `CTFd/__init__.py`                       | App factory, the entry point to everything              |
| `CTFd/config.ini`                        | Server configuration                                    |
| `CTFd/models/__init__.py`                | All database tables                                     |
| `CTFd/api/__init__.py`                   | List of API namespaces                                  |
| `CTFd/api/v1/challenges.py`              | Challenge listing and **flag submission** endpoint      |
| `CTFd/plugins/challenges/__init__.py`    | Challenge type base class and solve/fail recording      |
| `CTFd/plugins/flags/__init__.py`         | Flag comparison                                         |
| `CTFd/plugins/__init__.py`               | Plugin loader and plugin API                            |
| `CTFd/utils/decorators/__init__.py`      | Access-control decorators                               |
| `CTFd/utils/initialization/__init__.py`  | Request hooks and global template helpers               |
| `CTFd/themes/core/templates/`            | Player-facing HTML                                      |
| `docker-compose.yml`                     | Full production stack                                   |
