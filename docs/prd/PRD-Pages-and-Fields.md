# Pages and Fields Dictionary

**Introduction:** This document defines page-level UX requirements and lists all form fields with their validation rules for the AIEC platform (MVP). It is organized by user role and functional area. Each section describes the content of the page, key UI components, and the data fields (including data types, constraints, and whether they are required). This ensures consistency between the front-end implementation and the product requirements. It is meant for both developers (to implement forms) and contest operations (to understand what information is collected).

## 1) Portal (Public)

These pages are accessible without login, providing contest information and entry points for users to register or log in.

### 1.1 Home (Landing Page)

**Components:**
- **Hero Section:** Shows the contest name, current edition tagline or highlights, and prominent call-to-action buttons (e.g. “Register” and “Login”).
- **Timeline:** A summary of key dates for the current edition (registration open/close, submission deadlines for stages, event dates). This could be a horizontal timeline or list of phases.
- **News Feed:** Displays the latest ~5 news posts or announcements (title, date, and a short snippet) with link to read more.
- **FAQ Preview:** A selection of top 5–10 Frequently Asked Questions with brief answers, and a link to the full FAQ page.

**Data fields displayed:**
- contest.name (string) – e.g. “High School AI Entrepreneurship Contest”.
- edition.year (int) – e.g. 2025 Edition.
- edition.registration_open_at / registration_close_at (datetime) – used to show if registration is open or closed.
- For timeline: each stage.name and its key dates (start, deadline).
- For news items: news.title (string), news.published_at (datetime), news.snippet (string preview).

### 1.2 Edition Detail Page

A public page (or section) with detailed information about the contest edition.
- **Rules Download:** Link to download the contest handbook or rules PDF.
- **Submission Requirements Summary:** A read-only summary of what participants will need to submit (e.g. listing required materials for each stage, so teams know what to prepare).
- **Contact Info:** Contact email or support info for inquiries. Possibly an embedded form for questions.

*(Additional public pages may include a full FAQ page, an About page for OTLF, etc., but those are static content pages not described in detail here.)*

## 2) Participant Portal (Students/Teachers)

After login, students and teachers see a portal interface specific to their needs. This includes team management and submission workflow pages.

### 2.1 Dashboard

Upon logging in, a student or teacher lands on their Dashboard.
- **My Teams List:** If the user is part of one or more teams, list them here. Each entry shows team.name, the contest edition, and team status (e.g. DRAFT, APPLIED, ACTIVE, etc.). It may also show payment status if applicable (e.g. Paid/Pending, though payment is P1/out of scope for MVP).
- **Stage Checklist:** For the selected active team (if the user has one active team, or choose one), show a list of stages for that contest edition with status. Each stage entry might display stage.name, its deadline, and the team’s submission status for that stage (Not started / In Progress / Submitted / Locked). This gives teams a quick overview of what’s next and whether they have completed required submissions.
- If the user is not yet in a team, the dashboard should prompt them to **Create a Team** or **Join a Team** to get started.

### 2.2 Create Team Page

Form for creating a new team (accessible to Students or Teachers who are not already in a team for that edition).
**Fields:**
- edition_id (UUID or numeric ID) – hidden or selected via a dropdown if multiple contest editions are open. **Required.**
- team_name (string, 2–80 characters) – **Required.** The team name must be unique within the contest edition.
- team_intro (string, up to 500 characters) – Optional short introduction or tagline for the team.

**Validation:**
- **Team Name Uniqueness:** The system checks that no existing team in the edition has the same name.
- **Role Requirement:** Only users with the Student or Teacher role can create teams (Judges/Admins cannot). The creator will automatically become part of the team (as leader if student, or as mentor if a teacher is creating on behalf of students). In MVP, primarily students create teams; teacher-initiated team creation is an edge case.

On successful creation, a unique join_code is generated and displayed, and the team status is initially **DRAFT** (not yet applied).

### 2.3 Join Team Page

Form for joining an existing team via invite code.
**Fields:**
- join_code (string, typically 6–12 characters) – **Required.** The code provided by a team leader or the system.

