# AIEC Platform

This repository hosts the code and documentation for the **AI Entrepreneurship Contest (AIEC)** platform run by OTLF. The system powers registration and team formation, submission and judging workflows, and administrative functions for the annual AIEC competition. In addition to the source code, the repo contains the full product requirements, API specification, data dictionary, permissions matrix, release plan and architecture overview.

## Repository structure

```dir
OTLF-AIEC/
├── backend/aiec-api        # Java 17 Spring Boot API server
├── frontend/aiec-portal    # Vite/React front‑end application
├── infra                   # Docker Compose configuration for MySQL, MinIO and Mailpit
└── doc/                    # Product requirements, API spec, data dictionary, permissions matrix, release plan and architecture docs
```

## Key documents

The `doc/` folder contains the following artefacts:

- **Product Requirements**: `doc/prd/PRD-AIEC-Platform.md` lays out the functional requirements, user stories and high‑level workflows.
- **Pages & Fields Dictionary**: `doc/prd/PRD-Pages-and-Fields.md` enumerates form fields and validation rules.
- **API Specification**: `doc/api/API-Spec-MVP.md` documents the REST endpoints exposed by the backend.
- **Data Dictionary**: `doc/data-dictionary/Data-Dictionary-MVP.md` explains the database schema and key attributes.
- **Permissions Matrix**: `doc/security/Permissions-Matrix.md` details the role‑based access control model.
- **Release Plan**: `doc/release-plan/Release-Plan-3Weeks.md` summarises the three‑week rollout schedule.
- **Architecture Blueprint**: `doc/architecture/Repo-Architecture-Blueprint.md` diagrams the system and repo layout.

## Requirements

To build and run the platform locally you will need:

- **Java 17** and **Maven 3.8+** – for the Spring Boot backend. The server exposes REST APIs and uses Flyway for database migrations.
- **Node.js 18+** and **pnpm (v8+)** – for the React portal. The front‑end uses Vite and TypeScript. Install pnpm globally via npm install -g pnpm if not already installed. Scripts are defined in package.json[1].
- **Docker** and **Docker Compose v2** – recommended for starting MySQL, MinIO and Mailpit locally. Without Docker you will need your own MySQL 8.x, S3‑compatible storage and SMTP service.
- **Git** – to clone the repository.

## Getting started

Follow these steps to spin up the AIEC platform on your machine.

1. Clone the repository

    ```bash
    git clone <repository-url>
    cd OTLF-AIEC
    ```

2. Start supporting services with Docker Compose.

    The infra folder includes a Docker Compose configuration for MySQL, MinIO and Mailpit. From within that directory run:

    ```bash
    cd infra
    docker compose up -d
    ```

    This will launch:

    - **MySQL** on port 3306 using database aiec, user aiec and password aiec[2][3].
    - **MinIO** object storage on ports 9000 (S3 API) and 9001 (console) with access key aiec and secret key aiecsecret[4].
    - **Mailpit** for SMTP testing on 1025 (SMTP) and 8025 (web UI)[5].

    If you run your own database or object storage, update the connection settings in `backend/aiec-api/src/main/resources/application.yml` accordingly.

3. Build and run the backend

    (Re)Create the DB, let Flyway build it:

    ```bash
    mysql -h 127.0.0.1 -P 3306 -u root -p -e "DROP DATABASE IF EXISTS aiec; CREATE DATABASE aiec CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
    mysql -h 127.0.0.1 -P 3306 -u root -p -e "GRANT ALL PRIVILEGES ON aiec.* TO 'aiec'@'127.0.0.1';"
    ```

    Navigate to the API module and build the project:

    ```bash
    cd backend/aiec-api
    mvn clean package -DskipTests
    mvn spring-boot:run
    ```

    By default the API server listens on `http://localhost:8080`[2]. The application will run Flyway migrations on start‑up and expects to connect to MySQL at `localhost:3306`[2]. You can customise the JWT secret and other application properties in application.yml under the app section.

4. Install dependencies and run the front‑end

    Install front‑end dependencies via pnpm and start the Vite development server:

    ```bash
    cd frontend/aiec-portal
    pnpm install
    pnpm dev -- --host --port 5173
    ```

    The dev portal will be available at `http://localhost:5173`. If your API runs on a different host or port.

    When serving the built `dist` without the dev proxy, copy `frontend/aiec-portal/.env.example` to `.env` and set `VITE_API_BASE_URL` to point at the backend.

    ```bash
    cp .env.example .env
    pnpm build
    pnpm preview --host --port 4173
    ```

5. Seed data and test the workflow

    On first run there are no contests or users. Use the admin API endpoints documented in `doc/api/API-Spec-MVP.md` to create a contest and edition. Then register as a teacher and student via the portal, create a team, join using the invitation code, bind a mentor, submit an application and approve it via the admin interface. This smoke test validates the end‑to‑end flow.

6. Build for production

    To create an optimised static build of the portal run:

    ```bash
    pnpm build
    ```

    The compiled assets are output to the `dist/` directory. You can serve these with any static file server or bundle them into a Docker image for deployment alongside the API.

----

## Localhost ports summary

During local development, the following services are exposed:

| Service        | URL / Address            | Details                                    |
|----------------|--------------------------|--------------------------------------------|
| API server     | `http://localhost:8080`  | Spring Boot REST API[2].                   |
| Portal         | `http://localhost:5173`  | Vite/React front-end dev server.           |
| Portal         | `http://localhost:4173`  | Vite/React front-end dist server.          |
| MySQL          | `localhost:3306`         | Database used by the API[2].               |
| MinIO (S3)     | `http://localhost:9000`  | S3 endpoint for uploaded files[4].         |
| MinIO Console  | `http://localhost:9001`  | Management UI for MinIO[4].                |
| Mailpit UI     | `http://localhost:8025`  | Web UI for email testing[5].               |
| Mailpit SMTP   | localhost:1025           | SMTP endpoint for local email captures[5]. |

----

## Contributing

Contributions are welcome. Please fork the repository, create a feature branch, make your changes and open a pull request. Ensure that new features include appropriate tests and documentation.

----

## License

This project is licensed for use by OTLF. See the LICENSE file for details.

----

[1] package.json
https://github.com/OTLF-CA/OTLF-AIEC/blob/main/frontend/aiec-portal/package.json

[2] application.yml
https://github.com/OTLF-CA/OTLF-AIEC/blob/main/backend/aiec-api/src/main/resources/application.yml

[3] [4] [5] docker-compose.yml
https://github.com/OTLF-CA/OTLF-AIEC/blob/main/infra/docker-compose.yml
