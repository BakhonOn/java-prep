# Паттерны проектирования

> 📇 Справочник уровня middle. Формат: ответ по сути → механизм/структура → пример из реального кода → ❗ ловушка собеса.

**Всего вопросов: 16**

---

## 1. Что такое паттерны проектирования?

Паттерны проектирования — это типовые, проверенные практикой решения часто встречающихся проблем при проектировании программного обеспечения. Это не готовый код, который можно скопировать, а концептуальная схема-шаблон, адаптируемая под конкретную задачу. Паттерны возникли как обобщение опыта большого числа разработчиков и были систематизированы «Бандой четырёх» (GoF) в книге 1994 года.

Ключевая ценность паттернов — общий словарь. Когда разработчик говорит «здесь применён Strategy», коллега сразу понимает структуру: есть интерфейс алгоритма, есть несколько реализаций, есть контекст, который их использует. Паттерны помогают строить системы, устойчивые к изменениям, снижают связанность компонентов и делают архитектуру предсказуемой.

Паттерны — это ответ на конкретные «болевые точки»: размножение подклассов, жёсткая связанность, дублирование логики переходов между состояниями и т.д. Использовать паттерн ради паттерна — антипаттерн. Вопрос всегда «какую проблему он решает здесь и сейчас».

❗ На собесе проверяют не знание определения, а понимание мотивации. Типичный вопрос-ловушка: «В чём разница между паттерном и алгоритмом?» Алгоритм — конкретная последовательность шагов, решающая задачу; паттерн — концептуальное решение, которое реализуется по-разному в разных контекстах. Также могут спросить: «Почему иногда паттерны считают оверинжинирингом?» — потому что неуместное применение добавляет сложность без выгоды.

**→ Уточняющий вопрос:** Можешь назвать паттерн, который активно используется во фреймворках, которые ты применяешь в работе, и объяснить, какую проблему он там решает?

**↳ Ответ:** Spring использует Proxy для `@Transactional` и `@Cacheable` — решает проблему добавления cross-cutting concerns без изменения бизнес-кода. Template Method — `JdbcTemplate`, `RestTemplate`: инвариантная часть (управление соединением/ресурсом) зафиксирована, вариативная (SQL, callback) — снаружи. Это устраняет дублирование boilerplate и гарантирует правильное освобождение ресурсов.

---

## 2. Какие категории паттернов существуют?

GoF разделил 23 паттерна на три категории по характеру решаемой проблемы. **Порождающие** (Creational) отвечают на вопрос «как создать объект»: Singleton, Factory Method, Abstract Factory, Builder, Prototype. Они скрывают логику создания и делают систему независимой от конкретных типов создаваемых объектов.

**Структурные** (Structural) описывают, как объекты компонуются в более крупные структуры: Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy. Они помогают строить гибкие иерархии и добавлять поведение без изменения существующего кода. **Поведенческие** (Behavioral) регулируют взаимодействие и распределение ответственности между объектами: Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor.

В реальном коде Spring является энциклопедией паттернов: `ApplicationContext` — это Factory (создаёт бины) и Registry одновременно; `BeanPostProcessor` — Chain of Responsibility; `@Transactional` — Proxy; `JdbcTemplate` — Template Method; события Spring (`ApplicationEvent`) — Observer; `DispatcherServlet` — Front Controller (не GoF, но широко известный).

❗ На собесе часто просят «назвать паттерны из трёх категорий» — достаточно по 2-3 примера с объяснением, зачем они. Слабый ответ — просто перечисление названий без понимания цели. Сильный — «Builder решает проблему телескопических конструкторов», «Proxy нужен для прозрачного добавления cross-cutting concerns».

**→ Уточняющий вопрос:** Какой паттерн из каждой категории ты применял на практике и какую конкретную проблему он решил?

**↳ Ответ:** Порождающий: Builder для сложных DTO — устраняет телескопические конструкторы, делает объект иммутабельным. Структурный: Proxy через Spring AOP — добавляет логирование/метрики без изменения сервисного кода. Поведенческий: Strategy для выбора алгоритма (обработка оплаты по типу) — заменяет switch по типу на `Map<Тип, Стратегия>`, легко расширяется новым вариантом без правки существующего кода.

---

## 3. Что такое Singleton?

Singleton — порождающий паттерн, гарантирующий, что класс имеет только один экземпляр в JVM, и предоставляющий глобальную точку доступа к нему. Типичные применения: конфигурация приложения, пул соединений, логгер, кэш. Суть ограничения — `private` конструктор и статический метод доступа.

Классическая реализация через статическое поле — самая простая, но не ленивая:

```java
public class ConfigManager {
    private static final ConfigManager INSTANCE = new ConfigManager();

    private ConfigManager() {
        // загрузка конфигурации
    }

    public static ConfigManager getInstance() {
        return INSTANCE;
    }

    public String getProperty(String key) { /* ... */ return "value"; }
}
```

В Spring понятие Singleton переосмыслено: по умолчанию все бины имеют scope `singleton`, но это singleton в рамках `ApplicationContext`, а не ClassLoader. Если создать два контекста — будет два экземпляра. Spring управляет жизненным циклом через IoC-контейнер, и это предпочтительный способ работы с «одиночными» объектами в enterprise-коде.

