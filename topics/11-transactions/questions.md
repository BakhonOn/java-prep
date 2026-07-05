# Транзакции

> 📇 Справочник уровня middle. Формат: ответ по сути → как работает в PostgreSQL/Spring → реальный пример → ❗ на чём ловят на собесе.

**Всего вопросов: 22**

---

## 1. Расшифруйте каждую букву ACID

**Atomicity (Атомарность)** — транзакция выполняется целиком или не выполняется вовсе. Если при банковском переводе списание прошло, а зачисление упало — откатывается всё. **Consistency (Согласованность)** — БД переходит из одного валидного состояния в другое: все ограничения (foreign key, check, unique) соблюдаются после коммита. **Isolation (Изолированность)** — параллельные транзакции не видят незавершённые изменения друг друга (в рамках выбранного уровня изоляции). **Durability (Надёжность)** — после коммита данные сохранены даже при аварийном отключении сервера (WAL-журнал в PostgreSQL).

В PostgreSQL атомарность и долговечность обеспечиваются WAL (Write-Ahead Log): изменения сначала пишутся в журнал, затем в основные файлы данных. При сбое база восстанавливается по журналу. Consistency — ответственность разработчика: правильные constraints, триггеры, бизнес-правила. Isolation — задача MVCC (в PostgreSQL) или блокировок (в MySQL InnoDB).

```sql
-- Банковский перевод: оба UPDATE должны выполниться или ни один
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT; -- или ROLLBACK при ошибке
```

❗ На собесе часто путают Consistency с Isolation. Consistency — это про инварианты базы (constraints, правила), а не про видимость чужих изменений. Видимость — это Isolation. Ещё ловушка: Durability не гарантирует сохранение при отказе диска без RAID/репликации — только при корректном завершении.

**→ Уточняющий вопрос:** Как WAL в PostgreSQL обеспечивает Durability и что происходит при восстановлении после сбоя?

**↳ Ответ:** WAL — append-only журнал, куда изменения пишутся *до* модификации основных файлов данных (heap); при `COMMIT` достаточно fsync WAL-записи, heap можно обновить позже. При сбое PostgreSQL стартует с последней checkpoint-позиции и повторно применяет все WAL-записи закоммиченных транзакций, пропуская незакоммиченные — heap возвращается в согласованное состояние без ручного вмешательства. Правило: Durability держится на WAL-fsync, а не на немедленной записи в heap; отключение fsync (`fsync=off`) убивает D из ACID ради скорости.

---

## 2. Какие уровни изоляции транзакций существуют?

Стандарт SQL-92 определяет четыре уровня изоляции: **Read Uncommitted → Read Committed → Repeatable Read → Serializable** — от слабого к строгому. Каждый следующий уровень устраняет дополнительный класс аномалий, но снижает параллелизм и увеличивает накладные расходы.

Соответствие уровней и аномалий по стандарту:

| Уровень           | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------------------|:----------:|:-------------------:|:------------:|
| Read Uncommitted  | ✅ возможно | ✅ возможно         | ✅ возможно  |
| Read Committed    | ❌ нет      | ✅ возможно         | ✅ возможно  |
| Repeatable Read   | ❌ нет      | ❌ нет              | ✅ возможно* |
| Serializable      | ❌ нет      | ❌ нет              | ❌ нет       |

*PostgreSQL на уровне Repeatable Read также блокирует фантомы через MVCC — это сильнее стандарта.

В Spring уровень изоляции задаётся через `@Transactional(isolation = Isolation.READ_COMMITTED)`. Если указать `Isolation.DEFAULT` — используется дефолт СУБД. Важно: в PostgreSQL Read Uncommitted на практике работает как Read Committed (реализация через MVCC не поддерживает грязное чтение).

❗ Ловушка: кандидаты думают, что Repeatable Read в PostgreSQL допускает фантомы (по стандарту). Но PostgreSQL использует snapshot isolation, и фантомные чтения там тоже предотвращены. MySQL InnoDB решает это через gap locks, PostgreSQL — через MVCC-снимки.

**→ Уточняющий вопрос:** Чем MVCC в PostgreSQL отличается от блокировочного подхода MySQL при реализации Repeatable Read?

**↳ Ответ:** PostgreSQL MVCC: каждый SELECT видит «снимок» прошлого, читатели никогда не блокируют писателей и наоборот — нет lock contention при concurrent read+write. MySQL InnoDB тоже использует MVCC для обычных SELECT, но для `SELECT ... FOR UPDATE` переключается на current read с gap locks, блокируя диапазоны индекса и создавая риск deadlock'ов. Правило: PostgreSQL — «снимки без блокировок», MySQL — «снимки + блокировки диапазонов»; если видите deadlock из-за уровня изоляции — скорее всего MySQL с gap locks.

---

## 3. Что такое Read Uncommitted?

Read Uncommitted — самый слабый уровень изоляции: транзакция может читать данные, изменённые другой транзакцией, которая ещё не закоммитила (dirty read). Если та транзакция откатится — прочитанные данные окажутся несуществующими.

На практике используется крайне редко — только когда нужна максимальная скорость, а точность данных некритична (например, примерная статистика). В PostgreSQL этот уровень на деле работает как Read Committed: архитектура MVCC физически не позволяет видеть незакоммиченные строки.

```sql
-- Сессия 1: начала перевод, но не закоммитила
BEGIN;
UPDATE accounts SET balance = 50000 WHERE id = 1; -- было 1000

-- Сессия 2 (Read Uncommitted в MySQL):
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
SELECT balance FROM accounts WHERE id = 1; -- видит 50000 (грязные данные!)

-- Сессия 1:
ROLLBACK; -- откатилась — баланс снова 1000, но сессия 2 уже прочитала 50000
```

❗ Ловушка: кандидаты говорят «Read Uncommitted в PostgreSQL позволяет грязное чтение». Нет — PostgreSQL документально заявляет, что Read Uncommitted трактуется как Read Committed. Грязное чтение возможно только в MySQL/SQL Server.

**→ Уточняющий вопрос:** В каком реальном сценарии можно обоснованно использовать Read Uncommitted?

