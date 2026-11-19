# Firestore Rules patch — teacher progress mirrors

This patch is designed for the current `index.html` implementation on branch
`feature/teacher-dashboard-1`.

## Where to insert

Inside the existing block:

```firestore
match /groups/{groupId} {
  ...
}
```

Insert the following nested match **before the closing brace of that groups block**.
Do not replace the existing rules. Keep the final recursive deny rule unchanged.

## Rules to add

```firestore
    // Прогрес студента зберігається окремо в кожній групі.
    // Викладач читає лише записи студентів групи, призначеної йому.
    match /studentProgress/{studentUid} {
      function groupDoc() {
        return get(/databases/$(database)/documents/groups/$(groupId));
      }

      function isMemberStudent() {
        return request.auth != null
          && exists(/databases/$(database)/documents/groups/$(groupId))
          && studentUid in groupDoc().data.studentUids;
      }

      function isAssignedTeacher() {
        return request.auth != null
          && exists(/databases/$(database)/documents/groups/$(groupId))
          && groupDoc().data.teacherUid == request.auth.uid
          && studentUid in groupDoc().data.studentUids;
      }

      function isStudentAccount() {
        return request.auth != null
          && request.auth.uid == studentUid
          && request.auth.uid != 'NIXSOWQ2c1Tkh5TVGU6z9zkbhOp2'
          && (
            !exists(/databases/$(database)/documents/roles/$(request.auth.uid))
            || get(/databases/$(database)/documents/roles/$(request.auth.uid)).data.role == 'student'
          );
      }

      allow get: if isPlatformAdmin()
        || (isStudentAccount() && isMemberStudent())
        || isAssignedTeacher();

      allow create, update: if isPlatformAdmin()
        || (
          isStudentAccount()
          && isMemberStudent()
          && request.resource.data.studentUid == studentUid
          && request.resource.data.keys().hasOnly([
            'studentUid',
            'profile',
            'progress',
            'updatedAt'
          ])
        );

      allow delete: if isPlatformAdmin();
    }
```

## Expected access

- Student: can create/update/read only their own progress document inside a group they belong to.
- Assigned teacher: can read (not write) progress documents for students in that teacher's group.
- Administrator: full access.
- Other users: denied.
- No collection-wide `list` permission is granted; the current dashboard reads individual student documents by ID.

## Important

Do not publish these rules until they have been pasted into the Firebase Rules editor and passed its validation.
The app change writes progress to `groups/{groupId}/studentProgress/{studentUid}`; it no longer writes to a global `studentProgress/{studentUid}` collection.
