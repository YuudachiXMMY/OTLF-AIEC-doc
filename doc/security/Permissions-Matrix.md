# Permissions Matrix (MVP)

**Introduction:** This document defines the role-based access control (RBAC) for the AIEC platform. It lists what actions each user role is permitted to do, ensuring that students, teachers (mentors), judges, and admins each have appropriate access. The matrix helps developers implement security checks and helps contest organizers understand the visibility scope of each role.

**Legend (Action Types):**
- **R:** Read access (viewing data)
- **W:** Write access (create/update/delete)
- **X:** Execute or administer (special privileged actions, mainly admin-only)

**Scope qualifiers:**
- **SELF:** The user’s own data only.
- **TEAM:** The user’s own team’s data. (For mentors, the team they mentor; for students, their team.)
- **ASSIGNED:** Only items explicitly assigned to the user (for judges).
- **GLOBAL:** All data in the system (admin-level).

Below is the matrix of capabilities vs. roles:

| Capability | Student | Teacher/Mentor | Judge | Admin |
| --- | --- | --- | --- | --- |
| View public site content (home, FAQ, etc.) | R | R | R | R |
| Register an account / Login | W (SELF) | W (SELF) | – (N/A, created by admin) | – (N/A, created by admin) |
| Team Formation |  |  |  |  |
| Create a new team | W (TEAM) | W (TEAM) | – | X (GLOBAL) (can create teams for testing or on behalf) |
| Join an existing team | W (TEAM) | W (TEAM) | – | X (GLOBAL) (can assign users to teams) |
| Bind mentor to team | – (request via code) | W (TEAM) (mentor can join team) | – | X (GLOBAL) (admin can assign mentor) |
| Remove a member from team | W (TEAM) (leader only) | – | – | X (GLOBAL) |
| View team details (members, code) | R (TEAM) | R (TEAM) | – | R (GLOBAL) |
| Contest Application |  |  |  |  |
| Fill & submit team application | W (TEAM) | W (TEAM) | – | X (GLOBAL) (on behalf or override) |
| Approve/reject team application | – | – | – | X (GLOBAL) |
| View own application status & info | R (TEAM) | R (TEAM) | – | R (GLOBAL) |
| Stage Submissions |  |  |  |  |
| Create or edit stage submission (draft) | W (TEAM) | W (TEAM) | – | X (GLOBAL) (admin could edit if needed) |
| Submit/lock stage submission for judging | W (TEAM) | W (TEAM) | – | X (GLOBAL) (admin could force submit) |
| View own team’s submissions & feedback | R (TEAM) | R (TEAM) | – | R (GLOBAL) |
| Withdraw a submission (pre-deadline) | W (TEAM) | W (TEAM) | – | X (GLOBAL) |
| Judging |  |  |  |  |
| View assigned review packages (anon submissions) | – | – | R (ASSIGNED) | R (GLOBAL) |
| Download assigned submission files | – | – | R (ASSIGNED) | R (GLOBAL) |
| Submit scores & comments for assignment | – | – | W (ASSIGNED) | X (GLOBAL) (admin can input/edit scores if needed) |
| Decline a judging assignment (COI) | – | – | W (ASSIGNED) | X (GLOBAL) (admin can reassign) |
| View other judges’ scores or feedback | – | – | – | R (GLOBAL) |
| Results & Feedback |  |  |  |  |
| View own team’s results and judge feedback | R (TEAM) | R (TEAM) | – | R (GLOBAL) |
| Download feedback pack (own team) | R (TEAM) | R (TEAM) | – | R (GLOBAL) |
| View overall contest results (public winners) | R (GLOBAL)  (only after public release)* | R (GLOBAL) | R (GLOBAL) | R (GLOBAL) |
| Administrative Functions |  |  |  |  |
| Configure contests/editions/stages | – | – | – | W (GLOBAL) |
| Configure submission requirements | – | – | – | W (GLOBAL) |
| Configure rubrics | – | – | – | W (GLOBAL) |
| Manage users (assign roles, create judge/admin accounts) | – | – | – | W (GLOBAL) |
| Generate judge assignments | – | – | – | X (GLOBAL) |
| View judging progress (who submitted, etc.) | – | – | – | R (GLOBAL) |
| Aggregate scores and produce results | – | – | – | X (GLOBAL) |
| Publish stage results | – | – | – | X (GLOBAL) |
| Create/edit news posts (announcements) | – | – | – | W (GLOBAL) |
| Pin/unpin or delete news posts | – | – | – | W (GLOBAL) |
| View audit logs | – | – | – | R (GLOBAL) |

*(In the above, a dash "–" means no access by that role. X (GLOBAL) indicates only admins can perform, since those are high-privilege operations.)*

**Narrative Summary by Role:**
- **Student:** Can register/log in, create or join a team (one team per edition), collaborate with team on submissions, and view their own team’s status and feedback. Cannot see other teams’ data or any judging info. No access to admin or judge functions.
- **Teacher/Mentor:** Similar to student for team functions – can help fill in application and submissions for the team they mentor. Essentially same permissions on the team’s data as students have. Cannot create or join multiple teams (one team to mentor). No judging or admin privileges.
- **Judge:** Cannot create teams or view any team info except through their assignments. Judges only get read access to the submissions assigned to them (through the anonymous code). They can download files and submit scores for those assignments. They cannot see team identities or any data beyond their assignments. They cannot see results until those are publicly released (and even then, they may only see high-level results, not other judges’ individual scores). Judges have no access to admin functionality.
- **Admin:** Full read/write across the system. Admins set up contests, manage all teams and users, and oversee the judging process. They can impersonate or directly modify any data if needed (though direct data changes should be done through provided admin functions to log actions). Admins are the only ones who can view all submissions globally (for auditing) and all scores. They also manage publishing of results and system content like news. Admin actions are typically logged in audit logs. Multi-factor auth is recommended for admin accounts for security.

This matrix should be enforced both in the front-end (not showing disallowed UI elements) and in the back-end (authorization checks on each API). By adhering to these rules, we ensure data is only accessible to those who need it, preserving the contest’s fairness (e.g., teams can’t see each other’s work, judges can’t see team identities, etc.) and integrity of the competition.
