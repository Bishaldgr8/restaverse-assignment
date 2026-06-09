<img width="1716" height="817" alt="image" src="https://github.com/user-attachments/assets/f13859ec-6b25-461e-b3c0-069178632d4a" />
# Retaverse Employee Directory Search System

A high-performance, containerized internal tool to search, filter, and view company employees. Designed with a clean three-tier architecture in FastAPI, an optimized database schema in MySQL, and a premium SaaS search dashboard built in React, Vite, TypeScript, and TailwindCSS.

---

## Architecture

This application adheres to **Clean Architecture** and **SOLID principles** to ensure robust testability, clear separation of concerns, and future extensibility.

### System Topology

```
┌───────────────────────────────────────────────────────────────────────────┐
│                           Client Browser                                  │
│  [ React Frontend (Vite, TS) ] ──(Debounced 500ms API Search Requests)──┐ │
└─────────────────────────────────────────────────────┬─────────────────────┼─┘
                                                      │ (HTTP/JSON)         │
                                                      ▼                     │
┌───────────────────────────────────────────────────────────────────────────┼─┐
│                       FastAPI REST Gateway Service                        │ │
│                                                                           │ │
│  ┌────────────────────────┐      ┌────────────────────────┐               │ │
│  │     Route Handlers     │ ───> │     Service Layer      │               │ │
│  │ (app/api/routes/*.py)  │      │ (app/services/*.py)    │               │ │
│  └────────────────────────┘      └───────────┬────────────┘               │ │
│                                              │                            │ │
│                                              ▼                            │ │
│                                  ┌────────────────────────┐               │ │
│                                  │    Repository Layer    │               │ │
│                                  │ (app/repositories/*.py)│               │ │
│                                  └───────────┬────────────┘               │ │
└──────────────────────────────────────────────┼────────────────────────────┼─┘
                                               │ (SQLAlchemy ORM Queries)   │
                                               ▼                            │
┌───────────────────────────────────────────────────────────────────────────┼─┘
│                           Database Service                                │
│                     [ MySQL Storage Container ]                           │
└───────────────────────────────────────────────────────────────────────────┘
```

- **API Route Layer**: Thin HTTP controller functions. They deal exclusively with serializing data, executing path parameters validation, and returning JSON response formats. No business logic resides here.
- **Service Business Layer**: Orchestrates query definitions, processes validation, handles pagination parameters boundary clamping, maps ORM models to schemas, and translates DB anomalies into meaningful system exceptions.
- **Repository Data Layer**: The absolute boundary of MySQL query construction. Using SQLAlchemy's SQL compilation API prevents execution of raw queries and eliminates SQL Injection vectors.

---

## Folder Structure

