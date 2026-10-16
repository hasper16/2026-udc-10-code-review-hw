# Code Review: PR-1 — Сторінкування для списку позик

## 1. Резюме
У PR додано нову публічну функцію `listLoansPage(db, user, { page, pageSize })` в `app/src/loans.js` для пагінації списку позик та один юніт-тест у `app/test/loans.test.js`.
Проте реалізація містить критичну помилку **off-by-one** при обчисленні зміщення сторінки (через яку перша сторінка пропускає перші `pageSize` записів і в тестовому сценарії повертає порожній масив `items: []`), не обмежує `pageSize` зверху, не оновлює `app/CHANGELOG.md`, а доданий тест перевіряє лише метадані й не перевіряє сам масив `res.items`.

Загальний вердикт: **ЗАБЛОКОВАНО (BLOCK)**.

## 2. Блокуючі зауваження (Blocking Issues)

- **[Правило 7] Помилка off-by-one у пагінації та відсутність обмеження `pageSize`**: При 1-індексованій нумерації сторінок (`page = 1` за замовчуванням) початковий індекс обчислюється як `page * pageSize` замість `(page - 1) * pageSize`. У результаті для `page = 1, pageSize = 20` функція починає вибірку з індексу `20`, повністю пропускаючи перші 20 позик (першу сторінку). Крім того, відсутня валідація нижньої межі (`page >= 1`, `pageSize >= 1`) та верхній ліміт на `pageSize` (захист від передачі надвеликих значень).
  - **Файл та контекст**: `app/src/loans.js` (рядки 10–14)
  - **Цитата з коду**:
    ```js
    export function listLoansPage(db, user, { page = 1, pageSize = 20 } = {}) {
      const all = listLoans(db, user);
      const start = page * pageSize;
      return { items: all.slice(start, start + pageSize), page, pageSize, total: all.length };
    }
    ```
  - **Як виправити**: Використовувати формулу `const start = (page - 1) * pageSize;` та додати валідацію/обмеження параметрів пагінації (наприклад, обмежити `pageSize` максимумом `MAX_PAGE_SIZE = 100` і перевіряти `page >= 1`).

- **[Правило 8] Верифікаційний розрив у тестах (відсутня перевірка `res.items`)**: Новий тест перевіряє лише метадані (`res.total`, `res.page`, `res.pageSize`), які просто повертаються з вхідних аргументів та `all.length`, але жодним чином не перевіряє вміст чи довжину `res.items`. Саме тому тест «зелений», хоча при 7 записах у тестовій БД виклик `listLoansPage(createDb(), iryna, { page: 1, pageSize: 20 })` фактично повертає `res.items = []`.
  - **Файл та контекст**: `app/test/loans.test.js` (рядки 36–41)
  - **Цитата з коду**:
    ```js
    test("paged list reports the total and the page it returned", () => {
      const res = listLoansPage(createDb(), iryna, { page: 1, pageSize: 20 });
      assert.equal(res.total, 7);
      assert.equal(res.page, 1);
      assert.equal(res.pageSize, 20);
    });
    ```
  - **Як виправити**: Перевіряти конкретні повернуті елементи `res.items` (наприклад, `assert.deepEqual(res.items.map((l) => l.id), ...)` на 1-й та 2-й сторінках із невеликим `pageSize`, наприклад `pageSize: 3`).

- **[Правило 6] Відсутній запис у `app/CHANGELOG.md`**: Додано нову публічну функцію `listLoansPage` в `app/src/loans.js`, однак файл `app/CHANGELOG.md` у цьому PR не оновлено.
  - **Файл та контекст**: `app/CHANGELOG.md` / `app/src/loans.js` (рядок 10)
  - **Цитата з коду**:
    ```js
    export function listLoansPage(db, user, { page = 1, pageSize = 20 } = {}) {
    ```
  - **Як виправити**: Додати опис нової публічної функції `listLoansPage(db, user, { page, pageSize })` та формату її відповіді `{ items, page, pageSize, total }` до `app/CHANGELOG.md`.

- **[Правило 1, Правило 9, Правило 10] Невідповідність авторизації заявленій специфікації (Staff UI) та відсутність тесту на `AccessDenied`**: В описі `PR.md` («Список позик для персоналу вже на 3 000+ рядків, у UI все гальмує») та JSDoc-коментарі явно вказано, що функція створена для інтерфейсу персоналу (`Paged variant for the staff UI`). Однак замість перевірки ролі персоналу через `requireRole(user, STAFF)` (як у сусідній функції `overdueReport`), код делегує виклик у `listLoans(db, user)`, який дозволяє виклик і звичайним читачам (`member`), а при `!user` кидає загальний `Error("user required")` замість `AccessDenied`. Також відсутній тест на відхилення доступу (`AccessDenied`) для авторизованого користувача без прав персоналу (`olya`).
  - **Файл та контекст**: `app/src/loans.js` (рядки 9–11)
  - **Цитата з коду**:
    ```js
    /** Paged variant for the staff UI: { items, page, pageSize, total }. */
    export function listLoansPage(db, user, { page = 1, pageSize = 20 } = {}) {
      const all = listLoans(db, user);
    ```
  - **Як виправити**: Якщо функція призначена для staff UI — додати `requireRole(user, STAFF)` на початку `listLoansPage` і додати негативний тест із перевіркою `AccessDenied` для `olya` (`role: "member"`). Якщо ж функція має підтримувати обидві ролі — уточнити опис/JSDoc, забезпечити узгоджену перевірку прав та покрити тестами поведінку для `member` і неавторизованого користувача.

## 3. Зауваження та дрібниці (Non-blocking / Nits)
- **[Правило 12] Зайве клонування всього масиву перед пагінацією (Fowler Smell)**: Виклик `listLoans(db, user)` спочатку створює поверхневі копії усіх записів (`rows.map((l) => ({ ...l }))` — за описом PR це 3 000+ об'єктів), після чого `listLoansPage` відкидає всі записи, крім однієї сторінки (`all.slice(...)`). Доцільніше спочатку відфільтрувати масив, обчислити `total`, зробити `.slice(start, start + pageSize)` і клонувати лише елементи поточної сторінки.

## 4. Аналіз тестів та верифікації
- Єдиний доданий тест `paged list reports the total and the page it returned` має критичний верифікаційний розрив: він перевіряє лише `total`, `page` і `pageSize`, повністю ігноруючи `res.items`.
- У тесті використано `pageSize: 20` на базі з 7 записів, що навіть за умови перевірки `items` не дозволило б протестувати перехід між сторінками (сторінка 1 vs сторінка 2) та граничні умови.
- Відсутні тести на:
  1. Точний склад `res.items` (наприклад, `page: 1, pageSize: 3` -> ID `[1, 2, 3]`, `page: 2, pageSize: 3` -> ID `[4, 5, 6]`, `page: 3, pageSize: 3` -> ID `[7]`).
  2. Відхилення доступу (`AccessDenied`) для користувача з невідповідною роллю (Правило 9).
  3. Граничні значення `page` та `pageSize` (перевищення максимального `pageSize`, некоректні або нульові значення).

## 5. Вердикт
- [ ] Approve
- [x] Changes Requested (Blocked)