**↳ Ответ:** Read Uncommitted оправдан для быстро меняющихся приближённых метрик, где погрешность приемлема: счётчик просмотров страницы, онлайн-пользователей на дашборде, real-time топ популярных товаров. Ошибка на 1-2% не влечёт финансовых последствий, зато исчезает lock contention. Правило: если данные — «ориентир, а не истина» и потеря точности не нарушает инвариантов — Read Uncommitted даёт максимальный параллелизм без блокировок.

---

## 4. Что такое Read Committed?

Read Committed — уровень изоляции, при котором транзакция видит только **закоммиченные** данные. Грязное чтение исключено, но возможны неповторяющееся чтение и фантомы: если другая транзакция закоммитила изменение между двумя SELECT в одной транзакции — результаты будут разными.

Это дефолтный уровень в PostgreSQL и Oracle. В PostgreSQL реализован через MVCC: каждый SELECT получает снимок состояния БД на момент начала этого конкретного запроса (не транзакции). Это значит, что два SELECT в одной транзакции могут видеть разные данные, если между ними была чужая фиксация.

```java
// Spring: явно указываем READ_COMMITTED (или полагаемся на дефолт PostgreSQL)
@Transactional(isolation = Isolation.READ_COMMITTED)
public void processOrder(Long orderId) {
    Order order = orderRepo.findById(orderId).orElseThrow();
    // Если другая транзакция изменила заказ между двумя вызовами findById — 
    // мы увидим обновлённые данные при повторном чтении
    reserveStock(order);
}
```

❗ Ловушка: при Read Committed в PostgreSQL снимок берётся на момент каждого отдельного запроса, а не на момент начала транзакции. Поэтому `SELECT ... FOR UPDATE` в цикле может поймать обновлённые данные — это неожиданное поведение для тех, кто привык к MySQL.

**→ Уточняющий вопрос:** Как Read Committed защищает от dirty read на уровне MVCC в PostgreSQL — что именно происходит с незакоммиченными версиями строк?

**↳ Ответ:** Каждая версия строки (heap tuple) хранит `xmin` — ID транзакции-создателя и `xmax` — ID удалившей/обновившей. При SELECT PostgreSQL строит snapshot и проверяет: строка видима только если `xmin`-транзакция уже закоммичена (статус проверяется в `pg_xact`/clog). Незакоммиченная строка физически лежит на диске, но её xmin имеет статус «in progress» — snapshot её просто не включает. Правило: dirty read в PostgreSQL невозможен архитектурно, не зависит от уровня изоляции.

---

## 5. Что такое Repeatable Read?

Repeatable Read гарантирует, что если транзакция прочитала строку, повторное чтение той же строки вернёт то же значение — даже если другая транзакция изменила и закоммитила её между чтениями. Снимок берётся один раз — на момент начала **транзакции** (не каждого запроса).

В PostgreSQL Repeatable Read реализован через MVCC-снимок всей транзакции: все SELECT видят одно и то же состояние БД на момент первого запроса в транзакции. Кроме того, PostgreSQL предотвращает и фантомные чтения на этом уровне (что выше стандарта SQL-92). Если другая транзакция изменила строку, которую вы пытаетесь обновить — PostgreSQL выбросит `ERROR: could not serialize access due to concurrent update`.

```sql
-- PostgreSQL, уровень Repeatable Read
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1; -- 1000
-- Другая транзакция делает UPDATE и COMMIT: balance = 500
SELECT balance FROM accounts WHERE id = 1; -- всё равно 1000 (снимок!)
COMMIT;
```

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public BigDecimal calculateTotalForAudit(Long userId) {
    // Оба SELECT гарантированно видят одно состояние — важно для финансовых отчётов
    BigDecimal balance = accountRepo.getBalance(userId);
    BigDecimal reserved = reservationRepo.getReservedAmount(userId);
    return balance.subtract(reserved);
}
```

❗ Ловушка: многие думают, что Repeatable Read в PostgreSQL допускает фантомы (так написано в стандарте). Это неверно для PostgreSQL — его snapshot isolation блокирует и фантомы. Но в MySQL InnoDB фантомы на Repeatable Read всё ещё возможны в некоторых сценариях (например, при `SELECT` без блокировки vs `INSERT` в другой транзакции).

**→ Уточняющий вопрос:** Что произойдёт в PostgreSQL на уровне Repeatable Read, если две транзакции одновременно попытаются обновить одну строку?

**↳ Ответ:** Первая транзакция захватывает row-level lock и выполняет UPDATE. Вторая ждёт снятия блокировки. Когда первая коммитит — PostgreSQL обнаруживает, что строка изменилась с момента snapshot'а второй транзакции и выбрасывает `ERROR: could not serialize access due to concurrent update`. Правило: при Repeatable Read конкурентный UPDATE → один из двух получает ошибку; приложение должно быть готово к retry.

---

## 6. Что такое Serializable?

Serializable — строжайший уровень изоляции: транзакции выполняются так, как будто они идут строго последовательно, одна за другой. Все аномалии (dirty read, non-repeatable read, phantom read) исключены. Ценой является снижение параллелизма и возможные ошибки сериализации (`ERROR: could not serialize access`).

В PostgreSQL Serializable реализован через SSI (Serializable Snapshot Isolation): база отслеживает зависимости чтения-записи между транзакциями и при обнаружении цикла (что означало бы нарушение сериализуемости) откатывает одну из них. Это эффективнее классических предикатных блокировок. Приложение должно быть готово к retry при получении `SerializationFailureException`.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferWithSerializable(Long fromId, Long toId, BigDecimal amount) {
    // Гарантия: никакие параллельные транзакции не создадут аномалий
    Account from = accountRepo.findById(fromId).orElseThrow();
    Account to = accountRepo.findById(toId).orElseThrow();
    if (from.getBalance().compareTo(amount) < 0) throw new InsufficientFundsException();
    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));
    // При конфликте сериализации Spring выбросит исключение — нужен retry
}
```

❗ Ловушка: Serializable не означает «только один поток работает с БД». Транзакции выполняются параллельно, но результат должен быть эквивалентен какому-то последовательному порядку. При конфликте одна транзакция получает ошибку сериализации и должна быть повторена — если в коде нет retry-логики, транзакция просто упадёт.

**→ Уточняющий вопрос:** Чем SSI (Serializable Snapshot Isolation) в PostgreSQL отличается от традиционных предикатных блокировок и почему это важно для производительности?

