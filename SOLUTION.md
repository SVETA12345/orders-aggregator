# Задание 1. Скидки в статистике

## Шаги

1. **Миграция БД**: создала новый файл `migrations/002_discounts.sql`, который добавляет колонку `amount_paid_kop`.

2. **Парсинг внешнего заказа** (`src/services/orders.ts`):
   - В zod-схему `ExternalOrder` добавила `discountPercent` и `amountPaid`.
   - В `normalize()`: `amountPaidKop = toKop(parseRub(amountPaid ?? amount))` (переводим цену в копейки).

3. **Модель и репозиторий** — `src/repositories/orders.ts`:
   - В `Order` добавила `amountPaidKop`.

4. **Сервис totals** — `src/services/totals.ts`:
   - Суммировала отдельно `amountKop` (gross) и `amountPaidKop` (net) в копейках; `discount = gross - net` тоже в копейках.
   - Конвертировала всё в рубли через `fromKop`.

5. **Тесты**:
   - Обновила существующие проверки в `test/totals.test.ts`. Добавила новые поля.
   - Добавила новый тест на скидку.

6. **Проверка**: `npm run migrate`, `npm run typecheck`, `npm test`.

---

# 2 Задание Жалоба от аналитиков

## 1. Ошибка с переводом времени

1. Для начала узнаем из какого часового пояса `c_7`:

   ```bash
   curl http://localhost:4000/customers/c_7
   ```

   ```json
   { "id": "c_7", "timezone": "Europe/Berlin" }
   ```

   На летнее время в Берлине переводят время в последнее воскресенье марта 29.03.2026 (+ 2 часа, вместо +1).

2. Выведем таблицу:

   ```bash
   docker exec -it orders-aggregator-postgres-1 psql -U postgres -d orders
   ```

   ```sql
   SELECT id, created_at, amount_kop, amount_paid_kop
   FROM orders
   WHERE customer_id = 'c_7' AND status = 'paid'
     AND created_at >= '2026-03-27' AND created_at < '2026-04-01'
   ORDER BY created_at;
   ```

   ```
    id     |       created_at       | amount_kop | amount_paid_kop
   ---------+------------------------+------------+-----------------
    o_02090 | 2026-03-27 00:30:46+00 |     770400 |          639432
    o_02097 | 2026-03-27 01:45:27+00 |     252400 |          234732
    o_02131 | 2026-03-27 10:00:27+00 |     178000 |          172660
    o_05025 | 2026-03-27 10:13:56+00 |     604700 |          483760
    o_04992 | 2026-03-27 15:01:58+00 |      61600 |           61600
    o_02184 | 2026-03-27 22:15:44+00 |     162700 |          146430
    o_02188 | 2026-03-27 23:20:42+00 |     449100 |          336825
   (вывелось 46 строк)
   ```

   Как сейчас считает:

   ```bash
   curl "http://localhost:3000/customers/c_7/daily-totals?from=2026-03-27&to=2026-03-31"
   ```

   ```json
   [
     {
       "date": "2026-03-27",
       "total": 24736,
       "gross": 24736,
       "discount": 4071.21,
       "net": 20664.79
     },
     {
       "date": "2026-03-28",
       "total": 47995,
       "gross": 47995,
       "discount": 3484.31,
       "net": 44510.69
     },
     {
       "date": "2026-03-29",
       "total": 58871,
       "gross": 58871,
       "discount": 6490.33,
       "net": 52380.67
     },
     {
       "date": "2026-03-30",
       "total": 52201,
       "gross": 52201,
       "discount": 7875.93,
       "net": 44325.07
     },
     {
       "date": "2026-03-31",
       "total": 43576,
       "gross": 43576,
       "discount": 6430.29,
       "net": 37145.71
     }
   ]
   ```

   Проанализируем, что должно быть после перевода на летнее время:

   | id      | правильный день | считает код | 
   | ------- | --------------- | ----------- | 
   | o_02357 | 30.03           | 29.03       | 
   | o_02448 | 31.03           | 30.03       | 
   | o_02547 | 01.04           | 31.03       | 

   Исправим это. Шаги:
   1. **Нашла место бага**
      `src/services/totals.ts:28` — offset вычислялся один раз, «на сейчас»:

      ```ts
      const offset = tzOffsetMinutes(timezone, new Date());
      ```

      И применялся ко всем заказам диапазона.

   2. **Исправила**
      Убрала переменную `offset` и стала считать смещение для даты каждого заказа:

      ```ts
      for (const o of rows) {
        // offset зависит от даты заказа: летом/зимой он разный (DST)
        const date = dateAtOffset(o.createdAt, tzOffsetMinutes(timezone, o.createdAt));
        ...
      }
      ```

      Теперь заказ от 29 марта (летнее время, +2ч) получает правильный offset, а не ноябрьский (+1ч).

   3. **Добавила тест на DST**
      В `test/totals.test.ts` новый тест «учитывает переход на летнее время (DST)».

