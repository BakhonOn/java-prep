# Исключения

> 📇 Справочник уровня middle. Формат: ответ по сути → детали механизма → пример из реального кода → ❗ ловушка на собесе → уточняющий вопрос.

**Всего вопросов: 29**

---

## 1. В чём разница между checked и unchecked exceptions?

Checked-исключения — наследники `Exception`, за исключением `RuntimeException` и его потомков. Компилятор требует либо обработать их в `try-catch`, либо объявить в сигнатуре метода через `throws`. Unchecked-исключения — наследники `RuntimeException` — не требуют ни обработки, ни объявления: они сигнализируют о программных ошибках, которые нужно не ловить, а исправлять в коде.

На практике: `IOException`, `SQLException` — checked; `NullPointerException`, `IllegalArgumentException`, `IllegalStateException` — unchecked. В современных фреймворках (Spring, Hibernate) принято бросать unchecked, чтобы не засорять сигнатуры сервисного слоя конструкцией `throws XyzException` на каждом методе.

```java
// Checked — компилятор требует обработки
public void readFile(String path) throws IOException {
    Files.readAllBytes(Path.of(path));
}

// Unchecked — компилятор молчит
public User getUser(Long id) {
    return userRepo.findById(id)
        .orElseThrow(() -> new EntityNotFoundException("User not found: " + id));
}
```

❗ Частая ловушка: кандидаты путают «unchecked = не надо ловить» с «unchecked = нельзя ловить». Ловить можно любое; вопрос в том, обязан ли компилятор. Также нередко считают, что `Error` — это тоже unchecked exception, хотя `Error` вообще не является `Exception`.

**→ Уточняющий вопрос:** Почему большинство современных фреймворков (Spring, Hibernate) предпочитают unchecked exceptions, и какие у этого недостатки?

**↳ Ответ:** Checked exceptions нарушают принципы многослойной архитектуры: `SQLException` из репозитория не должна «светиться» в контроллере через `throws`-цепочку. Unchecked позволяют пробрасывать ошибку до `@ControllerAdvice` без загрязнения промежуточных сигнатур. Недостаток: потеря compiler safety — ошибку могут случайно «проглотить» или не обработать вовсе, а документация API менее очевидна без `throws` в сигнатуре.

---

## 2. Что такое checked exception и когда его использовать?

Checked exception — исключение, которое компилятор считает «ожидаемым» и требует явно учесть в коде вызывающей стороны. Философия: если ситуация реально ожидаема и вызывающий код может и должен на неё отреагировать — checked правомерен. Классический пример: `FileNotFoundException` при чтении конфига означает, что вызывающий код должен либо предоставить путь по умолчанию, либо завершить работу с понятным сообщением.

На практике checked уместен в низкоуровневых библиотеках (IO, JDBC, сетевое взаимодействие), но в бизнес-слое избыточен. Если метод сервиса объявляет `throws SQLException, IOException`, каждый клиент вынужден либо обрабатывать, либо пробрасывать эти технические детали, что нарушает инкапсуляцию слоёв. Поэтому на уровне репозитория SqlException оборачивают в unchecked `DataAccessException`.

```java
// Checked уместен в инфраструктурном слое
public Properties loadConfig(String path) throws IOException {
    try (InputStream is = new FileInputStream(path)) {
        Properties props = new Properties();
        props.load(is);
        return props;
    }
}

// В сервисном слое — оборачиваем в unchecked
public AppConfig getConfig() {
    try {
        return parse(loadConfig("app.properties"));
    } catch (IOException e) {
        throw new ConfigurationException("Cannot load app config", e);
    }
}
```

❗ Ловушка: ответить «checked — всегда хорошо, потому что компилятор помогает». Интервьюер ждёт понимания компромисса: checked даёт safety на уровне API, но создаёт boilerplate и протекание деталей реализации через слои.

**→ Уточняющий вопрос:** Как Spring обрабатывает `SQLException` из JDBC и почему это архитектурно правильно?

**↳ Ответ:** Spring JDBC перехватывает `SQLException` и транслирует её в иерархию unchecked `DataAccessException` (JdbcTemplate делает это автоматически). Это правильно архитектурно: сервисный слой не зависит от конкретного типа БД, замена PostgreSQL на Oracle не меняет сигнатуры методов сервиса. Плюс `DataAccessException` — богатая иерархия: `DataIntegrityViolationException`, `DeadlockLoserDataAccessException` и т.д. — можно обрабатывать точечно при необходимости.

---

## 3. Что такое unchecked exception (RuntimeException)?

`RuntimeException` и его наследники — исключения, возникающие из-за ошибок в логике программы: обращение к null, выход за границу массива, передача некорректного аргумента. Их не нужно объявлять в `throws` и не принято ловить для «восстановления» — правильная реакция: исправить код, а не добавить `catch`.

Unchecked — стандарт для кастомных исключений в бизнес-логике. `InsufficientFundsException extends RuntimeException` бросается из `AccountService` и перехватывается `@ControllerAdvice` на уровне HTTP-слоя, не засоряя промежуточные методы `throws`-объявлениями.

```java
// Кастомное unchecked для доменного слоя
public class OrderNotFoundException extends RuntimeException {
    private final Long orderId;

    public OrderNotFoundException(Long orderId) {
        super("Order not found: " + orderId);
        this.orderId = orderId;
    }

    public Long getOrderId() { return orderId; }
}

// Бросаем в сервисе — нет throws в сигнатуре
public Order getOrder(Long id) {
    return orderRepo.findById(id)
        .orElseThrow(() -> new OrderNotFoundException(id));
}
```

❗ Ловушка: «unchecked не надо обрабатывать вообще». На уровне REST-контроллера их надо перехватывать глобально (через `@ControllerAdvice`), иначе клиент получит 500 без понятного сообщения об ошибке.

**→ Уточняющий вопрос:** Как правильно обрабатывать `RuntimeException` в Spring REST API, чтобы клиент получал структурированный ответ об ошибке?

**↳ Ответ:** Через `@RestControllerAdvice` + `@ExceptionHandler` — глобальный перехватчик исключений для всех контроллеров. Каждый тип исключения маппится на свой HTTP-статус и тело ответа. Контроллер и сервис не содержат try-catch — исключение «всплывает» до советника, где логируется и конвертируется в `ResponseEntity<ErrorResponse>`. Это реализует принцип separation of concerns: бизнес-логика отделена от обработки ошибок.

---

## 4. Какие исключения нужно обрабатывать обязательно?

Обязательны к обработке (или объявлению в `throws`) только checked exceptions — `Exception` и его наследники, за исключением ветки `RuntimeException`. Это контракт на уровне языка: компилятор не соберёт код, если checked-исключение вызываемого метода не учтено.

`Error` и `RuntimeException` обрабатывать не обязательно, хотя технически `catch (Throwable t)` допустим. На практике `Error` (OOM, StackOverflow) обрабатывают только в особых случаях — инфраструктурном коде, который должен хоть что-то залогировать перед смертью.

```java
// Компилятор требует обработки — это checked
try {
    connection = DriverManager.getConnection(url); // throws SQLException
} catch (SQLException e) {
    throw new DataAccessException("DB connection failed", e);
}

// RuntimeException — компилятор не требует, но в GlobalExceptionHandler ловим
@ExceptionHandler(OrderNotFoundException.class)
public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException ex) {
    return ResponseEntity.status(404).body(new ErrorResponse(ex.getMessage()));
}
```

❗ Ловушка: сказать «все исключения надо обрабатывать». Правильно: обязательны только checked. Остальные — по необходимости, в зависимости от архитектуры приложения.

**→ Уточняющий вопрос:** Что будет, если объявить `throws Exception` вместо конкретного типа — это хорошая практика?

**↳ Ответ:** Нет, это антипаттерн. `throws Exception` заставляет всех вызывающих либо ловить `Exception` (слишком широко), либо тоже объявлять `throws Exception` — это «заражает» всю цепочку вызовов. Кроме того, API теряет информативность: непонятно, какие ошибки реально возможны. Checkstyle и SonarQube помечают это как violation. Исключение: `main()` метода и тестовые методы, где `throws Exception` — допустимое упрощение.

---

## 5. Что находится в вершине иерархии исключений?

В вершине иерархии — класс `Throwable`. От него наследуются два прямых потомка: `Error` и `Exception`. От `Exception` наследуется `RuntimeException`. Только потомков `Throwable` можно бросать (`throw`) и ловить (`catch`).

