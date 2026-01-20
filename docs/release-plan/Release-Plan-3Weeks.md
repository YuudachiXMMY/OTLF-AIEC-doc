# Release Plan — 3 Weeks (MVP)

**Target:** MVP上线在三周内 (MVP launch within 3 weeks, covering Portal, Teams, Submissions, Judges, Admin, Aggregation, Publish)

**Introduction:** This release plan outlines a rapid 3-week development cycle to deliver the MVP features of the AIEC platform. Each week has specific focus areas and goals, with defined deliverables and P0 (highest priority) user stories. The plan assumes a small focused dev team and fast iteration. It is meant for the product and engineering team to track progress and ensure all critical components are delivered by the end of Week 3.

## Week 1 — Foundations & Core CRUD

**Goals:** Set up the fundamental structure of the project and implement core entities and basic flows. By end of week, have user auth in place and the basic team formation and contest config working.
- Complete repository re-architecture (establish the project structure, CI, etc.).
- Implement authentication and RBAC end-to-end (so roles are recognized and protected on front/back-end).
- Basic contest configuration: Admin can create an edition and stages.
- Team formation features: students and teachers can register, create teams, join teams; team data persists in DB.
- Ensure database migrations are configured and can initialize the schema.

**Deliverables (Definition of Done):**
- **Backend:** User registration & login (JWT) working; roles enforced on sample endpoints.
- **Backend:** Basic Contest/Edition/Stage APIs – admin can create an edition with stages. (This can be tested via an API client if no UI yet.)
- **Backend:** Team APIs – create team, join team, get team info.
- **Frontend:** Basic React app skeleton with routing for roles (after login, route student/teacher to team dashboard placeholder, judge to assignments placeholder, admin to admin console placeholder).
- **Frontend:** Implement registration and login pages (with form validation, calls to API, and storing token).
- **Frontend:** Implement a simple Team creation page and Join Team page, and the dashboard listing team info. The UI doesn’t have to be pretty yet, but functional.
- **DB:** Flyway migration for initial schema (users, contests, editions, stages, teams, team_members, etc. – enough to support above features).
- **Testing:** Manual test that a student can register, log in, create a team, a second student can join via code, and admin can create an edition via API. Verify data in database.

**P0 User Stories (Week 1):**
- **P0-AUTH-01:** *“As a student or teacher, I can register an account and log in so that I can access the platform.”* – (Covers backend login/reg and basic front-end pages).
- **P0-ADMIN-01:** *“As an admin, I can create a contest edition and its stages via the admin interface.”* – (Even if via a simple form or API tool for now).
- **P0-TEAM-01:** *“As a student, I can create a new team for a contest edition and get an invite code.”*
- **P0-TEAM-02:** *“As a student, I can join a team using an invite code given by my team leader.”*
- **P0-TEAM-03:** *“As a team leader, I can see my team’s details and members on my dashboard.”*

By end of Week 1, we should have a working skeleton: users can log in, form teams, and admin can define contest basics. This sets the stage for submissions and judging in Week 2.

## Week 2 — Submissions & Judge Review

**Goals:** Implement the submission upload system and the judge assignment and scoring functionality. Also enable file handling (upload/download) and ensure the portal and judge UI for these aspects are in place.
- Build out stage submission feature: teams can upload required files, save drafts, and submit before deadline.
- Integrate file storage (MinIO) for handling file uploads via presigned URLs.
- Implement judge assignment generation (back-end logic) and provide a way to assign judges to submissions (could be auto or triggered via admin API).
- Develop the judge’s interface: view assignments, download submission materials, fill in score rubric, submit feedback.
- Basic enforcement of scoring rules (e.g. comment length).
- Begin implementing result calculation logic (if time permits, or leave for early Week 3).