❗ Ключевая ловушка: «Singleton Spring и GoF Singleton — это одно и то же?» Нет. GoF Singleton обеспечивает единственность через `private` конструктор на уровне ClassLoader. Spring Singleton — это область видимости бина внутри контейнера. Также проверяют: как Singleton нарушается через рефлексию (`setAccessible(true)` на конструкторе) и через сериализацию (десериализация создаёт новый объект).

**→ Уточняющий вопрос:** Как защитить Singleton от разрушения через рефлексию и сериализацию?

**↳ Ответ:** От рефлексии — только Enum: JVM запрещает вызов конструктора enum через рефлексию. Для класса: бросать исключение в конструкторе если INSTANCE уже не null. От сериализации — добавить `readResolve()`: `private Object readResolve() { return INSTANCE; }` — десериализация вернёт существующий экземпляр вместо создания нового. Enum решает обе проблемы автоматически — это главная причина рекомендации Блоха.

---

## 4. Как реализовать потокобезопасный Singleton?

Существует несколько подходов с разными компромиссами. **Enum-singleton** — рекомендация Джошуа Блоха («Effective Java», Item 3): потокобезопасен по спецификации JVM, защищён от рефлексии (попытка вызвать конструктор через рефлексию бросает `IllegalArgumentException`) и сериализации (JVM гарантирует единственность enum-константы):

```java
public enum DatabaseConnectionPool {
    INSTANCE;

    private final List<Connection> pool = new ArrayList<>();

    public Connection acquire() { /* ... */ return pool.get(0); }
    public void release(Connection c) { pool.add(c); }
}

// использование
DatabaseConnectionPool.INSTANCE.acquire();
```

**Holder Idiom** (Bill Pugh Singleton) — ленивая инициализация без синхронизации за счёт гарантий загрузчика классов: статический вложенный класс загружается только при первом обращении к `INSTANCE`, и JVM гарантирует thread-safe инициализацию статических полей:

```java
public class AppConfig {
    private AppConfig() {}

    private static class Holder {
        static final AppConfig INSTANCE = new AppConfig();
    }

    public static AppConfig getInstance() {
        return Holder.INSTANCE;
    }
}
```

**Double-checked locking** с `volatile` — третий вариант, подробнее в следующем вопросе.

❗ Классическая ошибка на собесе — реализация DCL без `volatile`. Интервьюер специально ждёт этого упущения. Второй вопрос-ловушка: «Почему synchronized на методе `getInstance()` — плохое решение?» Потому что каждый вызов будет захватывать монитор, а конкуренция там только при первой инициализации — это избыточная синхронизация.

**→ Уточняющий вопрос:** Объясни, почему Holder Idiom работает без явной синхронизации — какая гарантия JVM это обеспечивает?

**↳ Ответ:** JLS §12.4.2 гарантирует: статические поля класса инициализируются атомарно при первой загрузке класса ClassLoader'ом под Class Initialization Lock. Вложенный класс `Holder` загружается только при первом обращении к `Holder.INSTANCE` — это lazy. JVM обеспечивает happens-before между инициализацией и первым чтением. Синхронизация не нужна — JVM делает это за тебя через механизм загрузки классов.

---

## 5. Что такое double-checked locking?

Double-checked locking (DCL) — оптимизация ленивой инициализации Singleton: первая проверка `null` выполняется без блокировки (быстрый путь), и только когда объект ещё не создан, захватывается монитор. После захвата — вторая проверка, потому что между первой проверкой и захватом монитора другой поток мог уже создать объект.

```java
public class ReportService {
    // volatile ОБЯЗАТЕЛЕН
    private static volatile ReportService instance;

    private ReportService() {}

    public static ReportService getInstance() {
        if (instance == null) {                    // проверка 1: без блокировки
            synchronized (ReportService.class) {
                if (instance == null) {            // проверка 2: под блокировкой
                    instance = new ReportService();
                }
            }
        }
        return instance;
    }
}
```

`volatile` здесь критически важен по двум причинам. Во-первых, он запрещает переупорядочивание инструкций: оператор `instance = new ReportService()` компилируется в три шага (выделить память → инициализировать объект → записать ссылку в поле), и JIT/CPU могут выполнить шаги 1 и 3 до шага 2. Без `volatile` другой поток увидит ненулевую ссылку на недоинициализированный объект. Во-вторых, `volatile` обеспечивает happens-before: запись в volatile-поле происходит-до любого последующего чтения этого поля.

До Java 5 (до JSR-133, пересмотревшего Java Memory Model) DCL был сломан даже с `volatile` — стоит упомянуть на собесе как демонстрацию знания истории JMM.

❗ Самая частая ловушка: «Зачем нужен `volatile`, ведь мы уже используем `synchronized`?» Синхронизированный блок защищает только поток, вошедший в него. Первая проверка (`if (instance == null)`) выполняется вне `synchronized`, поэтому без `volatile` не гарантирована видимость свежего значения.

**→ Уточняющий вопрос:** Что такое happens-before и как оно связано с volatile?