Полная иерархия: `Throwable → Error` (фатальные системные ошибки) и `Throwable → Exception → RuntimeException` (программные ошибки) / `Exception` (checked). Это важно понимать для построения catch-блоков: `catch (Exception e)` не поймает `OutOfMemoryError`, а `catch (Throwable t)` поймает всё.

```java
// catch (Exception) не ловит Error
try {
    riskyOperation();
} catch (Exception e) {
    // OOM сюда НЕ попадёт
    log.error("Exception caught", e);
} catch (Error e) {
    // фатальные ошибки JVM — редко нужно
    log.error("JVM Error, shutting down", e);
}
```

❗ Ловушка: утверждать, что `Error` — это checked исключение или что `catch (Exception e)` поймает всё. `Error` не является `Exception` — это отдельная ветка `Throwable`.

**→ Уточняющий вопрос:** Когда в реальном коде может быть обоснован `catch (Throwable t)`?

**↳ Ответ:** В инфраструктурном коде, где важно не потерять запись об ошибке даже при OOM: Thread.UncaughtExceptionHandler, планировщики задач (Quartz, Spring Scheduler — чтобы один упавший job не убил весь пул), top-level в main() для гарантированного логирования перед shutdown. Правило: поймать `Throwable`, залогировать, и если это `Error` — пробросить или немедленно завершить работу, не продолжать в неопределённом состоянии.

---

## 6. Что такое Throwable?

`Throwable` — абстрактный корневой класс всей иерархии исключений в Java. Любой объект, который можно бросить оператором `throw` или поймать в `catch`, обязан быть наследником `Throwable`. Класс содержит: сообщение (`message`), причину (`cause`), стек вызовов (`stackTrace`) и список подавленных исключений (`suppressedExceptions`).

Ключевые методы: `getMessage()` — текстовое описание, `getCause()` — исходное исключение при чейнинге, `getStackTrace()` — массив фреймов стека, `getSuppressed()` — подавленные исключения из try-with-resources, `printStackTrace()` — вывод стека в stderr.

```java
try {
    processPayment();
} catch (Throwable t) {
    // В продакшене так делают только в критической инфраструктуре
    String msg = t.getMessage();
    Throwable cause = t.getCause();
    StackTraceElement[] frames = t.getStackTrace();
    log.error("Critical failure: {}", msg, t);
    // getSuppressed() покажет закрытые ресурсы в TWR
    for (Throwable suppressed : t.getSuppressed()) {
        log.error("Suppressed: ", suppressed);
    }
}
```

❗ Ловушка: написать `catch (Throwable t)` в бизнес-коде и «проглотить» `Error`. Если поймали `OutOfMemoryError` и продолжаете работу, программа находится в неопределённом состоянии. Правило: `Error` ловить только чтобы залогировать и немедленно пробросить.

**→ Уточняющий вопрос:** Что содержит `Throwable.cause` и как это связано с exception chaining?

**↳ Ответ:** `cause` — ссылка на оригинальное исключение, которое стало причиной текущего. При `new MyException("msg", originalEx)` поле `cause` хранит `originalEx`. Exception chaining — это цепочка: ServiceException → DataAccessException → SQLException. Каждый уровень добавляет контекст своего слоя, не теряя информацию нижнего. В логах `Caused by:` — это и есть вывод цепочки `cause`.

---

## 7. В чём разница между Error и Exception?

`Error` представляет критические системные сбои, из которых приложение, как правило, не может восстановиться: `OutOfMemoryError`, `StackOverflowError`, `NoClassDefFoundError`. JVM бросает их при фундаментальных проблемах с окружением или ресурсами. По соглашению приложение не должно их ловить и обрабатывать — только логировать и умирать.

`Exception` — условия, с которыми приложение может справиться программно: файл не найден, сетевой таймаут, нарушение бизнес-правила. Разделение принципиально: `Error` сигнализирует о поломке среды выполнения, `Exception` — о проблеме в логике или вводе данных.

```java
// Error — не ловим в бизнес-коде, только логируем
public class JvmErrorHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        if (e instanceof Error) {
            log.error("Fatal JVM error in thread {}", t.getName(), e);
            // Оповестить мониторинг, затем позволить JVM упасть
        }
    }
}

// Exception — обрабатываем штатно
@ExceptionHandler(DataAccessException.class)
public ResponseEntity<ErrorResponse> handleDbError(DataAccessException ex) {
    log.error("Database error", ex);
    return ResponseEntity.status(503).body(new ErrorResponse("Database unavailable"));
}
```

❗ Ловушка: `StackOverflowError` при бесконечной рекурсии — частый пример на собесе. Кандидат пишет рекурсивный метод без базового случая и интервьюер спрашивает, какое исключение будет. Ответ: `StackOverflowError`, не исключение — его не нужно ловить, нужно починить код.

**→ Уточняющий вопрос:** Можно ли поймать `StackOverflowError`? Что произойдёт, если попытаться?

**↳ Ответ:** Технически да — `catch (StackOverflowError e)` скомпилируется и выполнится. Но это практически бесполезно: стек уже переполнен, и любой вызов метода внутри catch (включая `log.error`) снова вызовет `StackOverflowError`. После поимки JVM находится в нестабильном состоянии. Правильная реакция: зафиксировать факт и немедленно завершить процесс, не пытаясь восстановить работу.

---

## 8. Гарантируется ли выполнение блока finally?

`finally` выполняется почти всегда: и при нормальном завершении `try`, и при исключении, и при `return` из `try` или `catch`. Это делает его надёжным местом для освобождения ресурсов. Однако есть исключения: `System.exit()` немедленно останавливает JVM — `finally` не выполнится. Аналогично при `Runtime.halt()`, фатальном сбое JVM (kill -9), бесконечном цикле или зависании в `try`.

Важный нюанс: если в `try` есть `return`, то `finally` выполняется до фактического возврата. Если `finally` тоже содержит `return`, он перекрывает `return` из `try` — возвращается значение из `finally`. Это тихий и опасный баг.

```java
// Антипаттерн: return в finally перекрывает return из try
public int riskyMethod() {
    try {
        return 1; // Это значение будет потеряно
    } finally {
        return 2; // Всегда вернёт 2, даже при исключении в try
    }
}

// Корректное использование finally — освобождение ресурса
Connection conn = null;
try {
    conn = dataSource.getConnection();
    // работа с БД
} catch (SQLException e) {
    throw new DataAccessException("Query failed", e);
} finally {
    if (conn != null) {
        try { conn.close(); } catch (SQLException ignored) {}
    }
}
// Современный аналог — try-with-resources, предпочтительнее
```

❗ Ловушка: вопрос «что вернёт метод, если в try есть return 1, а в finally — return 2?» — классический вопрос на собесе. Ответ: 2. Ещё ловушка: утверждать, что `finally` гарантирован абсолютно, забыв про `System.exit()`.

**→ Уточняющий вопрос:** Почему `try-with-resources` предпочтительнее ручного `finally` для закрытия ресурсов?

**↳ Ответ:** Три причины: 1) не нужна проверка `if (resource != null)` перед закрытием — компилятор гарантирует вызов `close()`; 2) решает проблему потери исключений — если `try` и `close()` оба бросают, исключение из `close()` становится suppressed, а не заменяет основное; 3) обратный порядок закрытия при нескольких ресурсах гарантирован автоматически. Ручной `finally` требует вложенных try-catch и легко написать с ошибкой.

---

## 9. Что такое try-with-resources?

`try-with-resources` — синтаксический сахар, введённый в Java 7, для автоматического закрытия ресурсов. Ресурсы, объявленные в скобках после `try`, автоматически закрываются (вызывается `close()`) в конце блока — в порядке, обратном объявлению. Это устраняет необходимость в ручном `finally` и решает проблему потери исключения при закрытии ресурса.

Механизм: компилятор генерирует `finally`-блок с вызовом `close()`, причём если в `try` уже было исключение, исключение из `close()` не теряется, а добавляется как suppressed к основному. В Java 9 можно передавать в `try()` effectively final переменные, объявленные снаружи.