**Deliverables:**
- **Backend:** File upload endpoints (/files/presign, /files/complete) working with MinIO – able to upload and store file metadata.
- **Backend:** Submission endpoints – create submission, add items, submit final. Validate against stage requirements.
- **Backend:** Judge assignment generation method. Could be an admin API or an automatic assignment when submissions lock (MVP can use an API trigger).
- **Backend:** Endpoints for judges to fetch their assignments and submit scores. Enforce that they can only access their assigned projects.
- **Backend:** Rubric and scoring data structures in place – perhaps a simple predefined rubric to test with. Implement logic to calculate weighted score.
- **Frontend:** Team submission form UI – dynamically list required fields (for MVP, can hardcode for Preliminary stage if easier, but architecture should allow dynamic). Allow file uploads (integration with the presigned API), link inputs, etc. Provide a submit button that calls the API.
- **Frontend:** Judge dashboard – list of assignments with statuses.
- **Frontend:** Judge review page – shows submission content (download links for files, etc.) and provides a form for scores and comment. Basic UI for the rubric (like a list of criteria with input boxes) and a textarea for comment with word count.
- **Frontend:** Admin view – at least a rudimentary page to trigger “assign judges” (or this could be done via an API call outside UI for now). Possibly list all submissions and allow picking judges (MVP can random assign via backend without UI).
- **Testing:** Ensure a full flow: admin creates judges (either via seeding or a quick endpoint), assign judges to a stage; a team submits files; judges retrieve and submit scores. Test file upload and download works (e.g., a PDF can be uploaded and then downloaded by a judge). Ensure validation (e.g. cannot submit without required fields, judge cannot submit without comment).

**P0 User Stories (Week 2):**
- **P0-SUB-01:** *“As a team member, I can upload my team’s submission materials (files/text) for the current stage and submit them for evaluation.”*
- **P0-FILE-01:** *“As a user, I can upload files (such as PDFs, images) through the system and the files are stored reliably.”*
- **P0-JUDGE-01:** *“As a judge, I can see a list of my assigned submissions (anonymized) and for each, view the submitted materials and enter my scores and feedback.”*
- **P0-ADMIN-02:** *“As an admin, I can assign judges to all submissions in a stage so that each submission gets the required number of evaluations.”*

By end of Week 2, we should be able to simulate a contest round: teams submit, judges score. The major remaining piece will be aggregating those scores and displaying results.

## Week 3 — Aggregation, Results & Final Polishing

**Goals:** Finalize the scoring aggregation and results display. Implement the permissions matrix fully (security sweeps). Do end-to-end testing and fix any critical bugs. Prepare for deployment (e.g. containerize the app). If time permits, implement nice-to-have features like news announcements or additional refinements.
- Implement score aggregation logic with outlier removal and ranking.
- Build admin interface (or API) to run aggregation and then mark which teams advance or determine winners.
- Implement functionality to publish results (make them visible to teams, and optionally create a public announcement).
- Develop front-end for teams to see their results and feedback after publication.
- Complete the Permissions Matrix enforcement – double-check that no role can access data beyond their scope (e.g., test that a student cannot call judge APIs, etc.).
- Set up audit logging for key actions (optional P0 for security – at least for admin actions like publishing results).
- Run thorough tests through the full workflow from registration to final results.
- Prepare a smoke test script and ensure all steps can be executed on a fresh deploy.
- Deployment prep: Dockerize backend and frontend, ensure environment configs (DB connection, MinIO) are ready. Possibly deploy to a staging server or use Docker Compose to simulate production.