**↳ Ответ:** Happens-before — гарантия JMM: все эффекты памяти операции A видны при выполнении B, если A happens-before B. Запись в volatile поле устанавливает happens-before с любым последующим чтением этого поля любым потоком. В DCL: без volatile ненулевая ссылка может быть видна другому потоку, но поля объекта — ещё 0/null (переупорядочивание инструкций). Volatile запрещает это переупорядочивание.

---

## 6. Каковы проблемы с Singleton?

Singleton — один из самых критикуемых паттернов, нередко называемый антипаттерном. Главная проблема: **глобальное изменяемое состояние**. Любой код в приложении может обратиться к Singleton и незаметно изменить его состояние, порождая скрытые зависимости и делая поведение системы непредсказуемым.

**Проблемы при тестировании**: Singleton с `private` конструктором сложно заменить моком. Unit-тест, зависящий от Singleton с состоянием (например, кэш или счётчик), не изолирован — результат зависит от порядка выполнения тестов. С DI-фреймворками (Spring) проблема решается: инъецируется интерфейс, мок подставляется через `@MockBean`.

**Проблемы в многопоточности**: Синглтон с изменяемым состоянием — потенциальное место гонки данных. Spring-бины по умолчанию singleton — и если в них есть поля уровня экземпляра, которые изменяются в runtime (non-final), это баг. Классический пример:

```java
@Service
public class UserService {
    // ОШИБКА: изменяемое состояние в singleton-бине
    private User currentUser; // разные потоки будут перезаписывать это поле

    public void process(User user) {
        this.currentUser = user; // race condition
        // ...
    }
}
```

Правильный подход: Spring singleton-бины должны быть stateless или работать с thread-local/scope-ограниченным состоянием (request scope, session scope).

❗ Любимый вопрос интервьюера: «Ваш Spring-сервис — это Singleton. Можно ли хранить в нём состояние?» Правильный ответ: можно хранить неизменяемое (final) или thread-safe состояние (AtomicLong, ConcurrentHashMap). Хранить изменяемые поля, зависящие от конкретного запроса, — нельзя.

**→ Уточняющий вопрос:** Как Spring решает проблему stateful-зависимостей в singleton-бинах? Что будет, если singleton-бин заинжектит prototype-бин?

**↳ Ответ:** Singleton-бин инжектирует prototype один раз при создании — prototype фактически живёт как singleton (баг). Решения: `ObjectProvider<T>` (Spring вернёт новый при каждом `getObject()`), аннотация `@Lookup` (Spring переопределит метод через CGLIB для возврата нового прототипа), или `@Scope(proxyMode = TARGET_CLASS)` на prototype-бине. Правило: если prototype важен — используй ObjectProvider.

---

## 7. В чём разница между Factory Method и Abstract Factory?

Оба паттерна инкапсулируют создание объектов, но решают задачи разного масштаба. **Factory Method** — метод в классе (обычно абстрактный), который подклассы переопределяют для создания объекта нужного типа. Это паттерн через **наследование**: «дай подклассу решить, какой объект создавать». Один метод — один тип продукта.

```java
// Factory Method в Spring: FactoryBean<T>
public class ConnectionFactoryBean implements FactoryBean<DataSource> {
    @Override
    public DataSource getObject() {
        // подкласс/реализация решает, какой DataSource создать
        return new HikariDataSource(config);
    }
    @Override
    public Class<?> getObjectType() { return DataSource.class; }
}
```

**Abstract Factory** — отдельный объект-фабрика, создающий **семейство взаимосвязанных объектов**. Это паттерн через **композицию**: клиент получает фабрику и использует её, не зная конкретных типов. Ключевое слово: «семейство» — несколько типов объектов, которые должны быть согласованы между собой.

```java
// Abstract Factory: создаём согласованное семейство UI-компонентов
interface UIComponentFactory {
    Button createButton();
    TextField createTextField();
    Dialog createDialog();
}

class WindowsUIFactory implements UIComponentFactory {
    public Button createButton()       { return new WindowsButton(); }
    public TextField createTextField() { return new WindowsTextField(); }
    public Dialog createDialog()       { return new WindowsDialog(); }
}

class MacOSUIFactory implements UIComponentFactory {
    public Button createButton()       { return new MacButton(); }
    public TextField createTextField() { return new MacTextField(); }
    public Dialog createDialog()       { return new MacDialog(); }
}
```

В Spring `BeanFactory` и `ApplicationContext` реализуют идею Abstract Factory: `ApplicationContext` создаёт и возвращает бины разных типов, согласованных в рамках одного контекста конфигурации.

❗ Классическая путаница: Factory Method — это паттерн класса (наследование), Abstract Factory — паттерн объекта (делегирование). Если интервьюер спросит «когда одно, когда другое»: Factory Method — когда нужно вынести создание одного типа объекта в подкласс; Abstract Factory — когда нужно гарантировать совместимость группы создаваемых объектов.

**→ Уточняющий вопрос:** Как Abstract Factory соотносится с Dependency Injection? Можно ли считать DI-контейнер реализацией Abstract Factory?

