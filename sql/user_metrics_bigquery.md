# Метрики на рівні юзера: запити в синтаксисі BigQuery

Аналіз виконувався локально в DuckDB, тому в ноутбуці запити написані під нього. Тут ті самі запити в стандарті BigQuery.

Запити нижче написані одразу по сирій таблиці `transactions`, без проміжного очищеного шару, щоб їх можна було запустити на вихідних даних як є.

---

## 1. Загальний спенд юзера за весь час

Спендом рахую суму лише успішних транзакцій, фейл не є витратою.

**Спосіб 1, умовна агрегація:**

```sql
SELECT
  id_user,
  SUM(IF(status = 'success', amount, 0)) AS total_spend
FROM transactions
GROUP BY id_user
ORDER BY id_user;
```

**Спосіб 2, віконна функція:**

```sql
SELECT DISTINCT
  id_user,
  SUM(IF(status = 'success', amount, 0)) OVER (PARTITION BY id_user) AS total_spend
FROM transactions
ORDER BY id_user;
```

Обидва дають однаковий результат, але перший ефективніший: віконна функція рахує суму для кожного рядка і тільки потім `DISTINCT` згортає їх до одного на юзера.

Юзери без жодного успіху отримують 0, а не NULL. Це навмисно: `IF(..., amount, 0)` дає нуль для фейлів, тому `SUM` завжди повертає число. Якби замість цього писати `SUM(amount) ... HAVING status = 'success'` або відфільтрувати фейли у `WHERE`, такі юзери зникли б з результату зовсім.

---

## 2. Статус останньої транзакції кожного юзера

Останню транзакцію визначаю за `date_created`. У даних є 34 випадки, коли у одного юзера дві транзакції мають однаковий час, тому другим ключем сортування ставлю `id_order`. Без нього результат недетермінований.

**Спосіб 1, віконна функція:**

```sql
SELECT
  id_user,
  status AS last_status
FROM transactions
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY id_user
  ORDER BY date_created DESC, id_order DESC
) = 1
ORDER BY id_user;
```

**Спосіб 2, агрегація через ARRAY_AGG:**

```sql
SELECT
  id_user,
  ARRAY_AGG(status ORDER BY date_created DESC, id_order DESC LIMIT 1)[OFFSET(0)] AS last_status
FROM transactions
GROUP BY id_user
ORDER BY id_user;
```

`LIMIT 1` усередині `ARRAY_AGG` тут не косметика. Без нього BigQuery збирає в масив усі статуси юзера, а в даних є юзери з двома тисячами транзакцій.

**Спосіб 3, найкоротший:**

```sql
SELECT
  id_user,
  ANY_VALUE(status HAVING MAX date_created) AS last_status
FROM transactions
GROUP BY id_user
ORDER BY id_user;
```

Цей варіант найкоротший, але не дозволяє задати правило для нічиїх: при однаковому `date_created` він вибере довільний рядок. У ноутбуці я звірив його DuckDB-аналог `ARG_MAX(status, date_created)` з двома попередніми способами на всіх 40 210 юзерах, результати збіглися. Але це збіг на конкретних даних, а не гарантія, тому як основну відповідь беру спосіб 2.

---

## 3. Чи була у юзера хоч одна успішна оплата

**Спосіб 1, булева агрегація:**

```sql
SELECT
  id_user,
  LOGICAL_OR(status = 'success') AS has_any_success
FROM transactions
GROUP BY id_user
ORDER BY id_user;
```

**Спосіб 2, через лічильник:**

```sql
SELECT
  id_user,
  COUNTIF(status = 'success') > 0 AS has_any_success
FROM transactions
GROUP BY id_user
ORDER BY id_user;
```

Типова помилка тут написати `GROUP BY id_user, status`. Тоді юзер, у якого були і успіхи, і фейли, потрапить у результат двічі, і відповіді так чи ні не вийде. Питання ставиться по кожному юзеру, отже в `GROUP BY` має бути тільки `id_user`.

---

## 4. Чи можна одним скриптом без підзапитів і без WITH відповісти на всі три питання

Так, можна.

Усі три питання зводяться до агрегацій у межах одного й того самого угруповання по `id_user`. Зі спендом і наявністю успіху все просто, це звичайні агрегати. Складніше виглядає остання транзакція: вона схожа на задачу сортування, а не агрегації. Але `ARRAY_AGG` з `ORDER BY` всередині перетворює її на агрегат теж: збираємо статуси юзера у масив, відсортований від найновішого, і беремо перший елемент.

```sql
SELECT
  id_user,
  SUM(IF(status = 'success', amount, 0)) AS total_spend,
  ARRAY_AGG(status ORDER BY date_created DESC, id_order DESC LIMIT 1)[OFFSET(0)] AS last_status,
  LOGICAL_OR(status = 'success') AS has_any_success
FROM transactions
GROUP BY id_user
ORDER BY id_user;
```

Для порівняння, той самий результат через підзапити:

```sql
SELECT
  s.id_user,
  s.total_spend,
  l.last_status,
  s.has_any_success
FROM (
  SELECT
    id_user,
    SUM(IF(status = 'success', amount, 0)) AS total_spend,
    LOGICAL_OR(status = 'success') AS has_any_success
  FROM transactions
  GROUP BY id_user
) s
JOIN (
  SELECT id_user, status AS last_status
  FROM transactions
  QUALIFY ROW_NUMBER() OVER (
    PARTITION BY id_user
    ORDER BY date_created DESC, id_order DESC
  ) = 1
) l USING (id_user)
ORDER BY s.id_user;
```

Варіант без підзапитів коротший і читає таблицю один раз, тоді як варіант з підзапитами двічі. Обидва я звірив на даних, результати однакові.

---

## Відмінності діалектів, які тут задіяні

| Задача | DuckDB (виконував) | BigQuery (відповідь) |
|---|---|---|
| Елемент масиву | `arr[1]` | `arr[OFFSET(0)]` |
| Булева агрегація | `BOOL_OR(x)` | `LOGICAL_OR(x)` |
| Останнє значення за ключем | `ARG_MAX(a, b)` | `ANY_VALUE(a HAVING MAX b)` |
| Умовна сума | `SUM(CASE WHEN ... END)` | `SUM(IF(...))`, `CASE` теж працює |
| Фільтр в агрегаті | `COUNT(*) FILTER (WHERE ...)` | `FILTER` немає, тільки `COUNTIF` або `IF` |
| Різниця дат | `date_diff('day', a, b)` | `DATE_DIFF(b, a, DAY)`, аргументи навпаки |
| Початок тижня | `date_trunc('week', x)` | `DATE_TRUNC(x, WEEK(MONDAY))` |
| Каст до дати | `x::DATE` | `DATE(x)` |

`QUALIFY` і `COUNTIF` працюють однаково в обох, тому їх переписувати не треба.