**Deliverables:**
- **Backend:** Score aggregation API – given a stage, calculate final scores, drop outliers, write stage_results. Possibly auto-advance top teams for Preliminary->Semi if desired or just output rankings for admin to interpret.
- **Backend:** Publish results API – mark results as published, which could trigger emails (MVP maybe just flags for UI to display).
- **Backend:** Implement any missing pieces of Permissions (e.g., if not already, restrict queries so users only get their data; add middleware for role-checking on endpoints).
- **Backend:** Audit logging on critical endpoints (admin actions like assignment generation, result publishing, score overrides, etc.).
- **Frontend:** Results page for teams – show their score (if we reveal) and advancement status, and display judge feedback comments. Possibly a link to download a compiled feedback (could be just a nicely formatted page to print).
- **Frontend:** Admin panel improvements – display a dashboard of current stage progress (e.g. number of submissions, number of scores in, etc.), a button to run aggregation and see the outcome (list of teams with scores, perhaps highlight outliers), and a button to publish results. Also allow admin to create news posts for announcements if time.
- **Documentation & Handover:** Update README with instructions to run the system. Ensure the doc/ folder (PRD, API spec, etc.) is up-to-date (this document is part of that).
- **Testing:** Conduct an end-to-end run: simulate a contest with 1 edition, 1 stage of judging. Use multiple dummy accounts: a few students to form a team, a teacher, a couple of judges, an admin. Walk through: team registers -> submits application -> admin approves -> team submits preliminary materials -> admin assigns judges -> judges submit scores -> admin aggregates -> admin publishes results -> team views feedback. Fix any bugs encountered.
- **Buffer:** Use remaining time for any UI/UX touch-ups (not priority over functionality, but maybe add loading spinners, better error messages, etc.), and any deferred features (like News CMS if trivial, or an FAQ page content).
- **Deployment:** Ensure that by end of week, we can deploy the backend (maybe as a Spring Boot Jar or Docker container) and the frontend (static build or container). If using Docker Compose, verify it works on a clean environment.

**P0 User Stories (Week 3):**
- **P0-SCORE-01:** *“As an admin, I can aggregate all judges’ scores for a stage, and the system will automatically drop the highest and lowest scores when applicable**, producing final scores and rankings.”*
- **P0-PUB-01:** *“As an admin, I can publish the results of a stage so that teams can see whether they advanced and view judges’ feedback.”*
- **P0-TEAM-04:** *“As a team member, I can view my team’s results for a stage (score/rank or advancement) and read the feedback from judges once results are published.”*
- **P0-AUDIT-01:** *“As a system owner, I have an audit log of important actions (e.g., admin **approvals, score changes, result publication) for accountability.”*
- **P0-OPS-01:** *“As a developer/devops, I can deploy the application easily (via Docker) and run a smoke test to verify the core functionality in a production-like environment.”*

By the end of Week 3, the MVP should be feature-complete and ready for a pilot run. We will have a fully functioning platform capable of handling the contest’s primary workflow. Any features not achieved (marked P1 or beyond) are noted as deferred, and we should ensure they are either stubbed or gracefully hidden. The three-week timeline is aggressive, so scope control is crucial: focus on the must-haves first, as outlined above, and ensure quality in those, rather than half-implementing too many extras.

**If timeline slips or issues arise (Hard Cut considerations):**
We will prioritize core contest functionality. Features that can be deferred to post-MVP if necessary include the News/FAQ pages (could update content manually), the appeals handling (if any), support for multiple mentors, and advanced COI handling. The plan is designed such that dropping those won’t break the main flow. We should communicate any such cuts early to stakeholders.

Finally, before launch, conduct a final run-through with actual contest data (maybe a small test contest) to validate everything under realistic conditions. After Week 3, we should be ready to onboard real users for the contest.

## Hard Cut Items (Potential Postponements):

- **News CMS:** If not done, admins can update participants via email instead for the pilot.
- **Appeals/Clarifications:** Not in MVP. If teams have disputes or questions post-judging, handle offline.
- **Multiple Mentors per team:** MVP assumes one mentor. This can be extended later if needed.
- **Advanced COI logic:** MVP uses manual judge opt-out. More complex automation (like mapping schools to avoid assignment) is deferred.
- **UI Polish:** If time is short, some UI beautification or convenience features (like profile settings, password reset, etc.) might be skipped, focusing on functionality.

This release plan balances speed and risk by delivering the most critical pieces first and ensuring we always have a usable system at the end of each week (even if limited). By following this plan, we aim to have a working MVP for the AIEC contest ready in 3 weeks. Good luck to the dev team – let’s make it happen!
