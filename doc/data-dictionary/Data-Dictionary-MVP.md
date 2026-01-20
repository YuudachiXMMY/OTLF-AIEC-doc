# Data Dictionary (MVP)

**Introduction:** This document details the database schema for the AIEC Platform MVP. The database is MySQL 8.0+ with InnoDB storage and utf8mb4 character set. All tables are defined to support the contest management features (teams, submissions, judging, etc.). This section lists each table, its columns (with types and constraints), and relevant indexes/keys. It helps developers understand how data is structured and related, and assists database administrators in maintenance and optimization.

**General Notes:**
- Every table has a primary key id (BIGINT UNSIGNED AUTO_INCREMENT) unless otherwise noted.
- Common audit columns: most tables include created_at and updated_at timestamps for auditing. These are set by the application (or via triggers) on insert/update.
- Soft delete: We prefer explicit status flags over deleting rows. Only a few tables (e.g. news, users) might have a soft delete flag or status. Otherwise, data is retained with statuses like INACTIVE or ARCHIVED rather than removed.
- Naming conventions: Table names are plural (e.g. users, teams), columns use lowercase_with_underscores. Foreign keys are named like <referenced_table_singular>_id.

## 1) **users**

Stores all user accounts for the system (students, teachers, judges, admins).
- **id** – BIGINT UNSIGNED, primary key.
- **email** – VARCHAR(255), unique. User’s login email.
- **password_hash** – VARCHAR(255). Hashed password.
- **role** – ENUM('STUDENT','TEACHER','JUDGE','ADMIN'). The user’s role in the system.
- **name** – VARCHAR(80). The person’s full name.
- **status** – ENUM('ACTIVE','SUSPENDED'). Active by default; could be suspended by admin if needed.
- **mfa_enabled** – TINYINT (boolean). 1 if MFA is enabled for this user (likely only true for admins in MVP).
- **mfa_secret** – VARCHAR(64), nullable. Stores the base32 secret for TOTP if MFA is enabled (encrypted or encoded).

**Indexes:**
- uq_users_email – UNIQUE(email).

## 2) **contests**

Represents a contest series (e.g. AIEC). Likely only one row (AIEC) for our use, but structured for possible future contests.
- **id** – BIGINT, PK.
- **name** – VARCHAR(200). Contest name (e.g. "AI Entrepreneurship Contest").
- **slug** – VARCHAR(100), unique. Short identifier (e.g. "aiec").
- **intro** – TEXT. Introduction or description of the contest.
- **is_published** – TINYINT (boolean). If false, contest is hidden (for drafts).

*(We can treat contests mostly as static; the main varying entity is contest_editions.)*

## 3) **contest_editions**

Each row is a specific edition/year of a contest.
- **id** – BIGINT, PK.
- **contest_id** – BIGINT, FK to contests(id).
- **year** – INT. Year or sequence (e.g. 2025).
- **title** – VARCHAR(200). e.g. "2025 High School AIEC".
- **registration_open_at** – DATETIME. When team registration starts.
- **registration_close_at** – DATETIME. When team registration (applications) closes.
- **status** – ENUM('DRAFT','OPEN','CLOSED','ARCHIVED'). Status of the edition.
- DRAFT: being set up, not visible to users.
- OPEN: active contest (registration or submission ongoing).
- CLOSED: contest finished (could still be visible).
- ARCHIVED: older edition, mostly read-only.

**Indexes:**
- idx_edition_contest_year – INDEX(contest_id, year). (Unique together ideally, one contest shouldn’t have two editions with same year.)

## 4) **stages**