## 2) Ошибка с discountPercent

1. **Первый сигнал — счётчик `invalid` в ответе `/sync`**
   После прогона `/sync` посмотрела на ответ. `invalid > 0`, следовательно сервис молча выбросил часть заказов.

   ```bash
   curl -X POST http://localhost:3000/sync
   ```

   ```json
   {
     "pages": 52,
     "received": 5030,
     "applied": 0,
     "stale": 4853,
     "invalid": 177
   }
   ```

2. **Найти, где именно отбрасываются**
   `invalid` считаются в `src/services/sync.ts`. Там `invalid` увеличивается (`stats.invalid++`).
   Добавила `console.log(e)` для вывода ошибки в консоль:

   ```ts
   for (const raw of page.items) {
     try {
       const r = await this.orders.apply(raw);
       if (r === "stale") stats.stale++;
       else stats.applied++;
     } catch (e) {
       console.log("e", e);
       if (!(e instanceof ValidationError)) throw e;
       stats.invalid++;
     }
   }
   ```

   Ещё раз выполнила запрос (`http://localhost:3000/sync`). Вывелись ошибки, которые показали, что проблема в формате поля `discountPercent` (`ValidationError: invalid order: discountPercent`).

   Добавила дополнительный `console.log('raw', raw)`:

   ```ts
   catch (e) {
     console.log('e', e)
     console.log('raw', raw)
     if (!(e instanceof ValidationError)) throw e;
     stats.invalid++;
   }
   ```

   Ещё раз выполнила тот же запрос. Вывелось множество строк. Как пример, одна из них:

   ```js
   raw {
     id: 'o_01201',
     customerId: 'c_38',
     amount: 1757,
     discountPercent: '15',
     amountPaid: 1493.45,
     status: 'paid',
     createdAt: '2026-03-15T12:19:17.000Z',
     updatedAt: '2026-03-15T13:49:17.000Z'
   }
   ```

   Здесь `discountPercent` это строка.

3. **Сопоставила схему с реальными данными**
   Файл `services/orders`. Увидела, что `discountPercent` может быть только числом (`discountPercent: z.number().optional()`).
   Исправила: `discountPercent: z.union([z.number(), z.string()]).optional()`.
   Проверила, выполнив снова запрос `curl -X POST http://localhost:3000/sync`:

   ```json
   { "pages": 52, "received": 5030, "applied": 0, "stale": 5030, "invalid": 0 }
   ```

---

## Что ещё заметили в коде и в поведении внешнего API

- **`discountPercent` иногда приходит строкой** (`"33"`, `"15"`), а не числом — контракт из README (`number`) не соблюдается, документация отстаёт. Из-за этого при строгом `z.number()` 177 заказов отбрасывались как `invalid`.
- **Баг с таймзонами**: в `totals.ts` offset считался один раз «на сейчас» и применялся ко всем заказам — ломалось на переходе на летнее время (DST).
- **Гонка (race condition) в обработке вебхуков**: `OrdersService.apply()` делает неатомарные «прочитать, сравнить, записать». Вебхуки приходят с повторами и вперемешку (21 заказ × по 2–6 версий с разным `updatedAt`), и при параллельной обработке старая версия заказа может перезаписать новую. Из-за этого результат `daily-totals` между прогонами немного «плавает». Лечится `INSERT ... ON CONFLICT ... WHERE updated_at < excluded.updated_at`.
- **Не загружался `.env`** (не было `dotenv`) — сервис падал на fallback-порт `5432` вместо `5433`.

## Допущения (где требования неоднозначны)

- Если `amountPaid` отсутствует — скидки нет: `amountPaidKop = toKop(parseRub(amountPaid ?? amount))`, т.е. `net = gross`.
- День заказа считаем по `createdAt` (момент создания), а не по `updatedAt`/моменту оплаты.
- `discountPercent` принимаем как `number | string`, но не нормализуем и не сохраняем.
- Деньги храним и считаем в целых копейках; конвертация в рубли — через `fromKop`.
- Для старых строк БД (`amount_paid_kop IS NULL`) `net` при чтении равен `amount_kop`.

## Что бы доделали при наличии времени

- Инкрементальный `/sync`: хранить последний курсор и не обходить все страницы заново.
- Убрать гонку запросов