```java
// JDBC с try-with-resources — ресурсы закрываются в порядке: st → conn
public List<User> findAllUsers(DataSource dataSource) throws SQLException {
    try (Connection conn = dataSource.getConnection();
         PreparedStatement st = conn.prepareStatement("SELECT * FROM users");
         ResultSet rs = st.executeQuery()) {

        List<User> users = new ArrayList<>();
        while (rs.next()) {
            users.add(new User(rs.getLong("id"), rs.getString("name")));
        }
        return users;
    }
    // conn.close(), st.close(), rs.close() вызовутся автоматически
}

// Java 9+: effectively final переменная
Connection existing = dataSource.getConnection();
try (existing) { // передаём снаружи объявленную переменную
    // работа с existing
}
```

❗ Ловушка: думать, что `close()` вызывается только при успехе. Нет — он вызывается всегда, включая случай исключения в `try`. Ещё ловушка: не знать про подавленные исключения — если `close()` бросает исключение, оно не теряется, а присоединяется к основному через `addSuppressed()`.

**→ Уточняющий вопрос:** В каком порядке закрываются ресурсы при нескольких ресурсах в try-with-resources, и что произойдёт, если `close()` бросит исключение?

**↳ Ответ:** Закрываются в обратном порядке объявления — LIFO (последний открытый — первый закрытый). Для `try(A a; B b; C c)` порядок close: C → B → A. Если `close()` бросает исключение: если в `try`-блоке уже было исключение — исключение из `close()` уходит в suppressed; если `try` завершился нормально — исключение из `close()` становится основным и распространяется дальше.

---

## 10. Какие требования к ресурсам в try-with-resources?

Любой ресурс, используемый в `try-with-resources`, должен реализовывать интерфейс `AutoCloseable` (или его наследника `Closeable`). Этого достаточно — класс объявляет метод `close()`, и компилятор гарантирует его вызов.

Переменные ресурсов объявляются в скобках `try(...)` и закрываются в порядке, обратном объявлению: последний открытый — первый закрытый. Это важно для JDBC: `ResultSet` закроется раньше `PreparedStatement`, который закроется раньше `Connection`. В Java 9 можно передавать effectively final переменные, объявленные до блока try, без повторного объявления.

```java
// Кастомный ресурс для try-with-resources
public class DatabaseTransaction implements AutoCloseable {
    private final Connection conn;
    private boolean committed = false;

    public DatabaseTransaction(DataSource ds) throws SQLException {
        this.conn = ds.getConnection();
        this.conn.setAutoCommit(false);
    }

    public void commit() throws SQLException {
        conn.commit();
        committed = true;
    }

    @Override
    public void close() throws Exception {
        if (!committed) {
            conn.rollback(); // откат при исключении в try
        }
        conn.close();
    }
}

// Использование
try (DatabaseTransaction tx = new DatabaseTransaction(dataSource)) {
    userRepo.save(user);
    auditRepo.log(user.getId());
    tx.commit();
} // rollback + close вызовутся автоматически при исключении
```

❗ Ловушка: создать объект, реализующий `AutoCloseable`, но объявить переменную снаружи `try` — автоматического закрытия не будет. Ещё ловушка: объект в `try(...)` должен быть инициализирован — если инициализация бросит исключение, `close()` не вызовется для этого объекта (но вызовется для уже открытых ресурсов).

**→ Уточняющий вопрос:** Что произойдёт, если конструктор второго ресурса в `try(R1 r1 = ...; R2 r2 = ...)` бросит исключение — будет ли закрыт `r1`?

**↳ Ответ:** Да, `r1` будет закрыт. Компилятор генерирует код, отслеживающий, какие ресурсы успешно открыты: если `r2` не создался — `r1.close()` всё равно вызывается. Сам `r2` не существует (не был создан), его закрывать не нужно. Это важное поведение при работе с цепочками ресурсов (Connection → Statement → ResultSet): частичное открытие не ведёт к утечкам.

---

## 11. Что такое AutoCloseable интерфейс?

`AutoCloseable` — функциональный интерфейс из `java.lang`, введённый в Java 7 вместе с `try-with-resources`. Содержит единственный метод `void close() throws Exception`. Любой класс, реализующий этот интерфейс, может быть использован как ресурс в `try-with-resources`.

Метод `close()` объявлен с `throws Exception` (самый широкий checked тип), что даёт максимальную гибкость реализациям. На практике конкретные реализации сужают тип: `Closeable.close()` бросает только `IOException`, `Connection.close()` бросает `SQLException`.

```java
// Реализация AutoCloseable для управления внешним ресурсом
public class KafkaProducerWrapper implements AutoCloseable {
    private final KafkaProducer<String, String> producer;

    public KafkaProducerWrapper(Properties config) {
        this.producer = new KafkaProducer<>(config);
    }

    public void send(String topic, String message) {
        producer.send(new ProducerRecord<>(topic, message));
    }

    @Override
    public void close() {
        producer.flush();
        producer.close(); // гарантированно вызовется в try-with-resources
    }
}

try (KafkaProducerWrapper wrapper = new KafkaProducerWrapper(config)) {
    wrapper.send("orders", orderJson);
}
```

❗ Ловушка: метод `close()` в `AutoCloseable` не обязан быть идемпотентным (в отличие от `Closeable`). Если ваш `AutoCloseable` вызывается дважды, это может привести к ошибке. Документируйте поведение повторного вызова явно.

**→ Уточняющий вопрос:** В чём разница между `AutoCloseable` и `Closeable` — можно ли использовать `Closeable` вместо `AutoCloseable` в try-with-resources?

**↳ Ответ:** Да, `Closeable` расширяет `AutoCloseable`, поэтому любой `Closeable` работает в try-with-resources. Разница: `Closeable.close()` бросает только `IOException` и должен быть идемпотентным; `AutoCloseable.close()` может бросать любой `Exception` и не обязан быть идемпотентным. Используйте `Closeable` для IO-ресурсов, `AutoCloseable` — для остальных.

---

## 12. В чём разница между AutoCloseable и Closeable?

`Closeable` — более старый интерфейс (Java 5) из пакета `java.io`, предназначенный для потоков ввода-вывода. Он расширяет `AutoCloseable` и сужает контракт: его `close()` бросает только `IOException` (не любой `Exception`). По контракту `Closeable.close()` должен быть идемпотентным — повторный вызов не должен давать эффекта.

`AutoCloseable` — более общий интерфейс (Java 7), `close()` может бросать любой `Exception`. Не требует идемпотентности. Оба работают в `try-with-resources`. Для IO-ресурсов (потоки, readers) используйте `Closeable`; для остальных управляемых ресурсов — `AutoCloseable`.

```java
// Closeable — для IO, идемпотентный close
public class BufferedFileReader implements Closeable {
    private final BufferedReader reader;
    private boolean closed = false;

    public BufferedFileReader(Path path) throws IOException {
        this.reader = Files.newBufferedReader(path);
    }

    public String readLine() throws IOException {
        return reader.readLine();
    }

    @Override
    public void close() throws IOException {
        if (!closed) {
            reader.close();
            closed = true;
        }
        // Повторный вызов — ничего не делает (идемпотентен)
    }
}

// AutoCloseable — для не-IO ресурсов, идемпотентность не гарантирована
public class RedisConnection implements AutoCloseable {
    // ...
    @Override
    public void close() throws Exception {
        // Может бросить не-IOException, не обязан быть идемпотентным
        jedis.close();
    }
}
```

❗ Ловушка: кандидаты часто отвечают «разница только в типе исключения». Интервьюер ждёт упоминания идемпотентности: `Closeable` гарантирует безопасность повторного вызова `close()`, `AutoCloseable` — нет. Это важно для некоторых паттернов, где ресурс может быть закрыт в нескольких местах.

**→ Уточняющий вопрос:** Почему `InputStream` реализует `Closeable`, а не напрямую `AutoCloseable`?

**↳ Ответ:** Исторически: `InputStream` и `Closeable` появились в Java 1.0/5 задолго до `AutoCloseable` (Java 7). При введении try-with-resources в Java 7 сделали так, чтобы `Closeable` расширял `AutoCloseable` — это позволило всем существующим IO-классам автоматически работать в try-with-resources без изменений. Это обратная совместимость: миллионы существующих классов стали совместимы с новым синтаксисом без рефакторинга.

---

## 13. Можно ли создавать кастомные исключения?

Да, создание кастомных исключений — стандартная практика в Java-разработке. Наследуемся от `Exception` (checked) или `RuntimeException` (unchecked). Минимально необходимый набор конструкторов: с сообщением, с сообщением и причиной, только с причиной. Без конструктора с `cause` теряется информация об исходной ошибке при оборачивании.