**↳ Ответ:** DI-контейнер — это обобщённая Abstract Factory: создаёт согласованное семейство объектов (бинов) и управляет их зависимостями. Разница: GoF Abstract Factory создаёт объекты фиксированного семейства типов; DI-контейнер создаёт любой тип по запросу. Spring `ApplicationContext` — это Abstract Factory + Singleton Registry + Lifecycle Manager в одном, что делает его мощнее классического GoF паттерна.

---

## 8. Когда использовать Builder?

Builder решает проблему **телескопических конструкторов** — когда класс имеет много параметров, особенно опциональных, и создание объекта через конструктор становится нечитаемым и error-prone (`new User("John", null, null, true, false, 25, null)`). Паттерн разделяет конструирование объекта и его представление, позволяя строить объект пошагово.

```java
// Immutable Value Object с Builder
public final class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;

    private HttpRequest(Builder builder) {
        this.url       = Objects.requireNonNull(builder.url, "url required");
        this.method    = builder.method;
        this.headers   = Collections.unmodifiableMap(builder.headers);
        this.body      = builder.body;
        this.timeoutMs = builder.timeoutMs;
    }

    public static Builder builder(String url) { return new Builder(url); }

    public static class Builder {
        private final String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeoutMs = 5000;

        private Builder(String url) { this.url = url; }

        public Builder method(String method)  { this.method = method; return this; }
        public Builder header(String k, String v) { headers.put(k, v); return this; }
        public Builder body(String body)      { this.body = body; return this; }
        public Builder timeout(int ms)        { this.timeoutMs = ms; return this; }
        public HttpRequest build()            { return new HttpRequest(this); }
    }
}

// использование
HttpRequest request = HttpRequest.builder("https://api.example.com/users")
    .method("POST")
    .header("Authorization", "Bearer token")
    .body("{\"name\":\"John\"}")
    .timeout(3000)
    .build();
```

В реальном коде Builder повсюду: `StringBuilder`, `Stream.Builder`, Lombok `@Builder`, Spring `MockMvcRequestBuilders`, `UriComponentsBuilder`, `ResponseEntity.ok().header(...).body(...)`. Также `ProcessBuilder`, `HttpClient.newBuilder()` в Java 11+. Builder особенно ценен в сочетании с иммутабельностью: объект строится поэтапно в Builder, а финальный объект — неизменяем.

❗ Частая ловушка: «Чем Builder отличается от конструктора с множеством параметров?» Читаемость — только часть ответа. Главное: Builder позволяет сделать финальный объект иммутабельным (private конструктор, только Builder имеет доступ), добавить валидацию в `build()`, и строить объект условно (if/else в цепочке).

**→ Уточняющий вопрос:** В чём разница между GoF Builder и Fluent Builder (как в Lombok)? GoF Builder предполагает отдельный Director — зачем он нужен?

**↳ Ответ:** GoF Builder включает Director — класс, инкапсулирующий алгоритм сборки: один алгоритм, разные Builder'ы (XML Builder, JSON Builder — одна логика наполнения). Полезен когда порядок шагов важен и переиспользуется. Lombok @Builder — Fluent Builder без Director: клиент сам управляет последовательностью. Для простых DTO Director избыточен; для сложных документов с обязательным порядком шагов — оправдан.

---

## 9. Что такое Prototype pattern?

Prototype — порождающий паттерн, создающий новые объекты путём **копирования существующего прототипа**, а не через `new`. Применяется, когда создание объекта дорого (например, требует сложной инициализации или обращения к базе данных), а нужно много похожих объектов с небольшими отличиями. Или когда тип объекта известен только в рантайме.

В Java Prototype реализуется через интерфейс `Cloneable` и метод `clone()`, либо через копирующий конструктор (предпочтительно):

```java
// Вариант 1: через Cloneable (имеет подводные камни)
public class ReportTemplate implements Cloneable {
    private String title;
    private List<String> sections; // mutable field!

    @Override
    public ReportTemplate clone() {
        try {
            ReportTemplate copy = (ReportTemplate) super.clone();
            // ВАЖНО: super.clone() делает shallow copy!
            // List нужно копировать вручную (deep copy)
            copy.sections = new ArrayList<>(this.sections);
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(); // не произойдёт
        }
    }
}

// Вариант 2: копирующий конструктор (рекомендуется Блохом)
public class ReportTemplate {
    private String title;
    private List<String> sections;

    // копирующий конструктор
    public ReportTemplate(ReportTemplate source) {
        this.title    = source.title;
        this.sections = new ArrayList<>(source.sections); // deep copy
    }
}
```

В Spring Prototype scope — это прямая реализация паттерна: каждый запрос `getBean()` возвращает новый экземпляр бина. `@Scope("prototype")` на бине означает: «используй меня как прототип, создавай копию по запросу».

❗ Главная ловушка: **shallow copy vs deep copy**. `super.clone()` копирует примитивы и ссылки, но не объекты, на которые они указывают. Если в объекте есть коллекции или mutable-объекты, shallow copy — баг: обе копии будут разделять одну и ту же коллекцию. Интервьюер обязательно спросит про это.