```
retaverse project/
├── docker-compose.yml       # Starts Frontend, Backend and MySQL in one go
├── backend/
│   ├── app/
│   │   ├── api/
│   │   │   ├── routes/
│   │   │   │   ├── employees.py  # Route definitions for search & filters
│   │   │   │   └── health.py     # Liveness & readiness probes
│   │   │   └── dependencies.py   # FastAPI Dependency Injection
│   │   ├── core/
│   │   │   ├── config.py         # Type-safe pydantic-settings loading
│   │   │   └── exceptions.py     # Custom exceptions & global handlers
│   │   ├── database/
│   │   │   ├── base.py           # Declarative ORM model base
│   │   │   └── session.py        # Connection pooling and session factories
│   │   ├── models/
│   │   │   └── employee.py       # Employee model with strategic indexes
│   │   ├── repositories/
│   │   │   └── employee_repository.py # Repository database layer
│   │   ├── schemas/
│   │   │   └── employee.py       # Pydantic schemas (Request / Response validation)
│   │   ├── services/
│   │   │   └── employee_service.py # Service business logic layer
│   │   └── utils/
│   │       └── seeder.py         # Seeds 100 mock employees idempotently
│   ├── tests/                    # Fully isolated pytest suite
│   ├── Dockerfile                # Production multi-stage python image
│   ├── requirements.txt          # Python dependencies
│   ├── .env.example              # Backend template configs
│   └── seed.py                   # Standalone seed utility runner
└── frontend/
    ├── src/
    │   ├── api/
    │   │   └── axiosClient.ts    # Configured Axios instance with interceptors
    │   ├── components/
    │   │   ├── SearchBar.tsx     # Handles inputs, clears, recent history dropdown
    │   │   ├── EmployeeCard.tsx  # Layout for employee info with highlights
    │   │   ├── Pagination.tsx    # Scope and limit selection controls
    │   │   └── Loader.tsx        # Pulsing skeleton loaders
    │   ├── hooks/
    │   │   ├── useDebounce.ts    # Input debouncer hook
    │   │   ├── useEmployees.ts   # React Query employee search hook
    │   │   └── useDepartments.ts # Department list cache hook
    │   ├── pages/
    │   │   ├── HomePage.tsx      # Dashboard view containing directory
    │   │   └── NotFoundPage.tsx  # Custom 404 page
    │   └── main.tsx              # React mounting root
    ├── Dockerfile                # Production multi-stage nginx hosting image
    ├── nginx.conf                # Nginx production rules
    ├── tailwind.config.js        # Tailwind settings
    └── vite.config.ts            # Vite & Vitest block configurations
```

---

## Setup & Running

### Using Docker (Recommended)
You can build and start all components (MySQL Database, FastAPI Backend, and React Frontend) with a single command:

1. Clone or copy the repository to your desktop.
2. Build and launch the container ecosystem:
   ```bash
   docker-compose up --build
   ```
3. Open your browser and navigate to:
   - **Frontend App**: `http://localhost:3000`
   - **API Docs (Swagger)**: `http://localhost:8000/api/docs`

---

## Performance Optimizations

### 1. Connection Pooling
MySQL connections require socket handshakes and authentication checks, taking 50-100ms. Retaverse uses SQLAlchemy `QueuePool` with:
- **Pool Size (10)**: Keeps 10 connections permanently open for fast reuse.
- **Max Overflow (20)**: Dynamically spins up to 20 extra connections under sudden traffic bursts.
- **Pre-Ping**: Validates connection status before checkout, recycling dropped connections.

### 2. Strategic Database Indexing
To avoid CPU-heavy Full Table Scans (`O(n)` complexity), the MySQL schema includes three B-tree indexes:
- `idx_employee_name` for `name` query filtering.
- `idx_employee_department` for exact matching on departments.
- `idx_employee_name_dept` (Composite Index) to optimize search instances matching both constraints.

### 3. Frontend Search Debouncing
To prevent API server flooding, keypresses are debounced by **500ms** using a custom React Hook.
Typing "Rahul" will result in a single request sent once the user pauses, instead of 5 separate requests.

### 4. React Query Caching
Uses `@tanstack/react-query` to manage client state:
- Results are cached for `5 minutes` to eliminate duplicate requests on back-navigation.
- `keepPreviousData` is enabled to prevent loading flickering during pagination.

---

## API Documentation

### Search Employees
`GET /api/v1/employees`
Returns a paginated list of employees matching search term or department.

**Query Parameters:**
- `search` (string, optional): Search by name or department (case-insensitive, partial matching).
- `department` (string, optional): Filter by exact department.
- `page` (int, default 1): Page index.
- `limit` (int, default 20): Size of result chunk.
- `sort_by` (string, default "name"): Sort column (`name`, `department`, `designation`, `date_of_joining`).
- `sort_order` (string, default "asc"): Direction (`asc` or `desc`).

### Retrieve Departments
`GET /api/v1/employees/departments`
Returns all unique departments present in the database.

---

## Testing

### Running Frontend Tests (Vitest)
```bash
cd frontend
npm run test
```

### Running Backend Tests (pytest)
```bash
cd backend
python -m pytest tests/ -v
```
