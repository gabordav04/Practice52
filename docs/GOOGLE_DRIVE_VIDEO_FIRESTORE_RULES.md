# Google Drive video submissions — Firestore Rules patch

This is a **proposed additive patch**, not a replacement for the live Firestore Rules. It is designed against the rules structure supplied during the Practice 52 review: platform administrator UID `fjJOrOvLxkdzHH0RzeFTUE1tSPS2`, group field `teacherUid`, group member field `studentUids`, and a final catch-all deny.

Before using it, export the current complete Rules from Firebase Console, keep a backup, insert this block inside the existing `match /databases/{database}/documents` scope and before the final catch-all deny, then test in Rules Playground or the Emulator. Do not publish this snippet by itself.

```firestore
match /practiceSubmissions/{submissionId} {
  function isPlatformAdminForSubmission() {
    return request.auth != null
      && request.auth.uid == 'fjJOrOvLxkdzHH0RzeFTUE1tSPS2';
  }

  function groupPath(groupId) {
    return /databases/$(database)/documents/groups/$(groupId);
  }

  function groupExists(groupId) {
    return exists(groupPath(groupId));
  }

  function isSubmissionStudent() {
    return request.auth != null
      && resource.data.studentUid == request.auth.uid
      && groupExists(resource.data.groupId)
      && request.auth.uid in get(groupPath(resource.data.groupId)).data.studentUids;
  }

  function isSubmissionTeacher() {
    return request.auth != null
      && groupExists(resource.data.groupId)
      && get(groupPath(resource.data.groupId)).data.teacherUid == request.auth.uid;
  }

  function isValidDriveUrl(url) {
    return url is string
      && url.matches('https://(drive\\.google\\.com|docs\\.google\\.com)/.*');
  }

  function isValidStudentSubmission() {
    return request.auth != null
      && request.resource.data.keys().hasAll([
        'studentUid', 'groupId', 'practiceId', 'skillId', 'skillIndex',
        'skillTitle', 'driveUrl', 'accessConfirmed', 'status',
        'submittedAt', 'updatedAt'
      ])
      && request.resource.data.keys().hasOnly([
        'studentUid', 'groupId', 'practiceId', 'skillId', 'skillIndex',
        'skillTitle', 'driveUrl', 'accessConfirmed', 'status',
        'submittedAt', 'updatedAt'
      ])
      && request.resource.data.studentUid == request.auth.uid
      && request.resource.data.groupId is string
      && groupExists(request.resource.data.groupId)
      && request.auth.uid in get(groupPath(request.resource.data.groupId)).data.studentUids
      && request.resource.data.practiceId == 'practice-52'
      && request.resource.data.skillId is string
      && request.resource.data.skillIndex is int
      && request.resource.data.skillIndex >= 0
      && request.resource.data.skillIndex < 52
      && request.resource.data.skillTitle is string
      && isValidDriveUrl(request.resource.data.driveUrl)
      && request.resource.data.accessConfirmed == true
      && request.resource.data.status == 'submitted'
      && request.resource.data.submittedAt is timestamp
      && request.resource.data.updatedAt is timestamp;
  }

  function isValidTeacherReview() {
    return request.auth != null
      && (isPlatformAdminForSubmission() || isSubmissionTeacher())
      && request.resource.data.diff(resource.data).affectedKeys().hasOnly([
        'status', 'practicalGrade', 'teacherComment',
        'reviewedBy', 'reviewedAt', 'updatedAt'
      ])
      && request.resource.data.status in ['reviewed', 'resubmission_requested']
      && request.resource.data.reviewedBy == request.auth.uid
      && request.resource.data.reviewedAt is timestamp
      && request.resource.data.updatedAt is timestamp
      && request.resource.data.teacherComment is string
      && request.resource.data.teacherComment.size() <= 3000
      && (
        (request.resource.data.status == 'reviewed'
          && request.resource.data.practicalGrade is int
          && request.resource.data.practicalGrade >= 2
          && request.resource.data.practicalGrade <= 5)
        ||
        (request.resource.data.status == 'resubmission_requested'
          && !('practicalGrade' in request.resource.data))
      );
  }

  allow get, list: if isPlatformAdminForSubmission()
    || isSubmissionStudent()
    || isSubmissionTeacher();

  allow create: if isValidStudentSubmission();

  allow update: if isValidTeacherReview();

  allow delete: if isPlatformAdminForSubmission();
}
```

## Important compatibility notes

- The current UI uses `practiceId: 'practice-52'`. This rule deliberately matches that provisional ID; change both together when the real practice ID is finalized.
- The current group schema supports one assigned teacher through `teacherUid`. This patch does not silently introduce multi-teacher assignments.
- The UI's checkbox is a student declaration. Firestore Rules cannot verify Google Drive sharing permissions.
- The current UI creates a new document for every resubmission. Earlier attempts remain intact.
- This patch authorizes the submission record only. It does not grant access to the student's Google Drive file; the student must share the file with the assigned teacher in Google Drive.
- The application writes the practical grade to the group's `gradebook` and displays it in the teacher journal and the student's «Моя група» view. This Rules patch only authorizes that data access; it does not implement the UI or the write itself.
- No Storage Rules are required for this design because Practice 52 does not upload or store video bytes.