**→ Уточняющий вопрос:** Как правильно сделать deep copy объекта с вложенными mutable-объектами? Какие альтернативы `clone()` существуют?

**↳ Ответ:** Три подхода: 1) Копирующий конструктор — явно копируем каждое поле, вложенные mutable-объекты рекурсивно. 2) MapStruct/ModelMapper — генерирует код копирования через маппер. 3) Сериализация в байты и обратно (медленно, требует Serializable). `clone()` с Cloneable — антипаттерн по Блоху: broken design, final-поля не работают, требует перехвата Exception. Правило: копирующий конструктор — всегда лучшее решение.

---

## 10. Когда использовать Strategy?

Strategy — поведенческий паттерн, выделяющий семейство взаимозаменяемых алгоритмов в отдельные классы с общим интерфейсом. Контекст делегирует выполнение алгоритма объекту-стратегии, который можно подменить в рантайме. Цель: избавиться от разветвлённых `if/else` или `switch`, где каждая ветка — отдельный алгоритм.

```java
// Интерфейс стратегии
@FunctionalInterface
public interface DiscountStrategy {
    BigDecimal apply(BigDecimal price);
}

// Контекст
@Service
public class PricingService {
    private final Map<String, DiscountStrategy> strategies;

    public PricingService() {
        strategies = new HashMap<>();
        strategies.put("REGULAR",    price -> price);
        strategies.put("VIP",        price -> price.multiply(new BigDecimal("0.8")));
        strategies.put("EMPLOYEE",   price -> price.multiply(new BigDecimal("0.5")));
        strategies.put("PROMO_CODE", price -> price.subtract(new BigDecimal("100")));
    }

    public BigDecimal calculatePrice(BigDecimal basePrice, String customerType) {
        return strategies
            .getOrDefault(customerType, price -> price)
            .apply(basePrice);
    }
}
```

В Java Strategy повсюду: `Comparator<T>` — это Strategy для алгоритма сравнения (передаётся в `Collections.sort()`, `Stream.sorted()`). `ExecutorService` с разными реализациями — Strategy для управления потоками. Spring Security использует Strategy в `AuthenticationProvider`, `AccessDecisionStrategy`. В Java 8+ стратегии элегантно реализуются через лямбды, т.к. они реализуют функциональные интерфейсы.

❗ Ловушка: «Чем Strategy отличается от простого `if/else`?» Strategy не просто убирает ветвление — она открыта к расширению (Open/Closed Principle): добавить новую стратегию можно без изменения контекста. Второй вопрос: «Чем Strategy отличается от State?» — разбор в вопросе 13.

**→ Уточняющий вопрос:** Как Strategy связана с Open/Closed Principle? Покажи пример, где замена Strategy на `if/else` была бы плохим решением.

**↳ Ответ:** Strategy реализует OCP: добавить новый алгоритм = добавить новый класс, не трогая существующий код. Пример: система скидок с новым типом `LOYALTY_CARD` — со Strategy: добавить один класс. Без Strategy: модифицировать существующий switch, рискуя сломать другие ветки. Критично в бизнес-логике, где требования меняются часто — каждое изменение if/else требует регрессионного тестирования всей ветки.

---

## 11. Как реализован Observer в Java?

Observer (Наблюдатель) — поведенческий паттерн, в котором субъект (publisher/subject) хранит список подписчиков (observers/listeners) и автоматически уведомляет их при изменении своего состояния. Реализует слабую связанность: субъект знает только об интерфейсе наблюдателя, а не о конкретных классах.

В Spring Events — это Observer в корпоративном исполнении:

```java
// Событие (message)
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;
    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
    public Order getOrder() { return order; }
}

// Субъект (publisher)
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public Order createOrder(OrderRequest request) {
        Order order = processOrder(request);
        // публикация события — уведомление всех наблюдателей
        eventPublisher.publishEvent(new OrderCreatedEvent(this, order));
        return order;
    }
}

// Наблюдатели (subscribers) — можно добавлять без изменения OrderService
@Component
public class EmailNotificationListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        sendEmail(event.getOrder().getCustomerEmail());
    }
}

@Component
public class InventoryListener {
    @EventListener
    @Async // асинхронное уведомление
    public void onOrderCreated(OrderCreatedEvent event) {
        reserveInventory(event.getOrder());
    }
}
```

В стандартном JDK: `PropertyChangeListener`/`PropertyChangeSupport` в JavaBeans, `java.util.EventListener` в Swing, `Flow.Publisher`/`Flow.Subscriber` (Java 9+, реализация Reactive Streams). Старые `java.util.Observer`/`java.util.Observable` были deprecated в Java 9 из-за ограниченного дизайна (Observable — класс, не интерфейс; нет типизации событий).

❗ Ловушка: «Какие проблемы есть у Observer?» Memory leak через не-отписавшихся наблюдателей (субъект держит сильную ссылку); порядок уведомлений не определён; при синхронном уведомлении медленный observer блокирует всех. В Spring `@EventListener` + `@Async` решает проблему блокировки.

**→ Уточняющий вопрос:** Чем отличается Observer от Pub/Sub? В Observer субъект знает о наблюдателях — как это устраняется в Pub/Sub (например, через брокер событий)?

