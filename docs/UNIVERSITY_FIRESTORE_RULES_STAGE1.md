# PRACTICE 52 — Firestore Rules for Stage 1 university catalog

## Що змінюється

Новий адміністративний інтерфейс читає та записує чотири top-level колекції:
- `universities`
- `programs`
- `courses`
- `practices`

На першому етапі каталог доступний лише адміністратору PRACTICE 52. Це тимчасова безпечна модель: студентам і викладачам доступ відкриємо окремо після реалізації їхніх маршрутів і призначень.

## Перед вставленням

1. Відкрий чинні Firestore Rules у Firebase Console.
2. Переконайся, що в цьому ж service-блоці вже є функція `isPlatformAdmin()` з правильним UID адміністратора.
3. Додай наведені `match` блоки всередину `match /databases/{database}/documents`, але перед фінальним `match /{document=**}`.
4. Не видаляй і не замінюй чинні правила для `users`, `groups`, `roles`, `leaderboard`, `teacherInvites`, `nicknames`, `skillRatings` та `studentProgress`.
5. Опублікуй правила після перевірки синтаксису в Rules Playground/Emulator.

## Додати

```firestore
// Університетський каталог PRACTICE 52 — етап 1.
// Поки що читання та зміни доступні лише адміністратору.
match /universities/{universityId} {
  allow read, create, update, delete: if isPlatformAdmin();
}

match /programs/{programId} {
  allow read, create, update, delete: if isPlatformAdmin();
}

match /courses/{courseId} {
  allow read, create, update, delete: if isPlatformAdmin();
}

match /practices/{practiceId} {
  allow read, create, update, delete: if isPlatformAdmin();
}
```

## Перевірка після публікації

1. Увійти під основним акаунтом адміністратора.
2. Відкрити «Адміністрування» → «Університет і курси».
3. Натиснути «Створити / оновити структуру».
4. Переконатися, що відображається 6/6 курсів і практика магістратури 2 курсу.
5. Створити тестову практику-чернетку, перевірити її відображення, потім за потреби видалити її вручну з Firestore Console.
6. Увійти під студентом і викладачем: вони не повинні мати змоги читати/змінювати ці колекції на цьому етапі.

## Важливі обмеження

- Новий інтерфейс не змінює чинні документи груп, ролей або прогресу.
- Кнопка ініціалізації використовує Firestore batch з `set(..., {merge:true})`, тому повторний запуск оновлює визначені поля, не видаляючи інші поля цих документів.
- Документ практики 52 навичок містить лише посилання на legacy-джерело `groups/15711fr`; сама група та її прогрес не переміщуються.
- Не відкривати читання каталогу всім авторизованим користувачам, поки не будуть готові enrollment та teachingAssignments перевірки.
- Firestore правила об'єднуються через OR між усіма відповідними `allow` виразами. Переконайся, що в чинному файлі немає ширшого catch-all дозволу, який відкриває ці колекції іншим користувачам.