**↳ Ответ:** Традиционные предикатные блокировки блокируют диапазоны индекса при чтении — читатели мешают писателям, что сильно снижает параллелизм. SSI в PostgreSQL отслеживает граф rw-зависимостей между транзакциями и откатывает только те, что образуют цикл (= нарушение сериализуемости), не блокируя остальных. Правило: SSI — «разрешаем всё, штрафуем только нарушителей» vs предикатные блокировки — «запрещаем всё на всякий случай»; при редких конфликтах SSI на порядок эффективнее.

---

## 7. Что такое грязное чтение (Dirty Read)?

Грязное чтение — ситуация, когда транзакция A читает данные, изменённые транзакцией B, которая ещё **не зафиксировала** изменения. Если B откатится — A уже работала с несуществующими данными, что может привести к серьёзным ошибкам.

Классический пример: платёжная система. Транзакция B начала перевод 10 000 руб. и списала деньги со счёта (баланс = 0), но не закоммитила. Транзакция A в этот момент читает баланс = 0 и отказывает в новом платеже. Затем B откатывается (баланс снова = 10 000) — клиент был незаконно заблокирован. Предотвращается начиная с уровня Read Committed.

```sql
-- MySQL, демонстрация dirty read
-- Сессия B:
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
UPDATE accounts SET balance = 0 WHERE id = 42; -- не коммитим!

-- Сессия A (тоже READ UNCOMMITTED):
BEGIN;
SELECT balance FROM accounts WHERE id = 42; -- видит 0 — ГРЯЗНОЕ ЧТЕНИЕ
-- принимает решение на основе несуществующих данных

-- Сессия B:
ROLLBACK; -- баланс снова прежний, но сессия A уже приняла неверное решение
```

❗ Ловушка: в PostgreSQL грязное чтение невозможно в принципе — даже на уровне Read Uncommitted, потому что MVCC хранит версии строк и никогда не показывает незакоммиченные версии. Если скажете «PostgreSQL поддерживает грязное чтение на Read Uncommitted» — это ошибка.

**→ Уточняющий вопрос:** Как MVCC в PostgreSQL физически предотвращает грязное чтение — что хранится в строке таблицы?

**↳ Ответ:** Каждая heap-версия строки содержит системные поля `xmin` (txid транзакции, создавшей строку) и `xmax` (txid удалившей/обновившей). При чтении PostgreSQL сверяет `xmin` с активным snapshot: строка видима только если xmin-транзакция закоммичена *до* начала snapshot. Незакоммиченные строки физически существуют на диске, но их xmin-транзакция числится «in progress» — snapshot их просто игнорирует без каких-либо блокировок.

---

## 8. Что такое неповторяющееся чтение (Non-Repeatable Read)?

Неповторяющееся чтение — ситуация, когда транзакция A дважды читает одну строку и получает разные значения, потому что между чтениями транзакция B изменила эту строку и закоммитила изменение. Данные не «грязные» (B зафиксирована), но результат нестабилен внутри одной транзакции A.

Реальный пример: оформление заказа. Транзакция A читает цену товара (1000 руб.), выполняет расчёты, затем снова читает цену перед формированием счёта. Если за это время маркетинг изменил цену на 800 руб. — финальная сумма не совпадёт с тем, что пользователь видел. Решается уровнем Repeatable Read, где снимок фиксируется на начало транзакции.

```java
// Проблема на Read Committed:
@Transactional(isolation = Isolation.READ_COMMITTED)
public Invoice createInvoice(Long orderId) {
    BigDecimal price = productRepo.getPrice(orderId); // 1000 руб.
    // ... долгая бизнес-логика ...
    BigDecimal finalPrice = productRepo.getPrice(orderId); // может быть 800 руб.!
    return new Invoice(finalPrice); // несоответствие
}

// Решение — Repeatable Read:
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Invoice createInvoice(Long orderId) {
    BigDecimal price = productRepo.getPrice(orderId); // 1000 руб.
    // ... любые операции ...
    BigDecimal finalPrice = productRepo.getPrice(orderId); // гарантированно 1000 руб.
    return new Invoice(finalPrice);
}
```

❗ Ловушка: non-repeatable read касается **изменения** существующей строки (UPDATE), а phantom read — **появления новых строк** (INSERT/DELETE). Это разные аномалии с разными способами решения. Repeatable Read устраняет первое, но по стандарту не обязан устранять второе.

**→ Уточняющий вопрос:** В каких реальных бизнес-сценариях non-repeatable read приводит к финансовым ошибкам и как Hibernate помогает это обнаружить?

**↳ Ответ:** Типичный сценарий: расчёт инвойса читает цену, делает скидочный расчёт, потом читает цену снова для итога — если между двумя чтениями маркетинг изменил цену, клиент получает некорректную сумму. Hibernate помогает через `@Version`: при `save()` генерируется `WHERE version = <прочитанное>` — если строка обновилась с момента чтения, 0 rows affected → `OptimisticLockException`, сигнализируя о конфликте. Правило: для финансовых расчётов с несколькими SELECT — либо Repeatable Read, либо `@Version`.

---

## 9. Что такое фантомное чтение (Phantom Read)?

Фантомное чтение — ситуация, когда транзакция A выполняет один и тот же запрос с условием (например, `WHERE status = 'NEW'`) дважды и получает разное количество строк, потому что транзакция B добавила или удалила строки, подходящие под это условие.

Реальный пример: система обработки заказов. Транзакция A считает количество новых заказов — их 5. Затем транзакция B создаёт 3 новых заказа и коммитит. Транзакция A повторяет запрос — теперь 8. Это и есть фантомы. Phantom read устраняется на уровне Serializable. В PostgreSQL благодаря snapshot isolation фантомы устранены уже на уровне Repeatable Read.

```sql
-- PostgreSQL Repeatable Read — фантомы предотвращены:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM orders WHERE status = 'NEW'; -- 5

-- Другая транзакция: INSERT INTO orders ... (status='NEW') COMMIT;

SELECT COUNT(*) FROM orders WHERE status = 'NEW'; -- всё равно 5 (снимок!)
COMMIT;

-- MySQL Repeatable Read — фантомы возможны при смешанных блокировках:
-- Обычный SELECT не увидит фантомы (consistent read),
-- но SELECT ... FOR UPDATE может их увидеть (current read)
```