Contest stages (Application, Preliminary, Semi, Final, etc.) for each edition.
- **id** – BIGINT, PK.
- **edition_id** – BIGINT, FK to contest_editions(id).
- **name** – VARCHAR(100). Stage name (e.g. "Preliminary Round").
- **stage_type** – ENUM('REGISTRATION','PRELIM','SEMI','FINAL', 'OTHER'). A code for logic. 'REGISTRATION' for the team application stage, others for judging stages.
- **start_at** – DATETIME. When this stage opens (e.g. when teams can start submitting).
- **deadline_at** – DATETIME. Submission deadline (for REGISTRATION, the application deadline; for judging stages, submission deadline).
- **grace_minutes** – INT, default 0. How many minutes after deadline we still accept submissions (grace period).
- **lock_at** – DATETIME. When the stage truly locks. Typically equals deadline + grace. Could also be when judging locks for editing.
- **rubric_id** – BIGINT, FK to rubrics(id). The scoring rubric used for this stage (if stage involves judging). Null for Registration stage (no judging).

**Indexes:**
- idx_stage_edition_type – INDEX(edition_id, stage_type). (We assume each edition has one 'REGISTRATION', one 'FINAL', etc.)

## 5) **material_types**

Reference table listing possible types of submission materials. Populated with entries like Business Plan, Video, etc., which define acceptable file types, etc.
- **id** – BIGINT, PK.
- **code** – VARCHAR(50), unique. Short code for material (e.g. 'BP', 'PITCH_DECK', 'MVP_VIDEO').
- **label** – VARCHAR(100). Human-readable name (e.g. "Business Plan Document").
- **allowed_exts** – VARCHAR(200). Comma-separated list of allowed file extensions if type is file (e.g. ".pdf,.pptx").
- **max_file_mb** – INT. Max file size in MB (for file types).
- **max_pages** – INT, nullable. If the item is a document, optional limit on pages (if needed, e.g. limit PPT to 15 slides).
- **max_minutes** – INT, nullable. If item is a video, optional limit on duration in minutes.

(This table helps configure stage requirements without duplicating file rules each time.)

## 6) **stage_requirements**

Defines which material types are required or allowed for each stage. Essentially the many-to-many between stages and material_types, with extra rules.
- **id** – BIGINT, PK.
- **stage_id** – BIGINT, FK to stages(id).
- **material_type_id** – BIGINT, FK to material_types(id).
- **required_flag** – TINYINT. 1 if this material is required for submission, 0 if optional.
- **min_files** – INT, default 0. Minimum number of files required (for file type items). Usually 0 or 1.
- **max_files** – INT, default 1. Maximum number of files allowed for this item. E.g. maybe allow up to 3 supplementary images.
- **content_type** – ENUM('FILE','LINK','TEXT'). The type of content expected. Should usually match what the material_type implies (e.g. Business Plan is a FILE, Demo Video might be a LINK, etc.).
- **min_words** – INT, nullable. If content_type is TEXT, minimum word count.
- **max_words** – INT, nullable. If content_type is TEXT, maximum word count.

**Unique Constraints:**
- uq_stage_material – UNIQUE(stage_id, material_type_id). (Each material can appear once per stage.)

## 7) **teams**

Teams participating in a contest edition.
- **id** – BIGINT, PK.
- **edition_id** – BIGINT, FK to contest_editions(id). The edition this team is in.
- **name** – VARCHAR(80). Team name. Must be unique within the edition.
- **join_code** – VARCHAR(12). The invite code used for joining the team. Likely a random alphanumeric string. Unique per edition (to avoid collisions).
- **leader_user_id** – BIGINT, FK to users(id). The team leader’s user id (must be one of the team members who is a student).
- **mentor_user_id** – BIGINT, FK to users(id). The teacher/mentor’s user id (if assigned).
- **status** – ENUM('DRAFT','APPLIED','ACTIVE','WITHDRAWN','DISQUALIFIED').
- DRAFT: Team created but application not submitted.
- APPLIED: Application submitted, pending approval.
- ACTIVE: Application approved, team in contest.
- WITHDRAWN: Team voluntarily withdrew (or didn’t show up).
- DISQUALIFIED: Team removed by admins (for rules violation, etc.).
- **paid_status** – ENUM('UNPAID','PAID','REFUNDED','WAIVED'), optional. If the contest had a fee, track payment. MVP may not use this field (default UNPAID or null).

**Unique Constraints:**
- uq_team_join_code – UNIQUE(edition_id, join_code). (Join code unique per contest edition.)
- uq_team_name – UNIQUE(edition_id, name). (Team name unique within edition.)

