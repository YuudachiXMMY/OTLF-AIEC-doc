# Repository Architecture Blueprint (Post-Refactor)

**Introduction:** This document describes the planned structure of the codebase and the deployment architecture for the AIEC platform after refactoring. The goal is to establish a clean, maintainable project layout that supports fast development and clear separation of concerns. We consider two approaches (monorepo vs. multi-repo) and outline essential engineering standards. This blueprint will guide developers in organizing code and config, and DevOps in setting up the deployment environment. It’s meant for the technical team to ensure consistency in the project’s structure and to plan future scaling or module additions.

## Option A – Monorepo (Recommended for MVP)

Maintain all components (backend, frontend, infrastructure, docs) in a single repository. This simplifies coordination during rapid development and ensures all pieces version together. Proposed structure:

OTLF-AIContest/
├── backend/
│   └── aiec-api/            # Spring Boot API server module
│       ├── src/main/java/... (controllers, services, models)
│       ├── src/main/resources/ (config files like application.yml, DB migrations)
│       └── pom.xml
├── frontend/
│   ├── aiec-portal/         # React front-end for students/teachers (public portal + team dashboard)
│   ├── aiec-judge/          # (Optional) separate React app for Judge portal (MVP could combine with portal)
│   ├── aiec-console/        # (Optional) separate React app for Admin console (or combine with portal under admin routes)
│   └── shared-ui/           # (Optional) shared components or library if multiple front-end apps
├── infra/
│   ├── db/                  # Database initialization/migration scripts (if not in backend)
│   ├── docker/              # Dockerfiles and Docker Compose configs
│   │   └── docker-compose.yml (for local dev: MySQL, MinIO, etc.)
│   └── k8s/                 # Kubernetes manifests (if deploying to K8s, optional)
├── doc/
│   ├── prd/                 # Product requirement docs (like this PRD and pages/fields doc)
│   ├── api/                 # API spec
│   ├── data-dictionary/     # DB schema documentation
│   ├── security/            # Security & permissions docs
│   ├── release-plan/        # Release planning docs
│   └── architecture/        # Architecture & technical docs (like this blueprint)
└── README.md (overview and instructions)

**Notes:** In this monorepo, the front-end could be one single-page app that includes conditional rendering for all roles (with protected routes for judge and admin sections). Alternatively, we maintain separate build outputs (portal vs. judge vs. admin) if those diverge significantly, but they can still live in one repo for now.

We expect to have a single MySQL database and a MinIO instance for storage in dev/prod. The infra/docker-compose.yml brings those up for local development.

A monorepo makes it easier to run end-to-end tests and to ensure all modules are using compatible versions of any shared contracts (like API spec or data models if we generate TS types from Java models or vice versa).

## Option B – Split Repos (Future consideration)

If the project grows (e.g., separate teams handling front-end and back-end, or open sourcing one part and not others), we might split into multiple repositories:
- aiec-backend – Contains the Spring Boot API.
- aiec-frontend – Contains the React front-end(s).
- aiec-docs – Documentation (could publish via GitHub Pages).
- aiec-infra – Infrastructure as code (docker, k8s, deployment scripts).

For MVP, Option A is more efficient. Option B might be adopted later if needed for team workflows or if parts of the system are decoupled.

## Mandatory Engineering Standards (MVP and beyond)