**Validation:**
- The join code must exist and correspond to an active team that is still accepting members. If the code is invalid or the team is full, show an error.
- Team size constraint: joining should be disallowed if the team already has the max number of members (8).
- A user cannot join two teams in the same edition – if they are already in a team for that contest, prevent joining another.
- Role check: If a Teacher (mentor) is using the code, allow it (they will be set as the team’s mentor if not already assigned). If a Student is using it, they become a team member. (If an Admin tries, probably not applicable; admins manage via console instead.)

On success, add the user to the team and show them the team management page.

### 2.4 Team Management Page

This page shows details of the team and allows certain actions by team members.
**Read-Only Information:**
- Team Name, Edition.
- Team Join Code (display for reference, so members can invite others).
- List of team members (names and maybe role tags if assigned). The mentor (teacher) is identified as such.
- Team Status (DRAFT vs ACTIVE, etc.). If DRAFT (application not submitted yet) or if the application was rejected, that might be indicated.

**Actions/Buttons:**
- **Copy Invite Code:** Easy way to copy the join code to clipboard to share with new members. (Only available if team is still open for joining; might disable after application submitted or after a certain date.)
- **Remove Member:** Team leader can remove a member (a student) if needed. Possibly an “X” or remove button next to each student’s name. (Cannot remove the mentor via this interface – mentor can only be changed by admin or by another mentor joining, depending on policy.)
- **Assign Roles to Members:** (If not already done at application submission time) The team leader can edit each member’s role tags (Tech/Business/Impact/Design/Ops/Leader). MVP might simply collect these during application submission and display them here.
- If team has not submitted application yet: perhaps a link or button to “Complete Application” from this page.

### 2.5 Mentor Binding Page

If a team does not yet have a mentor, this interface helps them bind one (for instance, if a mentor registered separately after the team was created). This could be part of the Team Management page or a separate step in the Application form.
**Fields:**
- mentor_teacher_user_id (integer or UUID) – **Required.** In MVP, the simplest method is to have the mentor join via code themselves. Alternatively, an admin or team leader could select a registered teacher from a dropdown to assign as mentor. Given MVP scope, the expected flow is: the mentor uses the join code to join the team. So an explicit form field may not be used by students. (If needed, an admin interface can bind a mentor by selecting from list of teacher accounts.)

**Rules:**
- The user being bound must have the Teacher role (the system should verify the selected user or code belongs to a teacher account).
- Only 1 mentor per team in MVP. If a mentor is already assigned, binding a new one could replace the old one (with a confirmation).
- Mentor must be assigned before application submission is allowed (the system should enforce this in the application submission step).

### 2.6 Contest Application Page

Once the team is assembled and ready, they fill out the contest participation application (Registration stage submission). This is a one-time form per team.
**Fields:**
- **Project Abstract:** project_abstract (text, up to 500 words in English) – **Required.** This is a summary of the project idea, covering problem, AI solution, and business concept. It must be in English and cannot exceed 500 words (the system can count words).
- **Member Role Statements:** For each team member, a sub-section:
- user_id (hidden, identifies the member).
- role_tags (enum list) – **Required.** One or more roles the member will take. Options: **TECH**, **BIZ**, **IMPACT**, **DESIGN**, **OPS**, **LEADER**. (Leader can be combined with one of the others if the team leader also has a functional role.) These tags are based on the contest’s 3I framework (Tech, Business, Social Impact) plus Design and Operations for additional roles, and Leader to mark team captain.
- role_statement (text, 50–500 chars) – **Required.** A short description written by the member about what they will do on the team. Must be at least 50 characters to be meaningful.
- **Mentor Declaration:** mentor_declaration_ack (boolean) – **Required.** The team mentor (teacher) must have acknowledged the project. In practice, this can be a checkbox like “Our mentor has reviewed and approved this application.” The mentor’s presence on the team is required, so this box must be checked (true) to submit.
- **Integrity & Originality:** integrity_originality_ack (boolean) – **Required.** A checkbox like “We affirm that this project is our original work and we will abide by all contest rules.” Must be true to submit.

