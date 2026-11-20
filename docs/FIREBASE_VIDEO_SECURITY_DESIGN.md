# PRACTICE 52 — security design for video submissions and the group gradebook

Status: implementation design only. **Do not publish these rules or treat this file as production security policy without testing against the live Firebase project and existing rules.**

## Findings from the currently supplied Firestore Rules

- Group membership is stored in `groups/{groupId}.studentUids`.
- The currently assigned teacher is stored in `groups/{groupId}.teacherUid`.
- The platform administrator is the UID checked by `isPlatformAdmin()`.
- Students can write their own `groups/{groupId}/studentProgress/{studentUid}` document.
- Group update rules permit admin, assigned teacher and group leader to add one eligible student.
- There is no cross-group uniqueness check. A client-side preflight query is not an authorization boundary.
- No production Storage Rules or Cloud Functions configuration has been supplied for review.

## Recommended record model

### `practiceSubmissions/{submissionId}`

Required fields:
- `studentUid`, `groupId`, `practiceId`, `skillId`
- `storagePath` (private object path, not a public download URL)
- `status`: `submitted`, `resubmission_requested`, `reviewed`
- `createdAt`, `updatedAt`

The client may create a submission only for the authenticated student and a group they belong to. A student must not be able to set a review, grade, reviewer identity or reviewed status.

### `groups/{groupId}/gradebook/{entryId}`

One entry per student + practice + skill. Suggested fields:
- `studentUid`, `practiceId`, `skillId`
- `theoryPassed`, `theoryPassedAt`
- `submissionStatus`, `latestSubmissionId`
- `practicalGrade` (integer 2–5 or null)
- `teacherComment`, `reviewedBy`, `reviewedAt`

The gradebook is readable by members of the group and its assigned teacher. Students have read-only access. Only the assigned teacher or platform administrator may write practical review fields.

## Security decisions required before production implementation

1. **Trusted exam result:** the current exam is evaluated in client-side JavaScript. A student can potentially manipulate client state or Firestore writes. A secure authoritative theory mark needs a trusted backend (for example, a callable Cloud Function) that validates the attempt and writes the gradebook entry. Merely checking `learningProgress[skill].exam === true` in a Firestore rule is not sufficient if the student can write that progress document.
2. **One active group:** Firestore Rules cannot safely enforce uniqueness by querying all group documents. Use a trusted transactional backend and a canonical membership record (for example `studentMemberships/{studentUid}`) updated atomically with the group membership. Preserve existing `studentUids` during a staged migration.
3. **Teacher scope:** current rules support one `teacherUid` per group. Multiple assigned teachers require a separate trusted assignment collection and rules that validate active assignments.
4. **Private video access:** Storage Rules must authorize upload/read by student ownership, group membership and active teacher assignment. Do not store public download URLs in the gradebook.
5. **Deployment:** do not replace live Firestore or Storage Rules until the complete current rules have been exported, merged, syntax-checked and tested in the Firebase Emulator/Rules Playground.

## Proposed private Storage object path

`practiceSubmissions/{groupId}/{studentUid}/{practiceId}/{skillId}/{submissionId}/video`

The path is private. The storage object should be limited to approved video MIME types and a defined maximum size. The exact size limit and allowed formats should be selected before writing final Storage Rules.

## Required test matrix

- Unauthenticated user cannot read or upload videos.
- Student A can upload and read only their own submission in their active group.
- Student A cannot read Student B's video or create metadata for Student B.
- Student cannot change `practicalGrade`, `reviewedBy`, `reviewedAt` or teacher comment.
- Assigned teacher can read videos and gradebook entries only for assigned groups/practices.
- Unassigned teacher cannot read or grade submissions.
- Student can read the group gradebook but cannot write it.
- Admin can review and correct records according to explicit admin policy.
- A student cannot join a second active group through admin, teacher, group leader, invite or direct Firestore writes.
- A successful theory exam produces a gradebook theory mark through trusted backend only.
- Existing group `15711fr`, user accounts and progress documents remain unchanged by the rollout.
