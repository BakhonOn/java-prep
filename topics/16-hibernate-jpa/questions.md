# Hibernate / JPA

> 📇 Справочник уровня middle. Формат: ответ по сути → как работает внутри → пример → ❗ ловушка на собесе → уточняющий вопрос.

**Всего вопросов: 30**

---

## 1. Что такое проблема N+1 и как её решить?

Проблема N+1 возникает, когда для получения списка из N сущностей выполняется 1 запрос на сам список, а затем ещё N отдельных запросов — по одному для каждой ленивой связи. Например, загружаем 100 заказов, а при обходе цикла Hibernate выполняет ещё 100 SELECT для каждого `order.getUser()`.

Внутри Hibernate: при обращении к ленивому прокси-объекту срабатывает `LazyLoadingInterceptor`, который открывает новый запрос к БД. Это происходит незаметно — никакого явного кода, просто обход коллекции в цикле.

Основные способы решения:
- **JOIN FETCH** в JPQL: `SELECT o FROM Order o JOIN FETCH o.user` — один запрос с JOIN
- **@EntityGraph**: декларативный аналог JOIN FETCH, задаётся аннотацией или динамически
- **@BatchSize**: подгружает связи пачками (`IN (id1, id2, ...)`) вместо по одной
- **DTO-проекция**: вообще не грузить Entity, выбрать только нужные поля одним запросом

```java
// Репозиторий — решение через JOIN FETCH
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Проблема: N+1 при обращении к order.getUser() в цикле
    List<Order> findAll();

    // Решение 1: JOIN FETCH
    @Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
    List<Order> findByStatusWithUser(@Param("status") OrderStatus status);

    // Решение 2: EntityGraph
    @EntityGraph(attributePaths = {"user", "items", "items.product"})
    List<Order> findByStatus(OrderStatus status);
}

// Сервис, где N+1 возникает незаметно
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;

    // ПЛОХО: N+1 — для каждого order идёт SELECT на user
    public List<String> getUserEmailsForOrders() {
        return orderRepository.findAll().stream()
            .map(o -> o.getUser().getEmail()) // <-- здесь N запросов
            .collect(Collectors.toList());
    }

    // ХОРОШО: один запрос с JOIN FETCH
    public List<String> getUserEmailsForOrdersFix() {
        return orderRepository.findByStatusWithUser(OrderStatus.ACTIVE).stream()
            .map(o -> o.getUser().getEmail())
            .collect(Collectors.toList());
    }
}
```

❗ **Ловушка:** N+1 не всегда бросается в глаза в коде — она проявляется только в логах SQL. На собесе могут показать метод с `findAll()` + обходом коллекций и спросить «что здесь не так». Умейте читать `show_sql=true` и объяснять, почему 100 записей дали 101 запрос.

**→ Уточняющий вопрос:** Чем отличается решение через JOIN FETCH от @BatchSize, и когда @BatchSize предпочтительнее?

---

## 2. В чём разница между Lazy и Eager загрузкой?

**Lazy (ленивая)** загрузка означает, что связанные объекты не загружаются вместе с родителем — вместо них создаётся прокси-объект (Javassist/ByteBuddy заглушка). Реальный SQL-запрос выполняется только при первом обращении к данным прокси внутри открытой сессии.

**Eager (жадная)** загрузка означает немедленную загрузку связи вместе с родителем, обычно через JOIN или дополнительный SELECT сразу при `find`/`findAll`. Данные доступны без открытой сессии.

Умолчания JPA: `@OneToMany` и `@ManyToMany` — **LAZY** (связи-коллекции могут быть большими). `@ManyToOne` и `@OneToOne` — **EAGER** (одиночный объект, обычно нужен).

```java
@Entity
public class Order {
    @Id
    private Long id;

    // LAZY по умолчанию — коллекция не загружается сразу
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;

    // EAGER по умолчанию — user загружается вместе с Order
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "user_id")
    private User user;
}
```

❗ **Ловушка:** EAGER на `@ManyToOne` кажется безобидным, но если у `User` есть EAGER `@OneToMany` — будет неожиданный каскад загрузок. Также EAGER с `findAll()` даёт JOIN на каждую EAGER-связь, что может порождать декартово произведение и дублирование строк.

**→ Уточняющий вопрос:** Что произойдёт, если у `@OneToMany(fetch=EAGER)` будет 2 коллекции одновременно — получите ли вы ошибку или просто медленный запрос?

---

## 3. Когда использовать Lazy, а когда Eager loading?

Правило большого пальца для Middle: **держите всё LAZY по умолчанию** и явно подгружайте нужные связи через JOIN FETCH или @EntityGraph там, где это требуется. Это даёт контроль над запросами и избегает скрытых проблем с производительностью.

EAGER оправдан в редких случаях: связь типа `@ManyToOne` к небольшому справочному объекту (например, `@ManyToOne` к `Currency` или `Status`), который нужен практически в каждом use case. Даже тогда лучше явный JOIN FETCH по необходимости, чем EAGER глобально.

Признаки того, что EAGER — плохой выбор: сущность используется в разных контекстах (иногда нужна связь, иногда нет), связь — коллекция (потенциально большая), на связи стоит ещё EAGER — риск цепочки загрузок.

❗ **Ловушка:** `@ManyToOne` и `@OneToOne` — EAGER по умолчанию. Если у вас `Order` → `User` → `Role` (все EAGER), один `findById(Order)` тянет весь граф объектов. Переопределяйте на LAZY явно: `@ManyToOne(fetch = FetchType.LAZY)`.

**→ Уточняющий вопрос:** Если вы сделали все связи LAZY, как вы гарантируете нужную подгрузку в конкретном методе репозитория, не меняя глобальные настройки?

---

## 4. Что такое LazyInitializationException и как её избежать?

`LazyInitializationException` бросается Hibernate, когда происходит попытка инициализировать ленивый прокси-объект вне активной сессии/контекста персистентности. Сессия закрылась (транзакция завершилась), а код контроллера или сериализатор пытается обратиться к `order.getItems()`.

Внутри: ленивый прокси хранит ссылку на `SessionImplementor`. При обращении к данным он проверяет, открыта ли сессия. Если нет — `LazyInitializationException`.

Правильные способы избежать:
1. **JOIN FETCH / @EntityGraph** в репозитории — загрузить всё нужное в рамках транзакции
2. **DTO-проекция** — не возвращать Entity из слоя сервиса наружу вообще
3. **`@Transactional`** на методе сервиса — держит сессию открытой, пока метод не вернёт результат
4. **`Hibernate.initialize(entity.getItems())`** — явная принудительная инициализация внутри транзакции

Антипаттерн — **Open Session In View (OSIV)**: держит сессию открытой до конца HTTP-запроса (в Spring Boot включён по умолчанию: `spring.jpa.open-in-view=true`). Устраняет исключение, но создаёт N+1 в представлении и скрытые запросы к БД в слое контроллера/сериализации.

