# API Specification (MVP)

**Introduction:** This document describes the RESTful API endpoints provided by the AIEC platform backend. It covers endpoints for authentication, public contest info, team and submission management, judge operations, and admin functions. All endpoints are prefixed with a base path and use standard HTTP methods. The API returns data in a consistent JSON envelope format. This spec is intended for front-end developers and any external systems that might integrate with the AIEC backend.

## Conventions

- **Base URL:** /api (e.g., if deployed locally, http://localhost:8080/api).
- **Authentication:** Uses Bearer JWT tokens. Include Authorization: Bearer <token> header for protected endpoints after login.
- **Response Envelope:** On success, responses are typically JSON of the form:
{ "code": 0, "message": "ok", "data": {...} }
where data contains the requested resource. On errors,
{ "code": <error_code>, "message": <error_message>, "data": null }.
(Exact format can be adjusted, but consistency is key.)
- **Common Error Codes:**
- 400 – Bad Request (validation_error, missing fields, etc.)
- 401 – Unauthorized (invalid or missing JWT)
- 403 – Forbidden (authenticated but not allowed, e.g., wrong role)
- 404 – Not Found (resource doesn’t exist)
- 409 – Conflict (e.g., duplicate entry, violation of unique constraint)
- 500 – Server error (unexpected issues)
- All request bodies for POST/PUT are JSON. Dates/times are in ISO8601 format (UTC).
- The API is versioned under /api (for MVP, version 1 implicitly; could use /api/v1/ in future).

## 1) Authentication & User Accounts

Endpoints for user registration, login, and admin multi-factor auth.

### POST /api/auth/register

**Description:** Register a new user (Student or Teacher). Judges and Admins are typically not self-registered in MVP.
**Request Body:**

{
  "role": "STUDENT" \| "TEACHER",
  "name": "<Full Name>",
  "email": "<email>",
  "password": "<password>"
}

- role – must be either "STUDENT" or "TEACHER" (case-insensitive).
- name – full name of the user (string).
- email – user’s email (will be the login username). Must be unique.
- password – plaintext password (will be hashed server-side). Minimum length policy (e.g. 8 chars) enforced.

**Response:** on success, returns the created user’s basic info and an auth token (if auto-login after register is desired), e.g.:

{
  "code": 0,
  "message": "ok",
  "data": {
    "user": { "id": 123, "role": "STUDENT", "name": "Alice", "email": "alice@example.com" },
    "token": "<jwt-token>"
  }
}

Alternatively, the API could require a separate login after registration; in that case it might just return user info without token.

### POST /api/auth/login

**Description:** Authenticate an existing user.
**Request Body:**

{ "email": "<email>", "password": "<password>" }

**Response:** On success:

{
  "code": 0,
  "message": "ok",
  "data": {
    "token": "<jwt-token>",
    "user": { "id": 123, "role": "STUDENT", "name": "Alice", "email": "alice@example.com" }
  }
}

The token is a JWT to be used in subsequent requests. The user object is provided for convenience so the client knows the role, etc.

On failure (wrong credentials): 401 Unauthorized with message.

### POST /api/auth/mfa/enable *(Admin only)*

**Description:** Enable TOTP-based 2FA for the admin’s account. (This endpoint requires the admin to be logged in with their password already.)
**Request Body:** *none* (or an empty JSON object).
**Response:**

{
  "code": 0,
  "message": "ok",
  "data": {
    "otpauth_uri": "otpauth://totp/...secret",
    "secret_masked": "ABCD...1234"
  }
}

The otpauth_uri can be used to generate a QR code for Google Authenticator, and secret_masked is a partially masked secret for display. The admin will scan the QR or input the secret in their Authenticator app.

### POST /api/auth/mfa/verify *(Admin only)*

**Description:** Complete enabling of MFA by verifying a code from the authenticator app.
**Request Body:**

{ "code": "<6-digit or 8-digit code>" }

**Response:**

{ "code": 0, "message": "ok", "data": { "mfa_enabled": true } }

If the code is correct. If incorrect, returns 400 with message "Invalid MFA code".

*(In the future, similar endpoints can be used for login with MFA – e.g., after normal login, if MFA is enabled, require a verify step. But MVP might not enforce MFA on login, only setup.)*

## 2) Public Endpoints (Contest Info)

These endpoints do not require authentication. They provide contest information to populate the public site.

### GET /api/public/editions

**Description:** Get list of contest editions (and possibly contests) that are available.
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": [
    { "editionId": 5, "contestName": "AIEC", "year": 2025, "title": "2025 AIEC Contest",
      "registrationOpen": "2025-01-01T00:00:00Z", "registrationClose": "2025-02-15T23:59:59Z",
      "status": "OPEN"
    },
    { "editionId": 6, "contestName": "AIEC", "year": 2026, "title": "2026 AIEC Contest", "status": "DRAFT" }
  ]
}