## Required tests before publishing Rules

1. Student A can create and read their own submission when in the referenced group.
2. Student A cannot create a submission for another student or a group they do not belong to.
3. Student A cannot read Student B's submission.
4. Student cannot update or delete a submitted record or set any teacher review fields.
5. Assigned teacher can list and read submissions for their group.
6. Unassigned teacher cannot read or review those submissions.
7. Assigned teacher can set grade 2, 3, 4, or 5 and an optional comment.
8. Assigned teacher can request a resubmission without a grade.
9. Teacher cannot change the student UID, group, skill, Drive URL, or submission timestamps.
10. Unauthenticated users cannot read or write submissions.
11. Administrator can read, review, and delete records.
12. Existing group, studentProgress, role, invite, and university rules continue to behave as before.


## Additive rules for the group gradebook

The teacher UI writes one gradebook document per student and skill at:

`groups/{groupId}/gradebook/{studentUid}_{skillIndex}`

Add this nested block **inside the existing** `match /groups/{groupId}` block, alongside (not inside) the existing `studentProgress` match. It relies on the existing `isPlatformAdmin()` and `isAssignedTeacher()` helpers already present in the supplied rules. The student read check is scoped to the parent group's membership list.

```firestore
match /gradebook/{entryId} {
  function groupExistsForGradebook() {
    return exists(/databases/$(database)/documents/groups/$(groupId));
  }

  function isAssignedTeacherForGradebook() {
    return request.auth != null
      && groupExistsForGradebook()
      && get(/databases/$(database)/documents/groups/$(groupId)).data.teacherUid == request.auth.uid;
  }

  function isMemberReadingOwnGrade() {
    return request.auth != null
      && groupExistsForGradebook()
      && resource.data.studentUid == request.auth.uid
      && request.auth.uid in get(
        /databases/$(database)/documents/groups/$(groupId)
      ).data.studentUids;
  }

  function isAuthorizedGradebookWriter() {
    return request.auth != null
      && (isPlatformAdmin() || isAssignedTeacherForGradebook());
  }

  allow get, list: if isPlatformAdmin()
    || isAssignedTeacherForGradebook()
    || isMemberReadingOwnGrade();

  allow create: if isAuthorizedGradebookWriter()
    && groupExistsForGradebook()
    && request.resource.data.studentUid is string
    && request.resource.data.studentUid in get(
      /databases/$(database)/documents/groups/$(groupId)
    ).data.studentUids
    && request.resource.data.practiceId == 'practice-52'
    && request.resource.data.skillId is string
    && request.resource.data.skillIndex is int
    && request.resource.data.skillIndex >= 0
    && request.resource.data.skillIndex < 52
    && request.resource.data.skillTitle is string
    && request.resource.data.practicalGrade is int
    && request.resource.data.practicalGrade >= 2
    && request.resource.data.practicalGrade <= 5
    && request.resource.data.submissionId is string
    && request.resource.data.teacherUid == request.auth.uid
    && request.resource.data.teacherComment is string
    && request.resource.data.teacherComment.size() <= 3000
    && request.resource.data.updatedAt is timestamp;

  allow update: if isAuthorizedGradebookWriter()
    && request.resource.data.studentUid == resource.data.studentUid
    && request.resource.data.practiceId == resource.data.practiceId
    && request.resource.data.skillId == resource.data.skillId
    && request.resource.data.skillIndex == resource.data.skillIndex
    && request.resource.data.skillTitle is string
    && request.resource.data.practicalGrade is int
    && request.resource.data.practicalGrade >= 2
    && request.resource.data.practicalGrade <= 5
    && request.resource.data.submissionId is string
    && request.resource.data.teacherUid == request.auth.uid
    && request.resource.data.teacherComment is string
    && request.resource.data.teacherComment.size() <= 3000
    && request.resource.data.updatedAt is timestamp
    && request.resource.data.diff(resource.data).affectedKeys().hasOnly([
      'skillTitle', 'practicalGrade', 'submissionId',
      'teacherUid', 'teacherComment', 'updatedAt'
    ]);

  function isAuthorizedGradebookRemovalForResubmission() {
    let submissionPath = /databases/$(database)/documents/practiceSubmissions/$(resource.data.submissionId);
    return isAssignedTeacherForGradebook()
      && existsAfter(submissionPath)
      && getAfter(submissionPath).data.status == 'resubmission_requested'
      && getAfter(submissionPath).data.groupId == groupId
      && getAfter(submissionPath).data.studentUid == resource.data.studentUid
      && getAfter(submissionPath).data.skillIndex == resource.data.skillIndex;
  }

  // Викладач може прибрати активну оцінку лише в batch,
  // який одночасно переводить відповідне подання у повторне виконання.
  allow delete: if isPlatformAdmin()
    || isAuthorizedGradebookRemovalForResubmission();
}
```

The gradebook block defines its own `isAssignedTeacherForGradebook()` helper from the parent group's `teacherUid`; it does not depend on a helper nested in a sibling match block. Use the explicit checks in `allow create` and `allow update` as the operative policy. Test first-time grade creation and re-grading, plus student self-read and cross-student denial in the Emulator.