```java
// ПЛОХО: транзакция кончается в сервисе, контроллер получает detached entity
@Service
public class OrderService {
    @Transactional(readOnly = true)
    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow(); // items — lazy proxy
    }
}
// В контроллере: order.getItems() -> LazyInitializationException (если OSIV выключен)

// ХОРОШО: загружаем нужное в рамках транзакции и возвращаем DTO
@Service
public class OrderService {
    @Transactional(readOnly = true)
    public OrderDto getOrder(Long id) {
        Order order = orderRepository.findByIdWithItems(id).orElseThrow();
        return OrderDto.from(order); // маппим пока сессия открыта
    }
}

// Репозиторий
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

❗ **Ловушка:** Spring Boot по умолчанию включает OSIV (`open-in-view=true`), и `LazyInitializationException` не возникает. На продакшне с `open-in-view=false` всё ломается. Убедитесь, что понимаете настройку и умеете работать без OSIV.

**→ Уточняющий вопрос:** Почему OSIV считается антипаттерном и какие конкретные проблемы он создаёт на продакшне?

---

## 5. Какие стратегии fetch существуют в Hibernate?

JPA определяет два `FetchType`: **LAZY** и **EAGER** — они задают «когда грузить». Hibernate дополнительно предоставляет `FetchMode`, определяющий «как грузить»:

- **SELECT** (по умолчанию) — отдельный SELECT-запрос при инициализации прокси
- **JOIN** — загружает через LEFT OUTER JOIN вместе с родителем (игнорирует FetchType.LAZY при использовании HQL без FETCH)
- **SUBSELECT** — для коллекций: загружает все коллекции одним запросом с подзапросом `WHERE parent_id IN (SELECT ...)`

Помимо этого, `@BatchSize(size=N)` задаёт батчинг: вместо SELECT по одному ID собирает их в `WHERE id IN (id1, id2, ..., idN)`.

```java
@Entity
public class User {
    @Id
    private Long id;

    // FetchMode.SUBSELECT: когда грузятся несколько User, все их orders
    // загружаются одним подзапросом
    @OneToMany(mappedBy = "user")
    @Fetch(FetchMode.SUBSELECT)
    private List<Order> orders;

    // BatchSize: грузить роли пачками по 20
    @ManyToMany
    @BatchSize(size = 20)
    private Set<Role> roles;
}
```

❗ **Ловушка:** `FetchMode.JOIN` на `@OneToMany` не делает EAGER-загрузку через JOIN в JPQL-запросах — там нужен явный `JOIN FETCH`. `FetchMode.JOIN` работает только при `session.get()` / `entityManager.find()`.

**→ Уточняющий вопрос:** Когда SUBSELECT предпочтительнее JOIN FETCH при загрузке коллекций?

---

## 6. Что делает аннотация @BatchSize?

`@BatchSize(size = N)` — оптимизация Hibernate, которая объединяет N инициализаций ленивых прокси в один запрос с `WHERE id IN (id1, id2, ..., idN)`. Вместо N отдельных SELECT получаем `ceil(N/batch)` запросов.

Работает двумя способами: на уровне коллекции (`@OneToMany`) — при инициализации коллекций нескольких родителей; на уровне сущности (`@BatchSize` на классе) — при инициализации прокси для отдельных связанных объектов.

Хорошо сочетается с LAZY-загрузкой как «дешёвая» альтернатива JOIN FETCH там, где JOIN FETCH невозможен (несколько коллекций одновременно, пагинация).

```java
@Entity
@BatchSize(size = 25) // при загрузке прокси User — пачки по 25
public class User {
    @Id
    private Long id;
    private String email;

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    @BatchSize(size = 10) // при инициализации orders нескольких User — по 10 за раз
    private List<Order> orders;
}

// При обходе 50 пользователей и обращении к orders:
// Без @BatchSize: 50 отдельных SELECT
// С @BatchSize(10): 5 SELECT с IN-клаузой на 10 элементов
```

❗ **Ловушка:** `@BatchSize` не устраняет N+1 полностью — только уменьшает количество запросов. Если нужна гарантия одного запроса, используйте JOIN FETCH. Также BatchSize не работает с `JOIN FETCH` одновременно.

**→ Уточняющий вопрос:** Как настроить @BatchSize глобально для всех коллекций в Hibernate, не ставя аннотацию на каждую?

---

## 7. Опишите жизненный цикл Entity в Hibernate

Entity проходит через четыре состояния. **Transient** — объект создан через `new`, Hibernate про него ничего не знает, нет записи в БД, нет идентификатора (или есть, но не из БД). **Persistent** — объект под управлением `EntityManager`/`Session`, имеет идентификатор, изменения автоматически отслеживаются (dirty checking) и синхронизируются с БД при flush. **Detached** — сессия закрылась или объект явно отсоединён (`evict`/`detach`), идентификатор есть, но изменения больше не отслеживаются. **Removed** — объект помечен к удалению (`remove`), DELETE будет выполнен при flush, но объект ещё в памяти.

```java
@Service
@Transactional
public class UserService {
    public void lifecycle(Long id) {
        // Transient
        User user = new User("Alice", "alice@example.com");

        // Transient -> Persistent
        entityManager.persist(user); // теперь под управлением EM, id присвоен

        // Persistent: dirty checking — изменение будет сохранено без явного save
        user.setEmail("alice2@example.com"); // UPDATE при flush

        // flush + commit -> Persistent -> Detached
        // (после @Transactional метода)

        // Получить Persistent снова
        User managed = entityManager.find(User.class, id);

        // Persistent -> Removed
        entityManager.remove(managed); // DELETE при flush
    }
}
```

❗ **Ловушка:** Переход из Detached обратно в Persistent — через `merge`, а не `persist`. Вызов `persist` на detached-объекте с существующим id может вызвать `EntityExistsException` или непредсказуемое поведение в зависимости от провайдера.

**→ Уточняющий вопрос:** Что происходит с объектом в состоянии Removed, если вы снова вызовете persist() на нём до flush?

---

## 8. Что такое состояния: transient, persistent, detached, removed?

**Transient**: объект создан `new User()`, Hibernate не знает о его существовании. Нет строки в БД, нет связи с `PersistenceContext`. Если потеряете ссылку — объект просто будет собран GC.

**Persistent**: объект находится в `PersistenceContext` (первом уровне кэша). Hibernate снял «снимок» состояния при загрузке/persist. При flush сравнивает текущее состояние со снимком и генерирует SQL для изменений. Именно поэтому `save()` в Spring Data JPA не обязателен для обновления — достаточно изменить поле.

**Detached**: объект существует в памяти, имеет id, но сессия закрылась. Изменения не отслеживаются. Типичный сценарий: Entity вернули из `@Transactional` метода — она сразу становится Detached. Вернуть в Persistent — через `merge()`.

**Removed**: `entityManager.remove(entity)` переводит Persistent → Removed. DELETE SQL откладывается до flush. Объект ещё доступен в коде, но после flush/commit становится Transient (id обнуляется в Hibernate, хотя объект Java жив).

❗ **Ловушка:** В Spring с `@Transactional` сущности, возвращённые из метода репозитория, остаются Persistent до конца транзакции — изменения автоматически сохраняются. Это может быть нежелательным побочным эффектом: изменили поле «для вывода», а Hibernate сохранил в БД.

**→ Уточняющий вопрос:** Как предотвратить случайное сохранение изменений в Persistent-объекте внутри транзакции, если вы меняете поля только для локального использования?

---

## 9. Что такое кэш первого уровня в Hibernate?

Кэш первого уровня (First Level Cache, L1) — это `PersistenceContext`, встроенный в каждую `Session`/`EntityManager`. Он включён всегда и не отключается. Гарантирует **identity guarantee**: в рамках одной сессии одна сущность по одному id — всегда один и тот же Java-объект.

Работает как Map: ключ — `(EntityClass, id)`, значение — managed-объект + «снимок» (для dirty checking). При повторном `find(User.class, 1L)` в той же сессии SQL не выполняется — объект берётся из L1.

Важно понимать ограничения: L1 живёт только в рамках одной сессии (одной транзакции в обычном Spring-приложении). Он не общий между потоками, не переживает закрытие сессии. При обработке большого batch-запроса L1 растёт без ограничений — может вызвать `OutOfMemoryError`.

```java
@Transactional
public void demonstrateL1Cache(Long userId) {
    User u1 = userRepository.findById(userId).get(); // SELECT выполнен
    User u2 = userRepository.findById(userId).get(); // SELECT НЕ выполнен, из L1

    System.out.println(u1 == u2); // true — один и тот же объект!

    // Проблема при batch-обработке: L1 накапливает объекты
    for (int i = 0; i < 100_000; i++) {
        processAndSave(i);
        if (i % 50 == 0) {
            entityManager.flush(); // синхронизировать
            entityManager.clear(); // очистить L1, иначе OOM
        }
    }
}
```

❗ **Ловушка:** При пакетной обработке данных в одной транзакции L1 накапливает все загруженные/созданные объекты. Без периодического `entityManager.clear()` или разбивки на меньшие транзакции — `OutOfMemoryError`.

**→ Уточняющий вопрос:** Как кэш первого уровня влияет на результат JPQL-запроса, если вы уже изменили объект в текущей сессии, но не сделали flush?

---

## 10. Что такое кэш второго уровня и когда его использовать?

Кэш второго уровня (Second Level Cache, L2) — это кэш уровня `SessionFactory`, общий для всех сессий и потоков в рамках одного приложения. В отличие от L1, он опционален и требует явной настройки. Реализации: Ehcache, Caffeine, Hazelcast, Infinispan.

L2 хранит не Java-объекты, а «гидратированное состояние» (массив значений полей), что позволяет избежать проблем с разделяемым состоянием между потоками. При запросе сущности Hibernate сначала проверяет L1, затем L2, и только потом идёт в БД.

Когда использовать: данные часто читаются и редко изменяются (справочники: категории, валюты, страны), данные не критичны к абсолютной актуальности (допустимо небольшое устаревание), нагрузка на БД высокая и нужно её снизить. Не использовать: для часто изменяемых данных (заказы, транзакции), для данных с жёсткими требованиями к согласованности.

```java
// Сущность для L2-кэша
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class ProductCategory {
    @Id
    private Long id;
    private String name;
}