Only editions with status OPEN (or upcoming) might be listed publicly. This is used to show which contest edition is currently active or open for registration.

### GET /api/public/editions/{editionId}

**Description:** Get detailed info for a specific contest edition (public data).
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": {
    "editionId": 5,
    "contestName": "AIEC",
    "year": 2025,
    "title": "2025 High School AI Entrepreneurship Contest",
    "description": "...",  // maybe contest intro
    "stages": [
       { "stageId": 10, "name": "Registration", "type": "REGISTRATION", "startAt": "...", "deadlineAt": "..." },
       { "stageId": 11, "name": "Preliminary Round", "type": "PRELIM", "startAt": "...", "deadlineAt": "..." },
       ...
    ],
    "rulesDocUrl": "https://.../rulebook.pdf"
  }
}

This could be used for an edition detail page or for the portal to display timeline etc.

### GET /api/public/news?editionId={id}&page={n}&pageSize={m}

**Description:** Get a paginated list of news posts. If editionId is provided, filter news for that contest edition (or global news if applicable).
**Response Data (example):**

{
  "code": 0, "message": "ok",
  "data": {
    "news": [
      { "newsId": 101, "title": "Semi-Finalists Announced", "publishedAt": "2025-03-01T12:00:00Z", "snippet": "Congratulations to our semi-finalists..." },
      { "newsId": 102, "title": "Final Round Schedule", "publishedAt": "...", "snippet": "The final presentations will be held at ..."}
    ],
    "page": 1,
    "pageSize": 10,
    "total": 15
  }
}

Only published news (publishedAt not null) are returned.

### GET /api/public/news/{newsId}

**Description:** Get the full content of a specific news post.
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": {
    "newsId": 101,
    "title": "Semi-Finalists Announced",
    "content": "<html or markdown content of the news post>",
    "publishedAt": "2025-03-01T12:00:00Z",
    "editionId": 5
  }
}

This provides the body content which might be HTML or Markdown that the front-end will render.

## 3) Team & Application Endpoints (Participant role)

These endpoints require Authorization: Bearer <token> of a user with role Student or Teacher (or Admin for administrative override).

### POST /api/editions/{editionId}/teams

**Description:** Create a new team in the given edition. The caller becomes the team leader (if Student) or mentor (if Teacher).
**Request Body:**

{ "team_name": "<name>", "team_intro": "<intro(optional)>" }

- team_name: required, must be unique within edition.
- team_intro: optional short description.

**Response:**

{
  "code": 0, "message": "ok",
  "data": { "team_id": 201, "join_code": "XYZ123" }
}

Also the team’s initial status (probably DRAFT). The join_code is the one to share with others.

**Authorization:** Must be logged in. If a user already in a team for that edition, should be forbidden (one team per user per edition policy).

### POST /api/teams/join

**Description:** Join a team using an invite code.
**Request Body:**

{ "join_code": "XYZ123" }

**Response:**

{ "code": 0, "message": "ok", "data": { "team_id": 201 } }

Indicates success and the team joined. The user is now a member of that team.
**Errors:** If code invalid or team full, returns appropriate error message (404 or 400).

### GET /api/teams/{teamId}

**Description:** Get details of a team. (Accessible to team members or admins; others should get 403.)
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": {
    "team": {
      "id": 201,
      "name": "Team Alpha",
      "editionId": 5,
      "status": "DRAFT",
      "members": [
         { "userId": 10, "name": "Alice", "role": "STUDENT", "roleTags": ["LEADER","TECH"] },
         { "userId": 11, "name": "Bob", "role": "STUDENT", "roleTags": ["BIZ"] },
         ...
         { "userId": 5, "name": "Mr. Smith", "role": "TEACHER", "roleTags": [] }
      ],
      "joinCode": "XYZ123",
      "mentorUserId": 5
    }
  }
}