Кастомные исключения дают доменный смысл ошибкам, позволяют нести дополнительный контекст (ID сущности, код ошибки), а в Spring — точечно маппить на HTTP-коды через `@ExceptionHandler` или `@ResponseStatus`.

```java
// Кастомное исключение для доменного слоя
public class InsufficientFundsException extends RuntimeException {
    private final Long accountId;
    private final BigDecimal requested;
    private final BigDecimal available;

    public InsufficientFundsException(Long accountId, BigDecimal requested, BigDecimal available) {
        super(String.format(
            "Insufficient funds on account %d: requested %.2f, available %.2f",
            accountId, requested, available
        ));
        this.accountId = accountId;
        this.requested = requested;
        this.available = available;
    }

    // Конструктор с cause — для оборачивания
    public InsufficientFundsException(Long accountId, BigDecimal requested,
                                       BigDecimal available, Throwable cause) {
        super("Insufficient funds on account " + accountId, cause);
        this.accountId = accountId;
        this.requested = requested;
        this.available = available;
    }

    // Геттеры для GlobalExceptionHandler
    public Long getAccountId() { return accountId; }
    public BigDecimal getRequested() { return requested; }
    public BigDecimal getAvailable() { return available; }
}

// В GlobalExceptionHandler
@ExceptionHandler(InsufficientFundsException.class)
public ResponseEntity<ErrorResponse> handleInsufficientFunds(InsufficientFundsException ex) {
    return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
        .body(new ErrorResponse("INSUFFICIENT_FUNDS", ex.getMessage()));
}
```

❗ Ловушка: создать исключение без конструктора с `cause`. Тогда при оборачивании низкоуровневой ошибки (`SQLException`) в кастомную теряется стек оригинальной ошибки, и диагностика в продакшне становится крайне сложной.

**→ Уточняющий вопрос:** Сколько конструкторов должно быть у хорошо спроектированного кастомного исключения и почему?

**↳ Ответ:** Минимум два: `(String message)` и `(String message, Throwable cause)`. Первый — для «первичного» броска, второй — для оборачивания другого исключения (exception chaining). Без конструктора с `cause` вы не можете корректно обернуть исходную ошибку, теряется стек. Опционально: `(Throwable cause)` и безаргументный. Для доменных исключений с контекстом — ещё конструктор с бизнес-параметрами (как `OrderNotFoundException(Long orderId, Throwable cause)`).

---

## 14. Когда стоит создавать свои исключения?

Кастомные исключения оправданы в трёх случаях: когда нужна доменная семантика (стандартные типы не выражают суть ошибки), когда нужно нести дополнительный контекст (ID объекта, код ошибки для клиента), и когда нужна точечная обработка — разные HTTP-коды или разные стратегии восстановления для разных классов ошибок.

Создавать кастомное исключение только ради нового имени (например, `MyRuntimeException extends RuntimeException` без доп. данных) — избыточно. `IllegalArgumentException`, `IllegalStateException` покрывают большинство программных ошибок. Правило: если поведение обработчика не отличается от стандартного типа — не создавайте.

```java
// Оправдано: несёт контекст, нужна отдельная обработка
public class EntityNotFoundException extends RuntimeException {
    private final String entityType;
    private final Object entityId;

    public EntityNotFoundException(String entityType, Object entityId) {
        super(entityType + " not found with id: " + entityId);
        this.entityType = entityType;
        this.entityId = entityId;
    }
}

// Оправдано: иерархия для семейства ошибок
public abstract class PaymentException extends RuntimeException {
    private final String transactionId;
    // ...
}
public class PaymentDeclinedException extends PaymentException { ... }
public class PaymentTimeoutException extends PaymentException { ... }

// В обработчике — разные HTTP-статусы:
// EntityNotFoundException → 404
// PaymentDeclinedException → 422
// PaymentTimeoutException → 503
```

❗ Ловушка: создавать слишком детальную иерархию исключений (10+ классов для одного модуля). Это усложняет код без выгоды. Хороший признак правильного решения: каждое исключение в иерархии маппится на отдельный ответ/стратегию обработки.

**→ Уточняющий вопрос:** Как организовать иерархию исключений в многомодульном приложении — выносить ли базовые классы в общий модуль?

**↳ Ответ:** Да, базовые типы (например, `ApplicationException`, `EntityNotFoundException`) стоит выносить в общий `core`/`common` модуль. Это позволяет `@ControllerAdvice` в web-модуле обрабатывать исключения из любого доменного модуля без циклических зависимостей. Специфичные исключения (`OrderNotFoundException`, `PaymentDeclinedException`) живут в своих модулях и наследуют от общих базовых. Не выносите слишком много — только то, что реально нужно нескольким модулям.

---

## 15. Что лучше: наследоваться от Exception или RuntimeException?

Современный консенсус в Java-сообществе — предпочитать `RuntimeException` для кастомных исключений в бизнес-логике. Причины: нет засорения сигнатур методов `throws`, нет принудительного оборачивания в промежуточных слоях, лучшая читаемость кода. Это позиция Spring, Hibernate, и большинства современных фреймворков.

`Exception` (checked) оправдан, когда: метод — публичный API библиотеки (не приложения), вызывающий действительно обязан обработать ошибку (не просто может), и восстановление — реалистичный сценарий. Если вы пишете бизнес-приложение, а не библиотеку — почти всегда `RuntimeException`.

```java
// Checked — оправдан для API, где клиент ОБЯЗАН обработать
public interface FileProcessor {
    void process(Path file) throws FileProcessingException; // checked
}

// Unchecked — правильный выбор для бизнес-логики в приложении
@Service
public class OrderService {
    public Order createOrder(OrderRequest request) {
        if (!inventory.isAvailable(request.getProductId())) {
            throw new ProductUnavailableException(request.getProductId()); // unchecked
        }
        // ...
    }
}

// GlobalExceptionHandler в Spring перехватывает всё
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ProductUnavailableException.class)
    public ResponseEntity<ErrorResponse> handle(ProductUnavailableException ex) {
        return ResponseEntity.status(409).body(new ErrorResponse(ex.getMessage()));
    }
}
```

❗ Ловушка: сказать «надо всегда checked, потому что компилятор помогает». Интервьюер ждёт обоснованного ответа с пониманием компромисса. Checked нарушает принцип открытости/закрытости: добавление нового checked-исключения в метод ломает всех вызывающих.

**→ Уточняющий вопрос:** Как решает эту проблему Lombok `@SneakyThrows` и почему это спорная практика?

**↳ Ответ:** `@SneakyThrows` использует generics-трюк через `Unsafe` — обманывает компилятор, но не JVM (JVM не проверяет типы исключений при броске). Спорность: нарушает контракт языка — вызывающий код не знает, что метод может бросить checked исключение, не может его поймать по типу. Хорошо в лямбдах для Stream API (где checked не допускается), но в публичном API это скрытая мина. Если команда не вся знает о `@SneakyThrows` — потратят часы на отладку.

---

## 16. Что такое stack trace?

Stack trace — снимок стека вызовов на момент создания исключения (в конструкторе `Throwable`). Представляет собой массив `StackTraceElement`, каждый из которых содержит: имя класса, имя метода, имя файла и номер строки. Читается снизу вверх: в самом низу — точка входа (main или поток), в самом верху — место, где было создано исключение.

JVM захватывает стек при создании объекта исключения (`new SomeException()`), а не при его броске (`throw`). Это важно: если создать исключение в одном месте и бросить в другом, стек покажет место создания.

```java
// Stack trace читается сверху вниз по происхождению ошибки:
// java.lang.NullPointerException: Cannot invoke "String.length()" because "str" is null
//     at com.example.service.UserService.processName(UserService.java:42)  ← место ошибки
//     at com.example.service.UserService.updateUser(UserService.java:28)   ← вызвавший
//     at com.example.controller.UserController.update(UserController.java:55)
//     at ...

// При exception chaining стек показывает оба уровня:
// ServiceException: Failed to process user
//     at com.example.service.UserService.processName(UserService.java:45)
// Caused by: java.sql.SQLException: Connection timeout
//     at com.example.repo.UserRepository.findById(UserRepository.java:33)
```