// application.properties
// spring.jpa.properties.hibernate.cache.use_second_level_cache=true
// spring.jpa.properties.hibernate.cache.region.factory_class=
//   org.hibernate.cache.jcache.JCacheRegionFactory
```

❗ **Ловушка:** L2-кэш и кластеризация — отдельная история. При нескольких инстансах приложения каждый имеет свой L2, что приводит к рассинхронизации. Нужен распределённый кэш (Hazelcast, Redis через Redisson) или `READ_ONLY` стратегия для неизменяемых данных.

**→ Уточняющий вопрос:** Какие стратегии конкурентного доступа (CacheConcurrencyStrategy) существуют и чем READ_WRITE отличается от NONSTRICT_READ_WRITE?

---

## 11. Как настроить кэш второго уровня?

Настройка состоит из трёх шагов: подключить зависимость провайдера кэша, включить L2 в конфигурации Hibernate, пометить нужные сущности и коллекции аннотациями.

Провайдер кэша (Ehcache/JCache):
```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <classifier>jakarta</classifier>
</dependency>
```

Конфигурация `application.properties`:
```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
spring.jpa.properties.hibernate.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
```

Аннотации на сущностях:
```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;

    @OneToMany(mappedBy = "product")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // кэш коллекции тоже нужен!
    private List<ProductImage> images;
}

// Кэш запросов
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    @QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
    List<Product> findByCategory(String category);
}
```

❗ **Ловушка:** Если пометить сущность `@Cacheable`, но не пометить коллекцию `@Cache` — коллекция не кэшируется. Два отдельных решения для сущности и её коллекций. Также кэш запросов кэширует только идентификаторы, а не объекты — нужен и L2 для сущностей.

**→ Уточняющий вопрос:** Что происходит с L2-кэшем при вызове `entityManager.clear()` или закрытии сессии?

---

## 12. Что такое dirty checking в Hibernate?

Dirty checking — механизм автоматического обнаружения изменений в Persistent-объектах и генерации соответствующих UPDATE-запросов при flush. Разработчику не нужно явно «сохранять» изменения — достаточно изменить поле.

Как работает внутри: при переводе объекта в Persistent-состояние (persist, merge, find) Hibernate сохраняет «снимок» (snapshot) — копию значений всех полей. При flush обходит все Persistent-объекты в PersistenceContext, сравнивает текущее состояние со снимком поле за полем, и если есть разница — генерирует UPDATE.

По умолчанию Hibernate использует рефлексию для сравнения, что может быть медленным при большом количестве объектов. Альтернатива — `@DynamicUpdate` (генерировать UPDATE только для изменённых колонок) и байткод-инструментирование (сравнение без рефлексии).

```java
@Entity
@DynamicUpdate // UPDATE только изменённых колонок — полезно для широких таблиц
public class User {
    @Id
    private Long id;
    private String email;
    private String name;
    // 20+ других полей...
}