The members list could include role tags if application submitted, or empty if not yet set. The joinCode might be included for team members to see (this is sensitive, but if you’re on the team it’s okay to share for inviting others).

### PUT /api/teams/{teamId}

**Description:** Update team info (e.g., change team name or intro).
**Request Body:**

{ "team_name": "<new name>", "team_intro": "<new intro>" }

All fields optional; only provided fields are updated.
**Response:**

{ "code": 0, "message": "ok", "data": { "team_id": 201, "team_name": "...", "team_intro": "..." } }

**Authorization:** Only team leader or admin should be allowed to update team name/intro. (The backend should check that the user is the team leader or has admin role.)

### POST /api/teams/{teamId}/mentor/bind

**Description:** Assign a mentor to the team (if not already assigned).
**Request Body:**

{ "mentor_teacher_user_id": <userId> }

This would likely be an admin function or team leader function if we allow specifying a mentor. In MVP, the usual flow is the mentor joins via code, not via this endpoint. So this endpoint might only be used by admins to attach a teacher to a team if needed.
**Response:**

{ "code": 0, "message": "ok", "data": { "mentor_user_id": 5 } }

**Authorization:** Admin or possibly team leader if providing an existing teacher’s userId (but verifying that user is teacher role).

### POST /api/teams/{teamId}/application/submit

**Description:** Submit the contest application for the team (Registration stage).
**Request Body:**

{
  "project_abstract": "<500-word abstract>",
  "member_role_statements": [
    { "user_id": 10, "role_tags": ["LEADER","TECH"], "role_statement": "I'll build the AI model..." },
    { "user_id": 11, "role_tags": ["BIZ"], "role_statement": "I'll work on the business plan..." },
    ...
  ],
  "mentor_declaration_ack": true,
  "integrity_originality_ack": true
}

**Response:**

{ "code": 0, "message": "ok", "data": { "application_id": 301, "status": "SUBMITTED" } }

After this, the team’s application is pending admin approval.
**Validation:** The backend will validate team membership count, presence of mentor, uniqueness of roles, etc., as per Section 2.6 rules. If any rule fails, returns 400 with details (e.g., “Team must have a mentor before submitting”, “At least one member must have TECH role”, etc.).

### GET /api/teams/{teamId}/application

**Description:** Get the submitted application details (for viewing/editing if rejected).
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": {
    "application": {
      "id": 301,
      "teamId": 201,
      "status": "SUBMITTED",
      "submittedAt": "2025-02-01T10:00:00Z",
      "projectAbstract": "...",
      "memberRoleStatements": [
        { "userId": 10, "roleTags": ["LEADER","TECH"], "roleStatement": "I'll build the AI model..." },
        ...
      ],
      "mentorDeclarationAck": true,
      "integrityOriginalityAck": true,
      "reviewedBy": null,
      "reviewedAt": null,
      "rejectionReason": null
    }
  }
}

If status is APPROVED or REJECTED, those fields would be filled and possibly a rejection reason provided. This allows team to see outcome or to edit if rejected (though editing might just mean re-calling submit with new data).

## 4) Stages & Submissions (for Participants)

These endpoints handle retrieving stage info and handling submissions uploads.

### GET /api/teams/{teamId}/stages

**Description:** List all stages of the contest edition relevant to the team, with a summary of the team’s submission status in each.
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": [
    { "stage_id": 11, "name": "Preliminary Round", "deadline_at": "2025-03-01T23:59:59Z", "submission_status": "SUBMITTED" },
    { "stage_id": 12, "name": "Semi-Final", "deadline_at": "2025-04-01T23:59:59Z", "submission_status": "NOT_OPEN" },
    ...
  ]
}

submission_status could be values like NOT_OPEN (stage not started yet), OPEN (stage open but no submission yet), DRAFT (team has a draft saved), SUBMITTED (team submitted), LOCKED (deadline passed, submission locked), or N/A if not applicable (e.g. team didn’t advance to that stage).

### GET /api/stages/{stageId}/requirements