❗ Ловушка: в PostgreSQL фантомы на Repeatable Read невозможны, но в MySQL InnoDB — могут быть при использовании `SELECT ... FOR UPDATE` или `SELECT ... LOCK IN SHARE MODE` (current read). Многие кандидаты не знают об этом различии между consistent read и current read в InnoDB.

**→ Уточняющий вопрос:** Как gap locks в MySQL InnoDB предотвращают фантомные чтения при использовании `SELECT ... FOR UPDATE`?

**↳ Ответ:** `SELECT ... FOR UPDATE` переключает InnoDB с MVCC-consistent read на current read — видит актуальные данные, не снимок. Чтобы другая транзакция не могла вставить строку в выбранный диапазон, InnoDB ставит gap lock на промежутки индекса между существующими значениями. Правило: gap lock «запрещает INSERT в диапазон» — если вы читаете `WHERE id BETWEEN 10 AND 20`, никто не вставит `id=15` до вашего COMMIT; именно gap locks и порождают deadlock'и в MySQL на Repeatable Read.

---

## 10. Что такое потерянное обновление (Lost Update)?

Потерянное обновление (Lost Update) — ситуация, когда две транзакции читают одно значение, затем обе его обновляют: первая транзакция записывает результат, затем вторая перезаписывает его своим значением, **не учитывая** изменение первой. Обновление первой транзакции «потеряно».

Классический пример: два менеджера одновременно обновляют количество товара на складе. Оба читают остаток = 100. Первый уменьшает на 30 → пишет 70. Второй уменьшает на 20 → пишет 80 (не 70!). В итоге вместо 50 на складе значится 80. Решается двумя способами: **пессимистичная блокировка** (`SELECT ... FOR UPDATE`) или **оптимистичная блокировка** (`@Version` в Hibernate).

```java
// Оптимистичная блокировка через @Version — Hibernate бросит OptimisticLockException
@Entity
public class Product {
    @Id Long id;
    int stock;
    @Version long version; // автоматически инкрементируется при UPDATE
}

// Если два потока одновременно прочитали version=1 и попытались сохранить:
// первый UPDATE: SET stock=70, version=2 WHERE id=1 AND version=1 — успех
// второй UPDATE: SET stock=80, version=2 WHERE id=1 AND version=1 — 0 rows affected → исключение

// Пессимистичная блокировка:
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Product findByIdForUpdate(@Param("id") Long id);
// Транслируется в: SELECT ... FROM products WHERE id=? FOR UPDATE
```

❗ Ловушка: Lost Update — это **не** аномалия стандартных уровней изоляции SQL-92. Read Committed не защищает от него при паттерне «read-modify-write» в приложении. Нужны либо блокировки, либо `@Version`. Многие кандидаты думают, что Repeatable Read автоматически решает проблему — это неверно для сценария двух параллельных UPDATE.

**→ Уточняющий вопрос:** Чем отличается поведение Hibernate при `OptimisticLockException` от `PessimisticLockException` и как правильно реализовать retry?

**↳ Ответ:** `OptimisticLockException` — транзакция откатилась при flush (конфликт version), retry нужно с самого начала бизнес-метода в новой транзакции. `PessimisticLockException` — не удалось получить блокировку (timeout или deadlock), транзакция может быть ещё активна, но безопаснее тоже стартовать заново. Правило: retry-цикл должен быть *снаружи* `@Transactional`-метода — никогда не retry внутри открытой транзакции, иначе вы работаете в уже «сломанном» контексте.

---

## 11. Какой уровень изоляции по умолчанию в PostgreSQL?

По умолчанию в PostgreSQL используется **Read Committed**. Это означает, что каждый SQL-запрос внутри транзакции получает свежий снимок данных на момент начала именно этого запроса, а не всей транзакции. Таким образом, два последовательных SELECT в одной транзакции могут видеть разные данные, если между ними другая транзакция зафиксировала изменения.

Это сознательный компромисс между изолированностью и производительностью: Read Committed даёт хороший параллелизм без dirty reads. Для большинства OLTP-сценариев этого достаточно. Если бизнес-логика требует стабильного снимка (финансовые отчёты, многошаговые вычисления) — нужно явно повышать до Repeatable Read.

```java
// В Spring: явно задать уровень для конкретного метода
@Transactional(isolation = Isolation.READ_COMMITTED) // явно, хотя это дефолт для PG
public void defaultBehavior() { ... }

@Transactional(isolation = Isolation.REPEATABLE_READ) // когда нужен стабильный снимок
public FinancialReport generateReport(LocalDate date) { ... }
```

❗ Ловушка: если в Spring указать `isolation = Isolation.DEFAULT` — Spring использует дефолт конкретной СУБД. Это значит, что один и тот же Spring-код будет вести себя по-разному на PostgreSQL (Read Committed) и MySQL (Repeatable Read). Это частая причина багов при смене или миграции БД.

**→ Уточняющий вопрос:** Как можно изменить уровень изоляции по умолчанию для всего приложения в Spring Boot и насколько это оправданно?

**↳ Ответ:** Через свойство пула соединений: `spring.datasource.hikari.transaction-isolation=TRANSACTION_REPEATABLE_READ` — HikariCP установит уровень на каждом connection. Но глобальное повышение редко оправданно: Serializable увеличивает число ошибок сериализации (нужен retry), Repeatable Read в MySQL добавляет gap lock deadlock'и. Правило: точечно указывайте `@Transactional(isolation=...)` там, где нужна повышенная изоляция — не поднимайте планку для всего приложения ради нескольких методов.

---

## 12. Какой уровень изоляции по умолчанию в MySQL?

По умолчанию в MySQL (InnoDB) используется **Repeatable Read**. Это означает, что снимок данных фиксируется на момент первого запроса в транзакции и все последующие SELECT видят одно и то же состояние. MySQL реализует Repeatable Read через комбинацию MVCC (для consistent reads) и gap locks (для блокировочных чтений).

Выбор Repeatable Read как дефолта обусловлен историческими причинами — это был необходимый уровень для корректной работы binlog-репликации в старых версиях MySQL (statement-based replication). Сейчас с row-based replication это ограничение снято, но дефолт остался. На практике это означает более строгое поведение, чем PostgreSQL, но также и более высокий риск deadlock'ов из-за gap locks.