**↳ Ответ:** Observer: субъект хранит список наблюдателей напрямую → прямая связь. Pub/Sub: между издателем и подписчиком стоит брокер (Kafka, RabbitMQ, Event Bus) → издатель не знает о подписчиках совсем. Spring Events — это Observer; Kafka — Pub/Sub. Преимущество Pub/Sub: полная развязка, масштабируется на несколько сервисов. Цена: дополнительная инфраструктура и сложность трассировки.

---

## 12. В чём преимущество Decorator перед наследованием?

Decorator — структурный паттерн, добавляющий объекту новое поведение динамически путём его оборачивания в объект-обёртку, реализующий тот же интерфейс. Ключевое отличие от наследования: поведение добавляется **в рантайме**, а не фиксируется **в compile-time**; обёртки можно комбинировать произвольно.

Проблема наследования: если есть `InputStream` и нужны варианты с буферизацией, шифрованием и сжатием, наследование породит комбинаторный взрыв классов (`BufferedEncryptedCompressedInputStream`). Decorator решает это элегантно:

```java
// Интерфейс
interface TextProcessor {
    String process(String text);
}

// Базовая реализация
class PlainTextProcessor implements TextProcessor {
    public String process(String text) { return text; }
}

// Декораторы
class TrimDecorator implements TextProcessor {
    private final TextProcessor delegate;
    TrimDecorator(TextProcessor delegate) { this.delegate = delegate; }
    public String process(String text) { return delegate.process(text.trim()); }
}

class UpperCaseDecorator implements TextProcessor {
    private final TextProcessor delegate;
    UpperCaseDecorator(TextProcessor delegate) { this.delegate = delegate; }
    public String process(String text) { return delegate.process(text).toUpperCase(); }
}

class HtmlEscapeDecorator implements TextProcessor {
    private final TextProcessor delegate;
    HtmlEscapeDecorator(TextProcessor delegate) { this.delegate = delegate; }
    public String process(String text) { return delegate.process(escapeHtml(text)); }
}

// Динамическая комбинация в рантайме
TextProcessor processor = new UpperCaseDecorator(
    new TrimDecorator(
        new HtmlEscapeDecorator(new PlainTextProcessor())
    )
);
```

В JDK Decorator — это весь пакет `java.io`: `new BufferedReader(new InputStreamReader(new FileInputStream("file.txt")))`. Каждый класс оборачивает предыдущий и добавляет поведение. Spring Security использует Decorator в цепочке фильтров: каждый `Filter` оборачивает следующий.

❗ Ловушка: «Decorator и Proxy — в чём разница?» Структурно они идентичны (оба оборачивают объект с тем же интерфейсом). Разница в намерении: **Decorator добавляет поведение** (расширяет функциональность), **Proxy контролирует доступ** (добавляет прослойку: ленивая загрузка, логирование, security-проверки) без изменения функциональности.

**→ Уточняющий вопрос:** Как Spring AOP соотносится с паттерном Decorator? Чем AOP-прокси отличается от ручного Decorator?

**↳ Ответ:** Spring AOP — это Decorator реализованный через динамический прокси: добавляет поведение (транзакция, кэш, логирование) без изменения целевого объекта. Разница от ручного Decorator: AOP-прокси создаётся автоматически по аннотациям/pointcut, не требует кода. Ручной Decorator — явный, compile-time, прозрачный в debug. AOP — implicit, runtime, сложнее отлаживать (стек трейс через прокси). AOP не работает при self-invocation; ручной Decorator — всегда.

---

## 13. В чём разница между State и Strategy?

State и Strategy структурно идентичны: оба инкапсулируют поведение в отдельный объект и делегируют ему вызовы. Разница — в **намерении и семантике**.

**Strategy**: клиент **выбирает** алгоритм и **явно устанавливает** его. Стратегии независимы друг от друга, не знают о существовании других стратегий, не управляют переключением. Алгоритм — это просто вариант выполнения задачи.

**State**: объект **сам меняет** своё поведение при изменении **внутреннего состояния**. Состояния часто знают друг о друге и сами инициируют переходы. Контекст не знает, как именно происходит переход — это ответственность состояний.

```java
// State: заказ меняет поведение в зависимости от своего статуса
interface OrderState {
    void pay(Order order);
    void ship(Order order);
    void cancel(Order order);
}

class NewOrderState implements OrderState {
    public void pay(Order order) {
        processPayment(order);
        order.setState(new PaidOrderState()); // само инициирует переход
    }
    public void ship(Order order)   { throw new IllegalStateException("Сначала оплатите"); }
    public void cancel(Order order) { order.setState(new CancelledOrderState()); }
}

class PaidOrderState implements OrderState {
    public void pay(Order order)    { throw new IllegalStateException("Уже оплачено"); }
    public void ship(Order order)   {
        shipOrder(order);
        order.setState(new ShippedOrderState()); // переход к следующему состоянию
    }
    public void cancel(Order order) { refund(order); order.setState(new CancelledOrderState()); }
}

class Order {
    private OrderState state = new NewOrderState();
    void setState(OrderState state) { this.state = state; }
    void pay()    { state.pay(this); }
    void ship()   { state.ship(this); }
    void cancel() { state.cancel(this); }
}
```