**Submission Rules:**
- Mentor must be bound to the team before submission (if not, prompt “Please add a mentor to your team before submitting”).
- Team size must meet min/max (5–8). If not, they cannot submit (e.g. “Team must have between 5 and 8 members to submit”).
- Required role coverage: The system checks that among role_tags of members, the essential categories are covered – at least one **TECH**, one **BIZ**, one **IMPACT**, and one **LEADER**. (Leader might be a single person who could also be the one covering tech or biz, etc., that’s fine as long as roles are assigned appropriately.) This ensures interdisciplinary balance as per contest rules.
- All text fields must be within limits (e.g. abstract <= 500 words, each role statement >= 50 chars). Possibly provide a word/char count UI.
- Both acknowledgment checkboxes must be checked.

Upon submission, the Application is saved with status SUBMITTED. The team can no longer edit these fields unless an admin rejects the application (which would reset status to DRAFT for resubmission).

### 2.7 Stage Submission Pages

For each contest stage (Preliminary Round, Semi-Final, Final, etc.), teams that are eligible (i.e. active and/or advanced to that stage) will have a submission page.

If a stage is open for submissions, the team’s dashboard or stage list will show a **“Submit”** or **“View Submission”** button. Clicking it goes to the submission page for that stage.

**Stage Submission Page Features:**
- **Submission Version List:** Show past submission versions if any (v1, v2, ... with timestamps). If a submission has been made, list them so the team can see what was submitted. The latest version status (DRAFT or SUBMITTED) is indicated.
- **Create New Version:** If the stage is still open and the team wants to update their submission, they can start a new version. (MVP could allow at least one submission; if multiple submissions are allowed by contest rules, this is how they’d do it.)

When creating or editing a submission (version):

**Common Submission Fields/Metadata:**
- submission_id (auto-generated) – identifies the submission record (for an existing draft or new one).
- stage_id – identifies which stage this submission is for. (The stage context is usually known from the page, so not a user input, but included in the API calls.)
- version_no – the system-assigned version number (1 for first submission, 2 for resubmission, etc.).
- status – DRAFT or SUBMITTED (the form will be in DRAFT until user finalizes).

**Submission Items:** The page dynamically lists the required and optional items for this stage, based on the stage configuration. For each item, the form will differ depending on content_type:
- If item.content_type = FILE: The form shows a file upload control. Allowed file extensions and max size are indicated (from allowed_exts and max_file_mb for that material type). If multiple files are allowed for that item (min_files, max_files), the UI allows multiple attachments (e.g. “Upload up to 3 images”).
- If item.content_type = LINK: The form shows a URL input field (or multiple if needed). Possibly also instructions on what kind of link (e.g. “YouTube video link for demo”).
- If item.content_type = TEXT: The form shows a textarea for input. It should enforce any min_words / max_words constraints for that item. For example, an **Ethics Statement** item might be TEXT with min_words=150, max_words=200.

Each item is identified by its material_type_code (like BP, MVP_VIDEO, etc.) and label. The user provides the required content for each: upload file(s), paste a link, or enter text as appropriate.

**Example Stage (Preliminary) Requirements:**
- Business Plan – content_type: FILE (PDF), required, max 1 file, allowed_exts “.pdf,.docx”, max_pages perhaps 15 (if checking), etc.
- Prototype Video – content_type: LINK (URL to video), required, e.g. YouTube link.
*(In this case, these two would be listed on the page for Preliminary round.)*

**Example Stage (Finals) Requirements:**
- Pitch Deck – FILE (PowerPoint or PDF), required.
- Live Demo Video – FILE or LINK, maybe optional if live presentation is primary.
- Technical Note – TEXT or FILE, required (e.g. fill a table of AI technical details or upload a doc).
- Ethics Statement – TEXT, required (150–200 words).
- ... etc. (just illustrative)