❗ Ловушка: захват стека — дорогая операция. В высоконагруженных системах создание исключений в «горячих» путях кода (hot path) может стать узким местом из-за `fillInStackTrace()`. Иногда переопределяют этот метод, чтобы отключить захват стека для технических исключений контроля потока.

**→ Уточняющий вопрос:** Почему создание исключений — дорогая операция и как это можно оптимизировать?

**↳ Ответ:** Дорого из-за `fillInStackTrace()` — при создании каждого `Throwable` JVM обходит весь стек вызовов и сохраняет его (десятки-сотни фреймов). Это ~1-10 мкс. Оптимизации: 1) переопределить `fillInStackTrace()` чтобы возвращал `this` без захвата стека — для технических исключений «потока управления» (например, sentinel-объект для выхода из Stream); 2) создать статические final экземпляры исключений без стека; 3) использовать `Optional` вместо исключений для ожидаемых «не нашли» случаев.

---

## 17. Что делает метод printStackTrace()?

`printStackTrace()` выводит полную информацию об исключении в `System.err`: тип исключения, сообщение, стек вызовов и, при наличии, цепочку причин (`Caused by:`). По умолчанию выводит в stderr, но есть перегрузки: `printStackTrace(PrintStream)` и `printStackTrace(PrintWriter)` для вывода в произвольный поток.

В продакшн-коде этот метод — антипаттерн. Вывод идёт в stderr без структуры, без временных меток, без уровней логирования, смешивается с другими потоками в многопоточном коде. Правильный подход — передать исключение в логгер: `log.error("Описание проблемы", exception)`.

```java
// АНТИПАТТЕРН — не делать в продакшне
try {
    processOrder(order);
} catch (Exception e) {
    e.printStackTrace(); // Выводит в stderr, теряется в потоке вывода
}

// ПРАВИЛЬНО — через SLF4J с передачей исключения вторым аргументом
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

try {
    processOrder(order);
} catch (OrderProcessingException e) {
    log.error("Failed to process order {}: {}", order.getId(), e.getMessage(), e);
    // Logback/Log4j2 автоматически выведет stack trace
    throw e; // пробрасываем дальше или оборачиваем
}
```

❗ Ловушка: `log.error(e.getMessage())` без передачи самого исключения — теряется stack trace в логах. Правильно: `log.error("message", exception)` — второй аргумент-Throwable обрабатывается логгером специально и выводит полный стек.

**→ Уточняющий вопрос:** Почему `log.error(ex.getMessage())` — плохая практика, и чем это отличается от `log.error("context", ex)`?

**↳ Ответ:** `log.error(ex.getMessage())` логирует только строку сообщения — stack trace полностью теряется. В логах появится строка без контекста, и когда ошибка произойдёт в продакшне, будет невозможно понять откуда она пришла. `log.error("context", ex)` — SLF4J видит что последний аргумент `Throwable` и автоматически добавляет полный стек с `Caused by:`. Правило: всегда передавайте объект исключения в логгер, не только его сообщение.

---

## 18. Как правильно логировать исключения?

Правило одно: всегда передавать объект исключения в логгер последним аргументом — `log.error("Описание ситуации", exception)`. SLF4J и реализации (Logback, Log4j2) специально обрабатывают последний аргумент типа `Throwable` и выводят полный stack trace. Без этого в логах будет только сообщение без стека — диагностика невозможна.

Второй принцип: логировать на правильном уровне и в правильном месте. Исключение нужно логировать один раз — там, где оно обрабатывается окончательно (обычно в `GlobalExceptionHandler`). Если пробрасываете исключение дальше — не логируйте; если оборачиваете — тоже; логируйте только в финальной точке обработки. Двойное логирование одного исключения засоряет логи.

```java
// В сервисном слое — НЕ логируем, просто оборачиваем и пробрасываем
public Order processOrder(OrderRequest request) {
    try {
        return orderRepo.save(buildOrder(request));
    } catch (DataAccessException e) {
        // Логировать НЕ надо — GlobalExceptionHandler залогирует
        throw new OrderCreationException("Failed to create order", e);
    }
}

// В GlobalExceptionHandler — логируем ОДИН РАЗ
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(OrderCreationException.class)
    public ResponseEntity<ErrorResponse> handleOrderCreation(
            OrderCreationException ex, HttpServletRequest request) {
        log.error("Order creation failed. Path: {}, Cause: {}",
            request.getRequestURI(), ex.getMessage(), ex); // ex — последний аргумент!
        return ResponseEntity.status(500)
            .body(new ErrorResponse("ORDER_CREATION_FAILED", ex.getMessage()));
    }
}
```

❗ Ловушка: логировать исключение на каждом уровне стека (в репозитории, сервисе и контроллере). Результат — одна ошибка даёт 3–5 идентичных строк в логах, усложняя поиск. Логируйте один раз — в точке финальной обработки.

**→ Уточняющий вопрос:** Как отслеживать исключения через несколько микросервисов — какие инструменты используются для distributed tracing?

**↳ Ответ:** Через propagation трейс-контекста: каждый запрос получает уникальный `traceId`, который передаётся в заголовках между сервисами (`X-B3-TraceId` или W3C `traceparent`). Logback/Log4j2 через MDC автоматически добавляют `traceId` в каждую строку лога. Инструменты: Zipkin, Jaeger (хранят трейсы и строят граф зависимостей), Micrometer Tracing (Spring Boot 3+), OpenTelemetry (стандарт). При ошибке по `traceId` можно найти все связанные логи во всех сервисах.

---

## 19. Что такое оборачивание (wrapping) исключений?

Оборачивание (exception wrapping) — перехват низкоуровневого исключения и проброс нового, более осмысленного для текущего слоя, с сохранением оригинала как `cause`. Это один из ключевых приёмов построения многослойной архитектуры: `SQLException` в репозитории оборачивается в `DataAccessException` в сервисе, который оборачивается в `OrderProcessingException` на уровне бизнес-логики.

Цель: изолировать слои друг от друга. Контроллер не должен знать о `SQLException` — это деталь реализации репозитория. При правильном оборачивании с `cause` полная цепочка (и все стеки) сохраняется в `getCause()` и выводится в логах как `Caused by:`.

```java
// Репозиторий — техническое исключение
public User findById(Long id) {
    try {
        return jdbcTemplate.queryForObject(SQL_FIND_BY_ID, USER_ROW_MAPPER, id);
    } catch (EmptyResultDataAccessException e) {
        // Оборачиваем с сохранением cause
        throw new UserNotFoundException(id, e);
    } catch (DataAccessException e) {
        throw new RepositoryException("Failed to query user " + id, e);
    }
}

// Сервис — бизнес-исключение
public UserDto getUser(Long id) {
    try {
        return mapper.toDto(userRepository.findById(id));
    } catch (UserNotFoundException e) {
        throw e; // пробрасываем как есть — это уже бизнес-исключение
    } catch (RepositoryException e) {
        throw new ServiceUnavailableException("User service temporarily unavailable", e);
    }
}

// В логах будет видна вся цепочка:
// ServiceUnavailableException: User service temporarily unavailable
//     at UserService.getUser(UserService.java:35)
// Caused by: RepositoryException: Failed to query user 42
//     at UserRepository.findById(UserRepository.java:28)
// Caused by: DataAccessException: ...
```

❗ Ловушка: обернуть без `cause` — `throw new ServiceException(e.getMessage())`. Теряется оригинальный стек, и в логах будет только верхнеуровневое исключение без первопричины. Всегда передавайте исходное исключение: `new MyException("message", originalException)`.

**→ Уточняющий вопрос:** Как получить полную цепочку `cause` программно — есть ли стандартный метод для этого?

**↳ Ответ:** Стандартного метода в JDK нет. Паттерн: цикл `while (t.getCause() != null) t = t.getCause()` — даёт root cause. В Apache Commons Lang есть `ExceptionUtils.getRootCause(e)` и `ExceptionUtils.getThrowableList(e)` (вся цепочка как список). Guava: `Throwables.getRootCause(e)`. Осторожно с циклами в цепочке `cause` — в JDK есть защита (повторное присвоение `initCause` запрещено), но самодельные исключения теоретически могут создать цикл.

---

## 20. Почему не стоит глотать исключения (catch empty)?