## 8) **team_members**

Associative table linking users to teams (many-to-many, but realistically in our model a user can only be in one team per edition). Also stores each member’s role info for the application.
- **id** – BIGINT, PK.
- **team_id** – BIGINT, FK to teams(id).
- **user_id** – BIGINT, FK to users(id).
- **role_tags** – VARCHAR(200). Comma-separated or JSON list of role tags for this member (as provided in application). E.g. "LEADER,TECH". (P1: could normalize into a separate table if needed, but storing as text is simpler for MVP.)
- **role_statement** – TEXT. The member’s self-described role statement from the application. Could be null if not provided yet (before application).

**Unique Constraint:**
- uq_team_user – UNIQUE(team_id, user_id). (Each user can appear at most once in a team.)

*(We assume if a user leaves a team, we would remove or mark that entry. MVP might not implement leaving a team after joining except by admin removal.)*

## 9) **applications**

Stores the team’s contest application (initial registration submission).
- **id** – BIGINT, PK.
- **team_id** – BIGINT, FK to teams(id). One-to-one: each team has one application.
- **project_abstract** – TEXT. The submitted project abstract (up to 500 words).
- **status** – ENUM('SUBMITTED','APPROVED','REJECTED'). Current status of the application.
- **submitted_at** – DATETIME. When the team submitted the application.
- **reviewed_by** – BIGINT, FK to users(id), nullable. The admin user who reviewed (approved/rejected) the application.
- **reviewed_at** – DATETIME, nullable. When the admin reviewed (approved/rejected).
- **rejection_reason** – TEXT, nullable. If rejected, a message explaining why, so team can address it.

*(Note: We could also derive status from team.status (APPLIED vs ACTIVE etc.), but keeping here for clarity of the application process.)*

## 10) **submissions**

Represents a team’s submission for a particular stage. Allows multiple versions per stage if needed.
- **id** – BIGINT, PK.
- **team_id** – BIGINT, FK to teams(id).
- **stage_id** – BIGINT, FK to stages(id).
- **version_no** – INT. Version number of the submission for that stage (starting at 1).
- **status** – ENUM('DRAFT','SUBMITTED','LOCKED','INVALID').
- DRAFT: still being edited, not finalized.
- SUBMITTED: team marked this version as final (before deadline).
- LOCKED: past deadline or aggregated – no changes allowed. (System may change SUBMITTED to LOCKED when judging starts.)
- INVALID: (optional) if a submission was found to violate rules, could mark it invalid.
- **submitted_at** – DATETIME, nullable. When this version was submitted (null if still draft).

**Unique Constraint:**
- uq_submission_version – UNIQUE(team_id, stage_id, version_no). (A team can’t have two submissions with same version number for the same stage. Exactly one submission with version_no=1, etc.)
*(The latest submission for a stage would be the one with highest version_no, typically.)*

## 11) **submission_items**

The individual content items attached to a submission. For example, one submission might have two items: a PDF file and a video link.
- **id** – BIGINT, PK.
- **submission_id** – BIGINT, FK to submissions(id).
- **material_type_id** – BIGINT, FK to material_types(id). Indicates what kind of item this is.
- **content_type** – ENUM('FILE','LINK','TEXT'). Redundant but denormalized for convenience (should match the material_type’s nature).
- **text_value** – LONGTEXT, nullable. If this item is a TEXT type, the content goes here.
- **link_url** – VARCHAR(500), nullable. If this item is a LINK type, store the URL here.
- **word_count** – INT, nullable. If text_value is provided, store the word count for quick checking. (This can be computed, but storing can speed up validation, up to the implementation.)

**Unique Constraint:**
- uq_submission_material – UNIQUE(submission_id, material_type_id). (Each submission can only have one item per material type. If multiple files of same type are allowed, they would still be grouped under one item with multiple files via the next table.)

## 12) **files**

