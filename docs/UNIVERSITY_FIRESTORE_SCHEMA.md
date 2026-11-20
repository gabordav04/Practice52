# PRACTICE 52 — Firestore data model (draft for Stage 1)

Цей документ описує цільову схему. Це НЕ готовий Firebase Rules файл і НЕ команда для негайного ручного створення документів у production.

## Naming and IDs

Use stable, lowercase IDs where possible. Course IDs:
- `bachelor-1`, `bachelor-2`, `bachelor-3`, `bachelor-4`
- `master-1`, `master-2`

Suggested fixed IDs:
- University: `practice52-university`
- Program: `physical-therapy-rehabilitation`
- Existing 52-skill practice: `master-2-cardiorespiratory-professional-practice`

## Collections

### `universities/{universityId}`
```js
{
  name: string,
  slug: "practice52-university",
  status: "active",
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

### `programs/{programId}`
```js
{
  universityId: string,
  name: "Фізична терапія та реабілітація",
  status: "active",
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

### `courses/{courseId}`
```js
{
  universityId: string,
  programId: string,
  educationLevel: "bachelor" | "master",
  yearNumber: 1 | 2 | 3 | 4,
  title: string,
  status: "active",
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

### `practices/{practiceId}`
```js
{
  universityId: string,
  programId: string,
  courseId: string,
  title: string,
  description: string,
  status: "draft" | "active" | "archived",
  skillSetId: string | null,
  legacySource: string | null,
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

The 52-skill practice uses `courseId = master-2`, `skillSetId = practice-52-cardiorespiratory`, and `legacySource = groups/15711fr` only as a migration reference. Do not move or delete the current group document during Stage 1.

### `teachingAssignments/{assignmentId}`
```js
{
  teacherUid: string,
  universityId: string,
  programId: string,
  courseId: string,
  practiceId: string | null,
  scope: "course" | "practice",
  status: "active" | "revoked",
  createdBy: string,
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

Validation:
- `scope == "course"` requires `practiceId == null`.
- `scope == "practice"` requires a valid `practiceId` whose `courseId` matches.
- Revoked assignments must no longer grant access.

### `groups/{groupId}` (legacy-compatible extension)
Existing fields remain intact:
- `name`, `teacherUid`, `teacherName`, `studentUids`, `groupLeaderUid`, timestamps.

New optional fields:
```js
{
  universityId: string,
  programId: string,
  courseId: string,
  practiceIds: string[],
  groupScope: "practice" | "course"
}
```

For a group dedicated to one practice, set `groupScope = "practice"` and `practiceIds = [practiceId]`.
For a course-level group that may attend several practices, set `groupScope = "course"` and link the practices through membership records below. Do not assume the legacy `studentUids` array alone is enough to authorize every practice.

### `groupPracticeMemberships/{membershipId}`
```js
{
  groupId: string,
  practiceId: string,
  universityId: string,
  courseId: string,
  status: "active" | "archived",
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

### `enrollments/{enrollmentId}`
```js
{
  studentUid: string,
  universityId: string,
  programId: string,
  courseId: string,
  practiceId: string,
  groupId: string | null,
  status: "pending" | "active" | "rejected" | "completed" | "withdrawn",
  enrollmentSource: "admin" | "teacher" | "invite" | "application",
  createdBy: string | null,
  approvedBy: string | null,
  enrolledAt: Timestamp | null,
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

Manual course promotion is represented by an administrator-approved change to the student's active course enrollment; keep historical completed enrollments for audit.

### `practiceInvites/{inviteId}`
```js
{
  tokenHash: string,
  universityId: string,
  programId: string,
  courseId: string,
  practiceId: string,
  groupId: string,
  mode: "auto_enroll" | "application",
  status: "active" | "revoked" | "expired",
  createdBy: string,
  expiresAt: Timestamp,
  maxUses: number | null,
  useCount: number,
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

The raw token must not be stored in Firestore. Resolve and consume invite tokens through a trusted backend/Cloud Function or another server-validated flow; never let an unauthenticated client write its own enrollment status as active.

## Progress isolation

Target key: student + practice. Recommended future path:
`practiceProgress/{practiceId}/students/{studentUid}`

Current path `groups/{groupId}/studentProgress/{studentUid}` remains authoritative until Stage 2 migration is designed, backed up, and verified. Do not dual-write yet: it can create divergent progress records.

## Access-control principles

- Client-side visibility is not authorization.
- Every read/write must validate authentication, role, assignment, enrollment, course and practice scope.
- A course-level teacher assignment grants access only to practices whose `courseId` matches.
- A practice-level assignment grants access only to that practice.
- Students may read only their own active enrollments and the roster information allowed for their group.
- Group leaders may only add students to the specific group/practice scope delegated to them.
- Admin is identified using the project's existing verified admin identity; do not copy a guessed UID into new rules.
- Firestore rules are additive across matching `allow` statements. Audit all existing group and progress rules before adding new permissions.

## Stage 1 seed manifest

```json
{
  "universityId": "practice52-university",
  "universityName": "PRACTICE 52 — Університет",
  "programId": "physical-therapy-rehabilitation",
  "programName": "Фізична терапія та реабілітація",
  "courses": [
    {"id":"bachelor-1","educationLevel":"bachelor","yearNumber":1},
    {"id":"bachelor-2","educationLevel":"bachelor","yearNumber":2},
    {"id":"bachelor-3","educationLevel":"bachelor","yearNumber":3},
    {"id":"bachelor-4","educationLevel":"bachelor","yearNumber":4},
    {"id":"master-1","educationLevel":"master","yearNumber":1},
    {"id":"master-2","educationLevel":"master","yearNumber":2}
  ],
  "practices": [
    {
      "id":"master-2-cardiorespiratory-professional-practice",
      "courseId":"master-2",
      "title":"Професійна практика з фізичної терапії та реабілітації при порушеннях діяльності серцево-судинної та дихальної системи",
      "skillSetId":"practice-52-cardiorespiratory",
      "legacySource":"groups/15711fr"
    }
  ]
}
```

Це manifest для контрольованого seed-скрипту на наступному кроці; він сам по собі не змінює production Firestore.