Пустой `catch`-блок или `catch` только с комментарием — один из самых опасных антипаттернов в Java. Исключение произошло — значит, что-то пошло не так. Если проглотить его, программа продолжает работу в потенциально некорректном состоянии, а разработчик даже не узнает о проблеме. Отладка такого кода превращается в кошмар: симптомы появляются далеко от источника.

Минимально допустимо — залогировать. Ещё лучше — обработать или пробросить. Единственный оправданный случай тихого игнорирования: намеренное подавление несущественной ошибки при закрытии ресурса (второй `close()` в `finally`) — и то только с комментарием, объясняющим почему.

```java
// АНТИПАТТЕРН — глотаем исключение
try {
    user.setLastLogin(parseDate(dateString));
} catch (ParseException e) {
    // TODO: handle this
    // Пользователь продолжит работу без обновления lastLogin
    // Никто не узнает что dateString был невалидным
}

// ДОПУСТИМО — логируем и принимаем решение
try {
    user.setLastLogin(parseDate(dateString));
} catch (ParseException e) {
    log.warn("Cannot parse login date '{}' for user {}, using current time",
        dateString, user.getId(), e);
    user.setLastLogin(LocalDateTime.now()); // fallback
}

// ЕДИНСТВЕННЫЙ оправданный случай тихого игнорирования:
Connection conn = null;
try {
    conn = getConnection();
    // ...
} finally {
    if (conn != null) {
        try {
            conn.close();
        } catch (SQLException e) {
            // Намеренно игнорируем: основная работа уже завершена,
            // ошибка закрытия соединения не критична
            log.debug("Failed to close connection", e);
        }
    }
}
```

❗ Ловушка: интервьюер может спросить «а когда оправдано игнорировать исключение?». Правильный ответ: при закрытии ресурса в `finally`, когда основная операция уже завершена — и даже тогда логировать на DEBUG. Никогда не «глотать» в бизнес-коде.

**→ Уточняющий вопрос:** Есть ли инструменты статического анализа, которые находят проглоченные исключения в коде?

**↳ Ответ:** Да: SonarQube (правило `S108` — пустые catch-блоки, `S1166` — перехваченное исключение не используется), SpotBugs (детектор `DE_MIGHT_IGNORE`), PMD (правило `EmptyCatchBlock`). Checkstyle тоже можно настроить. В IntelliJ IDEA встроена проверка пустых `catch`-блоков с предупреждением. Интеграция в CI через Maven/Gradle plugin — эти правила срабатывают на pull request и блокируют мёрж.

---

## 21. Что делает ключевое слово throws?

`throws` в сигнатуре метода — объявление контракта: «этот метод может бросить указанные checked-исключения, и вызывающий код должен их обработать». Это компиляторный механизм, который заставляет клиентов API явно решить: поймать исключение или объявить его в своём `throws`.

`throws` — только декларация, не обработка. Метод может объявить `throws IOException`, но никогда его не бросить. Компилятор не проверяет соответствие того, что реально бросается, объявленному `throws`, — только то, что все thrown checked-исключения объявлены. Для unchecked (`RuntimeException`, `Error`) `throws` не обязателен, но иногда используется для документирования.

```java
// throws — декларация для вызывающего кода
public interface ReportGenerator {
    void generate(Report report) throws ReportGenerationException, IOException;
}

// Имплементация — может бросать то, что в интерфейсе объявлено
public class PdfReportGenerator implements ReportGenerator {
    @Override
    public void generate(Report report) throws ReportGenerationException, IOException {
        try {
            byte[] content = renderPdf(report);
            Files.write(getOutputPath(report), content);
        } catch (RenderException e) {
            throw new ReportGenerationException("PDF render failed", e);
        }
        // IOException от Files.write пробрасывается напрямую
    }
}

// Вызывающий обязан обработать или пробросить
public void generateMonthlyReport() throws IOException {
    try {
        generator.generate(buildReport());
    } catch (ReportGenerationException e) {
        log.error("Report generation failed", e);
        throw new ServiceException("Cannot generate report", e);
    }
    // IOException пробрасывается выше через throws
}
```

❗ Ловушка: думать, что `throws Exception` — безопасный вариант «объявить всё сразу». На практике это плохой дизайн: вызывающий код не знает, какие конкретно исключения ожидать. Checkstyle и SonarQube ругаются на `throws Exception` в продакшн-коде.

**→ Уточняющий вопрос:** Можно ли в переопределённом методе объявить более широкий `throws`, чем в родительском?

**↳ Ответ:** Нет — это правило ковариантности исключений. Переопределённый метод может объявить только те же или более узкие checked исключения, или вообще не объявлять их. Нельзя добавить новый checked тип или расширить до `Exception`, если родитель бросает `IOException`. Причина: код, работающий через тип родителя, должен быть уверен в полноте своих catch-блоков. Для unchecked ограничений нет — их не нужно объявлять.

---

## 22. Можно ли пробросить checked exception из метода без throws?

Напрямую — нет. Компилятор требует, чтобы любое checked-исключение, которое может вылететь из метода, было либо поймано, либо объявлено в `throws`. Стандартный способ обойти это — обернуть в `RuntimeException`.

Существуют обходные пути: Lombok `@SneakyThrows` использует generics-трюк (`<T extends Throwable>`) и обманывает компилятор (но не JVM), позволяя бросить checked без объявления. `Unsafe.throwException()` делает то же самое. Оба подхода спорны: они нарушают контракт языка и могут удивить разработчиков, ожидающих стандартного поведения.

```java
// Стандартный способ — обернуть в unchecked
public void processFile(Path path) {
    try {
        Files.readAllBytes(path);
    } catch (IOException e) {
        throw new UncheckedIOException(e); // стандартный wrapper из java.io
    }
}

// Lombok @SneakyThrows — бросает checked без объявления
@SneakyThrows(IOException.class)
public void processFileSneaky(Path path) {
    Files.readAllBytes(path); // IOException бросится без объявления
}

// Generics-трюк без Lombok (для понимания механизма)
@SuppressWarnings("unchecked")
public static <T extends Throwable> void sneakyThrow(Throwable t) throws T {
    throw (T) t; // Компилятор думает T extends Throwable, JVM не проверяет
}
```

❗ Ловушка: `@SneakyThrows` — частый ответ на этом вопросе, но нужно знать риски. Если метод бросает `IOException` через `@SneakyThrows` без объявления, вызывающий код не сможет его поймать как `IOException` стандартным `catch` — только как `Exception` или `Throwable`.

**→ Уточняющий вопрос:** Почему `UncheckedIOException` появился в Java 8 и в чём его преимущество перед `RuntimeException(e)`?

**↳ Ответ:** Java 8 ввёл Stream API и лямбды, где checked исключения запрещены. `Files.lines()` и другие IO-операции в потоках требовали обёртки. `UncheckedIOException` — семантически точная обёртка: по имени класса понятно, что это IO-ошибка, её можно поймать отдельным `catch(UncheckedIOException e)` и извлечь оригинальный `IOException` через `getCause()`. `new RuntimeException(ioEx)` — работает, но теряет семантику: нельзя точечно поймать IO-ошибки среди других `RuntimeException`.

---

## 23. Что произойдёт, если в блоке finally тоже возникнет исключение?

Если в `finally` бросается исключение, оно полностью заменяет исключение из `try`-блока. Оригинальное исключение теряется — оно не присоединяется как `cause`, не логируется автоматически, просто исчезает. В логах появится исключение из `finally`, а реальная первопричина останется неизвестной.

Это одна из самых коварных ловушек в Java. Реальный сценарий: `try` бросил `SQLException` при выполнении запроса, `finally` бросил `SQLException` при попытке вернуть соединение в пул — в логах будет только второй, а первый потерян. Именно поэтому рекомендуется в `finally`-блоках не допускать исключений или ловить их внутри `finally`.

```java
// Опасный код: исключение из finally скрывает исключение из try
public void riskyOperation() throws Exception {
    try {
        throw new RuntimeException("Real problem"); // ← это теряется!
    } finally {
        throw new RuntimeException("Finally problem"); // ← это дойдёт до вызывающего
    }
}

// Правильный подход: ловим исключения внутри finally
public void safeOperation(Connection conn) throws SQLException {
    try {
        executeQuery(conn);
    } catch (SQLException e) {
        throw e;
    } finally {
        try {
            conn.close();
        } catch (SQLException closeEx) {
            // Логируем, но не перебрасываем — не хотим потерять исходное
            log.warn("Failed to close connection", closeEx);
        }
    }
}

// try-with-resources решает эту проблему: исключение из close() → suppressed
try (Connection conn = dataSource.getConnection()) {
    executeQuery(conn);
}
// Если executeQuery бросил E1 и close() бросил E2:
// E1 дойдёт до вызывающего, E2 будет в E1.getSuppressed()
```

