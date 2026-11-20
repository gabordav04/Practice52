# PRACTICE 52 — Google Drive video submissions

Status: first implementation on branch `feature/google-drive-video-submissions`. The UI is implemented in the application source, but it is **not deployed**. Firestore access rules and live-project testing are still required before students can use it.

## Product decision

Practice 52 does not host student video files. Each student stores their own video in their own Google Drive and submits a Drive link inside the relevant skill in Practice 52. Practice 52 stores only submission metadata and review information in Firestore.

The student experience remains inside Practice 52:
1. Open a practical skill.
2. Paste a Google Drive link.
3. Confirm that the assigned teacher has been granted Viewer access.
4. Submit for review.
5. See the current submission status.

The teacher experience remains inside Practice 52:
1. Open the student's profile in the teacher dashboard.
2. Open the submitted video from the relevant skill.
3. Give a practical grade from 2 to 5 and an optional comment, or request a replacement video.

The actual video opens in Google Drive in a new browser tab. A checkbox is only the student's confirmation; the application does not verify Google Drive sharing permissions.

## Firestore record

Collection: `practiceSubmissions/{autoId}`

Student-created fields:
- `studentUid`: authenticated student UID
- `groupId`: the student's active group ID
- `practiceId`: currently `practice-52` for the existing 52-skill practice
- `skillId`: stable identifier such as `skill-01`
- `skillIndex`: zero-based skill index used by the current app
- `skillTitle`: displayed skill title
- `driveUrl`: HTTPS URL on `drive.google.com` or `docs.google.com`
- `accessConfirmed`: true after the student confirms that the teacher has Viewer access
- `status`: `submitted`
- `submittedAt`, `updatedAt`: server timestamps

Teacher-only review fields:
- `status`: `reviewed` or `resubmission_requested`
- `practicalGrade`: integer 2–5 when reviewed
- `teacherComment`: optional string
- `reviewedBy`: teacher UID
- `reviewedAt`: server timestamp

Every resubmission creates a new document, preserving earlier attempts. A new attempt does not overwrite the previous video or its review.

## Required Firestore authorization

The existing rules supplied for review deny unmatched paths by default. Add a dedicated `match /practiceSubmissions/{submissionId}` block before the final catch-all deny. The block must enforce all of the following:

- Unauthenticated users cannot read or write submissions.
- A student may create a document only when `studentUid == request.auth.uid`, the referenced group exists, and the student's UID is in that group's `studentUids`.
- On create, allow only the student submission fields listed above. Require a valid HTTPS Drive/Docs host, `accessConfirmed == true`, and `status == 'submitted'`.
- A student may read only their own submission documents.
- A student cannot update or delete a submitted document, and cannot set or change any review field.
- An assigned teacher may read submissions only for a group whose `teacherUid` equals their UID.
- Only the assigned teacher or platform administrator may update review fields. Enforce `practicalGrade` as integer 2–5, and restrict changed keys to review fields.
- Do not allow clients to change `studentUid`, `groupId`, `practiceId`, `skillId`, `skillIndex`, `driveUrl`, or submission timestamps after creation.
- Do not allow a student to set `reviewed`, `resubmission_requested`, `practicalGrade`, `reviewedBy`, or `reviewedAt`.

The complete live Firestore Rules file must be exported and merged before deployment. Do not replace the existing rules with a partial snippet. Test rules in Firebase Emulator/Rules Playground for student A, student B, assigned teacher, unassigned teacher, administrator, and unauthenticated access.

## Current limitations / rollout notes

- The application UI does not upload files to Google Drive; the student uploads the file to Drive first and pastes its link.
- The application cannot independently confirm that the teacher can open the link. Google Drive sharing is managed by the student.
- The application does not consume platform video storage or Google Drive API quota.
- This first implementation writes grades into the submission record. The existing group journal UI/schema still needs a separate integration so the grade appears in the group journal automatically.
- The current project uses a legacy 52-skill practice. `practiceId = practice-52` is a provisional stable identifier and must be mapped to the final university practice ID before a future data migration.
- No Firestore Rules, Storage Rules, Cloud Functions, existing accounts, group memberships, or progress documents are changed by this branch.
- Do not merge or deploy until the rules patch and end-to-end test plan are complete.
