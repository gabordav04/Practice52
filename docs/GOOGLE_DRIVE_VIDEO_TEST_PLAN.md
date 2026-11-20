# PRACTICE 52 — video submission and gradebook test plan

Status: manual test plan for PR #10. No Firebase Emulator or production test has been run yet.

## Preconditions
- Export and back up the complete current Firestore Rules from Firebase Console.
- Integrate the proposed rules in `GOOGLE_DRIVE_VIDEO_FIRESTORE_RULES.md` into the complete rules file; do not publish the snippets alone.
- Validate the complete rules in the Firebase Rules Playground or Emulator before publishing.
- Use test accounts and test group documents. Do not use real student records or modify existing progress in group `15711fr`.
- Ensure the test teacher account is `test_teacher`, uses a separate email, and is assigned to the test group through `teacherUid`.
- Ensure test Student A and Student B are members of the test group; Student C belongs to a different group.

## Access-control matrix

| Actor | Read own submission | Read another student's submission | Submit video | Review / grade | Read own gradebook entry | Read another student's grade |
|---|---:|---:|---:|---:|---:|---:|
| Student A | Allow | Deny | Allow | Deny | Allow | Deny |
| Student B | Allow | Deny | Allow | Deny | Allow | Deny |
| Assigned `test_teacher` | Allow for assigned group | Allow for assigned group | Deny | Allow for assigned group | Allow for assigned group | Allow for assigned group |
| Unassigned teacher | Deny | Deny | Deny | Deny | Deny | Deny |
| Administrator | Allow | Allow | Allow for own student membership | Allow | Allow | Allow |
| Unauthenticated visitor | Deny | Deny | Deny | Deny | Deny | Deny |

Note: administrator video submission is available only when the administrator account is also a member of the target group as a student. The administrator's review permissions remain separate.

## Functional scenarios
1. Student A submits a valid HTTPS Drive link and confirms viewer access.
2. Reject non-HTTPS links, non-Google hosts, and submission without the access confirmation.
3. Student A can see their own latest submission status and cannot see Student B's link.
4. Assigned teacher opens Student A's submission and can save each grade from 2 through 5 with a comment.
5. Verify one batch updates the submission and writes the stable gradebook record for the same student and skill.
6. Verify the journal shows theory status and practical grade in separate columns.
7. Student A sees their own practical grade in «Моя група»; Student B cannot read Student A's grade.
8. Assigned teacher can request a replacement video. The active gradebook entry is removed and the previous submission record remains.
9. Student A submits a replacement video. The new attempt is a separate record; previous attempts remain.
10. Verify a teacher assigned to another group cannot read, review, or grade submissions for this group.
11. Verify invalid grade values (1, 6, decimal, string) are denied by Firestore Rules, not only hidden by the UI.
12. Verify a student cannot update review fields, change their UID/group/skill/Drive URL, or delete a submission.

## Existing-data regression checks
- Existing group `15711fr` remains unchanged.
- Existing `studentProgress` documents and user progress remain readable and unchanged.
- Existing group membership, teacher assignment, role, invitation, nickname, university catalog and skill-rating rules continue to behave as before.
- No production data is seeded, migrated, deleted, or rewritten during these tests.

## Known test-environment gap
The repository currently has no `firebase.json`, `firestore.rules`, `firestore.indexes.json`, `package.json`, or emulator test harness. The proposed snippets are not a complete rules file. Therefore the automated Emulator Suite cannot be run from this repository as it currently stands. This is an explicit blocker, not a passed test.