Stores metadata for each uploaded file. Actual file content is in object storage; this table tracks references and metadata.
- **id** – BIGINT, PK.
- **storage_key** – VARCHAR(500). The path/key in the object storage where file is stored. This, combined with the storage bucket, locates the file.
- **original_name** – VARCHAR(255). The original filename as uploaded (for reference/display).
- **mime_type** – VARCHAR(120). The content type (e.g. application/pdf).
- **size_bytes** – BIGINT. File size in bytes.
- **sha256** – CHAR(64). SHA-256 hash of the file content (optional but useful for integrity checks and deduplication).
- **uploaded_by** – BIGINT, FK to users(id). Who uploaded the file.
- **virus_scan_status** – ENUM('PENDING','CLEAN','INFECTED'). Default 'PENDING' if we implement scanning; for MVP, might set to 'CLEAN' by default or skip.

**Indexes:**
- idx_files_sha256 – INDEX(sha256). Could be used to detect duplicate files (if two identical files uploaded, we might reuse or just for info).

## 13) **submission_item_files**

Join table for many-to-many between submission_items and files, in case an item allows multiple files. (If each item was strictly one file, we could have put file_id in submission_items, but some items might allow multiple files, hence this table.)
- **id** – BIGINT, PK.
- **submission_item_id** – BIGINT, FK to submission_items(id).
- **file_id** – BIGINT, FK to files(id).

*(No unique needed beyond PK; one submission_item can have multiple file entries.)*

## 14) **anonymous_mappings**

Maps a team to an anonymous code for a specific stage (used for blind judging).
- **id** – BIGINT, PK.
- **edition_id** – BIGINT, FK to contest_editions(id). (Redundant if stage_id given, but for ease of querying by edition.)
- **stage_id** – BIGINT, FK to stages(id).
- **team_id** – BIGINT, FK to teams(id).
- **anonymous_code** – VARCHAR(32). Random code (could be alphanumeric or numeric). Should be unique per stage.

**Unique Constraints:**
- uq_anon_stage_team – UNIQUE(stage_id, team_id). (Each team gets only one code per stage.)
- uq_anon_stage_code – UNIQUE(stage_id, anonymous_code).

*(We generate these when needed, e.g., at Preliminary submission lock time, or at assignment generation time.)*

## 15) **judge_assignments**

Assignments of judges to review a particular anonymous project in a stage.
- **id** – BIGINT, PK.
- **stage_id** – BIGINT, FK to stages(id).
- **anonymous_code** – VARCHAR(32). The code identifying the project (from anonymous_mappings). Could also store mapping id instead for referential integrity, but using code directly is convenient for judges.
- **judge_user_id** – BIGINT, FK to users(id). The judge assigned.
- **status** – ENUM('ASSIGNED','IN_PROGRESS','SUBMITTED','DECLINED').
- ASSIGNED: Default when first assigned.
- IN_PROGRESS: if we want to mark when a judge has started (optional, maybe when they save a draft).
- SUBMITTED: judge completed scoring.
- DECLINED: judge refused due to conflict or other reason.
- **assigned_at** – DATETIME. When assignment was created.
- **declined_reason** – TEXT, nullable. If status=DECLINED, the reason the judge gave for declining.

**Unique Constraint:**
- uq_assignment – UNIQUE(stage_id, anonymous_code, judge_user_id). (One judge per project per stage once.)
*(We might also ensure a judge isn’t assigned the same team across different stages if blind, but that’s contest-specific policy not enforced here.)*

## 16) **rubrics**

Stores a scoring rubric definition (which can be reused for stages).
- **id** – BIGINT, PK.
- **name** – VARCHAR(120). Name of the rubric (e.g. "Preliminary Scoring Rubric").
- **scoring_scale_min** – DECIMAL(5,2). Minimum score value per criterion (e.g. 1.00).
- **scoring_scale_max** – DECIMAL(5,2). Maximum score value (e.g. 10.00).
- **min_comment_words** – INT. Minimum words for the overall comment for this rubric’s stage.

## 17) **rubric_criteria**