```sql
-- Проверить текущий уровень изоляции в MySQL:
SELECT @@transaction_isolation; -- REPEATABLE-READ

-- Изменить для сессии:
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Изменить глобально (в my.cnf):
-- transaction-isolation = READ-COMMITTED
```

❗ Ловушка: разница дефолтов PostgreSQL (RC) vs MySQL (RR) — классический вопрос на собесе. Если Spring-приложение переносится с PostgreSQL на MySQL без явного указания `isolation` — поведение транзакций изменится. Также в MySQL Repeatable Read с gap locks может вызывать deadlock'и там, где в PostgreSQL их не было.

**→ Уточняющий вопрос:** Почему MySQL исторически выбрал Repeatable Read как дефолт и как это связано с binlog-репликацией?

**↳ Ответ:** При statement-based binlog репликации (исторический дефолт MySQL) один и тот же SQL-запрос должен давать идентичный результат на master и slave. На Read Committed `INSERT ... SELECT` мог видеть разные строки из-за параллельных коммитов — репликация ломалась и данные расходились. Repeatable Read + gap locks стабилизировал выполнение запросов. Правило: с row-based replication (MySQL 5.1+) это ограничение снято, но дефолт RR остался как legacy — и теперь приносит с собой gap lock deadlock'и в современных приложениях.

---

## 13. Что такое Propagation в Spring?

Propagation (распространение транзакции) — это поведение Spring при вызове `@Transactional`-метода: что делать с транзакцией, если она уже существует в текущем потоке. Spring поддерживает семь значений: `REQUIRED`, `REQUIRES_NEW`, `SUPPORTS`, `NOT_SUPPORTED`, `MANDATORY`, `NEVER`, `NESTED`.

Управление транзакциями реализовано через `TransactionSynchronizationManager`, который хранит текущую транзакцию в `ThreadLocal`. При входе в `@Transactional`-метод Spring AOP-прокси проверяет `ThreadLocal`: есть ли активная транзакция. В зависимости от Propagation либо присоединяется к ней, либо создаёт новую, либо приостанавливает текущую.

```java
@Service
public class OrderService {
    @Transactional(propagation = Propagation.REQUIRED) // дефолт
    public void placeOrder(Order order) {
        orderRepo.save(order);
        auditService.log("ORDER_PLACED", order.getId()); // вызов другого @Transactional
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW) // независимая транзакция
    public void log(String event, Long entityId) {
        auditRepo.save(new AuditLog(event, entityId));
        // Если OrderService откатится — этот лог всё равно сохранится
    }
}
```

❗ Ловушка: `Propagation.SUPPORTS` присоединяется к транзакции если она есть, но если нет — выполняется без транзакции (а не создаёт новую). Это не то же самое, что `REQUIRED`. Многие путают SUPPORTS с REQUIRED.

**→ Уточняющий вопрос:** Когда стоит использовать `Propagation.MANDATORY` и что произойдёт если вызвать такой метод без активной транзакции?

**↳ Ответ:** `MANDATORY` подходит для методов, которые обязаны вызываться только внутри уже открытой транзакции — например, helper-методы с критичными записями, которые не имеют смысла без внешнего контекста. При вызове без активной транзакции Spring немедленно бросает `IllegalTransactionStateException`. Правило: MANDATORY — это «охранный assert» на уровне транзакций; если метод может вызываться самостоятельно — используйте REQUIRED, а не MANDATORY.

---

## 14. Что делает Propagation.NESTED?

`Propagation.NESTED` создаёт вложенную транзакцию внутри существующей с помощью **savepoint**. При откате вложенной транзакции откат происходит только до savepoint — внешняя транзакция продолжает работу. Если откатывается внешняя транзакция — откатываются и все вложенные.

Реализовано через `SAVEPOINT` в SQL: Spring создаёт savepoint перед вызовом вложенного метода, и при исключении выполняет `ROLLBACK TO SAVEPOINT`. Требует поддержки savepoint в драйвере JDBC (PostgreSQL и MySQL поддерживают). Важно: NESTED не создаёт отдельную транзакцию в БД — это одна транзакция с точкой отката.

```java
@Transactional
public void processOrderWithOptionalNotification(Long orderId) {
    Order order = orderRepo.findById(orderId).orElseThrow();
    order.setStatus(OrderStatus.CONFIRMED);
    orderRepo.save(order); // основная операция

    try {
        notificationService.sendEmail(order); // NESTED — может упасть без потери заказа
    } catch (Exception e) {
        // Откат только до savepoint: email не отправлен, но заказ сохранён
        log.warn("Notification failed, order still confirmed", e);
    }
}

@Transactional(propagation = Propagation.NESTED)
public void sendEmail(Order order) {
    // Если упадёт — внешняя транзакция продолжит работу
    emailRepo.save(new EmailLog(order));
    emailClient.send(order.getCustomerEmail(), buildMessage(order));
}
```

❗ Ловушка: NESTED и REQUIRES_NEW часто путают. Ключевое отличие: **REQUIRES_NEW создаёт отдельную транзакцию** (коммитится независимо), **NESTED — savepoint внутри той же транзакции** (откат внешней отменит и вложенную). Если нужен независимый аудит-лог — только REQUIRES_NEW. Если нужна «попытка с возможностью отмены части» — NESTED.

**→ Уточняющий вопрос:** Что произойдёт с NESTED-транзакцией если внешняя транзакция откатится после успешного завершения вложенной?

**↳ Ответ:** NESTED использует SAVEPOINT внутри одной транзакции БД — это единая транзакция, просто с точкой отката. Откат внешней транзакции откатывает *всё*, включая изменения вложенной (даже если вложенная завершилась успешно, её savepoint — часть той же транзакции). Правило: NESTED ≠ независимый коммит; если нужно «сохранить аудит-лог даже при откате основной операции» — только REQUIRES_NEW (отдельная транзакция).

---

## 15. В чём разница между REQUIRED и REQUIRES_NEW?

**REQUIRED** (дефолт): если активная транзакция уже есть — присоединяется к ней; если нет — создаёт новую. Оба метода работают в **одной** транзакции, их судьбы связаны: если внутренний метод выбрасывает исключение — транзакция помечается как rollback-only и внешняя тоже будет откачена.