To ensure code quality and maintainability:
- **Database Migrations:** Use Flyway (already integrated) for all schema changes. Every DB change must be accompanied by a migration script in backend resources (db/migration). This ensures any environment can be updated to latest schema easily.
- **API Versioning:** All API endpoints are under /api. We might prefix with version (e.g., /api/v1) to allow evolution. MVP might keep a single version implicitly, but prepare to introduce /v2 if breaking changes in future.
- **Data Validation:** Use javax.validation (Bean Validation annotations) on request DTOs in Spring Boot for input validation (e.g. @NotNull, @Size, custom annotations for email format, etc.). This ensures backend rejects bad data consistently. Also implement central exception handling to return validation errors in a standard format.
- **Error Handling:** Implement a global exception handler in the API that catches exceptions and returns structured JSON errors. Log server-side exceptions with enough detail (but without leaking sensitive info to client).
- **Audit Logging:** As noted, important actions should produce an audit log entry. This could be done via an aspect or simply in service methods when performing such actions. The logs should be stored in the audit_logs table.
- **Security:** Use Spring Security for JWT authentication and method/URL-based authorization. Roles are to be mapped to authorities and checked for admin endpoints vs. user endpoints. Ensure to test that data is properly restricted (no user can access others’ resources by changing IDs, etc. – implement checks in service layer for that).
- **Object Storage Abstraction:** Interact with files via an abstraction so that switching from MinIO (S3 compatible) to another storage is easy. For example, use the AWS S3 SDK (pointing to MinIO endpoint) or a library like MinIO SDK. Keep bucket names/config in application.yml. All file keys should be structured (e.g., bucket “aiec-files”, key pattern like {editionId}/{teamId}/{filename} or a UUID to avoid any guessable pattern).

## Backend Technical Stack & Structure

- The Spring Boot project is on Java 17, which is modern and allows using newer language features. Use Maven for build, with packaging as a single jar (for Docker image).
- Organize code by feature module where possible (e.g., package contest for contest/edition related code, team for team mgmt, submission for submission logic, judging for judge-related, security for auth config, etc.). This modular approach within the project helps future extraction if needed.
- Use Spring Data JPA for database access. Define Entity models corresponding to tables from the Data Dictionary. Use Repository interfaces for CRUD. Service layer to implement business logic (like assignment generation, scoring). Controller layer for request handling and mapping DTOs.
- Use DTOs for inputs/outputs in controllers – do not expose JPA entities directly, to maintain separation and avoid lazy loading issues in JSON serialization. Map struct or manual mapping can be used.
## Frontend Technical Stack & Structure