@Service
@Transactional
public class UserService {
    public void updateEmail(Long userId, String newEmail) {
        User user = userRepository.findById(userId).orElseThrow();
        user.setEmail(newEmail); // <-- НЕТ явного save/update!
        // При коммите Hibernate автоматически выполнит:
        // UPDATE users SET email = ? WHERE id = ?
        // Без @DynamicUpdate: UPDATE users SET email=?, name=?, ... (все поля)
        // С @DynamicUpdate: UPDATE users SET email=? WHERE id=?
    }
}
```

❗ **Ловушка:** Dirty checking выполняется для ВСЕХ Persistent-объектов при каждом flush, даже если вы их не меняли. При большом PersistenceContext (тысячи объектов) flush становится дорогим. Используйте `@Transactional(readOnly = true)` для read-only методов — Hibernate пропускает dirty checking.

**→ Уточняющий вопрос:** Как `@Transactional(readOnly = true)` влияет на dirty checking и производительность?

---

## 13. Как работает механизм flush в Hibernate?

Flush — это синхронизация состояния PersistenceContext с базой данных: генерация и выполнение SQL-запросов (INSERT, UPDATE, DELETE). Flush **не является коммитом** — данные изменяются в рамках текущей транзакции, но не фиксируются до `commit`.

Режимы flush (`FlushModeType`):
- **AUTO** (по умолчанию): flush перед выполнением JPQL/SQL-запроса (если запрос может задеть изменённые данные) и перед коммитом транзакции
- **COMMIT**: flush только перед коммитом
- **ALWAYS**: flush перед каждым запросом
- **MANUAL**: flush только при явном `entityManager.flush()`

Порядок операций при flush: INSERT (в порядке зависимостей), UPDATE, удаление из junction-таблиц ManyToMany, INSERT в junction-таблицы, DELETE.

```java
@Service
@Transactional
public class OrderService {
    public void processOrder(OrderDto dto) {
        Order order = new Order(dto);
        entityManager.persist(order); // INSERT отложен

        // Flush auto: перед этим JPQL-запросом Hibernate увидит,
        // что orders затронуты -> сделает flush -> INSERT выполнится
        // -> запрос вернёт корректный результат
        long count = entityManager.createQuery(
            "SELECT COUNT(o) FROM Order o WHERE o.userId = :uid", Long.class)
            .setParameter("uid", dto.getUserId())
            .getSingleResult();

        // Явный flush: принудительно синхронизировать сейчас
        // entityManager.flush();

        // Изменение режима для batch-операций
        entityManager.setFlushMode(FlushModeType.COMMIT);
    }
}
```

❗ **Ловушка:** Частая ошибка — думать, что `flush()` == `commit()`. После `flush()` другие транзакции всё ещё не видят изменений (зависит от уровня изоляции), а откат транзакции отменит всё, включая уже выполненные flush-запросы.

**→ Уточняющий вопрос:** Почему FlushModeType.COMMIT может быть опасен при использовании JPQL-запросов внутри той же транзакции?

---

## 14. В чём разница между persist() и merge()?

`persist(entity)` — переводит **Transient** объект в Persistent. Объект не должен иметь id (или id должен быть сгенерирован). Hibernate регистрирует его в PersistenceContext, при flush выполнится INSERT. Важно: `persist` работает на переданном объекте — тот же объект становится managed.

`merge(entity)` — работает с **Detached** или Transient объектами. Создаёт или находит Persistent-копию, копирует в неё состояние переданного объекта, и **возвращает эту копию**. Исходный объект остаётся Detached. Если объект с таким id существует в PersistenceContext — обновляет его; если есть в БД — загружает и обновляет; если нет нигде — создаёт новый.

```java
@Service
@Transactional
public class ProductService {

    public void persistExample() {
        Product product = new Product("Laptop", BigDecimal.valueOf(999));
        entityManager.persist(product);
        // product теперь managed, id присвоен
        product.setName("Gaming Laptop"); // dirty checking сохранит изменение
    }

    public Product mergeExample(Product detachedProduct) {
        // detachedProduct — пришёл из другой транзакции или десериализован
        Product managedProduct = entityManager.merge(detachedProduct);

        // КРИТИЧНО: managedProduct — это НОВЫЙ объект!
        // detachedProduct по-прежнему detached!
        managedProduct.setPrice(BigDecimal.valueOf(1099)); // сохранится
        detachedProduct.setPrice(BigDecimal.valueOf(0));   // НЕ сохранится!

        return managedProduct; // возвращаем managed-копию
    }

    // Spring Data JPA: save() использует persist для новых (id == null)
    // и merge для существующих (id != null)
    public void saveExample(Product product) {
        productRepository.save(product); // если id == null -> persist, иначе -> merge
    }
}
```

❗ **Ловушка:** `merge()` возвращает новый managed-объект — исходный аргумент остаётся Detached. Классическая ошибка: вызвали `merge(dto)`, продолжили работать с `dto`, а сохранения не произошло. Всегда работайте с возвращённым значением `merge`.

**→ Уточняющий вопрос:** Что происходит при вызове merge() для объекта с id, которого нет в базе данных?

---

## 15. Что делает метод refresh()?

`entityManager.refresh(entity)` — перезагружает состояние Persistent-объекта из базы данных, **перезаписывая все несохранённые изменения** в объекте. Используется для синхронизации с актуальными данными, когда другая транзакция могла изменить запись.

Практические сценарии применения: после нативного SQL-запроса (Hibernate не знает о его изменениях), после внешнего изменения данных в той же транзакции, при работе с триггерами БД (которые модифицируют строку после INSERT/UPDATE).

```java
@Service
@Transactional
public class InventoryService {

    public void refreshExample(Long productId) {
        Product product = entityManager.find(Product.class, productId);
        product.setStock(100); // изменение в памяти

        // Нативный запрос изменил данные в БД напрямую
        entityManager.createNativeQuery(
            "UPDATE products SET stock = stock - 10 WHERE id = ?")
            .setParameter(1, productId)
            .executeUpdate();

        // Hibernate не знает об изменении нативного запроса
        // product.getStock() всё ещё == 100 в памяти

        entityManager.refresh(product);
        // Теперь product.getStock() == актуальное значение из БД
        // Несохранённое изменение (100) потеряно!
    }
}
```

❗ **Ловушка:** `refresh()` безвозвратно потеряет все несохранённые изменения в объекте. Если вы делали `product.setName(...)` и потом `refresh()` — изменение имени пропадёт. Убедитесь, что это именно то, что вам нужно.

**→ Уточняющий вопрос:** Как refresh() взаимодействует с каскадными операциями и нужно ли явно добавлять CascadeType.REFRESH для дочерних сущностей?

---

## 16. Что такое EntityManager и чем он отличается от Session?

`EntityManager` — стандартный интерфейс JPA (Jakarta Persistence API), определённый спецификацией. `Session` — нативный API Hibernate, расширяющий `EntityManager`. Под капотом Spring Data JPA использует Hibernate Session как реализацию EntityManager.

Основные отличия:
- **Переносимость**: `EntityManager` работает с любым JPA-провайдером (EclipseLink, OpenJPA); `Session` — только Hibernate
- **API**: `Session` богаче — добавляет `saveOrUpdate()`, `createCriteria()` (устарело), `createFilter()`, нативные методы работы с кэшем (`evict`, `contains`)
- **Получение**: `EntityManager` инжектируется через `@PersistenceContext`; `Session` получается через `entityManager.unwrap(Session.class)`

```java
@Repository
public class UserRepository {

    @PersistenceContext
    private EntityManager entityManager; // JPA стандарт

    // Стандартные JPA операции
    public User findById(Long id) {
        return entityManager.find(User.class, id);
    }

    // Доступ к Hibernate Session для специфических операций
    public void evictFromCache(User user) {
        Session session = entityManager.unwrap(Session.class);
        session.evict(user); // убрать из L1 кэша

        // Или использовать StatelessSession для batch-операций без L1
        StatelessSession statelessSession = session.getSessionFactory()
            .openStatelessSession();
    }

    // JPQL (JPA стандарт)
    public List<User> findByEmail(String email) {
        return entityManager.createQuery(
            "SELECT u FROM User u WHERE u.email = :email", User.class)
            .setParameter("email", email)
            .getResultList();
    }
}
```

❗ **Ловушка:** В Spring Boot почти всегда используйте `EntityManager` — это правильная абстракция. Если нужны нативные Hibernate-возможности, получайте `Session` через `unwrap`. Не создавайте Session вручную через `SessionFactory.openSession()` в Spring-контексте — не будет участвовать в Spring-транзакции.

**→ Уточняющий вопрос:** Что такое StatelessSession в Hibernate и когда его стоит использовать вместо обычной Session?

---

## 17. Как реализовать оптимистичную блокировку в JPA?

Оптимистичная блокировка основана на предположении, что конфликты редки: не блокируем строку при чтении, но при записи проверяем, не изменил ли кто-то данные пока мы их обрабатывали. Реализуется через версионное поле `@Version`.

При UPDATE Hibernate добавляет в WHERE условие по версии: `UPDATE users SET email=?, version=? WHERE id=? AND version=?`. Если строка не обновилась (0 rows affected) — значит другая транзакция изменила данные, Hibernate бросает `OptimisticLockException`.

Типы версионных полей: `int`/`Integer`/`long`/`Long` (инкремент на 1), `Timestamp` (текущее время — менее надёжно из-за точности).

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private int stock;

    @Version
    private Long version; // Hibernate управляет автоматически
}

@Service
@Transactional
public class ProductService {

    public void decreaseStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId).orElseThrow();
        if (product.getStock() < quantity) {
            throw new InsufficientStockException();
        }
        product.setStock(product.getStock() - quantity);
        // При flush: UPDATE products SET stock=?, version=2 WHERE id=? AND version=1
        // Если другая транзакция успела изменить: OptimisticLockException
    }
}

// Обработка конфликта в сервисном слое
@Service
public class OrderService {
    @Retryable(value = OptimisticLockingFailureException.class, maxAttempts = 3)
    @Transactional
    public void placeOrder(Long productId, int qty) {
        productService.decreaseStock(productId, qty);
    }
}
```