**REQUIRES_NEW**: всегда создаёт **новую независимую** транзакцию, текущая приостанавливается (suspend). У неё свой коммит и откат, не зависящие от внешней. Используется когда нужно сохранить данные независимо от судьбы вызывающей транзакции — классический пример: аудит-лог должен сохраниться, даже если основная операция откатилась.

```java
@Service
public class PaymentService {
    @Autowired AuditService auditService;

    @Transactional // REQUIRED: создаёт транзакцию T1
    public void processPayment(Payment payment) {
        paymentRepo.save(payment);
        auditService.logPaymentAttempt(payment); // REQUIRES_NEW: создаёт T2, T1 приостановлена

        if (payment.getAmount().compareTo(limit) > 0) {
            throw new LimitExceededException(); // T1 откатится, но T2 уже закоммичена!
        }
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logPaymentAttempt(Payment payment) {
        auditRepo.save(new AuditLog("PAYMENT_ATTEMPT", payment)); // сохранится в любом случае
    }
}
```

❗ Ловушка: REQUIRES_NEW может вызвать deadlock. Транзакция T1 заблокировала строку, T2 пытается заблокировать ту же строку — взаимная блокировка. Кроме того, T2 **не видит незакоммиченных изменений T1** (они из разных транзакций), что иногда неожиданно для разработчика.

**→ Уточняющий вопрос:** В каком сценарии REQUIRES_NEW может вызвать deadlock и как это предотвратить?

**↳ Ответ:** REQUIRES_NEW приостанавливает внешнюю транзакцию T1, но T1 *не освобождает* свои блокировки. Если T2 (REQUIRES_NEW) пытается заблокировать строку, которую уже держит T1, — deadlock: T1 ждёт окончания T2, T2 ждёт освобождения блокировки T1. Правило: REQUIRES_NEW метод должен работать только с данными, которые T1 ещё не блокировала — выносите его в таблицы с другой областью данных (например, только в `audit_log`).

---

## 16. Что такое аннотация @Transactional?

`@Transactional` — аннотация Spring для декларативного управления транзакциями. При её использовании Spring создаёт AOP-прокси вокруг бина: перед вызовом метода открывается транзакция (или присоединяется к существующей), после успешного завершения — коммит, при исключении — откат (в зависимости от типа исключения).

Под капотом работает `PlatformTransactionManager` (например, `JpaTransactionManager`). Spring перехватывает вызов через CGLIB-прокси (или JDK dynamic proxy для интерфейсов), получает или создаёт соединение с БД, устанавливает нужный уровень изоляции и propagation, выполняет метод, обрабатывает результат. Все операции с БД в рамках одного потока используют одно соединение благодаря `TransactionSynchronizationManager` и `ThreadLocal`.

```java
@Service
public class AccountService {
    @Transactional(
        isolation = Isolation.READ_COMMITTED,
        propagation = Propagation.REQUIRED,
        rollbackFor = Exception.class,
        timeout = 30,         // откат если метод выполняется > 30 сек
        readOnly = false
    )
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepo.findById(fromId).orElseThrow();
        Account to = accountRepo.findById(toId).orElseThrow();
        from.debit(amount);
        to.credit(amount);
        // Spring сделает COMMIT автоматически при выходе из метода
    }
}
```

❗ Ловушка: `@Transactional` работает только через прокси — только при вызове метода **из другого бина**. Вызов `@Transactional`-метода из того же класса (self-invocation) через `this` минует прокси и транзакция не откроется. Это одна из самых частых ошибок в реальных проектах.

**→ Уточняющий вопрос:** Как именно Spring создаёт прокси для `@Transactional` — CGLIB или JDK proxy, и в чём разница?

**↳ Ответ:** JDK dynamic proxy работает только для классов с интерфейсом — создаёт объект, реализующий тот же интерфейс и делегирующий вызовы. CGLIB создаёт подкласс целевого класса на лету — работает без интерфейса, но требует, чтобы класс и методы не были `final`. Spring Boot по умолчанию предпочитает CGLIB (`proxyTargetClass=true`). Правило: `final class` или `final method` → CGLIB не сможет создать подкласс → `@Transactional` молча не работает.

---

## 17. На каком уровне можно использовать @Transactional?

`@Transactional` можно ставить на **public-методы** и на **классы** (тогда применяется ко всем public-методам класса). Spring AOP создаёт прокси, который может перехватывать только публичные методы — это ограничение CGLIB/JDK proxy.

На `private` и `protected` методах аннотация не работает: прокси их не перехватит. Аннотация на уровне класса задаёт дефолтные настройки, которые можно переопределить на уровне конкретного метода. Хорошая практика — аннотировать на уровне класса и переопределять там, где нужны специфические настройки (например, `readOnly = true` для методов чтения).

```java
@Transactional(readOnly = true) // дефолт для всего класса — только чтение
@Service
public class ProductService {

    public List<Product> findAll() { ... } // readOnly = true (от класса)

    public Product findById(Long id) { ... } // readOnly = true (от класса)

    @Transactional // переопределяем: readOnly = false, для записи
    public Product save(Product product) { ... }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updatePrice(Long id, BigDecimal price) { ... } // своя propagation

    // private — @Transactional НЕ работает, прокси не перехватит:
    private void internalUpdate(Product p) { ... }
}
```

❗ Ловушка: аннотация `@Transactional` на интерфейсе работает только с JDK dynamic proxy (когда Spring создаёт прокси через интерфейс). При использовании CGLIB (когда Spring Boot проксирует классы напрямую) аннотация на интерфейсе игнорируется. Рекомендация Spring — всегда ставить `@Transactional` на реализацию, не на интерфейс.

**→ Уточняющий вопрос:** Что происходит если поставить `@Transactional` на protected-метод и вызвать его из подкласса того же бина?

**↳ Ответ:** Spring AOP по умолчанию не регистрирует pointcut для protected-методов — аннотация `@Transactional` на них игнорируется. Даже если CGLIB технически переопределит метод в подклассе-прокси, Spring не добавит transactional advice. При вызове из подкласса через `this` — ещё и self-invocation проблема. Правило: `@Transactional` только на public методах; для protected применяйте AspectJ weaving, если это действительно нужно.

---

## 18. Что такое rollback в транзакциях?

