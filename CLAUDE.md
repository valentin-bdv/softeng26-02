# CLAUDE.md

Guidance for Claude Code when working in this repository (Development Globe, NTUA Software Engineering 2026-2027, team softeng26-02).

## Project summary

**Development Globe** is an interactive world map showing how countries have evolved over the last 50 years. The user picks an indicator (life expectancy, GDP per capita, CO₂ emissions...) and a year, scrolls through the years, opens a country profile, compares countries, and builds a custom index by weighting several indicators.

- **Data sources:** World Bank Indicators API (yearly time series per country) and GISCO Countries from Eurostat (country boundaries, GeoJSON), joined on the ISO3 country code.
- **Two architectures, same application:** the course requires building it twice.
  - **Layered:** backend, REST API, CLI calling the API, one responsive web frontend (desktop + smartphone).
  - **MVC:** backend (Model + Controller), REST API, two separate web Views (desktop and smartphone).
- **Use cases:** UC1 world map for one indicator and one year, UC2 animation over the years, UC3 country profile, UC4 country comparison, UC5 custom index and ranking, UC6 CSV/JSON export, UC7 data import administration. UC1–UC4 are implemented in the frontend.
- **Deadlines:** 30/11/2026 for the SRS and the conceptual, activity and state diagrams; 10/01/2027 for everything else (code, tests, remaining diagrams, Visual Paradigm models).
- **Reference documents:** the course instructions `documentation/project_softeng2026_v01_en.pdf` and the team plan `documentation/Development_Globe_Project_Plan.md`. When they disagree, the course instructions win.
- **Language:** everything in the repository (code, comments, documentation, commits, issues, pull requests) is written in English.

## Repository structure

```
CLAUDE.md
README.md
documentation/
├── requirements/            srs-softeng26-02.md (SRS, Markdown with embedded PlantUML)
├── diagram-as-code/
│   ├── style.puml           shared PlantUML style
│   ├── layered/             every diagram, layered version
│   └── mvc/                 every diagram, MVC version
├── visual-paradigm/         softeng26-02-layered.vpp, softeng26-02-mvc.vpp
├── apis/                    OpenAPI specs and Postman scripts
├── Development_Globe_Project_Plan.md
└── project_softeng2026_v01_en.pdf
layered/
├── backend/   ├── cli/   ├── frontend/   ├── data/   └── testing/
mvc/
├── backend/   ├── frontend/   ├── data/   └── testing/
```

The folder names and locations are imposed by the course: do not rename, move or add top-level folders. Empty folders contain a `.gitkeep` file, which can be removed once the folder has real content.

### Naming rules

- **Diagrams:** `softeng26-02-[diagram_id].puml`, stored in **both** `diagram-as-code/layered/` and `diagram-as-code/mvc/`, with differences reflecting each architecture. The same `diagram_id` is used in both folders.
- **Diagram ids:** `uc01-activity` ... `uc07-activity`, `uc02-state`, `state-custom-index`, `state-import-job`, `data-conceptual`; later `class`, `api`, `component`, `deployment`.
- **SRS:** `documentation/requirements/srs-softeng26-02.md`, a single file for the whole team.
- **PlantUML:** every diagram starts with `!include ../style.puml`. Diagrams and documents must be PlantUML or Markdown: no Word files and no diagrams drawn as images. A generated `.svg` may sit next to its `.puml` for display in Markdown, but the `.puml` is the source.
- **Entity names** in diagrams and code follow the conceptual data model: Country, Indicator, Observation, CustomIndex, IndexComponent, ImportJob.

## Tech stack

These choices are final. They replace any open question in `documentation/Development_Globe_Project_Plan.md`. Do not add another language, framework, database or major library without the team's agreement: if a task needs a choice not listed here, ask the user and propose adding it to this section.

### Both architectures

- The same stack is used in `layered/` and `mvc/`, but they are two independent codebases: never import code from one into the other.
- Both use the same database schema and the same data dump.
- Local ports: API `8000`, frontend `5173`, database `5432`. Only one architecture runs at a time.

### Backend and REST API (`layered/backend`, `mvc/backend`)

| Concern | Choice |
|---|---|
| Language | Python 3.12 |
| Web framework | FastAPI, served by Uvicorn |
| Validation and settings | Pydantic v2, pydantic-settings (configuration from a `.env` file, never committed) |
| Database access | SQLAlchemy 2.0 ORM, GeoAlchemy2 for geometries, psycopg 3 driver |
| Migrations | Alembic |
| HTTP client (data import) | httpx |
| Geometry handling | Shapely |
| Dependencies | `venv` + `requirements.txt` (runtime) and `requirements-dev.txt` (tests, lint), versions pinned |
| Lint and format | Ruff |

- API base path: `/api`. Responses are JSON; exports (UC6) also accept `?format=csv`.
- Errors: FastAPI `HTTPException` with the status codes used in the diagrams (400 invalid input, 404 unknown resource, 503 database unavailable).
- In `mvc/backend`, the Model is the SQLAlchemy layer and the Controllers are the FastAPI routers; in `layered/backend`, keep separate `api` (routers), `services` (business logic) and `repositories` (database access) packages.
- API documentation: export the OpenAPI spec generated by FastAPI to `documentation/apis/softeng26-02-openapi-layered.yaml` and `documentation/apis/softeng26-02-openapi-mvc.yaml`.