The criteria (dimensions) for a rubric. Each criterion represents one aspect being scored.
- **id** – BIGINT, PK.
- **rubric_id** – BIGINT, FK to rubrics(id).
- **dimension** – VARCHAR(80). The broader category of the criterion (e.g. "Technical", "Business", etc.). Could be null or same as name if not grouping. This might be used to group criteria in display or for reference.
- **name** – VARCHAR(120). The specific name of the criterion (e.g. "Technical Feasibility and Innovation").
- **weight** – DECIMAL(6,4). The weight of this criterion in the total score (e.g. 0.2500 for 25%). All criteria weights under a rubric should sum to 1.0 (100%).

*(If needed, we could have max_score per criterion if they differ, but in this design, presumably each criterion uses the rubric’s overall scale. If some criteria were out of, say, 5 and others out of 10, we’d incorporate that into weight or scale accordingly. MVP assumes uniform scale per rubric.)*

## 18) **judge_scores**

Stores a judge’s submitted scoring for an assignment (one entry per judge per project per stage). This could be seen as a join of judge_assignments with their scores, but keeping separate can simplify storing the numeric scores and comment.
- **id** – BIGINT, PK.
- **stage_id** – BIGINT, FK to stages(id). (Redundant but for easy queries by stage.)
- **anonymous_code** – VARCHAR(32). Which project was scored (redundant copy from assignment for ease of lookup).
- **judge_user_id** – BIGINT, FK to users(id).
- **total_weighted_score** – DECIMAL(7,2). The total score this judge gave, after applying weights. (If each criterion score is 1-10 and weights sum to 1, total will be in [1,10]. We allow 7,2 to cover up to 999.99 just in case scales differ.)
- **overall_comment** – LONGTEXT. The judge’s written feedback.
- **comment_word_count** – INT. Word count of the comment (for enforcement and reference).
- **submitted_at** – DATETIME. When the judge submitted their scores.

**Unique Constraint:**
- uq_judge_score – UNIQUE(stage_id, anonymous_code, judge_user_id). (Each judge can submit only one scoring for a given project in a stage.)

*(If a judge edits their score, we update the same record rather than insert new.)*

## 19) **judge_score_items**

Stores the individual criterion scores as part of a judge’s scoring.
- **id** – BIGINT, PK.
- **judge_score_id** – BIGINT, FK to judge_scores(id).
- **criterion_id** – BIGINT, FK to rubric_criteria(id).
- **score_value** – DECIMAL(5,2). The numeric score the judge gave for this criterion.
- **weight_snapshot** – DECIMAL(6,4). The weight of this criterion at the time of scoring (copied from rubric_criteria). We store it here to preserve what weight was used to compute total_weighted_score, in case rubric changes later or for easy recalculation.

*(No unique constraints; judge_score_id + criterion_id uniquely identifies each row logically, but one judge_score will have multiple rows here.)*

## 20) **score_aggregation**

Stores the aggregated results for a stage (scores after combining judges). Each entry is a team’s aggregated outcome in a stage.
- **id** – BIGINT, PK.
- **stage_id** – BIGINT, FK to stages(id).
- **anonymous_code** – VARCHAR(32). The project code. (For reference; could also link to team via mapping but keeping anon to remain blind in data if needed.)
- **final_score** – DECIMAL(7,2). The final averaged score after aggregation (and any weight scaling).
- **dropped_score_ids_json** – JSON. Records which judge_scores (IDs or judge IDs) were dropped as outliers, if any. E.g. [701, 709] or null if none.
- **calculated_at** – DATETIME. Timestamp when aggregation was done.

*(This table might be optional – we could directly compute and fill stage_results, but having it can keep detailed info about how the score was formed.)*

## 21) **stage_results**

High-level results of a stage for each team. This is more about advancement and awards rather than scores (though we might include rank or score for completeness).
- **id** – BIGINT, PK.
- **stage_id** – BIGINT, FK to stages(id).
- **team_id** – BIGINT, FK to teams(id).
- **advance_flag** – TINYINT. 1 if the team advanced to next stage (for non-final stages), 0 if not. Null or 0 for final stage (no next stage).
- **rank_no** – INT, nullable. The team’s rank in this stage (1 = highest score, etc.). If not needed for all, can be null.
- **published_at** – DATETIME, nullable. When this result was published/released to participants (populated when admin hits publish).