**Description:** Get the list of submission requirements for a given stage (what items need to be submitted).
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": [
    { "material_type_code": "BP", "label": "Business Plan", "content_type": "FILE", "required": true, "max_files": 1, "allowed_exts": ".pdf,.docx", "max_size_mb": 10 },
    { "material_type_code": "MVP_VIDEO", "label": "Prototype Video", "content_type": "LINK", "required": true },
    { "material_type_code": "PITCH_DECK", "label": "Pitch Deck", "content_type": "FILE", "required": false, "max_files": 1, "allowed_exts": ".pptx,.pdf", "max_size_mb": 20 },
    { "material_type_code": "AI_NOTE", "label": "AI Technical Note", "content_type": "TEXT", "required": false, "min_words": 100, "max_words": 300 }
  ]
}

This allows the frontend to dynamically build the submission form for that stage.

### POST /api/stages/{stageId}/submissions

**Description:** Create a new submission record (version) for the given stage for the current team. (The teamId could be inferred from auth, but to be safe we might include it or determine from user’s membership).
**Request:** (No body needed aside from path)
**Response:**

{ "code": 0, "message": "ok", "data": { "submission_id": 401, "version_no": 1, "status": "DRAFT" } }

This indicates a draft submission has been created. The team can now attach files and fill in content for this submission.

### PUT /api/submissions/{submissionId}

**Description:** Add or update the items (files/links/text) in a submission draft. This is used to save a draft or update content before final submit.
**Request Body:**

{
  "items": [
    { "material_type_code": "BP", "content_type": "FILE", "file_ids": [111] },
    { "material_type_code": "MVP_VIDEO", "content_type": "LINK", "link_url": "https://youtu.be/...", "text_value": null },
    { "material_type_code": "AI_NOTE", "content_type": "TEXT", "text_value": "We used GPT-3 for ...", "link_url": null }
  ]
}

Explanation: For each item, identify by material_type_code (must match the stage’s requirement list). Provide the content based on type:
- For FILE: instead of sending the file directly, we send one or multiple file_ids which were obtained via the file upload process (see Files API). E.g., file_ids: [111, 112] if multiple files allowed.
- For LINK: provide the link_url as a string.
- For TEXT: provide the text_value.
*(The backend will store these in the **submission_items** table. **file_ids** would result in entries in a join table linking submission_items to files.)*

**Response:**

{ "code": 0, "message": "ok", "data": { "submission_id": 401, "status": "DRAFT", "items_count": 3 } }

Meaning the items were saved (still draft status). If any validation fails (e.g. file type not allowed, text too long, etc.), returns 400 with error message.

### POST /api/submissions/{submissionId}/submit

**Description:** Finalize and submit the current draft of the submission. This locks in the version.
**Request Body:** *none* (or could accept a final confirmation payload).
**Response:**

{ "code": 0, "message": "ok", "data": { "status": "SUBMITTED", "submitted_at": "2025-03-01T22:30:00Z" } }

After this, the submission is marked submitted. If within grace period or before deadline, it’s accepted. The backend should verify that all required items are present (if something required is missing, it should error and not allow submission). If the deadline has passed, it may return 400 or 403 indicating late submission not allowed (unless grace period).

Teams can still possibly create a new version if the stage is open and multiple submissions are allowed. In that case, calling the POST to /stages/{stageId}/submissions again would create version_no 2, etc.

### GET /api/stages/{stageId}/submissions?teamId={teamId}

**Description:** (Optional) Get the list of submission versions for a team in a given stage.
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": [
    { "submission_id": 401, "version_no": 1, "status": "SUBMITTED", "submitted_at": "2025-03-01T22:30:00Z" },
    { "submission_id": 402, "version_no": 2, "status": "DRAFT", "submitted_at": null }
  ]
}

This could be used to display version history. (MVP might not need to expose via API if not used, but helpful if front-end wants to show multiple attempts.)

## 5) File Uploads

We use a two-step file upload to integrate with object storage securely.

### POST /api/files/presign

**Description:** Request a presigned URL to upload a file.
**Request Body:**

{ "filename": "BusinessPlan.pdf", "mime_type": "application/pdf", "size_bytes": 234567 }

**Response:**

{
  "code": 0, "message": "ok",
  "data": {
    "upload_url": "https://minio.example.com/aiec/…signed-url…",
    "storage_key": "aiec/2025/teams/201/111-BusinessPlan.pdf",
    "headers": { "Authorization": "AWS4-HMAC-SHA256 ..."}
  }
}