❗ **Ловушка:** `OptimisticLockException` (JPA) оборачивается Spring в `ObjectOptimisticLockingFailureException`. Транзакция при этом помечается как rollback-only — повторить нужно в НОВОЙ транзакции. Не ловите исключение внутри той же `@Transactional` и не пытайтесь продолжить.

**→ Уточняющий вопрос:** Как реализовать retry при OptimisticLockException в Spring и почему retry должен происходить в новой транзакции?

---

## 18. Как реализовать пессимистичную блокировку в JPA?

Пессимистичная блокировка физически блокирует строку в БД при чтении, не давая другим транзакциям её изменить до снятия блокировки. Используется при высокой вероятности конфликтов или когда нельзя терять обновление.

Типы блокировок:
- `PESSIMISTIC_READ` — `SELECT ... FOR SHARE` (другие могут читать, но не писать)
- `PESSIMISTIC_WRITE` — `SELECT ... FOR UPDATE` (эксклюзивная блокировка)
- `PESSIMISTIC_FORCE_INCREMENT` — `FOR UPDATE` + инкремент версии

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Через аннотацию в Spring Data JPA
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}

@Service
@Transactional
public class InventoryService {

    public void reserveStock(Long productId, int quantity) {
        // Блокируем строку на время транзакции
        Product product = productRepository.findByIdForUpdate(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        // Другие транзакции ждут, пока эта не завершится
        if (product.getStock() < quantity) {
            throw new InsufficientStockException();
        }
        product.setStock(product.getStock() - quantity);
        // При коммите блокировка снимается
    }

    // Или через EntityManager с таймаутом
    public Product lockWithTimeout(Long productId) {
        Map<String, Object> hints = new HashMap<>();
        hints.put("jakarta.persistence.lock.timeout", 5000); // 5 секунд
        return entityManager.find(Product.class, productId,
            LockModeType.PESSIMISTIC_WRITE, hints);
    }
}
```

❗ **Ловушка:** Пессимистичная блокировка — потенциальный дедлок. Если две транзакции блокируют строки в разном порядке, они зависнут друг в друге. БД обнаружит дедлок и убьёт одну из транзакций. Всегда блокируйте в одном и том же порядке и устанавливайте таймаут.

**→ Уточняющий вопрос:** В чём принципиальное различие сценариев применения оптимистичной и пессимистичной блокировок, и как выбрать подходящую?

---

## 19. Что такое @Version и зачем она нужна?

`@Version` — аннотация JPA, которая помечает поле как версионное для реализации оптимистичной блокировки. Hibernate автоматически управляет этим полем: инкрементирует при каждом UPDATE и включает в WHERE при обновлении для проверки.

Механизм работы: при загрузке сущности Hibernate запоминает версию. При flush UPDATE включает `AND version = :загруженная_версия`. Если БД вернула 0 обновлённых строк — кто-то уже изменил запись, Hibernate бросает `OptimisticLockException`.

Дополнительный эффект: `@Version` также защищает от проблемы «потерянного обновления» (lost update) в многопоточных сценариях без реальной блокировки БД.

```java
@Entity
public class BankAccount {
    @Id
    private Long id;

    private BigDecimal balance;

    @Version
    private int version; // начинается с 0, Hibernate инкрементит автоматически

    // Никогда не устанавливайте version вручную — только Hibernate!
    // Не включайте version в equals/hashCode — это техническое поле
}

// Сценарий конфликта:
// Транзакция 1: читает BankAccount{balance=1000, version=5}
// Транзакция 2: читает BankAccount{balance=1000, version=5}
// Транзакция 2: обновляет -> UPDATE ... SET balance=900, version=6 WHERE version=5 -> OK
// Транзакция 1: пытается обновить -> UPDATE ... WHERE version=5 -> 0 rows -> OptimisticLockException
```

❗ **Ловушка:** Если вы обновляете сущность через нативный SQL или JPQL `UPDATE`, `@Version` не сработает автоматически — вы должны вручную инкрементировать версию в запросе. Нативные операции обходят механизм Hibernate.

**→ Уточняющий вопрос:** Что произойдёт, если использовать @Version с Timestamp вместо числового типа — какие проблемы это создаёт?

---

## 20. Как работают каскадные операции (Cascade)?

Каскадирование — распространение операций JPA (persist, merge, remove и др.) от родительской сущности на связанные дочерние. Без каскада каждую сущность нужно явно передавать в EntityManager.

Внутри: при выполнении операции над родителем Hibernate проверяет, есть ли в маппинге `cascade`, и если да — рекурсивно применяет ту же операцию к коллекции/связанным объектам.

Правило проектирования: каскад имеет смысл только там, где жизненный цикл дочернего объекта полностью определяется родительским (агрегат). Например, `Order` → `OrderItem`: без заказа позиция не существует. Но `Order` → `User`: пользователь живёт независимо — каскад REMOVE здесь катастрофа.

```java
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private User user; // НЕТ каскада — пользователь независим

    @OneToMany(mappedBy = "order",
               cascade = CascadeType.ALL,  // все операции каскадируются
               orphanRemoval = true)        // удалить элемент из коллекции = удалить из БД
    private List<OrderItem> items = new ArrayList<>();

    // Хелпер-метод для корректного управления двунаправленной связью
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null); // orphanRemoval удалит из БД
    }
}

@Service
@Transactional
public class OrderService {
    public Order createOrder(User user, List<ProductDto> products) {
        Order order = new Order(user);
        products.forEach(p -> order.addItem(new OrderItem(p)));

        // persist(order) автоматически сделает persist для каждого OrderItem
        return orderRepository.save(order);
    }
}
```

❗ **Ловушка:** `CascadeType.ALL` включает `REMOVE`. Если вы переиспользуете дочерние сущности между несколькими родителями (например, `Product` shared между `OrderItem`), CASCADE REMOVE удалит `Product` при удалении `OrderItem`. Всегда думайте, является ли связь агрегатом или ссылкой.

**→ Уточняющий вопрос:** Чем CascadeType.REMOVE отличается от orphanRemoval = true и когда нужны оба?

---

## 21. Какие типы Cascade существуют?

JPA определяет 6 типов через `javax.persistence.CascadeType`:

- **PERSIST** — `persist()` родителя каскадируется на дочерние (автосохранение новых детей)
- **MERGE** — `merge()` родителя каскадируется (обновление detached-дочерних)
- **REMOVE** — `remove()` родителя удаляет дочерние (осторожно!)
- **REFRESH** — `refresh()` родителя перезагружает дочерние из БД
- **DETACH** — `detach()` родителя отсоединяет дочерние от контекста
- **ALL** — все вышеперечисленные

Hibernate добавляет собственные типы через `org.hibernate.annotations.CascadeType`:
- **SAVE_UPDATE** — для `saveOrUpdate()` (нативный Hibernate)
- **REPLICATE**, **LOCK** — специфические Hibernate-операции

```java
@Entity
public class BlogPost {