❗ Ловушка: именно поэтому `try-with-resources` предпочтительнее ручного `finally` для ресурсов. TWR решает эту проблему через `addSuppressed()`: оба исключения сохраняются, основное — как главное, исключение из `close()` — как suppressed.

**→ Уточняющий вопрос:** Как `try-with-resources` решает проблему потери исключения из `finally` — что именно делает компилятор под капотом?

**↳ Ответ:** Компилятор генерирует `finally`-блок с try-catch вокруг `close()`. Если в `try` было исключение (`primaryEx != null`) и `close()` тоже бросил — вызывается `primaryEx.addSuppressed(closeEx)`, и дальше бросается `primaryEx`. Если `try` завершился нормально — исключение из `close()` бросается напрямую. Это принципиальное отличие от ручного `finally`, где исключение из `finally` просто замещало исходное без сохранения.

---

## 24. Что такое suppressed exceptions?

Suppressed exceptions — механизм, введённый в Java 7 вместе с `try-with-resources` для решения проблемы потери исключений при закрытии ресурсов. Если в блоке `try` возникло исключение и при закрытии ресурса (вызов `close()`) тоже возникло исключение — оба сохраняются. Исключение из `try` остаётся основным и доходит до вызывающего, а исключение из `close()` добавляется к нему через `addSuppressed()`.

Получить подавленные исключения: `throwable.getSuppressed()` — возвращает массив. Можно добавлять вручную: `primaryException.addSuppressed(secondaryException)`. Это полезно при реализации собственных ресурсов или при оборачивании множественных ошибок.

```java
// Демонстрация suppressed через кастомный ресурс
public class BrokenResource implements AutoCloseable {
    @Override
    public void close() throws Exception {
        throw new Exception("Close failed");
    }
}

try {
    try (BrokenResource r = new BrokenResource()) {
        throw new RuntimeException("Main problem");
    }
} catch (RuntimeException e) {
    System.out.println("Main: " + e.getMessage());           // Main: Main problem
    System.out.println("Suppressed count: " + e.getSuppressed().length); // 1
    System.out.println("Suppressed: " + e.getSuppressed()[0].getMessage()); // Close failed
}

// Ручное добавление suppressed для агрегации ошибок
public void processAll(List<Task> tasks) {
    RuntimeException aggregate = null;
    for (Task task : tasks) {
        try {
            task.execute();
        } catch (RuntimeException e) {
            if (aggregate == null) {
                aggregate = new RuntimeException("Some tasks failed");
            }
            aggregate.addSuppressed(e);
        }
    }
    if (aggregate != null) throw aggregate;
}
```

❗ Ловушка: при ручном `finally` (до Java 7 или без `try-with-resources`) suppressed не работает автоматически — исключение из `finally` просто замещает исходное. Только `try-with-resources` автоматически добавляет в suppressed. Не знать об `getSuppressed()` при диагностике ошибок в логах — значит пропустить важную часть картины.

**→ Уточняющий вопрос:** Как увидеть suppressed exceptions в stack trace — выводит ли их стандартный `printStackTrace()`?

**↳ Ответ:** Да, `printStackTrace()` выводит suppressed с маркером `Suppressed:` (в отличие от `Caused by:` для cause). Logback и Log4j2 тоже выводят suppressed при логировании через `log.error("msg", ex)`. Программно: `ex.getSuppressed()` возвращает массив. Если используете агрегатор логов (ELK, Splunk) — убедитесь, что парсер настроен на обработку многострочных стектрейсов с `Suppressed:` секциями.

---

## 25. Можно ли несколько catch блоков для одного try?

Да, один `try`-блок может иметь произвольное количество `catch`-блоков. Они проверяются сверху вниз, и выполняется только первый подходящий — тот, чей тип совместим с брошенным исключением (является тем же типом или суперклассом). После срабатывания одного `catch` остальные не проверяются.

Порядок `catch`-блоков критически важен: если более общий тип (родитель) стоит раньше специфичного (потомка) — компилятор выдаст ошибку «exception already caught», потому что до специфичного блока никогда не дойдёт. Это гарантия корректности на уровне компилятора.

```java
public ResponseEntity<?> handleRequest(Long orderId) {
    try {
        Order order = orderService.findOrder(orderId);
        Payment payment = paymentService.charge(order);
        return ResponseEntity.ok(payment);

    } catch (OrderNotFoundException e) {
        // Специфичный — первым
        log.warn("Order not found: {}", orderId);
        return ResponseEntity.notFound().build();

    } catch (PaymentDeclinedException e) {
        // Другой специфичный тип
        log.warn("Payment declined for order {}: {}", orderId, e.getMessage());
        return ResponseEntity.status(422).body(new ErrorResponse(e.getMessage()));

    } catch (ServiceUnavailableException e) {
        // Инфраструктурная ошибка
        log.error("Service unavailable for order {}", orderId, e);
        return ResponseEntity.status(503).body(new ErrorResponse("Service unavailable"));

    } catch (Exception e) {
        // Общий — последним как fallback
        log.error("Unexpected error for order {}", orderId, e);
        return ResponseEntity.status(500).body(new ErrorResponse("Internal error"));
    }
}
```

❗ Ловушка: написать `catch (Exception e)` первым, а ниже более специфичные типы. Компилятор выдаст ошибку компиляции: «Exception has already been caught». Это одна из немногих проблем иерархии исключений, которую компилятор отлавливает сам.

**→ Уточняющий вопрос:** Что произойдёт, если в `catch`-блоке возникнет другое исключение — будут ли проверены оставшиеся `catch`-блоки того же `try`?

**↳ Ответ:** Нет. Если в catch-блоке возникает новое исключение, оставшиеся catch-блоки того же try не проверяются. Новое исключение распространяется вверх по стеку вызовов как обычное необработанное исключение. Выполнится `finally`-блок (если есть), но другие `catch`-блоки пропускаются. Это важно: если в `catch` пишете код с возможными исключениями — оборачивайте его в отдельный try-catch.

---

## 26. Что такое multi-catch (catching multiple exceptions)?

Multi-catch — синтаксис Java 7, позволяющий перечислить несколько типов исключений в одном `catch` через символ `|`. Используется, когда для нескольких исключений одинаковая обработка — устраняет дублирование кода. Переменная исключения в multi-catch неявно `final`: переприсваивание запрещено компилятором.

Ограничение: нельзя использовать типы, находящиеся в отношении наследования (например, `catch (Exception | RuntimeException e)` — ошибка компиляции, так как `RuntimeException` уже покрывается `Exception`). Байткод multi-catch аналогичен отдельным `catch`-блокам с одинаковым телом — это чисто синтаксический сахар.

```java
// Multi-catch устраняет дублирование
// ДО Java 7:
try {
    processData();
} catch (IOException e) {
    log.error("IO error", e);
    throw new ProcessingException("Data processing failed", e);
} catch (ParseException e) {
    log.error("Parse error", e);
    throw new ProcessingException("Data processing failed", e);
}

// С Java 7 multi-catch:
try {
    processData();
} catch (IOException | ParseException e) {
    // e — неявно final, нельзя делать e = new IOException()
    log.error("Processing error: {}", e.getMessage(), e);
    throw new ProcessingException("Data processing failed", e);
}

// В Spring GlobalExceptionHandler:
@ExceptionHandler({UserNotFoundException.class, OrderNotFoundException.class})
public ResponseEntity<ErrorResponse> handleNotFound(RuntimeException ex) {
    return ResponseEntity.status(404).body(new ErrorResponse(ex.getMessage()));
}
```

❗ Ловушка: попытаться изменить переменную `e` внутри multi-catch. `e` в multi-catch — неявно `final` (в отличие от обычного `catch`). Попытка присвоить `e = new IOException()` приведёт к ошибке компиляции. Интервьюеры иногда спрашивают именно об этом.

**→ Уточняющий вопрос:** Почему переменная в multi-catch неявно final — как это связано с тем, как компилятор компилирует multi-catch в байткод?