Rollback — откат всех изменений, сделанных в рамках транзакции, к состоянию, которое было до её начала. В PostgreSQL это реализуется через MVCC: при откате старые версии строк (xmin/xmax) остаются видимыми для других транзакций, а изменения текущей транзакции просто перестают быть видимыми (txid помечается как aborted в clog). Никакого физического «возврата» данных нет — просто меняется видимость версий.

В Spring rollback происходит автоматически при выбросе исключения из `@Transactional`-метода. Spring перехватывает исключение в AOP-advice, проверяет, должен ли данный тип исключения вызывать rollback (по правилам rollbackFor/noRollbackFor), и вызывает `transactionManager.rollback(status)`. Явный rollback можно вызвать через `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.

```java
@Transactional
public void processOrder(Order order) {
    orderRepo.save(order);
    stockService.reserveItems(order); // может бросить исключение

    // Явная пометка на откат без выброса исключения:
    if (!paymentService.charge(order)) {
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        // метод завершится нормально, но транзакция откатится
    }
}
```

❗ Ловушка: если транзакция помечена как `rollback-only` (например, внутренним методом), а внешний код пытается её закоммитить — Spring выбросит `UnexpectedRollbackException`. Это частая неожиданность при Propagation.REQUIRED: внутренний метод поймал исключение, пометил транзакцию как rollback-only, внешний метод думает что всё хорошо и транзакция закоммитится — но нет.

**→ Уточняющий вопрос:** Что такое `UnexpectedRollbackException` и в каком сценарии с Propagation.REQUIRED она возникает?

**↳ Ответ:** Внутренний `@Transactional(REQUIRED)` метод выбросил исключение — Spring пометил транзакцию как `rollback-only`. Внешний метод поймал исключение через `try-catch` и думает, что всё нормально — но при попытке закоммитить получает `UnexpectedRollbackException`, потому что транзакция уже «мертва». Правило: если внутренний REQUIRED-метод выбросил исключение и вы его поглотили — транзакция всё равно не коммитится; единственное решение — использовать REQUIRES_NEW для независимого внутреннего метода.

---

## 19. Какие исключения по умолчанию вызывают rollback?

По умолчанию Spring выполняет rollback только при **unchecked-исключениях** (`RuntimeException` и его наследниках) и при `Error`. Checked-исключения (наследники `Exception`, кроме `RuntimeException`) по умолчанию **не вызывают** rollback — Spring считает их частью нормального бизнес-потока.

Это правило унаследовано из EJB-спецификации и нередко удивляет разработчиков. Логика такая: checked исключения предполагают, что вызывающий код их обработает, а значит — транзакцию не обязательно откатывать. На практике это приводит к неожиданным коммитам при бизнес-ошибках, если разработчик не указал `rollbackFor`.

```java
// Пример ловушки: checked исключение НЕ вызывает rollback
@Transactional
public void createUser(User user) throws UserAlreadyExistsException {
    userRepo.save(user);         // сохранено
    emailService.sendWelcome(user); // бросает checked UserAlreadyExistsException
    // Транзакция ЗАКОММИТИТСЯ, несмотря на исключение!
    // user сохранён в БД, хотя метод завершился с ошибкой
}

// RuntimeException — rollback произойдёт:
@Transactional
public void transfer(Long from, Long to, BigDecimal amount) {
    debit(from, amount);
    if (!hasCredit(to)) throw new IllegalStateException("Account blocked"); // rollback!
    credit(to, amount);
}
```

❗ Ловушка: это самая частая ошибка с транзакциями в Spring. Разработчик бросает `IOException`, `SQLException`, собственный `BusinessException extends Exception` — и транзакция коммитится. Данные частично сохраняются. Решение: либо `rollbackFor = Exception.class`, либо делать бизнес-исключения unchecked.

**→ Уточняющий вопрос:** Почему в современных Java-проектах принято делать кастомные бизнес-исключения unchecked, и как это связано с транзакциями?

**↳ Ответ:** Checked exceptions засоряют сигнатуры методов и нарушают принцип единственной ответственности: каждый промежуточный слой вынужден либо объявлять `throws`, либо оборачивать. В контексте транзакций: unchecked автоматически вызывает rollback по умолчанию в Spring — не нужно помнить про `rollbackFor`. Правило: `extends RuntimeException` → и API чище, и транзакции безопаснее (rollback by default); `extends Exception` → нужен явный `rollbackFor` везде, иначе partial commit.

---

## 20. Как настроить rollback для checked exceptions?

Для включения rollback для checked-исключений используется параметр `rollbackFor` в `@Transactional`. Можно указать конкретный класс или общий `Exception.class` для отката при любом исключении. Обратный параметр — `noRollbackFor` — исключает конкретный тип из правил rollback.

Это критично при работе с `IOException`, `SQLException` и кастомными бизнес-исключениями типа `InsufficientFundsException extends Exception`. Хорошая практика в крупных проектах — задать `rollbackFor = Exception.class` на уровне класса через `@Transactional` на классе и переопределять там, где нужно другое поведение.

```java
// Явно для конкретного исключения:
@Transactional(rollbackFor = InsufficientFundsException.class)
public void transfer(Account from, Account to, BigDecimal amount)
        throws InsufficientFundsException {
    if (from.getBalance().compareTo(amount) < 0) {
        auditService.logFailedTransfer(from, amount); // логируем
        throw new InsufficientFundsException("Not enough funds"); // теперь вызовет rollback
    }
    from.debit(amount);
    to.credit(amount);
}

// Для любого исключения (самый безопасный вариант):
@Transactional(rollbackFor = Exception.class)
public void processPayment(Payment payment) throws PaymentException {
    paymentRepo.save(payment);
    gatewayClient.charge(payment); // throws PaymentException (checked)
    // rollback гарантирован при любом исключении
}

// Исключить конкретное checked исключение из rollback:
@Transactional(rollbackFor = Exception.class,
               noRollbackFor = AuditLoggingException.class)