- upload_url: a pre-authorized URL where the file can be PUT.
- storage_key: the path/key where the file will be stored in the bucket. (Client should not modify this; after uploading, the client will report this back to complete.)
- headers: any additional headers required for the PUT (if any, e.g. for S3 might include an Authorization header or nothing if already in URL).

**Client flow:** The client then performs an HTTP PUT of the file’s binary data to upload_url (outside of this API). If that succeeds (HTTP 200 from S3/MinIO), the file is stored.

### POST /api/files/complete

**Description:** Notify the server that file upload is complete and to record the file in the database.
**Request Body:**

{
  "storage_key": "aiec/2025/teams/201/111-BusinessPlan.pdf",
  "original_name": "BusinessPlan.pdf",
  "mime_type": "application/pdf",
  "size_bytes": 234567,
  "sha256": "<hash-of-file>"
}

*(The **sha256** could be computed by client for integrity, or omitted if not easily done on client.)*
**Response:**

{ "code": 0, "message": "ok", "data": { "file_id": 111 } }

The file_id is an internal ID that can now be used in submission items. The server stores metadata mapping that storage_key to the file record. If a virus scan pipeline is in place, virus_scan_status might be PENDING until scanning done; for MVP, we mark CLEAN by default or skip scanning integration.

After this, the client can then call the submission update API to attach this file_id.

### GET /api/files/{fileId}/download

**Description:** Get a signed URL to download a file (for viewing by authorized users).
**Response:**

{
  "code": 0, "message": "ok",
  "data": { "signed_url": "https://minio.example.com/aiec/...<signed>..." }
}

The client will then use this URL to actually fetch the file. This ensures the file is only accessible to users who should have access (the server should verify that the requesting user indeed has rights to that file, e.g. it's part of their team’s submission or they are an admin or assigned judge). The signed URL typically expires after a short time.

## 6) Judge Review Endpoints (Judge role)

These endpoints are for judges reviewing and scoring.

### GET /api/judge/assignments?stageId={stageId}

**Description:** List assignments for the authenticated judge (optionally filtered by stage).
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": [
    { "assignment_id": 501, "anonymous_code": "P-7A3", "stage_id": 11, "stage_name": "Preliminary Round", "status": "ASSIGNED", "due_at": "2025-03-10T23:59:59Z" },
    { "assignment_id": 502, "anonymous_code": "P-4F8", "stage_id": 11, "stage_name": "Preliminary Round", "status": "SUBMITTED", "due_at": "2025-03-10T23:59:59Z" }
  ]
}

Only the judge’s own assignments are returned.

### POST /api/judge/assignments/{assignmentId}/decline

**Description:** Decline an assignment due to conflict of interest or other reason.
**Request Body:**

{ "reason_code": "CONFLICT", "reason_text": "I mentored this team last year." }

reason_code could be a fixed set like "CONFLICT" or "NOT_AVAILABLE", etc., and reason_text provides details.
**Response:**

{ "code": 0, "message": "ok", "data": { "status": "DECLINED" } }

After this, the assignment will no longer appear in the judge’s active list (could move to a different list if needed). Admin will handle reassignment separately.

### GET /api/judge/projects/{anonymousCode}

**Description:** Retrieve the submission package and rubric for a given anonymous project code (if assigned to the judge). This is used to load the review page.
**Response Data:**

{
  "code": 0, "message": "ok",
  "data": {
    "submission": {
      "anonymous_code": "P-7A3",
      "stage_id": 11,
      "items": [
         { "material_type_code": "BP", "label": "Business Plan", "content_type": "FILE", "file_ids": [111], "file_names": ["BusinessPlan.pdf"] },
         { "material_type_code": "MVP_VIDEO", "label": "Prototype Video", "content_type": "LINK", "link_url": "https://youtu.be/...", "text_value": null },
         { "material_type_code": "AI_NOTE", "label": "AI Technical Note", "content_type": "TEXT", "text_value": "We used GPT-3 to build..." }
      ]
    },
    "rubric": {
      "rubric_id": 100,
      "criteria": [
         { "criterion_id": 1001, "name": "Commercial Viability", "max_score": 10, "weight": 0.30 },
         { "criterion_id": 1002, "name": "Technical Feasibility", "max_score": 10, "weight": 0.25 },
         { "criterion_id": 1003, "name": "User Validation", "max_score": 10, "weight": 0.20 },
         { "criterion_id": 1004, "name": "Presentation", "max_score": 10, "weight": 0.25 },
      ],
      "min_comment_words": 60
    }
  }
}