### Database and data (`layered/data`, `mvc/data`)

| Concern | Choice |
|---|---|
| Database | PostgreSQL 16 + PostGIS 3.4 |
| Local setup | Docker Compose, image `postgis/postgis:16-3.4` (one `docker-compose.yml` at the root of `layered/` and of `mvc/`) |
| Naming | `snake_case`, singular table names (`country`, `indicator`, `observation`, `custom_index`, `index_component`, `import_job`) |
| Geometry | `geometry(MultiPolygon, 4326)` |
| Data dump | `pg_dump` plain SQL, `softeng26-02-dump.sql` in `layered/data` and `mvc/data` |

- **World Bank:** Indicators API v2 (`https://api.worldbank.org/v2/`, `format=json`, paginated). Aggregates are the entries whose region is "Aggregates" (`region.id == "NA"`): never store them as countries.
- **GISCO:** Countries dataset, release 2024, scale 1:20M, EPSG:4326, GeoJSON (`CNTR_RG_20M_2024_4326.geojson`).
- **Join:** on the ISO3 code (`ISO3_CODE` in GISCO, `id` in the World Bank country list). Never join on 2-letter codes.
- **Missing values:** no row in `observation`; never store 0 or an empty string instead.

### CLI (`layered/cli` only)

- Python 3.12 + Typer, calling the REST API through httpx. The CLI never accesses the database directly.
- Command name: `globe`. API URL read from the `GLOBE_API_URL` environment variable (default `http://localhost:8000/api`).
- There is no CLI in the MVC architecture.

### Frontend (`layered/frontend`, `mvc/frontend`)

| Concern | Choice |
|---|---|
| Runtime and package manager | Node.js 22 LTS, npm |
| Framework and build | React 19 + Vite, JavaScript (no TypeScript) |
| Map | Leaflet + react-leaflet 5 |
| Colour scales | d3-scale + d3-scale-chromatic |
| Charts (UC3, UC4) | Recharts |
| Routing | React Router |
| API calls | the browser `fetch` API, grouped in one `api` module |
| Lint and format | ESLint + Prettier |

- **Layered:** one responsive application (desktop and smartphone), CSS media queries with a single breakpoint at 768px.
- **MVC:** one Vite project with two separate Views, `desktop.html` and `mobile.html`, each with its own components; only the `api` module is shared between them.

### Tests (`layered/testing`, `mvc/testing`)

- API functional tests: pytest + FastAPI `TestClient`, against a separate test database.
- CLI tests: pytest + Typer `CliRunner`.
- Performance tests: Locust.
- No frontend tests (not required by the course).

## GitHub workflow

Every piece of work is tracked by a GitHub issue, in the milestone of the current submission, on the project board "Development Globe". Grades depend on each member's recorded contribution, so this workflow must be followed for every task.

Repository: `valentin-bdv/softeng26-02`. All commands below use the `gh` CLI; if `gh` is missing or not logged in, tell the user to run `gh auth login` then `gh auth refresh -s project`.

### 1. Before starting a task

- Find the issue for the task: `gh issue list --assignee @me` (or `gh issue view <N>`).
- If no issue exists for the task, say so and offer to create one (title, checklist, assignee, label, milestone, project) before doing the work.
- Never work directly on `main`. Create a branch named after the task, starting from an up-to-date `main`:
  `git switch main && git pull --ff-only && git switch -c <type>/<short-name>`
  Types: `srs/`, `diagrams/`, `data/`, `docs/`, `feat/`, `fix/`, `test/`. Example: `diagrams/uc03-activity`.
- Move the issue to **In Progress** on the board:
  ```bash
  ITEM=$(gh project item-list 1 --owner valentin-bdv --format json --jq '.items[] | select(.content.number==<N>) | .id')
  gh project item-edit --project-id PVT_kwHOCzg4c84BtWnf --id "$ITEM" \
    --field-id PVTSSF_lAHOCzg4c84BtWnfzhrgfwo --single-select-option-id 47fc9ee4
  ```

### 2. While working

- When a checklist item of the issue is done, tick it in the issue body (`gh issue edit <N> --body-file <file>` with `- [ ]` changed to `- [x]`). Do not rewrite or remove the other lines.
- Commit often with clear messages, in English, using the team prefixes: `[ADD]`, `[UPDATE]`, `[FIX]`, `[REMOVE]`. Example: `[ADD] UC3 activity diagram (layered)`.
- Only commit files related to the issue.

### 3. Opening the pull request

- Nobody merges into `main` without a pull request reviewed by another member.
- The PR description must contain `Closes #<N>` so the issue closes automatically on merge.
- Add the same milestone as the issue and ask the user which teammate should review it.
- Do not merge the pull request yourself, and do not approve your own user's PR.

### 4. After the merge

- Switch back to `main`, pull, and delete the local branch: `git switch main && git pull --ff-only && git branch -d <branch>`.
- The closed issue moves to **Done** on the board automatically.

### Attribution

Do not add any AI attribution anywhere: no `Co-Authored-By: Claude` trailer in commits, no "Generated with Claude Code" line in pull requests, issues or comments. This file is the only place where Claude is mentioned.
