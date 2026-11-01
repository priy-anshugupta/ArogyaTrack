# ArogyaTrack

A full-stack healthcare platform connecting patients, doctors, pharmacists, and administrators through medical records, digital prescriptions, pharmacy dispensing, medication adherence, and disease surveillance.

[Repository](https://github.com/priy-anshugupta/ArogyaTrack) · [Issues](https://github.com/priy-anshugupta/ArogyaTrack/issues) · [Setup guide](docs/local-setup.md)

## Features

| Role | Main workflows |
| --- | --- |
| Patient | Manage profiles and health cards, view records and prescriptions, track medication doses and health measurements. |
| Doctor | Review patient history, record consultations and diagnoses, issue prescriptions, and review adherence. |
| Pharmacist | Scan prescriptions, manage medicine inventory, record dispensing, and work with invoices. |
| Administrator | Review doctor registrations, manage access, and inspect regional surveillance dashboards. |

Additional modules cover PDF documents, multilingual web resources, clinical decision support, and model training and evaluation. Availability depends on configuration, data, and the services you run.

**Status:** development and demonstration software. The active surveillance bundle uses synthetic training data. Simulation results do not establish real-world clinical accuracy. See the [ML architecture](docs/ml-system.md) and [validation report](docs/ml-validation-results.md).

## Architecture and stack

```text
Next.js web (3000)        Flutter mobile
           \                /
             Django REST API (8000)
                 |         |
            PostgreSQL   Redis / Celery
                 |
         Surveillance model bundles
```

Web authentication uses cookies with CSRF protection; mobile clients use bearer tokens. Django owns permissions and business logic. Celery handles background jobs, and Redis provides a task broker and shared cache.

| Layer | Technologies |
| --- | --- |
| Backend | Django 4.2, Django REST Framework, Simple JWT |
| Web | Next.js 14, React 18, TypeScript, Tailwind CSS |
| Mobile | Flutter; Dart SDK constraint `^3.10.3` |
| Storage and tasks | PostgreSQL, Redis, Celery |
| Surveillance | scikit-learn, XGBoost, versioned model manifests |
| Documents | ReportLab, QR codes, multilingual fonts |

## Repository layout

```text
backend/                       Django apps, migrations, API, tests, documents
frontend/                      Active Next.js web application
mobile_app/                    Flutter app and platform projects
ml_models/                     Generators, training, bundles, evaluation
archive/legacy-vite-admin-panel/ Archived web prototype
docs/                          Setup, ML documentation, historical demo reports
.github/workflows/             Backend, frontend, and container CI
docker-compose.yml             Redis, API, worker, scheduler, and web services
```

Start web development in `frontend/`. The archived prototype is retained for reference.

## Prerequisites

- Python **3.11** matches CI and the backend container. Use a compatible Python 3.10–3.12 environment locally.
- Node.js **20** matches CI; npm uses the committed lockfile.
- PostgreSQL and an empty development database.
- Redis for Celery and shared caching; shared caching is required with `DEBUG=False`.
- Flutter satisfying `mobile_app/pubspec.yaml` for mobile development.
- Docker with Compose for the optional container workflow.

## Local setup

### Clone

```bash
git clone https://github.com/priy-anshugupta/ArogyaTrack.git
cd ArogyaTrack
```

### Backend

```bash
cd backend
python -m venv venv
```

Activate the environment and copy its configuration:

```powershell
# Windows PowerShell
.\venv\Scripts\Activate.ps1
Copy-Item .env.example .env
```

```bash
# macOS / Linux
source venv/bin/activate
cp .env.example .env
```

```bash
python -m pip install -r requirements.txt
```

Create a PostgreSQL database named `health_surveillance`, or choose your own name. Edit `backend/.env` with matching credentials and your own `SECRET_KEY` and `JWT_SECRET`. Use `DEBUG=True` only locally. Unless overridden, Django settings use console email, so local OTP messages appear in the backend terminal.

```bash
python manage.py migrate
python manage.py setup_rbac
python manage.py createsuperuser
python manage.py runserver 8000
```

The API is at `http://localhost:8000/api/`; Django admin is at `http://localhost:8000/admin/`. Keep this terminal running.

### Web application

Open a second terminal at the repository root:

```bash
cd frontend
npm ci
```

Copy `.env.local.example` to `.env.local` with your shell's copy command and confirm:

```dotenv
NEXT_PUBLIC_API_URL=http://localhost:8000/api
```

```bash
npm run dev
```

Open `http://localhost:3000`. Demo-login buttons require matching accounts already seeded in the selected database; cloning does not create accounts. See [demo data](docs/demo-data.md).

### Background tasks

Start Redis. With the backend virtual environment active, run each command in a separate terminal from `backend/`:

```bash
celery -A config worker --loglevel=info
```

```bash
celery -A config beat --loglevel=info
```

Set `CELERY_BROKER_URL` and `CELERY_RESULT_BACKEND` to your Redis address. Compose supplies Linux containers for worker development on Windows.

## Docker Compose

Prepare `backend/.env`, then run:

```bash
docker compose up --build
```

Compose starts Redis, Django, Celery worker/beat, and Next.js. **PostgreSQL is not included.** Set `DATABASE_URL` to a database reachable from inside the containers. With Docker Desktop, a host database can typically use `host.docker.internal`; container `localhost` refers to the container itself.

The backend entrypoint runs migrations and collects static files. Uploaded media uses a named volume; validated model bundles are mounted read-only. Public frontend settings are supplied at image build time, so rebuild when changing them.

```bash
docker compose logs -f backend
docker compose down
```

## Mobile development

```bash
cd mobile_app
flutter pub get
```

Copy `.env.example` to `.env`. Set `API_BASE_URL` to a device-accessible API address: Android emulators commonly use `http://10.0.2.2:8000/api`; physical devices need your computer's LAN address. Set the backend's local `DEV_ALLOWED_HOSTS` accordingly and bind Django to `0.0.0.0:8000` for LAN testing.

```bash
flutter run
```

Platform builds need native toolchains. See the [mobile README](mobile_app/README.md).

## Configuration

| Template | Purpose |
| --- | --- |
| `backend/.env.example` | Django, database, Redis, SMTP, optional AI integration |
| `frontend/.env.local.example` | API address, public settings, demo and map configuration |
| `mobile_app/.env.example` | Device-accessible API address |

Keep credentials in local environment files, outside Git. `NEXT_PUBLIC_` values are visible to browser users.

With `DEBUG=False`, settings require `SECRET_KEY`, `JWT_SECRET`, `DATABASE_URL`, and `USE_REDIS_CACHE=true`. Configure allowed hosts, CORS, trusted CSRF origins, HTTPS, SMTP, and storage for deployment. Local templates are starting points rather than production configuration.

Optional AI features read `AI_API_KEY`, `AI_BASE_URL`, and `AI_MODEL`; consult the backend template and source for provider configuration.

## API map

These prefixes are registered in `backend/config/urls.py`. Most workflows require authentication and role-specific permissions.

| Prefix | Responsibility |
| --- | --- |
| `/api/auth/` | Authentication, registration, verification, account management |
| `/api/patients/` | Patient profiles and related records |
| `/api/doctors/` | Clinical records and doctor workflows |
| `/api/prescriptions/` | Prescriptions, medicines, verification, downloads |
| `/api/pharmacy/` | Inventory and dispensing |
| `/api/adherence/` | Schedules and adherence |
| `/api/surveillance/` | Regions, observations, forecasts, alerts, model pipeline |
| `/api/dashboard/` | Summaries, metrics, alerts, exports |
| `/api/cdss/` | Clinical decision support |
| `/api/documents/` | Document operations |

Consult each app's `urls.py`, serializers, and permissions for exact endpoints and payloads. Web API integration lives in `frontend/lib/api.ts`.

## Surveillance models

Follow [docs/ml-system.md](docs/ml-system.md) before training or activating bundles. Create a separate environment using `ml_models/requirements.lock.txt`, generate the documented input data, then run from `ml_models/`:

```bash
python train_all_refined.py --data-dir training_data_v6 --threads 4 --activate
```

The active manifest selects an immutable bundle under `saved_models/validated/`. Runtime checks verify integrity and compatibility. Match training and serving dependencies. Historical bundles and reports are reference artifacts; synthetic scores are not measured clinical performance.

## Validation and CI

With dependencies installed and the backend test database configured:

```bash
cd backend
python manage.py check
python manage.py makemigrations --check --dry-run
python manage.py test
```

In a separate terminal:

```bash
cd frontend
npm run lint
npm run type-check
npm run build
```

The workflow under `.github/workflows/` runs backend checks with PostgreSQL/Redis, frontend checks, and container builds. A clean checkout does not imply those checks have passed in your environment.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Database connection refused | PostgreSQL is running and credentials, database, host, and port match `.env`. |
| Backend rejects configuration | Review required settings and the intended `DEBUG` mode. |
| Browser cannot reach API | Check backend process, API URL, CORS, and trusted origins; restart Next.js after env changes. |
| Email does not arrive | Inspect console email output or configure SMTP. |
| Demo login fails | Seed matching accounts in the selected database. |
| Mobile cannot connect | Check reachable host, allowed hosts, and firewall. |
| Model output unavailable | Inspect active manifest, mount, dependency versions, and required input data. |

## Documentation

- [Local setup](docs/local-setup.md)
- [Backend setup](docs/backend-setup.md)
- [Frontend setup](docs/frontend-setup.md)
- [Demo data and historical reports](docs/demo-data.md)
- [ML system](docs/ml-system.md)
- [ML validation results](docs/ml-validation-results.md)
- [Authentication](JWT_AUTHENTICATION_GUIDE.md)
- [Medical records](MEDICAL_RECORDS_MODULE.md)

## Development workflow

Use focused branches and descriptive commits. Include migrations with schema changes, keep patient information and secrets outside Git, and run relevant checks. Report issues with the affected role, reproduction steps, expected behavior, and sanitized logs in the [issue tracker](https://github.com/priy-anshugupta/ArogyaTrack/issues).