The judge can use the file_ids with the file download API to fetch files. Or the API could inline signed URLs for each file for convenience (but that might be a lot of data; better to retrieve on demand). The rubric info is included so the frontend knows what criteria to display and enforce.

### POST /api/judge/scores

**Description:** Submit scores and feedback for a judged project.
**Request Body:**

{
  "stage_id": 11,
  "anonymous_code": "P-7A3",
  "rubric_item_scores": [
    { "criterion_id": 1001, "score_value": 8.5 },
    { "criterion_id": 1002, "score_value": 9.0 },
    { "criterion_id": 1003, "score_value": 7.0 },
    { "criterion_id": 1004, "score_value": 8.0 }
  ],
  "overall_comment": "The team presented a strong business case with decent technical implementation. Could improve user testing data."
}

The server will look up the judge’s assignment for that anonymous_code and stage, verify it’s the correct judge. It then saves the scores and comment. It also calculates the total weighted score internally (e.g., 8.5*0.3 + 9.0*0.25 + ... = X).
**Response:**

{ "code": 0, "message": "ok", "data": { "score_id": 600 } }

Where score_id is the record of the judge’s scoring (could be the judge_assignment id or a separate entry). After submission, the assignment’s status becomes SUBMITTED.

If the judge later calls this again (for editing), the backend can update the existing scores (if before aggregation lock). Possibly we may use PUT instead of POST to indicate update, but POST can be idempotent in this context if we design it to overwrite.

**Validation:** If any required criterion missing or comment too short, return 400 with details (the front-end should ideally prevent those anyway).

## 7) Admin Endpoints (Admin role)

These endpoints allow contest configuration and managing the contest process. All require admin JWT.

### POST /api/admin/editions

**Description:** Create a new contest edition.
**Request Body:**

{
  "contest_id": 1,
  "year": 2026,
  "title": "2026 AIEC Contest",
  "registration_open_at": "2026-01-01T00:00:00Z",
  "registration_close_at": "2026-02-15T23:59:59Z"
}

Other optional fields: description/intro, etc.
**Response:**

{ "code": 0, "message": "ok", "data": { "edition_id": 7, "status": "DRAFT" } }

The edition will initially be DRAFT until opened.

### POST /api/admin/editions/{editionId}/stages

**Description:** Create a new stage under an edition.
**Request Body:**

{
  "name": "Semi-Final",
  "stage_type": "SEMI",
  "start_at": "2025-03-20T00:00:00Z",
  "deadline_at": "2025-04-10T23:59:59Z",
  "grace_minutes": 60,
  "rubric_id": 100
}

(The rubric should be created beforehand via a rubric endpoint, or this call might include rubric details inline in one go – could be separate for simplicity.)
**Response:**

{ "code": 0, "message": "ok", "data": { "stage_id": 12 } }

After this, admin would likely call requirements endpoint to configure stage requirements.

*(Note: We might create separate endpoints to configure requirements and rubrics.)*

### POST /api/admin/stages/{stageId}/requirements

**Description:** Set the submission requirements for a stage. Possibly multiple calls or one call with bulk data.
**Request Body:** e.g.

{
  "requirements": [
    { "material_type_code": "BP", "required": true, "min_files": 0, "max_files": 1, "content_type": "FILE" },
    { "material_type_code": "MVP_VIDEO", "required": true, "content_type": "LINK" },
    { "material_type_code": "PITCH_DECK", "required": false, "content_type": "FILE" },
    { "material_type_code": "AI_NOTE", "required": false, "content_type": "TEXT", "min_words": 100, "max_words": 300 }
  ]
}

**Response:**

{ "code": 0, "message": "ok", "data": { "count": 4 } }

Meaning 4 requirements were set.
(This operation could also be done through a GUI with multiple requests; exact API design can vary.)

### POST /api/admin/rubrics