**Validation (when submitting):**
- All **required** items must have content provided. If any required file not uploaded or required text empty, the submit action is blocked.
- Files should be validated client-side for allowed type/size before upload. The backend will also enforce these constraints.
- If submission is after the stage deadline + grace period, it should be rejected (unless admin extended). The submit button might be disabled on UI after deadline.
- Once the team clicks **Submit** for that version, mark it SUBMITTED and record timestamp. After submission, they should not be able to modify that version’s content. If within the allowed time and multiple versions are allowed, they can start a new version. If not, they are done for that stage.

### 2.8 View Results & Feedback

After a stage’s results are published (especially after Preliminary and Semi-Final), teams can view their outcome and feedback.

For each stage (once available):
- **Stage Result:** Indicate if the team advanced or not (for elimination rounds) or what place they got (for final or any ranking). For example, “Status: **Advanced to Semi-Final**” or “Status: **Did Not Advance**” with maybe their rank or score. In final, it might show “Award: **Honorable Mention**” etc.
- **Final Score:** Optionally show the team’s total score for that stage (if contest policy allows). Possibly show the score breakdown by criterion if deemed appropriate (or just overall).
- **Feedback Comments:** List of anonymized judge feedbacks. For each judge (identified as “Judge 1”, “Judge 2”, ...), display their overall comment and perhaps their criterion scores. (The identity of judges is hidden, as per contest rules – feedback is given but judge names are not disclosed.)
- **Download Feedback Pack:** A button to download a compiled PDF of all feedback and scores. (MVP can generate a simple PDF or provide a print view.)

This page ensures teams get the benefit of judges’ insights. Only team members and their mentor can access their own feedback. Admin can see all teams’ results in the admin console.

## 3) Judge Portal

Judges log in to a dedicated interface that shows them their assignments and allows submitting scores.

### 3.1 Assignments List

**My Assignments:** This page lists all projects (per stage) assigned to the logged-in judge that are pending review or completed. For each assignment entry:
- anonymous_code – the code identifying the team’s submission (e.g. “Proj-7A”).
- stage_name – e.g. “Preliminary Round” or “Semi-Final”. If multiple contest editions, also identify edition.
- due_at – the scoring due date/time (usually the stage’s judging deadline).
- status – ASSIGNED (not yet started), IN_PROGRESS (maybe if partial saved), SUBMITTED (completed), or DECLINED (if judge excused themselves).

The judge can click on an assignment to open the review interface. There may also be an action to **Decline** right from this list (for COI) with a reason selector.

### 3.2 Submission Review Page

When a judge opens an assignment, they see the project’s submission package and the scoring form.
**Read-Only Project Info:**
- anonymous_code displayed prominently, plus possibly some basic info like contest edition and stage. No team-identifying info.
- The submission materials are presented for review: for each item, provide a link or embedded view:
- If it’s a file (PDF, etc.), offer a **Download** link (which generates a secure signed URL for the judge to download the file). Possibly embed PDF viewer if feasible.
- If it’s a link item (e.g. a video URL), present it as a hyperlink or embedded video player if possible.
- If it’s text item, display the text directly on the page.
- Essentially, judges should be able to access all parts of the team’s submission easily from this page.

- Also include any special instructions for judges (e.g. rubric hints or a note like “Please ensure you provide at least X words of feedback”).

**Rubric Form:**
Below the materials, the rubric for this stage is displayed as a form.

### 3.3 Score Submission Form

**Fields:**
- rubric_item_scores[]: A repeated field for each criterion in the rubric:
- criterion_id – hidden ID of the rubric criterion (e.g. “Technical Feasibility” criterion).
- Score input – could be a numeric slider or dropdown or free input (decimal allowed if e.g. scale 1–10 with one decimal). **Required**. Must fall within the rubric’s min/max (e.g. 1.00 to 10.00). The UI might show the range and step.
- overall_comment (text area) – **Required.** The judge’s overall comments and feedback to the team. Must meet the minimum word count for this stage (the requirement is enforced before submission).

**Validation:**
- All required criteria must have a score. If any missing, cannot submit.
- Score values must be within allowed range (if outside or not a number, show error). Possibly enforce one decimal precision if needed.
- The overall_comment must have at least the minimum number of words as configured for the stage (e.g. 80 words for Semi-Final). The form can display a live word count.
- If the judge tries to submit with too short a comment or missing scores, highlight those fields.