    // Комментарии — часть поста, ALL оправдан
    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments;

    // Теги — независимые объекты, только PERSIST и MERGE
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    private Set<Tag> tags;

    // Автор — полностью независим, каскада нет
    @ManyToOne
    private User author;
}
```

❗ **Ловушка:** `CascadeType.ALL` на `@ManyToOne` или `@ManyToMany` почти всегда ошибка. Представьте: удалили один `Order` → CASCADE REMOVE на `User` → удалился пользователь со всеми заказами. Используйте `ALL` только на `@OneToMany` в явных агрегатах.

**→ Уточняющий вопрос:** Что произойдёт, если использовать CascadeType.PERSIST на @ManyToMany с уже существующими тегами?

---

## 22. Что такое orphan removal?

`orphanRemoval = true` — флаг на `@OneToMany`/`@OneToOne`, который говорит Hibernate: если дочерний объект убран из коллекции (потерял родителя — стал «сиротой»), его нужно удалить из БД. Это поведение отличается от `CascadeType.REMOVE`.

Разница:
- `CascadeType.REMOVE` — удаляет детей при удалении **родителя**
- `orphanRemoval = true` — удаляет ребёнка при **удалении из коллекции** родителя (даже если сам родитель не удаляется)

На практике: `orphanRemoval = true` имплицитно включает `CascadeType.REMOVE`. Поэтому часто ставят `cascade = CascadeType.ALL, orphanRemoval = true` вместе.

```java
@Entity
public class Order {
    @Id
    private Long id;

    @OneToMany(mappedBy = "order",
               cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}

@Service
@Transactional
public class OrderService {

    public void removeItemFromOrder(Long orderId, Long itemId) {
        Order order = orderRepository.findById(orderId).orElseThrow();

        // orphanRemoval: убираем из коллекции -> Hibernate выполнит DELETE
        order.getItems().removeIf(item -> item.getId().equals(itemId));
        // Явного delete не нужно!
    }

    public void deleteOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        orderRepository.delete(order);
        // CascadeType.REMOVE: все OrderItem удалятся автоматически
    }
}
```

❗ **Ловушка:** `orphanRemoval` работает только через управляемый объект: нужно загрузить Order, убрать элемент из коллекции. Если сделать `DELETE FROM order_items WHERE id = ?` нативным SQL — orphanRemoval не при чём, но и Persistent-объект будет рассинхронизирован. Используйте `refresh` или работайте через JPA.

**→ Уточняющий вопрос:** Что произойдёт при orphanRemoval = true, если одна и та же дочерняя сущность ссылается на двух разных родителей?

---

## 23. Как правильно использовать @OneToMany и @ManyToOne?

Правильная реализация: двунаправленная связь с `@ManyToOne` как **владельцем** (там, где FK в БД) и `mappedBy` на стороне `@OneToMany`. Управление связью — через хелпер-методы для поддержания согласованности обоих сторон.

Ключевые правила:
1. `@JoinColumn` ставится на стороне `@ManyToOne` (владелец FK)
2. `@OneToMany` получает `mappedBy` — она зеркало, не владелец
3. `fetch = LAZY` на `@ManyToOne` явно (умолчание EAGER плохо)
4. Хелпер-методы `addChild`/`removeChild` синхронизируют обе стороны
5. `@OneToMany` с `List` и `@OrderColumn` или без порядка — зависит от нужд

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String email;

    // mappedBy = имя поля на стороне Order, указывающего на User
    @OneToMany(mappedBy = "user",
               cascade = CascadeType.ALL,
               orphanRemoval = true,
               fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();

    // Хелпер для согласованности двунаправленной связи
    public void addOrder(Order order) {
        orders.add(order);
        order.setUser(this);
    }

    public void removeOrder(Order order) {
        orders.remove(order);
        order.setUser(null);
    }
}

@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY) // явно LAZY!
    @JoinColumn(name = "user_id", nullable = false) // FK колонка
    private User user;

    // equals/hashCode через бизнес-ключ или id (не через user!)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order order = (Order) o;
        return id != null && id.equals(order.id);
    }
}
```

❗ **Ловушка:** Двунаправленная связь без хелпер-методов — источник багов. Если сделать `order.setUser(user)` без `user.getOrders().add(order)`, в текущей сессии коллекция `user.orders` окажется неактуальной. После flush/reload всё будет верно, но до этого — несогласованное состояние в памяти.

**→ Уточняющий вопрос:** Почему использование Set вместо List для @OneToMany коллекций может быть предпочтительным, и как это связано с equals/hashCode?

---

## 24. В чём особенности bidirectional relationships?

Двунаправленная связь (bidirectional) — это когда обе сущности имеют ссылки друг на друга: `User.orders` (OneToMany) и `Order.user` (ManyToOne). В отличие от однонаправленной, здесь есть понятие **владельца связи** — та сторона, где физически находится FK в БД, и которая управляет отношением.

**Владелец связи** — сторона без `mappedBy`. Именно изменения на стороне владельца Hibernate синхронизирует с БД. Изменения только на стороне с `mappedBy` игнорируются при flush.

**Синхронизация обеих сторон** — ответственность разработчика. Если установить только `order.setUser(user)`, а не добавить `user.getOrders().add(order)` — в рамках текущей сессии `user.getOrders()` не будет содержать этот заказ. После reload из БД всё OK, но текущая сессия содержит несогласованный граф.

```java
// Проблема без хелперов:
@Transactional
public void badExample() {
    User user = userRepository.findById(1L).get();
    Order order = new Order();
    order.setUser(user); // устанавливаем ManyToOne сторону (владелец)
    orderRepository.save(order); // это сохранится в БД

    // Но user.getOrders() в текущей сессии НЕ содержит order!
    System.out.println(user.getOrders().size()); // может быть 0 в кэше!
}

// Правильно с хелпером:
@Transactional
public void goodExample() {
    User user = userRepository.findById(1L).get();
    Order order = new Order();
    user.addOrder(order); // хелпер устанавливает обе стороны
    orderRepository.save(order);

    System.out.println(user.getOrders().size()); // корректно
}

// equals/hashCode критичны для Set-коллекций и корректной работы
// remove() из коллекции:
@Entity
public class Order {
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order that = (Order) o;
        // Используем бизнес-ключ или только id (с проверкой на null для transient)
        return id != null && Objects.equals(id, that.id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode(); // стабильный для transient и persistent
    }
}
```

❗ **Ловушка:** Проблема `equals`/`hashCode` у Hibernate-сущностей нетривиальна. При использовании `id` как основы: у transient-объекта id == null, все они «равны» по hashCode, что ломает HashSet/HashMap. Рекомендация Hibernate: `hashCode()` возвращает `getClass().hashCode()` (константный), а `equals` проверяет id с null-guard.