В Spring State Machine (spring-statemachine) — библиотека, реализующая паттерн State для сложных бизнес-процессов (статусы заказа, согласование документов, workflow).

❗ Ловушка: «Если структура одинакова, зачем вообще различать?» Ответ: потому что они решают разные проблемы. Strategy — про выбор алгоритма (сортировки, оплаты, отчёта). State — про управление жизненным циклом объекта (заказ, соединение, документ). Путаница приводит к неправильному распределению ответственности: в Strategy состояния не должны знать друг о друге, в State — могут и должны.

**→ Уточняющий вопрос:** Как State паттерн помогает избежать разрастания `if/else` в бизнес-логике? Покажи, как выглядел бы тот же код без State.

**↳ Ответ:** Без State: `void pay(Order o) { if (o.status == NEW) { ... } else if (o.status == PAID) { throw... } }` — и так для каждого метода (pay, ship, cancel). Добавить новый статус = модифицировать каждый метод. Со State: каждый статус — класс с нужным поведением, добавить статус = добавить один класс. Количество `if/else` в клиентском коде = 0. Правило: если поведение объекта зависит от его статуса и статусов много — State.

---

## 14. Какие типы Proxy существуют?

Proxy — структурный паттерн, предоставляющий суррогатный объект, контролирующий доступ к реальному объекту. Прокси реализует тот же интерфейс и перехватывает вызовы, добавляя логику до/после/вместо вызова. GoF выделяет несколько видов по назначению.

**Virtual Proxy** — ленивая загрузка дорогого объекта (создаётся только при первом обращении). Hibernate `PersistentBag` и lazy-loaded сущности — это Virtual Proxy: объект выглядит загруженным, но реальные данные достаются из БД только при обращении.

**Protection Proxy** — контроль доступа. Spring Security создаёт прокси вокруг бинов с `@PreAuthorize`: вызов перехватывается, проверяются права, и только потом делегируется реальному объекту.

**Remote Proxy** — представляет объект в другом адресном пространстве (RMI, gRPC-стабы).

В Java два механизма создания прокси в рантайме:

```java
// JDK Dynamic Proxy — только по интерфейсу
UserService userService = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class[]{UserService.class},
    (proxy, method, args) -> {
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(realUserService, args);
        System.out.println("After: " + method.getName());
        return result;
    }
);

// CGLIB — по классу (без интерфейса), создаёт подкласс
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserServiceImpl.class);
enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
    System.out.println("CGLIB before: " + method.getName());
    return proxy.invokeSuper(obj, args);
});
UserServiceImpl proxiedService = (UserServiceImpl) enhancer.create();
```

Spring AOP использует JDK Dynamic Proxy если бин реализует интерфейс, и CGLIB если нет. `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize` — все реализованы через Proxy. Поэтому `@Transactional` не работает при вызове метода из того же класса (self-invocation): вызов идёт на `this`, минуя прокси.

❗ Самая горячая ловушка: **self-invocation и @Transactional**. `methodA()` вызывает `methodB()` в том же бине — `@Transactional` на `methodB()` не сработает. Интервьюер обязательно спросит «почему?» — потому что вызов идёт на реальный объект, а не на прокси.

**→ Уточняющий вопрос:** Как решить проблему self-invocation в Spring? Назови хотя бы два способа.

**↳ Ответ:** 1) Вынести метод в отдельный бин — лучшее решение, улучшает дизайн. 2) Self-injection: `@Autowired private MyService self` — Spring внедрит прокси, вызовы через `self.method()` пойдут через прокси. 3) `AopContext.currentProxy()` с `exposeProxy=true` — антипаттерн, код осведомлён о своём прокси. 4) AspectJ compile-time weaving — устраняет ограничение прокси полностью. Правило: первый вариант — всегда лучший архитектурный выбор.

---

## 15. Что такое Iterator pattern?

Iterator — поведенческий паттерн, предоставляющий **единый интерфейс для последовательного обхода** элементов коллекции без раскрытия её внутреннего устройства. Клиент не знает, является ли коллекция массивом, деревом, связным списком или результатом SQL-запроса — интерфейс обхода одинаков.

В Java Iterator встроен в язык на уровне интерфейсов и синтаксиса:

```java
// Интерфейс Iterator<E>: hasNext(), next(), remove()
// Интерфейс Iterable<E>: iterator() — позволяет использовать for-each

// Кастомный Iterator для бинарного дерева (обход in-order)
public class BinaryTree<T extends Comparable<T>> implements Iterable<T> {
    private Node<T> root;

    @Override
    public Iterator<T> iterator() {
        return new InOrderIterator();
    }

    private class InOrderIterator implements Iterator<T> {
        private final Deque<Node<T>> stack = new ArrayDeque<>();

        InOrderIterator() { pushLeft(root); }

        private void pushLeft(Node<T> node) {
            while (node != null) { stack.push(node); node = node.left; }
        }

        @Override
        public boolean hasNext() { return !stack.isEmpty(); }

        @Override
        public T next() {
            if (!hasNext()) throw new NoSuchElementException();
            Node<T> node = stack.pop();
            pushLeft(node.right);
            return node.value;
        }
    }
}

// Теперь дерево работает в for-each
BinaryTree<Integer> tree = new BinaryTree<>();
for (int value : tree) {
    System.out.println(value); // in-order обход
}
```

