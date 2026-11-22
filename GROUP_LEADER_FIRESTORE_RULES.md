# PRACTICE 52 — Firestore rules for group leaders

Цей патч потрібен для функцій у `index.html` на гілці `feature/group-leader-membership`.

**Не вставляти навмання:** додайте наведені умови всередину чинного `match /groups/{groupId}` та перевірте весь файл Rules у Firebase Rules Playground/Emulator перед публікацією. Існуючі правила читання груп і `studentProgress` не видаляйте.

## Політика доступу

- Адміністратор: чинний повний доступ.
- Викладач групи (`teacherUid == request.auth.uid`): може змінювати `groupLeaderUid` лише на UID студента цієї групи або очищати поле; також може додавати студентів.
- Староста (`groupLeaderUid == request.auth.uid`): може лише додавати одного студента до `studentUids`; не може вилучати учасників, змінювати старосту чи інші поля.
- Інші користувачі: не можуть змінювати склад групи.

## Додати всередину match /groups/{groupId}

```firestore
function isP52Admin() {
  return request.auth != null
    && request.auth.uid == 'fjJOrOvLxkdzHH0RzeFTUE1tSPS2';
}

function isAssignedGroupTeacher() {
  return request.auth != null
    && resource.data.teacherUid == request.auth.uid;
}

function isCurrentGroupLeader() {
  return request.auth != null
    && resource.data.groupLeaderUid == request.auth.uid;
}

function isEligibleStudent(uid) {
  return uid is string
    && uid != resource.data.teacherUid
    && uid != resource.data.groupLeaderUid
    && (
      !exists(/databases/$(database)/documents/roles/$(uid))
      || get(/databases/$(database)/documents/roles/$(uid)).data.role == 'student'
    );
}

function teacherAppointsGroupLeader() {
  return isAssignedGroupTeacher()
    && request.resource.data.diff(resource.data).affectedKeys()
      .hasOnly(['groupLeaderUid', 'updatedAt'])
    && (
      !('groupLeaderUid' in request.resource.data)
      || request.resource.data.groupLeaderUid == null
      || request.resource.data.groupLeaderUid in resource.data.studentUids
    );
}

function teacherAddsStudent() {
  return isAssignedGroupTeacher()
    && request.resource.data.diff(resource.data).affectedKeys()
      .hasOnly(['studentUids', 'updatedAt'])
    && request.resource.data.studentUids.size() == resource.data.studentUids.size() + 1
    && request.resource.data.studentUids.hasAll(resource.data.studentUids)
    && request.resource.data.studentUids.toSet().difference(resource.data.studentUids.toSet()).size() == 1
    && isEligibleStudent(
      request.resource.data.studentUids.toSet()
        .difference(resource.data.studentUids.toSet()).toList()[0]
    );
}

function groupLeaderAddsStudent() {
  return isCurrentGroupLeader()
    && request.resource.data.diff(resource.data).affectedKeys()
      .hasOnly(['studentUids', 'updatedAt'])
    && request.resource.data.studentUids.size() == resource.data.studentUids.size() + 1
    && request.resource.data.studentUids.hasAll(resource.data.studentUids)
    && request.resource.data.studentUids.toSet().difference(resource.data.studentUids.toSet()).size() == 1
    && isEligibleStudent(
      request.resource.data.studentUids.toSet()
        .difference(resource.data.studentUids.toSet()).toList()[0]
    );
}

allow update: if isP52Admin()
  || teacherAppointsGroupLeader()
  || teacherAddsStudent()
  || groupLeaderAddsStudent();
```

## Перевірити перед публікацією

1. Адміністратор може додати студента в будь-яку групу.
2. Викладач може додати студента лише у свою групу.
3. Викладач може призначити старостою лише учасника своєї групи.
4. Староста може додати рівно одного нового студента за одну операцію лише у власну групу.
5. Староста не може вилучати студента, змінювати `teacherUid`, `groupLeaderUid` або інші поля.
6. Викладач іншої групи та звичайний студент не можуть змінити групу.
7. Староста після вилучення з групи або зняття призначення втрачає право додавання.

**Важливо:** якщо чинні правила містять окремий `allow update` для адміністратора, правила Firestore об'єднуються через логічне OR. Тому нові дозволи не повинні послабити інші поля: перевірте всі наявні `allow update/write` у цьому блоці. Також перевірте, що назва колекції ролей і схема ролей відповідають чинному проєкту.