**→ Уточняющий вопрос:** Почему использование @EqualsAndHashCode от Lombok на JPA-сущностях часто приводит к ошибкам, и что с этим делать?

---

## 25. Как избежать бесконечной рекурсии при сериализации Entity?

Бесконечная рекурсия возникает при сериализации (Jackson → JSON) двунаправленной связи: `User` сериализует `orders`, каждый `Order` сериализует `user`, который сериализует `orders`... → `StackOverflowError`.

Решения в порядке предпочтения:

**1. DTO (лучшая практика)** — никогда не сериализовать Entity напрямую. Маппить в DTO (MapStruct, вручную) без обратных ссылок.

**2. `@JsonManagedReference` / `@JsonBackReference`** — Jackson-аннотации: managed-сторона сериализуется, back-сторона — нет (только десериализация).

**3. `@JsonIgnore`** — пометить одну сторону связи как игнорируемую при сериализации.

**4. `@JsonIdentityInfo`** — сериализует объект полностью один раз, далее ссылка по id. Менее удобно для REST API.

```java
// Решение 1 (рекомендуемое): DTO
@Service
public class UserService {
    public UserDto getUser(Long id) {
        User user = userRepository.findByIdWithOrders(id).orElseThrow();
        return new UserDto(
            user.getId(),
            user.getEmail(),
            user.getOrders().stream()
                .map(o -> new OrderDto(o.getId(), o.getStatus()))
                .collect(Collectors.toList())
        );
    }
}

// Решение 2: Jackson аннотации (если DTO не вариант)
@Entity
public class User {
    @JsonManagedReference // сериализуется, управляет циклом
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}

@Entity
public class Order {
    @JsonBackReference // НЕ сериализуется в JSON, только десериализация
    @ManyToOne
    private User user;
}

// Решение 3: @JsonIgnore
@Entity
public class Order {
    @JsonIgnore
    @ManyToOne
    private User user;
}
```

❗ **Ловушка:** `@JsonBackReference` работает только в одну сторону — при десериализации JSON с объектом Order обратная ссылка на User не заполняется автоматически. DTO устраняет проблему полностью и является правильным подходом для production API.

**→ Уточняющий вопрос:** Какие проблемы создаёт прямое возвращение JPA-сущностей из REST-контроллеров помимо рекурсии?

---

## 26. Что такое JPQL и чем он отличается от SQL?

JPQL (Jakarta Persistence Query Language) — объектно-ориентированный язык запросов JPA, оперирующий **сущностями и их полями** вместо таблиц и колонок. Hibernate транслирует JPQL в SQL для конкретной БД, обеспечивая переносимость.

Ключевые отличия:
- JPQL работает с именами классов и полей Java, SQL — с именами таблиц и колонок
- JPQL переносим между БД (PostgreSQL, MySQL, H2); SQL — диалект конкретной БД
- JPQL возвращает управляемые Entity или проекции; SQL возвращает сырые данные
- JPQL поддерживает полиморфные запросы по иерархии наследования
- Нет DDL, только SELECT/UPDATE/DELETE

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // JPQL: имена классов и полей Java
    @Query("SELECT o FROM Order o " +
           "JOIN FETCH o.user u " +
           "JOIN FETCH o.items i " +
           "WHERE u.email = :email AND o.status = :status")
    List<Order> findByUserEmailAndStatus(
        @Param("email") String email,
        @Param("status") OrderStatus status);

    // Агрегатные функции
    @Query("SELECT new com.example.dto.UserOrderStats(u.email, COUNT(o), SUM(o.total)) " +
           "FROM Order o JOIN o.user u " +
           "GROUP BY u.email " +
           "HAVING COUNT(o) > :minOrders")
    List<UserOrderStats> findActiveUserStats(@Param("minOrders") long minOrders);

    // Нативный SQL — когда JPQL недостаточен (специфичные функции БД)
    @Query(value = "SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days'",
           nativeQuery = true)
    List<Order> findRecentOrdersNative();
}
```

❗ **Ловушка:** JPQL чувствителен к именам: `FROM Order` (класс), не `FROM orders` (таблица); `o.userId` (поле Java), не `o.user_id` (колонка). Ошибка в имени даёт `QuerySyntaxException` в рантайме, не в compile-time. Используйте Criteria API или Querydsl для compile-time безопасности.

**→ Уточняющий вопрос:** Когда стоит использовать нативный SQL вместо JPQL и какие ограничения у нативных запросов в JPA?

---

## 27. Что такое Criteria API и когда его использовать?

Criteria API — типобезопасный программный способ построения JPA-запросов через Java-объекты. В отличие от JPQL (строка, ошибки в рантайме), Criteria API проверяется компилятором. Особенно полезен для **динамических запросов**, когда набор условий определяется в рантайме.

Структура: `CriteriaBuilder` (фабрика), `CriteriaQuery<T>` (сам запрос), `Root<T>` (FROM), `Predicate` (условия WHERE), `Join` (JOIN).

Минусы: многословен, сложно читается. Для сложных динамических запросов часто используют **Querydsl** или **Spring Data JPA Specifications** как более удобную обёртку над Criteria API.

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    @PersistenceContext
    private EntityManager entityManager;

    // Динамический поиск: любые комбинации фильтров
    public List<Product> search(ProductFilter filter) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Product> query = cb.createQuery(Product.class);
        Root<Product> product = query.from(Product.class);

        List<Predicate> predicates = new ArrayList<>();

        if (filter.getName() != null) {
            predicates.add(cb.like(
                cb.lower(product.get("name")),
                "%" + filter.getName().toLowerCase() + "%"
            ));
        }
        if (filter.getMinPrice() != null) {
            predicates.add(cb.greaterThanOrEqualTo(
                product.get("price"), filter.getMinPrice()
            ));
        }
        if (filter.getCategory() != null) {
            Join<Product, Category> category = product.join("category");
            predicates.add(cb.equal(category.get("name"), filter.getCategory()));
        }

        query.where(predicates.toArray(new Predicate[0]));
        query.orderBy(cb.desc(product.get("createdAt")));

        return entityManager.createQuery(query).getResultList();
    }
}

// Spring Data JPA Specifications — удобнее для стандартных случаев
@Repository
public interface ProductRepository
    extends JpaRepository<Product, Long>, JpaSpecificationExecutor<Product> {}

// Использование
Specification<Product> spec = Specification
    .where(ProductSpecs.hasName(filter.getName()))
    .and(ProductSpecs.hasMinPrice(filter.getMinPrice()));
productRepository.findAll(spec, pageable);
```

❗ **Ловушка:** Criteria API с MetaModel (`Product_.name` вместо строки `"name"`) даёт полную безопасность типов, но требует генерации метамодели (через annotation processor). Без MetaModel используются строки — та же проблема, что в JPQL, только многословнее.

**→ Уточняющий вопрос:** Чем Spring Data JPA Specifications отличаются от прямого использования Criteria API и в каких случаях лучше использовать Querydsl?

---

## 28. Как использовать JOIN FETCH для решения проблемы N+1?

`JOIN FETCH` — директива JPQL, которая инструктирует Hibernate загрузить связь немедленно через JOIN, независимо от настройки `FetchType`. Это наиболее прямолинейное решение N+1 для конкретного запроса.