public void placeOrder(Order order) throws Exception { ... }
```

❗ Ловушка: `rollbackFor` работает по иерархии классов. `rollbackFor = Exception.class` включает откат для всех наследников Exception, включая RuntimeException (которые и так откатываются). Но `rollbackFor = IOException.class` не включает `FileNotFoundException` отдельно — это уже наследник, он тоже будет откачен. Кандидаты иногда думают, что нужно перечислять все конкретные подклассы.

**→ Уточняющий вопрос:** Как настроить глобальные правила rollback для всего приложения без указания `rollbackFor` на каждом методе?

**↳ Ответ:** Создайте мета-аннотацию: `@Transactional(rollbackFor = Exception.class)` → `@AppTransactional` и используйте её везде вместо стандартной. Альтернатива: настроить AOP-advice через `TransactionAttributeSource` с кастомным `DefaultTransactionAttribute(rollbackFor = Exception.class)`. Правило: мета-аннотация — самый чистый способ, не требует XML и не нарушает явность конфигурации; если большинство методов в проекте должны откатываться на любом исключении — зафиксируйте это один раз в центральной аннотации.

---

## 21. Что такое readonly транзакция?

`@Transactional(readOnly = true)` — подсказка Spring и Hibernate, что транзакция не будет изменять данные. Это даёт несколько оптимизаций: Hibernate отключает **dirty checking** (не сканирует managed entities на изменения перед flush), не делает flush в конце транзакции, и может устанавливать hint `readOnly` на JDBC-соединение.

PostgreSQL при `readOnly = true` не создаёт transaction snapshot для изменений (что чуть эффективнее). При репликации БД некоторые библиотеки маршрутизации соединений (например, `AbstractRoutingDataSource`) используют `readOnly` флаг для направления запросов на реплику чтения. Это ключевое применение в high-load системах.

```java
@Service
@Transactional(readOnly = true) // все методы класса — только чтение
public class ReportService {

    public List<Order> getOrdersByPeriod(LocalDate from, LocalDate to) {
        return orderRepo.findByDateBetween(from, to);
        // Hibernate НЕ будет отслеживать изменения этих сущностей
        // Экономия CPU на dirty checking, особенно при загрузке 1000+ сущностей
    }

    public DashboardStats getDashboard() {
        return statsRepo.buildStats(); // только SELECT, никакого flush
    }

    @Transactional // переопределяем: readOnly = false для записи
    public Order createOrder(OrderRequest req) {
        return orderRepo.save(new Order(req));
    }
}
```

❗ Ловушка: `readOnly = true` **не запрещает** выполнять INSERT/UPDATE — это лишь подсказка. Некоторые СУБД (Oracle) могут бросить исключение при попытке записи, но PostgreSQL — нет (если не выставлен `SET TRANSACTION READ ONLY` явно). Также readOnly не улучшает производительность само по себе без подключения к replica datasource — без маршрутизации это просто экономия dirty checking.

**→ Уточняющий вопрос:** Как реализовать маршрутизацию запросов на read-реплику с помощью `AbstractRoutingDataSource` и флага `readOnly`?

**↳ Ответ:** Создаём две DataSource (primary и replica), оборачиваем в `AbstractRoutingDataSource`, переопределяем `determineCurrentLookupKey()` — возвращаем ключ `"replica"` если `TransactionSynchronizationManager.isCurrentTransactionReadOnly() == true`. Spring устанавливает флаг readOnly в ThreadLocal *до* получения соединения из пула — маршрутизация применяется автоматически. Правило: `@Transactional(readOnly=true)` → replica, `@Transactional` → primary, без единой строки бизнес-кода.

---

## 22. Что произойдёт при вызове @Transactional метода из другого метода того же класса?

Транзакция **не откроется**. Spring управляет транзакциями через AOP-прокси: когда внешний клиент вызывает метод бина, вызов идёт через прокси-объект, который добавляет транзакционное поведение. Но когда метод внутри класса вызывает другой метод того же класса через `this` — вызов идёт напрямую, минуя прокси. Spring просто не знает об этом вызове.

Это называется **self-invocation problem**. Нет универсального красивого решения — каждый вариант имеет компромиссы. Три основных подхода: (1) вынести метод в отдельный Spring-бин; (2) инжектировать сам бин в себя (self-injection); (3) использовать `TransactionTemplate` программно.

```java
// ПРОБЛЕМА: self-invocation — транзакция не откроется для internalProcess
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        orderRepo.save(order);
        internalProcess(order); // вызов через this — прокси обходится!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW) // НЕ РАБОТАЕТ
    public void internalProcess(Order order) { ... }
}

// РЕШЕНИЕ 1: вынести в отдельный бин
@Service
public class OrderProcessor {
    @Transactional(propagation = Propagation.REQUIRES_NEW) // теперь работает
    public void process(Order order) { ... }
}

// РЕШЕНИЕ 2: self-injection (Spring разрешает циклические зависимости для этого)
@Service
public class OrderService {
    @Autowired
    @Lazy // чтобы избежать circular dependency при создании
    private OrderService self;

    @Transactional
    public void placeOrder(Order order) {
        orderRepo.save(order);
        self.internalProcess(order); // вызов через прокси!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void internalProcess(Order order) { ... }
}

// РЕШЕНИЕ 3: программное управление транзакцией
@Service
public class OrderService {
    @Autowired TransactionTemplate transactionTemplate;

    @Transactional
    public void placeOrder(Order order) {
        orderRepo.save(order);
        transactionTemplate.execute(status -> { // явная новая транзакция
            processInNewTransaction(order);
            return null;
        });
    }
}
```

❗ Ловушка: self-injection (решение 2) работает в Spring Boot, но выглядит странно и смущает коллег на code review. Лучший архитектурный подход — решение 1: разделить ответственности на два бина. Кроме того, кандидаты иногда думают что `@Transactional` на `private`-методе решит проблему — нет, не решит, прокси всё равно не перехватит.

**→ Уточняющий вопрос:** Есть ли способ включить self-invocation для `@Transactional` без вынесения в отдельный бин — например через AspectJ compile-time weaving?

**↳ Ответ:** Да — AspectJ compile-time (CTW) или load-time weaving (LTW) инструментирует байткод самих классов, а не создаёт proxy-обёртки. При этом вызовы между методами одного класса тоже перехватываются — self-invocation проблема исчезает полностью. Для активации: зависимость `spring-aspects`, `-javaagent:aspectjweaver.jar`, аннотация `@EnableLoadTimeWeaving`. Правило: AspectJ weaving решает проблему, но усложняет build/deploy pipeline; для 95% случаев архитектурно правильнее и проще вынести метод в отдельный бин.
