# Code Review: PR-3 — Експорт учасників у CSV

## 1. Резюме
Цей PR додає функцію `exportMembersCsv(db, user)` у `app/src/members.js` для експорту списку всіх учасників бібліотеки (включно з `email` та `phone`) у форматі CSV, запис у `app/CHANGELOG.md` та базові тести в `app/test/members.test.js`.

**Вердикт: ЗАБЛОКОВАНО (BLOCK)** — виявлено критичні порушення правил авторизації (будь-який звичайний учасник може вивантажити персональні дані всіх членів громади), повну відсутність обов'язкового аудиту доступу до персональних даних (PII), а також недостатнє покриття й строгість тестів.

## 2. Блокуючі зауваження (Blocking Issues)

- **[Правило 1] Відсутність перевірки ролі персоналу (`STAFF`) при експорті всіх учасників**: Функція `exportMembersCsv` перевіряє лише наявність об'єкта `user` (`if (!user) throw new AccessDenied()`), але не перевіряє його роль. У тому самому модулі навіть список без контактних даних (`listMembers`) вимагає `requireRole(user, STAFF)`, а `memberCard` забороняє читати чужі картки не-персоналу. У поточній реалізації будь-який авторизований читач із роллю `"member"` може викликати `exportMembersCsv` і отримати `email` та `phone` усіх учасників бібліотеки. Необхідно замінити перевірку на `requireRole(user, STAFF)`.
  - **Файл та контекст**: `app/src/members.js` (рядки 25–30)
  - **Цитата з коду**:
    ```javascript
    export function exportMembersCsv(db, user) {
      if (!user) throw new AccessDenied();
      const header = "id,name,email,phone,role";
    ```

- **[Правило 2] Відсутність логування доступу до персональних даних (`logAccess`)**: Функція повертає конфіденційні персональні дані (`email` та `phone`) усіх учасників із `db.members`, проте жодного разу не викликає `logAccess(db, user, memberId, purpose)` з `src/audit.js`. За стандартом команди кожне повернення `email` або `phone` зобов'язане фіксуватися в `db.accessLog` окремо для кожного учасника, чиї дані надано (наприклад, `logAccess(db, user, m.id, "members-csv-export")` для кожного `m` у `db.members`).
  - **Файл та контекст**: `app/src/members.js` (рядки 25–30)
  - **Цитата з коду**:
    ```javascript
    export function exportMembersCsv(db, user) {
      if (!user) throw new AccessDenied();
      const header = "id,name,email,phone,role";
      const lines = db.members.map((m) => [m.id, m.name, m.email, m.phone, m.role].join(","));
      return [header, ...lines].join("\n");
    }
    ```

- **[Правило 9] Відсутність тесту на відхилення доступу (`AccessDenied`) для авторизованого користувача з невідповідною роллю**: Негативний тест перевіряє лише анонімний виклик (`null`), тоді як за стандартом кожна нова функція з контролем доступу повинна мати тест, де доступ явно відхиляється для авторизованого користувача без необхідних прав (наприклад, `olya` з роллю `"member"`).
  - **Файл та контекст**: `app/test/members.test.js` (рядки 60–62)
  - **Цитата з коду**:
    ```javascript
    test("CSV export refuses an anonymous caller", () => {
      assert.throws(() => exportMembersCsv(createDb(), null), AccessDenied);
    });
    ```

- **[Правило 8] Недостатня строгість і повнота перевірок у тестах**: Позитивний тест перевіряє лише заголовок `lines[0]` та загальну кількість рядків `assert.equal(lines.length, 6)`, але взагалі не перевіряє вміст жодного рядка з даними учасників (чи правильний порядок полів і їхні значення) і не перевіряє наявність записів аудиту в `db.accessLog`.
  - **Файл та контекст**: `app/test/members.test.js` (рядки 53–58)
  - **Цитата з коду**:
    ```javascript
    test("CSV export has a header and one row per member", () => {
      const csv = exportMembersCsv(createDb(), iryna);
      const lines = csv.split("\n");
      assert.equal(lines[0], "id,name,email,phone,role");
      assert.equal(lines.length, 6);
    });
    ```

## 3. Зауваження та дрібниці (Non-blocking / Nits)
- **Небезпечне формування CSV без екранування спецсимволів (Edge Case)**: Конкатенація полів через `[m.id, m.name, m.email, m.phone, m.role].join(",")` зламає структуру CSV-файлу, якщо ім'я (`m.name`) міститиме кому, подвійні лапки або перенос рядка (наприклад, `"Приклад, Оля"`). Крім того, поля без екранування (зокрема телефони, що починаються з `+`, або імена з `=`, `-`, `@`) вразливі до CSV Injection під час відкриття у табличних процесорах. Рекомендується додати допоміжну функцію екранування значень за стандартом RFC 4180.
- **Невідповідність переліку полів в описі PR**: У `PR.md` зазначено експорт полів «ім'я, email, телефон, роль», тоді як код також включає першим стовпцем `id` (`"id,name,email,phone,role"`). Варто узгодити опис PR із фактичним контрактом.

## 4. Аналіз тестів та верифікації
- **Покриття авторизації (Verification Gap)**: Відсутній тест `assert.throws(() => exportMembersCsv(createDb(), olya), AccessDenied)`, через що критична діра в авторизації залишилася непоміченою.
- **Покриття аудиту PII (Verification Gap)**: Відсутній тест, який перевіряє, що після виклику `exportMembersCsv(db, iryna)` масив `db.accessLog` містить 5 записів (по одному на кожного учасника з `db.members` із `by: 3`).
- **Перевірка фактичних даних**: У тесті варто перевіряти конкретні сформовані рядки (наприклад, `assert.equal(lines[1], "1,Оля Приклад,olya@example.invalid,+380000000001,member")`), а не лише `lines.length`.

## 5. Вердикт
- [ ] Approve
- [x] Changes Requested (Blocked)