**Description:** Create a new rubric (with criteria).
**Request Body:**

{
  "name": "MVP Scoring Rubric",
  "scoring_scale_min": 1.0,
  "scoring_scale_max": 10.0,
  "min_comment_words": 80,
  "criteria": [
    { "dimension": "Business Viability", "name": "Commercial Viability", "weight": 0.30 },
    { "dimension": "Technical", "name": "Technical Feasibility & Innovation", "weight": 0.25 },
    { "dimension": "Validation", "name": "User Validation & Data", "weight": 0.20 },
    { "dimension": "Presentation", "name": "Pitch & Presentation", "weight": 0.25 }
  ]
}

**Response:**

{ "code": 0, "message": "ok", "data": { "rubric_id": 100 } }

Criteria are stored and linked to that rubric. The weights should sum to 1.0; if not, return error.

### POST /api/admin/applications/{applicationId}/approve

**Description:** Approve a team’s application.
**Response:**

{ "code": 0, "message": "ok", "data": { "status": "APPROVED" } }

This sets the application as approved and the team becomes ACTIVE. (We might also directly have /api/admin/teams/{teamId}/approve if easier.)

### POST /api/admin/applications/{applicationId}/reject

**Description:** Reject a team’s application.
**Request Body:**

{ "reason": "Team does not meet role requirements." }

**Response:**

{ "code": 0, "message": "ok", "data": { "status": "REJECTED" } }

The rejection reason is saved. Team can edit and resubmit application.

### POST /api/admin/stages/{stageId}/assignments/generate

**Description:** Generate judge assignments for a stage.
**Request Body:**

{ "judges_per_project": 3, "max_load_per_judge": 5 }

**Response:**

{ "code": 0, "message": "ok", "data": { "created_assignments": 60 } }

This number indicates how many assignment records were created. The server will internally implement the assignment algorithm. If not enough judges, maybe it still creates what it can and returns a warning or error.

### POST /api/admin/stages/{stageId}/aggregate

**Description:** Perform score aggregation for a stage.
**Request Body:**

{ "outlier_rule": "DROP_HIGHEST_LOWEST" }

(e.g., choose outlier removal strategy; MVP likely fixed as drop highest/lowest if conditions met.)
**Response:**

{ "code": 0, "message": "ok", "data": { "aggregated_projects": 20 } }

Meaning 20 teams’ scores were aggregated. The results are now stored (in stage_results, etc.). At this point, stage results exist but not yet published.

### POST /api/admin/stages/{stageId}/publish

**Description:** Publish the results of a stage to teams (and optionally public).
**Request Body:**

{ "visibility": "TEAM_ONLY" \| "PUBLIC" }

- "TEAM_ONLY" means each team can see their own result/feedback when they log in, but no public announcement via the API’s public endpoints.
- "PUBLIC" could trigger that the results (e.g. list of advancing teams or winners) are also available in a public endpoint or included in the public news.

**Response:**

{ "code": 0, "message": "ok", "data": { "published_at": "2025-03-15T12:00:00Z" } }

The stage_results entries will have this published_at timestamp set. Teams will now see results.

*(If partial publishing is needed, e.g., only publish finalists list but not scores, that would be handled by logic in what data is exposed via the API, rather than separate flags.)*

### POST /api/admin/news

**Description:** Create or update a news post. (We can use same endpoint for create/update by including id if updating, or separate PUT for update.)
**Request Body:**

{
  "edition_id": 5,
  "title": "Final Round Schedule",
  "content": "<p>The final presentations will be on May 5...</p>",
  "pinned": false
}

**Response:**

{ "code": 0, "message": "ok", "data": { "news_id": 105 } }

This creates a news post (published_at could be set automatically to now, or we add a field if scheduling).

*(Additional endpoints might include: GET/PUT for news, listing applications pending approval, listing all teams, etc., which can be added as needed.)*

**Note:** All the above admin endpoints are protected and should return 403 if a non-admin user attempts to call them. The Permissions Matrix document specifies which roles can access which endpoints.

This API spec covers the core functionality needed for the MVP. Future enhancements (marked as P1 or beyond) can introduce more endpoints (e.g., for appeals process, multiple mentors, advanced analytics, etc.), but those are out of scope for now. The front-end will use these endpoints to implement the workflows described in the PRD.