В реальном коде итераторы везде: `ResultSet` в JDBC — это Iterator над строками БД; `Stream<T>` в Java 8+ — «ленивый итератор» с функциональными операциями; `Scanner` — Iterator по токенам; `DirectoryStream<Path>` в NIO.2. Реактивные стримы (`Flow.Publisher`) можно рассматривать как push-based Iterator.

❗ Ловушка: «Чем Iterator отличается от for-each по массиву?» For-each по массиву компилируется в обычный цикл с индексом. For-each по `Iterable` компилируется в вызов `iterator()`, `hasNext()`, `next()`. Вторая ловушка: `ConcurrentModificationException` — если изменить коллекцию во время итерации (не через `iterator.remove()`), итератор это обнаружит через `modCount` и выбросит исключение.

**→ Уточняющий вопрос:** Почему `ConcurrentModificationException` называется best-effort? Гарантирует ли он, что обнаружит все concurrent modifications?

**↳ Ответ:** CME называется best-effort потому что Javadoc явно предупреждает: обнаружение не гарантировано. Механизм — `modCount` без синхронизации: в многопоточном коде другой поток может изменить коллекцию так, что modCount случайно совпадёт. В однопоточном коде при удалении предпоследнего элемента ArrayList CME тоже не возникает (итерация завершится без next()). Вывод: CME — отладочный сигнал, а не механизм защиты от concurrent access.

---

## 16. Какие антипаттерны вы знаете?

Антипаттерн — это типичное «решение», которое кажется логичным, но на практике создаёт больше проблем, чем решает. Знание антипаттернов не менее важно, чем знание паттернов: именно их вы будете видеть в legacy-коде и именно от них вас будут просить «отрефакторить».

**God Object / God Class** — класс, знающий слишком много и делающий слишком много. Нарушает SRP. Признак: `UserManager`, `SystemHelper`, `ApplicationUtils` с тысячами строк. В Spring: `@Service` с 50 методами и зависимостями от 15 других сервисов.

**Anemic Domain Model** — доменные объекты содержат только данные (геттеры/сеттеры), вся бизнес-логика вынесена в сервисы. Мартин Фаулер считает это антипаттерном: теряется смысл ООП, бизнес-правила размываются по сервисам. Характерно для типичного Spring CRUD с `@Entity` + `@Service`.

```java
// Anemic — плохо: логика в сервисе, объект — мешок данных
public class Order { private BigDecimal total; /* только геттеры/сеттеры */ }
public class OrderService {
    public void applyDiscount(Order order, double pct) {
        order.setTotal(order.getTotal().multiply(BigDecimal.valueOf(1 - pct)));
    }
}

// Rich Domain Model — лучше: логика в объекте
public class Order {
    private BigDecimal total;
    public void applyDiscount(double percentage) {
        if (percentage < 0 || percentage > 1) throw new IllegalArgumentException();
        this.total = total.multiply(BigDecimal.valueOf(1 - percentage));
    }
}
```

**Другие важные антипаттерны:**
- **Golden Hammer** — «у меня есть молоток, значит всё — гвоздь». Применение любимого инструмента/паттерна везде (NoSQL для всего, микросервисы для CRUD-приложения).
- **Premature Optimization** — «корень всех зол» (Кнут). Оптимизация до измерений и профилирования.
- **Magic Numbers/Strings** — `if (status == 3)` вместо `if (status == Status.SHIPPED)`.
- **Spaghetti Code** — запутанный поток управления без чёткой структуры.
- **Copy-Paste Programming** — дублирование логики вместо абстракции.
- **Service Locator** — антипаттерн в контексте DI: компонент сам достаёт зависимости из глобального реестра (`ApplicationContext.getBean()`), скрывая зависимости.

❗ На собесе часто спрашивают про **Anemic Domain Model** в контексте Spring: «Ваши Entity — anemic?» Это дискуссионный вопрос. Правильный ответ — зависит от сложности домена. Для простого CRUD это нормально; для сложной бизнес-логики anemic model ведёт к процедурному коду, размазанному по сервисам. Также спросят про **Service Locator vs DI** — почему DI лучше.

**→ Уточняющий вопрос:** Как отличить Singleton-паттерн от Singleton-антипаттерна? Когда Singleton оправдан, а когда он — признак плохого дизайна?

**↳ Ответ:** Singleton оправдан когда объект действительно один по природе: пул соединений, конфигурация, реестр. Антипаттерн — когда используется для «удобного глобального доступа» к изменяемому состоянию: скрытая зависимость, нарушает тестируемость. Тест: «могу ли я заменить этот Singleton моком в unit-тесте без изменения кода?» — если нет, это антипаттерн. DI решает задачу лучше: та же единственность экземпляра, но с явными зависимостями и тестируемостью.