*(Awards are handled separately or could be an extension of this table for final stage only. Possibly a separate awards table if multiple awards per team.)*

## 22) **news_posts**

Stores news or announcement posts for display in the portal.
- **id** – BIGINT, PK.
- **edition_id** – BIGINT, FK to contest_editions(id), nullable. If null, the news is global or not specific to an edition.
- **title** – VARCHAR(200).
- **content** – LONGTEXT. The body of the news (could be HTML/Markdown).
- **pinned_flag** – TINYINT. If true, this news should appear pinned (at top of list).
- **published_at** – DATETIME. If null, the news is a draft and not visible to users. If set, this is the date it was made public.

*(No additional index beyond maybe edition filter; list queries can use by edition and published_at desc.)*

## 23) **audit_logs**

Captures important actions in the system for audit trail.
- **id** – BIGINT, PK.
- **actor_user_id** – BIGINT, FK to users(id). The user who performed the action (if applicable; could be null for system events).
- **action_code** – VARCHAR(80). A code for the action, e.g. "USER_LOGIN", "TEAM_CREATE", "APPLICATION_APPROVE", "JUDGE_SCORE_SUBMIT", etc.
- **target_type** – VARCHAR(80). The entity affected, e.g. "TEAM", "APPLICATION", "SUBMISSION", "USER", etc.
- **target_id** – BIGINT. The ID of the entity affected (if applicable).
- **diff_json** – JSON, nullable. Details of the change in JSON form, if capturing before/after or relevant info. For example, could store the changed fields or the content of a judge’s submitted score, etc. This can be schema-less for flexibility.
- **ip** – VARCHAR(64). IP address of the actor (if applicable, for web requests).
- **user_agent** – VARCHAR(255). User agent string of the actor’s request (to track device/browser if needed).
- **created_at** – DATETIME. Timestamp when the action occurred (or when logged).

**Indexes:**
- idx_audit_actor_date – INDEX(actor_user_id, created_at). Allows querying actions by user or timeframe.

*(This table can grow large; might consider archiving old logs after some time.)*

**Relationships & Notable Constraints:**
- A **user** can be in **team_members** multiple times across different teams only if those teams are in different editions (the app logic prevents joining 2 teams in one edition). We do not enforce that at DB level, but application logic ensures user_id not in two teams of same edition.
- **mentor_user_id** in teams refers to a user with role TEACHER. Similarly, team leader typically role STUDENT (not enforced in schema, but by logic).
- When a team is approved (teams.status ACTIVE), there must be an application entry that is APPROVED. We rely on application status to track that.
- Deleting or changing any contest configuration (like removing a stage or criterion) after data exists can break things. The system should disallow removing things that have been used (or handle cascading carefully). MVP assume once contest is running, config is mostly fixed.
- All score-related data (assignments, judge_scores, score_items, aggregation, results) are tied to stage. If a team is disqualified or withdraws, we might not remove their data but mark them accordingly; their submissions could remain but not be counted in results depending on team.status.

**Index/Performance Considerations:**
- We expect frequent queries: e.g., “fetch all teams for an edition” (index on teams.edition_id).
- “fetch all members of a team” (index on team_members.team_id).
- “get all submissions for a stage” (index on submissions.stage_id).
- “get all assignments for a judge or for a stage” (indexes on judge_assignments.judge_user_id, stage_id).
- “score aggregation” will join judge_scores by stage and team, so data is keyed by anonymous_code in many places; ensure that’s indexed appropriately. Possibly an index on judge_scores by stage_id,anonymous_code.
- Many tables have composite unique keys to enforce data integrity which also serve as natural indexes for lookups.

This data model covers the core MVP requirements. It is structured to allow extension (e.g., adding new roles, new stages, additional submission types) with minimal changes. By following the dictionary during implementation, we ensure consistency between the code (entities/ORM) and the database schema. Future iterations might refine the schema (for instance, to handle multiple mentors via a linking table, or to support appeals), but those can be layered on without altering the fundamental structure described here.