Синтаксис: `SELECT a FROM Author a JOIN FETCH a.books` — один SQL с JOIN вместо 1+N SELECT.

Важные ограничения:
1. **Пагинация + JOIN FETCH коллекции** = `HHH90003004: HHH-000104` предупреждение: Hibernate загружает ВСЕ данные в память, а потом применяет пагинацию в Java, не в SQL. Для коллекций с пагинацией нужен `@BatchSize` или подзапросы.
2. **Несколько JOIN FETCH коллекций** в одном запросе → `MultipleBagFetchException` для List (используйте Set или несколько запросов)
3. JOIN FETCH возвращает дубли строк при one-to-many — нужен `DISTINCT` или `Set`

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // Решение N+1 для одной коллекции
    @Query("SELECT DISTINCT u FROM User u JOIN FETCH u.orders WHERE u.active = true")
    List<User> findActiveUsersWithOrders();

    // Проблема: два JOIN FETCH с List -> MultipleBagFetchException
    // @Query("SELECT u FROM User u JOIN FETCH u.orders JOIN FETCH u.roles")

    // Решение: два отдельных запроса (Hibernate объединит)
    @Query("SELECT DISTINCT u FROM User u JOIN FETCH u.orders WHERE u.id IN :ids")
    List<User> findWithOrders(@Param("ids") List<Long> ids);

    @Query("SELECT DISTINCT u FROM User u JOIN FETCH u.roles WHERE u.id IN :ids")
    List<User> findWithRoles(@Param("ids") List<Long> ids);

    // JOIN FETCH + пагинация = ПЛОХО для коллекций, ХОРОШО для @ManyToOne
    @Query("SELECT o FROM Order o JOIN FETCH o.user")
    Page<Order> findAllWithUser(Pageable pageable); // OK: ManyToOne, не коллекция
}

// EntityGraph — декларативная альтернатива JOIN FETCH
@EntityGraph(attributePaths = {"orders", "orders.items"})
List<User> findByActiveTrue();
```

❗ **Ловушка:** `JOIN FETCH` + `DISTINCT` в JPQL и SQL — разные вещи. `DISTINCT` в JPQL говорит Hibernate убрать дублирующиеся корневые объекты (User) из результата Java-списка, но в SQL без `DISTINCT` дублей может не быть. В Hibernate 6 `DISTINCT` в JPQL не добавляет `DISTINCT` в SQL автоматически.

**→ Уточняющий вопрос:** Почему нельзя использовать JOIN FETCH с пагинацией для коллекций и какой паттерн правильно решает эту задачу?

---

## 29. Что такое projection в JPA?

Проекция — выборка не всей сущности, а только нужных полей. Позволяет избежать загрузки лишних данных и проблем с ленивыми связями. JPA/Spring Data поддерживают несколько типов проекций.

**Interface-based projection** — самый простой: интерфейс с геттерами для нужных полей. Spring Data генерирует прокси.

**Class-based (DTO) projection** — JPQL `new` выражение или `@SqlResultSetMapping`. Полный контроль, нет прокси.

**Dynamic projection** — репозиторий возвращает разные типы проекций в зависимости от переданного `Class<T>`.

```java
// 1. Interface projection (Spring Data)
public interface UserSummary {
    Long getId();
    String getEmail();
    // Nested projection
    OrderSummary getLatestOrder();

    interface OrderSummary {
        Long getId();
        String getStatus();
    }
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findByActiveTrue(); // Spring Data сам построит запрос
}

// 2. DTO проекция через JPQL (рекомендую для сложных случаев)
public record UserOrderDto(Long userId, String email, Long orderId, BigDecimal total) {}

@Query("SELECT new com.example.dto.UserOrderDto(u.id, u.email, o.id, o.total) " +
       "FROM Order o JOIN o.user u WHERE o.status = 'ACTIVE'")
List<UserOrderDto> findActiveOrderSummaries();

// 3. Dynamic projection
public interface ProductRepository extends JpaRepository<Product, Long> {
    <T> List<T> findByCategoryId(Long categoryId, Class<T> type);
}
// Использование:
List<ProductDto> dtos = productRepository.findByCategoryId(1L, ProductDto.class);
List<ProductSummary> summaries = productRepository.findByCategoryId(1L, ProductSummary.class);
```

❗ **Ловушка:** Interface-based проекции с вложенными (Nested) связями всё равно могут вызывать N+1 — Spring Data может генерировать отдельные запросы для вложенных проекций. Проверяйте SQL в логах. DTO-проекция через JPQL `new` надёжнее и предсказуемее.

**→ Уточняющий вопрос:** Какую проекцию вы бы выбрали для REST API endpoint, который возвращает список пользователей с количеством их заказов, и почему?

---

## 30. Какие типы наследования поддерживает JPA?

JPA поддерживает три стратегии маппинга наследования Java-иерархии на реляционные таблицы:

**SINGLE_TABLE** (`@Inheritance(strategy = SINGLE_TABLE)`) — вся иерархия в одной таблице с дискриминатором. Быстро (нет JOIN), но много NULL-колонок. Полиморфные запросы — один SELECT. Нарушает 3NF.

**JOINED** (`@Inheritance(strategy = JOINED)`) — отдельная таблица для каждого класса, только его поля. Нормализовано, нет NULL. Полиморфный запрос требует JOIN всех таблиц. Медленнее при глубокой иерархии.

**TABLE_PER_CLASS** (`@Inheritance(strategy = TABLE_PER_CLASS)`) — таблица для каждого конкретного (не абстрактного) класса со всеми полями включая родительские. Нет JOIN при запросе конкретного типа, но полиморфный запрос требует UNION ALL — медленно.

```java
// SINGLE_TABLE — хорошо для небольших иерархий
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type", discriminatorType = DiscriminatorType.STRING)
public abstract class Payment {
    @Id @GeneratedValue private Long id;
    private BigDecimal amount;
}

@Entity
@DiscriminatorValue("CARD")
public class CardPayment extends Payment {
    private String cardNumber; // NULL для других типов
    private String cvv;
}

@Entity
@DiscriminatorValue("BANK")
public class BankTransferPayment extends Payment {
    private String iban; // NULL для других типов
}

// JOINED — хорошо для широких иерархий с уникальными полями
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Vehicle {
    @Id @GeneratedValue private Long id;
    private String brand;
}

@Entity
@PrimaryKeyJoinColumn(name = "vehicle_id")
public class Car extends Vehicle {
    private int doors; // в таблице cars
}
```

| Стратегия | Запрос полиморфный | NULL-поля | JOIN | Нормализация |
|---|---|---|---|---|
| SINGLE_TABLE | 1 SELECT | Много | Нет | Нет |
| JOINED | SELECT + JOIN | Нет | Да | Да |
| TABLE_PER_CLASS | UNION ALL | Нет | Нет | Частично |

❗ **Ловушка:** `TABLE_PER_CLASS` нельзя использовать с `GenerationType.IDENTITY` — каждая таблица имеет свой auto-increment, что нарушает уникальность id в полиморфных запросах. Используйте `SEQUENCE` или `TABLE` для генерации id.

**→ Уточняющий вопрос:** Как стратегия наследования влияет на производительность полиморфных запросов типа `SELECT p FROM Payment p WHERE p.amount > 1000`?