After filling in, the judge clicks **Submit Scores**. If before the deadline, they might get a confirmation and can still edit later (depending on implementation). The status of that assignment becomes SUBMITTED. Judges can log back in and edit their submitted feedback until the admin locks the scores (MVP optional feature – could allow editing until deadline passes).

**Conflict of Interest (COI) Decline:**
If the judge identifies a conflict or cannot evaluate, they use the **Decline Assignment** option instead of scoring. This might be a button “Decline due to conflict” which triggers:
- A prompt for decline_reason_code (enum, e.g. “KNOW_TEAM” (I know this team personally), “NOT_EXPERT” (not qualified in topic), etc.) and decline_reason_text for details. Both are required to confirm decline.
- Once confirmed, the assignment is marked DECLINED and the judge no longer needs to score it. The admin is alerted to reassign that project to another judge. Judges should do this as early as possible to not delay the process.

*(Judges do not have a concept of saving a draft vs submit in MVP aside from just not clicking submit. If needed, auto-save can be implemented for their convenience.)*

## 4) Admin Console

Admins have a comprehensive web interface to configure and monitor the contest.

### 4.1 Contests, Editions & Stages Management

Admins can create and edit contest structure:
- **Create Contest:** (If needed) add a new contest (name, description). For AIEC this may already exist, so not frequently used.
- **Create Edition:** Set up a new edition for a contest (e.g. “2025”). Input fields: year, title, registration open/close dates, etc. Status (Draft/Open/Closed/Archived). After creating an edition, it will initially be in Draft status (not visible to users) until ready.
- **Manage Stages:** Within an edition, admins can add stages. Fields for a stage: name (e.g. “Preliminary Round”), type (dropdown: Registration, Preliminary, Semi-Final, Final, etc.), start_at, deadline_at, grace_period (in minutes), and lock_at (which might auto-calc as deadline + grace). Also attach a rubric (select from existing or create new rubric on the fly).
- **Configure Requirements:** For each stage, define required submission items. Admin selects from predefined material types or creates new: e.g. choose “Business Plan” file, mark it required, 1 file max; choose “Pitch Deck” file, required or optional; “Demo Video Link”, etc. Define any min_words if TEXT type. This is done through a form that likely lists all available material types with checkboxes to include and fields for the constraints.
- **Configure Rubric:** Admin can create/edit rubrics. For a rubric: set name, scoring scale (e.g. min 1.00, max 10.00), minimum comment words. Then add criteria: each criterion has a dimension/category (could be something like “Business Viability” or “Technical Innovation”), a short name, and weight (as a fraction or percent that will sum to 1 or 100%). The weights of all criteria in a rubric should sum to 1.0 (or 100%). For example: Business Viability 0.30 (30%), Technical Feasibility 0.25, User Validation 0.20, Presentation 0.25 – summing to 1.0. The system should validate the total weight when saving. Once a rubric is attached to a stage and judging has started, editing it should be locked (to avoid changing mid-contest).

*(These configurations correspond to the underlying database tables **contests**, **contest_editions**, **stages**, **stage_requirements**, **rubrics**, **rubric_criteria**.)*

### 4.2 Team Applications Approval

Admins see a list of team **Applications** (those submitted in the Registration stage) that are pending review. For each application: team name, list of members, abstract, etc., and status SUBMITTED. Admin can click into an application to view details including the project abstract and all role statements, and whether mentor is present.

Actions:
- **Approve Application:** This marks the team as ACTIVE (they are officially entered into the contest). The application record’s status becomes APPROVED, and reviewed_by and reviewed_at are set. The team can then proceed to stage submissions. Possibly trigger an email to team confirming acceptance.
- **Reject Application:** Admin can reject if the application is incomplete or not meeting criteria. This requires providing a rejection reason/comment (e.g. “Abstract is too vague” or “Team missing required roles”). The application status becomes REJECTED, rejection_reason stored, and team is notified. The team should then be allowed to edit and resubmit their application. (In UI, a rejected application might allow the team to adjust their abstract or roster and resubmit.)