- Use Vite + React + TypeScript for the front-end. Organize the front-end project into pages and components by feature. E.g., have sub-folders for pages: pages/Login, pages/Dashboard, pages/JudgeAssignments, pages/AdminConsole, etc.
- Use a state management solution if needed (React Context or a lightweight store) for things like user auth state, but for MVP we can get by with passing props or using Context for the current user info and token.
- Networking: use a library like axios for API calls. Centralize API client code so that token is attached to requests and error responses are handled (maybe an interceptor to redirect to login on 401).
- Routing: use React Router. Define routes per role: e.g., /admin/* routes that only admins can access (check role and maybe redirect if not). Similarly, /judge/* for judge, /team/* for participant. The portal home and public pages can be at root or /public/*.
- For UI components, use a simple library (maybe Chakra UI or Ant Design) if it accelerates development, or basic custom components for forms and tables as needed. MVP can be basic but should be functional and not overly ugly – use a consistent simple CSS (could use Tailwind or Bootstrap for speed).
- The build output of the front-end (static files) will be served (in dev via Vite's dev server, in prod possibly by the Spring Boot app or a separate static server). For simplicity, we might integrate a production build of the React app into the Spring Boot resources (so the jar serves it). Alternatively, serve it via Netlify or another static hosting and have it call the API on a separate domain – but then handle CORS. In MVP, perhaps package frontend into Docker Nginx container if separate.
## Deployment Architecture

**Dev Environment:** Use Docker Compose for MySQL, MinIO, and Mailpit (for email testing). Developers run backend locally or in Docker, and same for frontend. We have environment config such that localhost:8080 is API and localhost:5173 is dev front-end, with CORS allowed for dev. .env files or application.yml handle local vs prod settings (e.g., DB credentials, MinIO keys).

**Production:** The target is to containerize the Spring Boot app (Dockerfile using a base OpenJDK image). The React app can be either served by the Spring Boot (if we include the static files) or by a separate Nginx container. For MVP, one container might suffice: a Spring Boot that serves API and static front-end (perhaps using Spring ResourceHandler for the front-end files). Alternatively, use two containers: one for API, one for static front-end. The static front-end could also be hosted on GitHub Pages or S3 + CloudFront, given it’s just static files, and configured to point to the API endpoint.

**Infrastructure:** We will likely deploy on a cloud VM or a container service. The components needed at runtime: - MySQL 8 database (we can use RDS or a managed DB, or a VM with MySQL).
- An S3-compatible storage (MinIO server or use AWS S3 in production). If using AWS S3, update config accordingly (access key, secret, bucket, region).
- The Spring Boot API container.
- If separate, a container or hosting for the front-end.
- Optionally, Mailpit can be replaced with a real SMTP service for sending emails (like AWS SES or any SMTP) for things like user notifications. MVP might not actually send emails except maybe registration confirmation or results announcements – which we can add if needed.

**Scaling considerations:** The MVP is for a single contest event with presumably a manageable number of users (maybe dozens of teams, a handful of judges). A single instance of the app should handle it easily. We ensure the system can be dockerized so scaling out (multiple app instances behind a load balancer) is possible if needed, though likely not required initially. The database should be sized to handle a few thousand entries without issues.

## Suggested Frontend Information Architecture

As mentioned, we have a few user groups using the system. The front-end routing should reflect that:
- **Public Portal:** Unauthenticated pages (home, info, login, register). Possibly at the root path / (home) and /login, /register, etc.
- **Student/Teacher Team Portal:** After login, these roles land in a “team portal”. If combined SPA, we can prefix routes like /app/* or use role-based logic. Alternatively, host at /team/ subpath. For clarity, could do e.g. http://site.com/team/dashboard, .../team/submission. This area includes team dashboard, team management, application form, submissions, results view.
- **Judge Portal:** Could be a separate section or separate app at /judge. E.g. http://site.com/judge goes to judge’s assignment list, etc. If same SPA, we just conditionally render based on role. For MVP, possibly easier to not split the build: just have one app that after login checks role and routes accordingly. But to avoid loading unnecessary student UI for judges, we might separate. It’s a trade-off. Given time, one app with conditional views is fine.
- **Admin Console:** Section at /admin with routes for contest setup, monitoring, etc. This should be secured (maybe add an extra login check or MFA step).

So one possible approach: Use a top-level route check – if user.role === 'ADMIN', show admin app component; if 'JUDGE', show judge app component; if 'STUDENT' or 'TEACHER', show team app component. Each of those components manages its sub-routes. This way we bundle one application but logically separate the UI by role.

## Additional: Naming Conventions & Coding Style

- **Git & Commits:** Use meaningful commit messages, prefixed with component if possible (e.g., “[backend] Implement team APIs”, “[frontend] Add judge scoring page”). Use feature branches for major features, merge via PR.
- **Code Style:** Backend Java – follow typical conventions (CamelCase classes, methods camelCase, constants UPPER_SNAKE). Use Lombok for getters/setters to reduce boilerplate if allowed. Frontend TS/JS – follow a style guide (e.g., Airbnb or default ESLint recommended). Use functional components and React hooks.
- **Configuration Management:** Externalize config that varies by environment. E.g., DB URL, MinIO creds, JWT secret – use environment variables or a config server in prod. Do not hardcode secrets in the repo. The repo can have a sample application.yml (or .env.example for front-end) but actual secrets injected via env in deployment.
- **Logging:** Use a logging framework (SLF4J with Logback in Spring Boot). Log at INFO for high-level events (server start, major actions), DEBUG for detailed dev troubleshooting. Ensure no sensitive info (like passwords or large data dumps) is logged.
- **Testing:** Aim to write unit tests for critical pieces (e.g., assignment generator, score aggregator). Given the tight timeline, integration tests might be minimal, but at least have a smoke test script as mentioned. We can add more automated tests post-MVP.
## Object Storage Structure

We will use a single S3 bucket (e.g., named aiec-submissions) for all uploaded files. Within the bucket, we structure keys logically:
For example: {editionYear}/team-{teamId}/{stage}-{stageId}/{filename}.
However, to avoid exposing teamId or stageId (which might not be guessable easily but still), an alternative is to use the database file record ID or a UUID for the file and not expose structure. E.g., upon upload request, generate a GUID and use that as key prefix. But for clarity in manual debugging, a structured path helps.

One approach:
- Use edition or contest as top folder: e.g., AIEC-2025/TEAM-201/BusinessPlan-v1.pdf.
- The storage_key in DB will have that full path. The MinIO (S3) endpoint plus bucket gives a full URL when needed.
In any case, the security is enforced by only giving signed URLs to authorized users, so even if keys have some readable info, it’s not accessible without a valid signature. Still, we’ll include random components in file names or use file IDs to avoid predictable patterns.

Example key scheme using IDs: edition-{editionId}/team-{teamId}/submission-{submissionId}/{fileId}-{origFileName}. This keeps it organized by contest structure but includes unique IDs so that even if two teams have same file name, they won’t clash, and keys aren’t easily guessable without knowing IDs.

MinIO is configured with an access key/secret (aiec/aiecsecret in dev as per README). In production, we will use secure credentials.

## Deployment & Environment Considerations

- We will deploy in an environment that supports Docker (could be AWS EC2, or container service). Ensure environment variables or config maps are used for: DB connection (URL, user, pass), MinIO/S3 credentials & bucket name, JWT secret, and any SMTP config if emails used.
- We will enable CORS on the backend for the front-end domain if they are separate. For MVP, if we serve front-end from same domain (just different paths), CORS isn’t an issue.
- Set up a basic monitoring or at least logging aggregation (maybe just ensure logs can be accessed, or use something like ELK stack if time permits later).
- We should also think about backup strategy for data (since this is contest data, ensure the MySQL DB is backed up or can be restored if crash). Using a managed DB service can offload this. For MinIO, regular backups of the volume or use a replicated storage.
## Diagram (High-Level)

Below is a simple high-level architecture diagram of the system components and interactions:

*High-Level System Architecture – The Spring Boot backend serves REST API requests, connects to MySQL for data and to MinIO for file storage. The React front-end (portal) communicates with the API via HTTP/JSON. Judges and Admin use the same API with role-based access. The system can be deployed via Docker containers. External dependencies include an SMTP server for notifications (Mailpit in dev) and object storage (MinIO/S3) for file uploads.*

*(In the diagram, arrows indicate communication: the front-end in browser calls the API; the API reads/writes to the database and to the object storage. Admin and Judge are just different users of the same front-end or different front-end modules but still calling the same API. Dev/ops tools like Docker and Flyway handle environment setup and DB migrations respectively.)*

## Team & Responsibilities

*(If needed, we can outline which team members handle which part, e.g., Alice on backend, Bob on frontend, etc. – but since this doc is more architecture, we skip detailed people assignments.)*

## Naming Conventions Recap

- **GitHub Repo:** OTLF-AIContest for monorepo.
- **Java Packages:** e.g., ca.otlf.aiec.* as base package (if using organization naming).
- **Database:** Already discussed – snake_case table and column names, singular for tables where it makes sense or plural consistently. The current design uses mostly plural table names. Indexes and constraints named clearly (as in dictionary).
- **API Endpoints:** Use American English, consistent plurality. Nouns for resources (e.g., /teams, /submissions), verbs only for actions that don’t fit CRUD (like /assignments/generate).
- **Front-end:** Use PascalCase for component names, camelCase for filenames or kebab-case depending on preference (just be consistent). E.g., component file TeamDashboard.jsx or team-dashboard.jsx – decide one style.

This architecture blueprint ensures the team knows how the project is organized and deployed. By following this, we facilitate collaboration (everyone knows where to put or find things) and pave the way for future expansion (additional stages, more contests, etc., can slot into the structure with minimal friction). With the MVP delivered, we can iterate on this foundation for further improvements.