**↳ Ответ:** В байткоде multi-catch компилируется как несколько отдельных catch-блоков с одинаковым телом, но общим типом переменной. Если бы `e` не была final, компилятор должен был бы определить её статический тип (либо `IOException`, либо `ParseException`) — это невозможно при общем handler. Запрет переприсваивания решает проблему: тип `e` в runtime — всегда тот, что реально был брошен, но компилятор работает с обобщённым типом (пересечением типов).

---

## 27. В каком порядке располагать catch блоки?

Правило одно: от более специфичных (потомков) к более общим (родителям). Компилятор Java принудительно соблюдает это правило для наследования: если `catch (Exception e)` стоит раньше `catch (IOException e)`, это ошибка компиляции — `IOException` никогда не будет достигнута.

На практике порядок отражает архитектуру обработки ошибок: сначала конкретные бизнес-исключения с конкретной реакцией, затем инфраструктурные, в конце — `Exception` как fallback. В `@ExceptionHandler` методах Spring порядок также важен: более конкретный обработчик должен иметь приоритет над общим.

```java
try {
    orderService.placeOrder(request);

} catch (InsufficientFundsException e) {
    // 1. Конкретное бизнес-исключение — специфичная реакция
    return ResponseEntity.status(422)
        .body(new ErrorResponse("INSUFFICIENT_FUNDS", e.getMessage()));

} catch (PaymentException e) {
    // 2. Родитель PaymentDeclinedException и других — после специфичных
    return ResponseEntity.status(402)
        .body(new ErrorResponse("PAYMENT_ERROR", e.getMessage()));

} catch (DataAccessException e) {
    // 3. Инфраструктурное — после бизнес-исключений
    log.error("DB error during order placement", e);
    return ResponseEntity.status(503).body(new ErrorResponse("DB_ERROR", "Try later"));

} catch (Exception e) {
    // 4. Общий fallback — всегда последним
    log.error("Unexpected error", e);
    return ResponseEntity.status(500).body(new ErrorResponse("INTERNAL_ERROR", "Unexpected error"));
}
```

❗ Ловушка: компилятор отлавливает нарушение только при прямом наследовании. Если два несвязанных исключения `A` и `B` попали в `catch (Exception e)` выше `catch (A e)` — компилятор не поможет, `A` просто никогда не будет поймана отдельным блоком. Статические анализаторы (SonarQube) умеют это находить.

**→ Уточняющий вопрос:** Как работает порядок `@ExceptionHandler` методов в Spring `@ControllerAdvice` — гарантирован ли порядок обработки?

**↳ Ответ:** Spring выбирает наиболее специфичный обработчик по иерархии типов исключений: если есть `@ExceptionHandler(OrderNotFoundException.class)` и `@ExceptionHandler(RuntimeException.class)`, Spring предпочтёт первый для `OrderNotFoundException`. Порядок нескольких `@ControllerAdvice` классов управляется `@Order` или `Ordered` интерфейсом. При нескольких методах с одинаковым типом поведение не определено — избегайте дублирования. Один класс `@ControllerAdvice` с чёткой иерархией обработчиков — правильный подход.

---

## 28. Можно ли повторно бросить исключение?

Да, пойманное исключение можно пробросить дальше оператором `throw`. Это стандартная практика: поймать для логирования или добавления контекста, затем бросить повторно — либо то же исключение, либо обёрнутое в новое.

Java 7 ввёл улучшенный rethrow (precise rethrow): если в `catch` поймано исключение по широкому типу (например, `Exception`) и оно перебрасывается без изменений, компилятор отслеживает реальные типы из `try`-блока. Это позволяет объявить более точный `throws` на методе, не дублируя все конкретные типы.

```java
// Простой rethrow — логируем и пробрасываем как есть
public void processOrder(Order order) throws OrderException {
    try {
        validate(order);
        persist(order);
    } catch (OrderException e) {
        log.warn("Order processing failed for orderId={}", order.getId(), e);
        throw e; // пробрасываем то же исключение
    }
}

// Rethrow с оборачиванием — добавляем контекст
public void importOrders(List<Order> orders) {
    for (int i = 0; i < orders.size(); i++) {
        try {
            processOrder(orders.get(i));
        } catch (OrderException e) {
            // Добавляем индекс к контексту ошибки
            throw new BatchImportException("Failed at order index " + i, e);
        }
    }
}

// Java 7 precise rethrow — компилятор знает, что реально бросается
public void execute() throws IOException, SQLException { // точные типы, не Exception
    try {
        if (mode == IO) readFile();    // throws IOException
        else queryDb();                // throws SQLException
    } catch (Exception e) {
        log.error("Operation failed", e);
        throw e; // компилятор знает: реально это IOException или SQLException
    }
}
```

❗ Ловушка: при rethrow через `throw e` стек исключения сохраняется — он указывает на место первоначального броска, а не на `throw e`. Если нужно обновить стек (например, при переносе исключения между потоками), есть метод `fillInStackTrace()`, но в обычном коде это не нужно.

**→ Уточняющий вопрос:** Что делает `exception.fillInStackTrace()` и когда его реально используют?

**↳ Ответ:** `fillInStackTrace()` перезахватывает текущий стек вызовов и записывает его в объект исключения, заменяя стек, захваченный при создании. Реальные случаи применения: 1) при переносе исключения между потоками (передали Future.get() и хотите стек потока-получателя, а не потока-отправителя); 2) переопределить в custom исключении, вернув `this` без захвата стека — для «управляющих» исключений в hot path, где производительность критична. В обычном бизнес-коде не нужен.

---

## 29. Что такое exception chaining?

Exception chaining (цепочка исключений) — механизм сохранения оригинального исключения при оборачивании. Новое исключение создаётся с ссылкой на причину (cause): `new MyException("message", originalException)`. Cause хранится в поле `Throwable.cause`, доступен через `getCause()`. При выводе stack trace каждое звено цепи выводится через `Caused by:`.

Это фундаментальный приём многослойной архитектуры. Без chaining: обернули `SQLException` в `ServiceException`, передав только `getMessage()` — стек оригинального исключения потерян, причина неизвестна. С chaining: полная картина в логах — от HTTP-запроса через сервис до конкретной SQL-ошибки.

```java
// Правильное оборачивание — cause передаётся явно
public Order findOrder(Long id) {
    try {
        return jdbcTemplate.queryForObject(SQL, ORDER_MAPPER, id);
    } catch (EmptyResultDataAccessException e) {
        throw new OrderNotFoundException(id, e); // ← cause
    } catch (DataAccessException e) {
        throw new RepositoryException("Failed to find order " + id, e); // ← cause
    }
}

// Кастомное исключение с поддержкой chaining
public class OrderNotFoundException extends RuntimeException {
    private final Long orderId;

    public OrderNotFoundException(Long orderId) {
        super("Order not found: " + orderId);
        this.orderId = orderId;
    }

    public OrderNotFoundException(Long orderId, Throwable cause) {
        super("Order not found: " + orderId, cause); // ← передаём cause в Throwable
        this.orderId = orderId;
    }
}

// В логах увидим полную цепочку:
// OrderNotFoundException: Order not found: 42
//     at OrderRepository.findOrder(OrderRepository.java:35)
//     at OrderService.getOrder(OrderService.java:28)
// Caused by: EmptyResultDataAccessException: Incorrect result size: expected 1, actual 0
//     at JdbcTemplate.queryForObject(JdbcTemplate.java:...)
```

❗ Ловушка: создать исключение через `new OrderNotFoundException(id)` без передачи `cause` при оборачивании. Это самая распространённая ошибка при работе с исключениями: `Caused by:` пропадёт из логов, и команда будет тратить часы на поиск первопричины в продакшне. Всегда проверяйте, что у вашего кастомного исключения есть конструктор с `Throwable cause`.

**→ Уточняющий вопрос:** Как обойти цепочку причин программно — как дойти до «корневой» причины (root cause) через несколько уровней `getCause()`?

**↳ Ответ:** Стандартный паттерн: `Throwable root = ex; while (root.getCause() != null) root = root.getCause();`. Apache Commons `ExceptionUtils.getRootCause(ex)` делает то же самое. Guava: `Throwables.getRootCause(ex)`. Используется при диагностике: часто root cause — это `SocketTimeoutException` или `SQLException`, а не обёртки уровнями выше. В `@ControllerAdvice` иногда проверяют root cause для более точного маппинга на HTTP-статус.