This ensures only eligible teams participate.

### 4.3 Judge Assignment Generation

For each stage that requires judging, the admin can use an **Assignments** tool:
- Select a stage (e.g. Preliminary Round).
- Input parameters: judges_per_project (N) and optionally max_load_per_judge (M). E.g., assign 3 judges per project, and don’t give any judge more than 5 projects.
- Click **Generate Assignments**. The system will then create assignment records for that stage. It should output how many assignments were created (e.g. 60 assignments for 20 projects and 10 judges). If the judge pool is not enough to fulfill criteria, the system might warn (e.g. not enough judges to cover all projects with 3 each). Admin may then adjust or manually assign additional judges.

After generation, admin can review assignment lists (which judge got which anonymous code). They can also manually adjust if needed (maybe an interface to remove or add a particular judge to a project – MVP optional). If any judges decline, this interface is where an admin sees DECLINED and can assign a different judge.

### 4.4 Scoring Progress & Monitoring

Admin dashboard for a stage in judging:
- Show a list of all assignments for that stage with their status. This can be aggregated per team or per judge. For example, a table: each judge and how many of their assignments are submitted vs pending. Or each team and how many judge scores are in vs missing.
- **Completion Rate:** e.g. “45/60 evaluations submitted (75%)”.
- **Overdue Judges:** After the judging deadline, highlight any judges who have not submitted and which assignments are pending. Admin can then follow up offline or reassign if absolutely needed.

This helps ensure all scores come in on time.

### 4.5 Score Aggregation & Results Publishing

Once judging is done for a stage:
- Admin clicks **Aggregate Scores** for the stage. The system performs the aggregation: calculates each team’s average score (applying weights and outlier removal) and determines advancement cutoffs if applicable. The results are stored in a score_aggregation table and stage_results table for teams. The admin is shown a summary, e.g., a ranking of teams by final score. If any manual adjustment is needed (for tie-breaks or borderline cases), admin can export the data or the system might allow marking an additional team to advance. (Any manual changes should be noted outside the system or in an override field.)
- Aggregation also locks the judge scores (so they can’t be changed after the fact). If a judge tries to edit after aggregation, the system should prevent it.
- After verifying results, Admin uses **Publish Results** action. Options: publish visibility can be “TEAM_ONLY” (default – only teams see their own result and feedback) or “PUBLIC” (if we want to publish a public leaderboard or list of finalists). Typically, for Preliminary and Semi, we might do team-only first, then a separate public announcement listing who advanced. For Finals, publishing might directly announce winners publicly.
- The Publish action time-stamps the results as published and perhaps triggers emails to participants (MVP can skip email or just rely on them logging in).

Publishing finalizes the stage. Teams can then view their results in their portal (as described in 2.8). Judges might also gain access to see who won after everything, depending on policy (not critical for MVP).

### 4.6 News/Announcements CMS

Admins can post news updates that appear on the public site or in user dashboards.
- Create News Post: Fields: edition_id (or global), title, content (rich text or Markdown), pinned_flag (whether it should stay at top), published_at. They can save as draft or publish immediately. Published news becomes visible via the public API and on the site.
- Edit/Delete News: Admins can edit content of a post or remove it. (If removed, it might be a soft delete – marking as deleted or just not returned in queries.)
- The news content can include contest announcements like “Semi-Finalists Announced” or general updates. This CMS is simple but important for communication.

### 4.7 Audit Logs Viewer

A page where admins can search and view audit log entries.
- Filter by actor (user), action type, target (e.g. team or stage), or date range.
- The results show entries like “2026-01-15 10:05:23 – Admin (user123) PUBLISHED results for Stage ID 45” or “2026-01-10 09:20:10 – Judge (user456) SUBMITTED scores for AnonymousCode ABC123”.
- Being able to review these logs helps in investigating any issues (for example, if a team claims their submission was altered, admins can see if any edits or admin overrides were done).

*(Additional admin functions that might exist but are minimal in MVP: user management (changing roles, resetting passwords), but these can also be done via database if not front-facing.)*
