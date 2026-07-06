# Spring / Spring Boot

> 📇 Справочник уровня middle. Формат: ответ по сути → как Spring реализует внутри → пример из реального приложения → ❗ ловушка на собесе.

**Всего вопросов: 29**

---

## 1. Что такое Dependency Injection?

Dependency Injection (DI) — паттерн проектирования, при котором объект не создаёт свои зависимости сам, а получает их извне. Класс объявляет, что ему нужно, а контейнер (Spring ApplicationContext) решает, как это предоставить. Это инверсия управления (IoC): контроль над созданием зависимостей передаётся от класса к контейнеру.

Внутри Spring реализует DI через BeanFactory/ApplicationContext: при старте контекста сканирует классы, строит граф зависимостей (BeanDefinition), инстанцирует бины в правильном порядке и связывает их между собой. Если зависимость не найдена или есть цикл — приложение упадёт на старте, а не в рантайме.

В реальном Spring Boot REST-сервисе это выглядит так: `OrderController` зависит от `OrderService`, тот — от `OrderRepository`. Ни один из них не создаёт другой через `new`. Spring сам выстраивает цепочку при запуске.

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) { // Spring внедрит сюда бин
        this.orderService = orderService;
    }

    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id);
    }
}
```

❗ На собесе часто спрашивают: "чем DI отличается от IoC?" — IoC — это принцип (передать контроль контейнеру), DI — конкретная реализация этого принципа. Путаница между ними сразу выдаёт поверхностное понимание.

**→ Уточняющий вопрос:** Какие типы DI поддерживает Spring и какой из них предпочтительнее?

**↳ Ответ:** Три типа: constructor injection (рекомендован), setter injection (для опциональных зависимостей), field injection через `@Autowired` на поле (антипаттерн). Constructor injection — предпочтительный: поля могут быть `final`, объект всегда полностью инициализирован, unit-тест пишется как `new MyService(mockDep)` без Spring-контекста. Field injection скрывает зависимости, ломает возможность финализации и делает класс тяжелотестируемым.

---

## 2. В чём разница между constructor, setter и field injection?

**Constructor injection** — зависимости передаются через конструктор. Поля можно сделать `final`, зависимости гарантированно не null, объект всегда полностью инициализирован. Это рекомендуемый способ.

**Setter injection** — Spring вызывает сеттер после создания объекта. Подходит для опциональных зависимостей (можно не внедрять). Минус: поле не может быть `final`, и объект может быть частично инициализированным.

**Field injection** — `@Autowired` прямо на поле. Spring устанавливает значение через рефлексию, обходя инкапсуляцию. Выглядит компактно, но скрывает зависимости класса, делает класс непригодным для unit-тестов без Spring-контекста, и поле тоже не может быть `final`.

```java
// Constructor injection — предпочтительно
@Service
public class PaymentService {
    private final PaymentRepository paymentRepository;
    private final NotificationService notificationService;

    public PaymentService(PaymentRepository paymentRepository,
                          NotificationService notificationService) {
        this.paymentRepository = paymentRepository;
        this.notificationService = notificationService;
    }
}

// Field injection — антипаттерн
@Service
public class PaymentService {
    @Autowired
    private PaymentRepository paymentRepository; // нельзя final, сложно тестировать
}
```

❗ На собесе могут показать класс с `@Autowired` на поле и спросить "что здесь не так". Ответ: нельзя написать unit-тест без `@SpringBootTest` или Mockito `@InjectMocks` с рефлексией, поле не final, зависимости неочевидны из публичного API класса.

**→ Уточняющий вопрос:** Как Spring тестирует constructor injection без поднятия контекста?

**↳ Ответ:** Никак — это и есть преимущество. При constructor injection зависимости — обычные параметры конструктора. Тест создаёт объект напрямую: `new OrderService(Mockito.mock(OrderRepository.class))`. Spring-контекст не нужен. Это делает тест быстрым (нет загрузки контекста) и изолированным. Только field injection требует `@SpringBootTest` или `@InjectMocks` с рефлексией для внедрения моков.

---

## 3. Какой тип injection рекомендуется использовать и почему?

Constructor injection — официальная рекомендация Spring-команды и лучшая практика индустрии. Причин несколько: зависимости обязательны и проверяются на старте, поля `final` дают иммутабельность и thread safety, для unit-тестов достаточно передать моки через конструктор без поднятия Spring-контекста.

Ещё одно важное свойство: Spring обнаруживает циклические зависимости при constructor injection на этапе старта и выбрасывает исключение. При field injection цикл не обнаруживается сразу — это скрытая проблема. Начиная со Spring Boot 2.6 циклические зависимости запрещены по умолчанию именно для того, чтобы принудить к правильному дизайну.

```java
// Чистый unit-тест без Spring — возможен только с constructor injection
class OrderServiceTest {
    @Test
    void shouldReturnOrder() {
        OrderRepository mockRepo = Mockito.mock(OrderRepository.class);
        OrderService service = new OrderService(mockRepo); // просто new!
        // ...
    }
}
```

❗ Если в проекте используется Lombok, `@RequiredArgsConstructor` автоматически генерирует конструктор для всех `final`-полей — это идиоматичная связка с constructor injection.

**→ Уточняющий вопрос:** Как Spring 2.6+ обрабатывает циклические зависимости, и как их исправить?

**↳ Ответ:** Spring Boot 2.6+ по умолчанию запрещает циклические зависимости (бросает BeanCurrentlyInCreationException на старте). Раньше Spring разрывал цикл через трёхфазное создание (создание → заполнение → инициализация), но это работало только с setter/field injection. Правильное исправление: пересмотреть дизайн — выделить третий сервис с общей логикой. Обходной путь: `spring.main.allow-circular-references=true` — но это признак архитектурной проблемы, не решение.

---

## 4. Что такое Bean в Spring?

Bean — это объект, жизненным циклом которого управляет Spring IoC-контейнер. Контейнер создаёт бин, внедряет его зависимости, вызывает методы инициализации, предоставляет его другим бинам, и в конце уничтожает. Без контейнера это обычный Java-объект.

Каждый бин описывается через `BeanDefinition` — метаданные: класс, scope, имя, зависимости, методы init/destroy. При создании ApplicationContext Spring читает все `BeanDefinition`, строит граф зависимостей и инстанцирует синглтон-бины (eagerly, если не `@Lazy`). По умолчанию имя бина = имя класса с маленькой буквы, или имя метода для `@Bean`.

В типичном Spring Boot приложении бинами являются все классы с `@Service`, `@Repository`, `@Controller`, `@Component`, а также объекты, объявленные через `@Bean` в конфигурационных классах (DataSource, RestTemplate, ObjectMapper и т.д.).

❗ Частый вопрос: "Все ли Spring-объекты — бины?" Нет. Сущности JPA (`@Entity`), DTO, вспомогательные POJO создаются через `new` и Spring ими не управляет. Путаница между managed bean и обычным объектом — типичная ошибка.

**→ Уточняющий вопрос:** Что такое BeanDefinition и где Spring его хранит?

**↳ Ответ:** `BeanDefinition` — метаданные о бине: имя, класс, scope, являются ли lazy, зависимости, методы init/destroy, аргументы конструктора. Хранится в `BeanDefinitionRegistry` (реализация — `DefaultListableBeanFactory`). Можно получить программно: `context.getBeanFactory().getBeanDefinition("myBean")`. Это "чертёж" объекта — Spring читает его при запросе бина и решает как создать экземпляр. `BeanFactoryPostProcessor` может изменять `BeanDefinition` до создания объектов.

---

## 5. Как создать Bean в Spring?

Три основных способа. **Аннотации на классе**: `@Component` и его специализации (`@Service`, `@Repository`, `@Controller`, `@RestController`) — Spring находит их через component scan и регистрирует как бины.

**Метод с `@Bean` в `@Configuration`-классе** — явное объявление, полный контроль над созданием. Используется для сторонних классов, которые нельзя аннотировать, или когда нужна сложная логика создания.

**Programmatic registration** — через `BeanDefinitionRegistry` или `ApplicationContext.registerBean()` (Spring 5+) — редко, для динамической регистрации.

```java
// Способ 1: аннотация на классе
@Service
public class UserService { ... }

// Способ 2: @Bean в конфигурации (для сторонних классов)
@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate rt = new RestTemplate();
        rt.setConnectTimeout(Duration.ofSeconds(5));
        return rt;
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}
```

❗ Распространённая ошибка: попытаться аннотировать `@Component` класс из сторонней библиотеки (например, библиотечный клиент). Исходник недоступен — нужен `@Bean` в конфигурации. Или забыть, что component scan работает только в пакете приложения и его подпакетах.

**→ Уточняющий вопрос:** Чем отличается `@Bean` в `@Configuration` от `@Bean` в `@Component`-классе?

**↳ Ответ:** В `@Configuration` (full mode) CGLIB проксирует класс — повторный вызов `@Bean`-метода изнутри возвращает тот же singleton из контейнера. В `@Component` (lite mode) прокси нет — вызов `@Bean`-метода создаёт новый объект каждый раз, как обычный Java-вызов. Это приводит к тому, что если два `@Bean`-метода в `@Component` вызывают третий — получают разные экземпляры. Правило: для конфигурации используй `@Configuration`, не `@Component`.

---

## 6. Что такое Bean Lifecycle?

Bean Lifecycle — это полный путь бина от создания до уничтожения под управлением Spring-контейнера. Контейнер не просто создаёт объект через `new`, а проводит его через цепочку шагов: создание, внедрение зависимостей, инициализация, использование, уничтожение. На каждом этапе можно встроить свой код.

Понимание lifecycle важно для правильной инициализации ресурсов (подключение к БД, прогрев кэша в `@PostConstruct`), корректного освобождения ресурсов (закрытие соединений в `@PreDestroy`), и понимания, почему prototype-бины ведут себя иначе чем singleton.

```java
@Service
public class CacheWarmupService {

    private final UserRepository userRepository;
    private Map<Long, User> cache;

    public CacheWarmupService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @PostConstruct
    public void init() {
        // Выполняется ПОСЛЕ внедрения зависимостей
        // userRepository здесь уже доступен
        cache = userRepository.findAll()
            .stream()
            .collect(Collectors.toMap(User::getId, u -> u));
    }

    @PreDestroy
    public void cleanup() {
        cache.clear();
    }
}
```

❗ Нельзя обращаться к другим бинам в конструкторе, если их DI ещё не завершён. Это причина, почему инициализационную логику выносят в `@PostConstruct`, а не в конструктор.

**→ Уточняющий вопрос:** В какой момент lifecycle создаётся AOP-прокси для бина?

**↳ Ответ:** На шаге `BeanPostProcessor.postProcessAfterInitialization()` — уже после `@PostConstruct` и `afterPropertiesSet`. Создаёт прокси `AnnotationAwareAspectJAutoProxyCreator`. Следствие: в `@PostConstruct` вы работаете с оригинальным объектом (не прокси) — вызовы внутри `@PostConstruct` не идут через AOP. Контейнер затем сохраняет прокси вместо оригинала, и внешние пользователи бина получают прокси.

---

## 7. Какие этапы жизненного цикла Bean?

Полная последовательность для singleton-бина:

1. **Instantiation** — Spring создаёт объект через конструктор
2. **Populate Properties** — внедряются зависимости (DI)
3. **Aware-интерфейсы** — вызываются `BeanNameAware.setBeanName()`, `BeanFactoryAware.setBeanFactory()`, `ApplicationContextAware.setApplicationContext()`
4. **BeanPostProcessor.postProcessBeforeInitialization()** — хуки до инициализации
5. **@PostConstruct** / `InitializingBean.afterPropertiesSet()` / `init-method`
6. **BeanPostProcessor.postProcessAfterInitialization()** — здесь создаётся AOP-прокси
7. **Бин готов** — используется приложением
8. **@PreDestroy** / `DisposableBean.destroy()` / `destroy-method` — при закрытии контекста

Ключевой момент: AOP-прокси создаётся на шаге 6, то есть уже после всей инициализации. Это объясняет, почему в `@PostConstruct` вы работаете с реальным объектом, а не прокси.

```java
@Component
public class LifecycleBean implements BeanNameAware, InitializingBean, DisposableBean {

    @Override
    public void setBeanName(String name) {
        System.out.println("Step 3: BeanNameAware, name=" + name);
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("Step 5: @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("Step 5b: InitializingBean");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("Step 8: @PreDestroy");
    }
}
```

❗ На собесе часто путают порядок `@PostConstruct` и `BeanPostProcessor`. BPP работает со всеми бинами в контексте, `@PostConstruct` — только с конкретным бином. Если бин является `BeanPostProcessor`, он создаётся раньше остальных.

**→ Уточняющий вопрос:** Почему BeanPostProcessor-бины создаются раньше обычных бинов?

**↳ Ответ:** Потому что BeanPostProcessor должен быть готов до того, как начнут создаваться обычные бины, которые он должен обрабатывать. Spring специально поднимает BPP-бины в отдельной фазе до основного refresh. Побочный эффект: у BPP нет возможности получить обработку от других BPP (они ещё не созданы). Предупреждение в логах "is not eligible for getting processed by all BeanPostProcessors" означает именно это.

---

## 8. Что такое BeanPostProcessor?

`BeanPostProcessor` — интерфейс с двумя методами: `postProcessBeforeInitialization()` и `postProcessAfterInitialization()`. Реализуя его, можно вмешиваться в процесс инициализации любого бина в контексте — модифицировать объект или заменять его другим (например, прокси).

На BeanPostProcessor построена большая часть магии Spring: `AutowiredAnnotationBeanPostProcessor` обрабатывает `@Autowired`, `CommonAnnotationBeanPostProcessor` обрабатывает `@PostConstruct`/`@PreDestroy`, `AnnotationAwareAspectJAutoProxyCreator` создаёт AOP-прокси после инициализации.

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before init: " + beanName);
        return bean; // вернуть тот же объект или другой
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("After init: " + beanName + " -> " + bean.getClass().getSimpleName());
        // Здесь AOP-прокси уже может подменить оригинальный объект
        return bean;
    }
}
```

❗ Распространённая ошибка: BeanPostProcessor-бины не могут иметь зависимости от обычных бинов через `@Autowired`, потому что BPP создаются раньше остальных. Spring предупредит об этом в логах ("is not eligible for getting processed by all BeanPostProcessors").

**→ Уточняющий вопрос:** Чем BeanPostProcessor отличается от BeanFactoryPostProcessor?

**↳ Ответ:** `BeanFactoryPostProcessor` (BFPP) вызывается до создания любых бинов — он работает с `BeanDefinition`-метаданными (может изменять scope, свойства, добавлять/удалять определения). Пример: `PropertySourcesPlaceholderConfigurer` — подставляет `${property}` значения из application.properties. `BeanPostProcessor` вызывается уже после создания каждого бина — работает с готовым объектом, может подменить его прокси. Правило: BFPP — до создания (чертёж), BPP — после создания (объект).

---

## 9. Что делают методы с аннотацией @PostConstruct и @PreDestroy?

`@PostConstruct` — метод, который Spring вызывает после создания бина и внедрения всех зависимостей, но до того, как бин станет доступен другим компонентам. Идеален для инициализации: подключение, прогрев кэша, валидация конфигурации. В момент вызова все `@Autowired`-поля и конструктор уже отработали.

`@PreDestroy` — метод, вызываемый перед уничтожением бина при закрытии ApplicationContext. Используется для освобождения ресурсов: закрытие соединений, flush буферов, отписка от событий.

Обе аннотации из пакета `jakarta.annotation` (JSR-250), то есть стандарт Java EE, а не Spring-специфика. Spring их поддерживает через `CommonAnnotationBeanPostProcessor`.

```java
@Service
public class KafkaConsumerService {

    private KafkaConsumer<String, String> consumer;

    @PostConstruct
    public void start() {
        consumer = new KafkaConsumer<>(buildConfig());
        consumer.subscribe(List.of("orders-topic"));
        // запуск опроса в отдельном потоке
    }

    @PreDestroy
    public void stop() {
        if (consumer != null) {
            consumer.close(Duration.ofSeconds(5)); // graceful shutdown
        }
    }
}
```

❗ `@PreDestroy` НЕ вызывается для prototype-бинов — Spring не отслеживает их после создания. Также не вызывается при аварийном завершении JVM (`kill -9`). Для надёжного cleanup регистрируйте shutdown hook или используйте `@Bean(destroyMethod = "close")`.

**→ Уточняющий вопрос:** Что произойдёт, если выбросить исключение в методе @PostConstruct?

**↳ Ответ:** Приложение не запустится. `@PostConstruct` вызывается в фазе инициализации бина — исключение распространяется вверх и вызывает `BeanCreationException`, что прерывает refresh ApplicationContext. Spring Boot завершит процесс с ненулевым кодом выхода. Это желательное поведение: если инициализация провалилась (например, не удалось подключиться к БД), лучше упасть при старте, чем работать в сломанном состоянии.

---

## 10. Что такое scope Bean?

Scope определяет, сколько экземпляров бина создаёт Spring и как долго они живут. Это мета-информация в BeanDefinition, которая влияет на стратегию получения бина из контейнера. При каждом запросе бина Spring проверяет scope и либо возвращает существующий экземпляр, либо создаёт новый.

Выбор scope — важное архитектурное решение: неправильный scope может привести к утечкам памяти (session-бин живёт дольше ожидаемого), состоянию гонки (singleton с изменяемым состоянием в многопоточной среде), или неожиданному поведению (prototype внедрён в singleton — один экземпляр на весь lifecycle singleton).

```java
@Component
@Scope("prototype") // новый экземпляр при каждом запросе
public class ReportGenerator {
    private final List<String> lines = new ArrayList<>(); // безопасно — у каждого свой список

    public void addLine(String line) { lines.add(line); }
    public String build() { return String.join("\n", lines); }
}
```

❗ Забытый scope — частая причина багов: разработчик создаёт класс с изменяемым состоянием, забывая, что singleton это состояние разделяет между всеми потоками. В Spring MVC это особенно опасно для контроллеров и сервисов.

**→ Уточняющий вопрос:** Что произойдёт, если prototype-бин внедрить в singleton через @Autowired?

**↳ Ответ:** Prototype фактически становится singleton. `@Autowired` внедряет зависимость один раз при создании singleton-бина. После этого в поле живёт один и тот же экземпляр prototype навсегда — независимо от того, что scope прописан как prototype. Для получения нового экземпляра при каждом вызове нужны: `ObjectProvider<T>`, `@Lookup`-метод, или явный `applicationContext.getBean()`.

---

## 11. Какие scope существуют в Spring?

**Основные (доступны всегда):**
- `singleton` — один экземпляр на ApplicationContext (по умолчанию)
- `prototype` — новый экземпляр при каждом запросе бина из контейнера

**Web-scope (только в web ApplicationContext):**
- `request` — один экземпляр на HTTP-запрос
- `session` — один экземпляр на HTTP-сессию
- `application` — один экземпляр на ServletContext (близко к singleton, но привязан к web)
- `websocket` — один экземпляр на WebSocket-сессию

Кастомные scope можно создавать, реализовав интерфейс `Scope` и зарегистрировав через `ConfigurableBeanFactory.registerScope()`. Spring Cloud добавляет `@RefreshScope` — бин пересоздаётся при обновлении конфигурации.

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String traceId = UUID.randomUUID().toString();

    public String getTraceId() { return traceId; }
}
// proxyMode нужен, чтобы singleton-бины могли внедрять request-scoped бины
```

❗ Web-scope бины нельзя напрямую внедрить в singleton — несоответствие жизненных циклов. Нужен `proxyMode = ScopedProxyMode.TARGET_CLASS` — Spring создаст прокси, который при каждом вызове достаёт нужный экземпляр из текущего контекста (request/session).

**→ Уточняющий вопрос:** Как работает ScopedProxyMode.TARGET_CLASS и когда он нужен?

**↳ Ответ:** Spring создаёт CGLIB-прокси вместо самого бина. Singleton получает этот прокси при внедрении. При каждом вызове метода через прокси — прокси обращается к текущему контексту (request/session) и достаёт оттуда реальный экземпляр бина. Нужен всегда, когда short-lived бин (request, session) внедряется в long-lived (singleton). Без `proxyMode` — ошибка при старте или один экземпляр на весь lifecycle singleton.

---

## 12. В чём разница между singleton и prototype scope?

**Singleton** — Spring создаёт ровно один экземпляр бина на весь ApplicationContext и кэширует его. Все компоненты, запрашивающие этот бин, получают одну и ту же ссылку. Создаётся при старте контекста (eager), если не помечен `@Lazy`. Spring управляет полным lifecycle: создание, `@PostConstruct`, использование, `@PreDestroy`.

**Prototype** — при каждом запросе бина из контейнера (`context.getBean()` или при внедрении) создаётся новый экземпляр. Spring создаёт и отдаёт его, но дальше НЕ отслеживает: `@PreDestroy` не вызывается, бин не кэшируется. Ответственность за уничтожение — на вызывающем коде.

Критическая проблема: если prototype внедрить в singleton через `@Autowired`, внедрение происходит один раз при создании singleton. Prototype фактически становится singleton. Для получения нового экземпляра каждый раз нужны `ApplicationContext.getBean()`, `ObjectProvider<T>`, `@Lookup`-метод или `javax.inject.Provider<T>`.

```java
@Service
public class OrderService {

    // ObjectProvider даёт новый prototype при каждом вызове getObject()
    private final ObjectProvider<ReportGenerator> reportGeneratorProvider;

    public OrderService(ObjectProvider<ReportGenerator> reportGeneratorProvider) {
        this.reportGeneratorProvider = reportGeneratorProvider;
    }

    public String generateReport(Long orderId) {
        ReportGenerator generator = reportGeneratorProvider.getObject(); // новый экземпляр!
        generator.addLine("Order: " + orderId);
        return generator.build();
    }
}
```

❗ Это одна из самых частых ловушек на собесе. Покажите, что понимаете: если просто сделать `@Autowired` на prototype-бин в singleton — получите singleton-поведение. Нужно явно запрашивать новый экземпляр.

**→ Уточняющий вопрос:** Как работает @Lookup-аннотация для получения prototype из singleton?

**↳ Ответ:** `@Lookup` на абстрактном или обычном методе — Spring через CGLIB переопределяет этот метод: реализация вызывает `applicationContext.getBean(returnType)`. Каждый вызов метода возвращает новый prototype-экземпляр. Сигнатура метода важна: возвращаемый тип = тип prototype-бина. Плюс перед `ObjectProvider`: нет явной зависимости от ApplicationContext в коде. Минус: класс не может быть final.

---

## 13. Что такое прокси в Spring?

Прокси в Spring — это объект-обёртка, который Spring подставляет вместо реального бина. Когда другие бины обращаются к "вашему сервису", они на самом деле вызывают методы прокси. Прокси выполняет дополнительную логику (открытие транзакции, проверку прав, обращение к кэшу) и делегирует вызов реальному объекту.

Spring использует два механизма создания прокси. **JDK Dynamic Proxy** — работает только если бин реализует интерфейс; прокси реализует тот же интерфейс и перехватывает вызовы через `java.lang.reflect.InvocationHandler`. **CGLIB** — создаёт подкласс бина, переопределяя методы; работает для классов без интерфейсов. Spring Boot по умолчанию использует CGLIB для всех `@Transactional`, `@Async`, `@Cacheable` бинов.

```java
// Исходный класс
@Service
public class UserService {
    @Transactional
    public User save(User user) { ... }
}

// Что Spring создаёт (упрощённо):
class UserService$$SpringCGLIB$$0 extends UserService {
    @Override
    public User save(User user) {
        // до: открыть транзакцию
        TransactionStatus tx = txManager.getTransaction(txDef);
        try {
            User result = super.save(user); // вызов реального метода
            txManager.commit(tx);
            return result;
        } catch (RuntimeException e) {
            txManager.rollback(tx);
            throw e;
        }
    }
}
```

❗ Прокси — ключ к пониманию self-invocation. Когда метод вызывает другой метод своего же класса через `this`, он обходит прокси, и никакого транзакционного/кэшируемого поведения не будет.

**→ Уточняющий вопрос:** Почему CGLIB не может проксировать final-классы и final-методы?

**↳ Ответ:** CGLIB создаёт подкласс проксируемого класса и переопределяет его методы, вставляя перехватчики. `final`-класс не может быть наследован — CGLIB не создаст подкласс. `final`-метод не может быть переопределён — прокси вызовет оригинальный метод напрямую, минуя перехватчик. Следствие: `@Transactional`, `@Cacheable` и другие AOP-аннотации молча не работают на final-методах. Spring Boot 3 + ByteBuddy частично решает это для конкретных случаев, но правило остаётся: не делай методы с AOP-аннотациями final.

---

## 14. Когда Spring создаёт прокси?

Spring создаёт прокси для бина, когда к нему нужно применить AOP-логику. Это происходит при наличии: `@Transactional` на методе/классе, `@Async`, `@Cacheable`/`@CacheEvict`/`@CachePut`, `@Scheduled`, кастомных AOP-аспектов с `@Aspect`, Spring Security (`@PreAuthorize`, `@Secured`).

Технически это делает `AnnotationAwareAspectJAutoProxyCreator` — специальный `BeanPostProcessor`. В методе `postProcessAfterInitialization()` он проверяет, есть ли для данного бина подходящие advisors. Если есть — оборачивает бин в прокси и возвращает прокси вместо оригинального объекта. Оригинальный объект хранится внутри прокси как target.

```java
// Всё это вызовет создание прокси:

@Service
@Transactional // прокси для управления транзакциями
public class AccountService { ... }

@Service
public class EmailService {
    @Async // прокси для асинхронного выполнения
    public void sendEmail(String to) { ... }

    @Cacheable("users") // прокси для кэширования
    public User getUser(Long id) { ... }
}
```

❗ Важное следствие: если вы вызываете `context.getBean(UserService.class)` — вернётся прокси, а не оригинальный объект. Если написать `userService.getClass()` — увидите что-то вроде `UserService$$EnhancerBySpringCGLIB$$abc123`. Это нормально, но иногда удивляет при отладке.

**→ Уточняющий вопрос:** Как узнать во время выполнения, является ли бин прокси или оригинальным объектом?

**↳ Ответ:** `AopUtils.isAopProxy(bean)` — true если прокси. `AopUtils.isCglibProxy(bean)` / `isJdkDynamicProxy(bean)` — уточнить тип. `bean.getClass().getName()` — содержит `$$EnhancerBySpringCGLIB$$` для CGLIB-прокси. `AopProxyUtils.ultimateTargetClass(bean)` — получить класс оригинального объекта за всеми слоями прокси. В тестах: `AopTestUtils.getTargetObject(bean)` — получить оригинальный объект для прямых вызовов.

---

## 15. Что такое AOP (Aspect-Oriented Programming)?

AOP — парадигма программирования для вынесения сквозной функциональности (cross-cutting concerns) из бизнес-логики. Сквозная функциональность — это поведение, которое одинаково нужно многим классам: логирование, транзакции, безопасность, аудит, метрики. Без AOP этот код дублируется в каждом методе.

Spring AOP реализован поверх прокси-механизма (не байткод-манипуляции, как AspectJ). Spring создаёт прокси для бинов с аспектами и перехватывает вызовы методов. Это проще AspectJ, но имеет ограничение: работает только для Spring-бинов и только для вызовов через прокси (self-invocation не перехватывается).

```java
@Aspect
@Component
public class LoggingAspect {

    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);

    // Pointcut: все публичные методы в пакете service
    @Around("execution(public * com.example.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed(); // вызов реального метода
        long time = System.currentTimeMillis() - start;
        log.info("{} выполнился за {} мс",
            joinPoint.getSignature().getName(), time);
        return result;
    }
}
```

❗ Spring AOP работает только на уровне вызовов методов через прокси. Если нужно перехватить вызов приватного метода или вызов внутри класса — Spring AOP не поможет. Нужен AspectJ с compile-time или load-time weaving. Это ключевое ограничение, о котором спрашивают на собесе.

**→ Уточняющий вопрос:** В чём разница между Spring AOP и AspectJ?

**↳ Ответ:** Spring AOP — прокси-based, работает только для Spring-бинов, только для вызовов методов через прокси, runtime. AspectJ — полноценный AOP-фреймворк: compile-time weaving (ткёт байткод при компиляции), load-time weaving, перехватывает любые join points (поля, конструкторы, static-методы, private-методы, self-invocation). Spring AOP проще и достаточен для 95% задач (транзакции, кэш, security). AspectJ нужен когда нужно перехватить то, что Spring AOP не может.

---

## 16. Что такое аспект, advice, pointcut, join point?

**Join point** — конкретная точка в выполнении программы, где может быть применён аспект. В Spring AOP это всегда вызов метода (в AspectJ — ещё обращение к полям, конструкторы и т.д.).

**Pointcut** — предикат/выражение, выбирающее набор join points. Например: "все методы с аннотацией `@Transactional`" или "все методы в классах, заканчивающихся на `Service`". Записывается через AspectJ expression language.

**Advice** — код, выполняемый при совпадении pointcut. Типы: `@Before` (до метода), `@After` (после, всегда), `@AfterReturning` (после успешного возврата), `@AfterThrowing` (при исключении), `@Around` (полный контроль: до и после, можно изменять аргументы и результат).

**Aspect** — модуль, объединяющий pointcut и advice. В Spring — класс с `@Aspect`.

```java
@Aspect
@Component
public class AuditAspect {

    // Pointcut: методы с аннотацией @Audited
    @Pointcut("@annotation(com.example.annotation.Audited)")
    public void auditedMethods() {}

    // Advice: выполняется после успешного возврата
    @AfterReturning(pointcut = "auditedMethods()", returning = "result")
    public void audit(JoinPoint joinPoint, Object result) {
        String methodName = joinPoint.getSignature().getName();
        String user = SecurityContextHolder.getContext()
            .getAuthentication().getName();
        auditLog.save(new AuditEntry(user, methodName, result));
    }

    // Around: полный контроль — можно изменить аргументы, результат, подавить исключение
    @Around("execution(* com.example.service.*.*(..)) && @annotation(Retryable)")
    public Object retry(ProceedingJoinPoint pjp) throws Throwable {
        for (int i = 0; i < 3; i++) {
            try { return pjp.proceed(); }
            catch (TransientException e) { /* retry */ }
        }
        return pjp.proceed();
    }
}
```

❗ Путают `@After` и `@AfterReturning`: `@After` выполняется всегда (как finally), `@AfterReturning` — только при нормальном завершении без исключения. `@Around` — самый мощный, но требует вызвать `joinPoint.proceed()`, иначе реальный метод не выполнится.

**→ Уточняющий вопрос:** В каком порядке применяются несколько аспектов на одном методе?

**↳ Ответ:** Порядок неопределён без явного указания. Управляется через `@Order(n)` на аспектах: меньший номер = высший приоритет = внешний слой (выполняется первым до, последним после). Аналогия: аспекты как матрёшки — `@Order(1)` — самая внешняя. Если аспекты без `@Order` — Spring использует произвольный порядок (часто алфавитный по имени класса, но не гарантировано). Для продакшна: всегда явно ставь `@Order` на аспекты если важен порядок (например, security должен быть до logging).

---

## 17. Что делает аннотация @Transactional?

`@Transactional` помечает метод (или весь класс) как выполняемый в транзакции. Spring создаёт прокси для бина, и при вызове аннотированного метода через прокси: получает или создаёт транзакцию (с учётом propagation), выполняет метод, коммитит при успешном завершении, откатывает при исключении.

По умолчанию откат происходит только для unchecked exceptions (`RuntimeException` и `Error`). Checked exceptions по умолчанию НЕ вызывают rollback — это одна из самых частых ловушек. Настраивается через `rollbackFor` и `noRollbackFor`.

Ключевые параметры: `propagation` (как вести себя при наличии/отсутствии активной транзакции), `isolation` (уровень изоляции), `readOnly` (hint для оптимизации — Hibernate не отслеживает dirty-checking), `timeout`.

```java
@Service
@Transactional(readOnly = true) // все методы read-only по умолчанию
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;

    // Переопределяем для write-методов
    @Transactional // propagation=REQUIRED по умолчанию
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        inventoryService.reserve(order); // если выбросит RuntimeException — rollback обоих
        return order;
    }

    // readOnly=true: Hibernate пропустит dirty-checking, оптимизация
    public List<Order> findByUser(Long userId) {
        return orderRepository.findByUserId(userId);
    }

    // Checked exception НЕ вызовет rollback без rollbackFor!
    @Transactional(rollbackFor = Exception.class) // явно указываем
    public void processPayment(Long orderId) throws PaymentException {
        // ...
    }
}
```

❗ Два самых частых вопроса-ловушки: (1) checked exception не откатывает транзакцию по умолчанию; (2) self-invocation — вызов `@Transactional`-метода из того же класса не создаст транзакцию.

**→ Уточняющий вопрос:** Что означает propagation=REQUIRES_NEW и когда его использовать?

**↳ Ответ:** `REQUIRES_NEW` — приостанавливает текущую транзакцию (если есть), создаёт новую независимую транзакцию, выполняет метод, коммитит/откатывает независимо от внешней. Типичный use case: аудит-лог. Если бизнес-транзакция откатилась — запись в аудит должна всё равно сохраниться. Без `REQUIRES_NEW` аудит-запись откатится вместе с основной операцией. Важно: требует разных бинов — self-invocation всё равно не работает.

---

## 18. Почему @Transactional не работает при self-invocation?

`@Transactional` работает через AOP-прокси. Когда внешний код вызывает `orderService.createOrder()`, вызов идёт через прокси, который открывает транзакцию. Но когда `createOrder()` внутри вызывает `this.anotherMethod()` — это прямой вызов `this`, он минует прокси. Прокси-объект и оригинальный объект — разные экземпляры в памяти.

Это фундаментальное ограничение прокси-подхода Spring AOP. В отличие от AspectJ (compile-time weaving), прокси перехватывает только внешние вызовы. Даже если `anotherMethod()` помечен `@Transactional`, аннотация будет проигнорирована при вызове через `this`.

```java
@Service
public class NotificationService {

    // Внешний вызов: прокси перехватит, транзакция откроется ✅
    @Transactional
    public void sendAndSave(Notification notification) {
        save(notification); // self-invocation! вызов через this, не через прокси ❌
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void save(Notification notification) {
        // REQUIRES_NEW НЕ сработает при вызове из sendAndSave через this
        // Обе операции выполнятся в одной транзакции (или вообще без неё)
        repository.save(notification);
    }
}
```

❗ Это самая частая ловушка по Spring на собесе. Многие понимают, что self-invocation не работает, но не могут объяснить почему. Ключ: прокси и оригинальный объект — разные объекты. `this` всегда ссылается на оригинальный объект, минуя прокси.

**→ Уточняющий вопрос:** Как бы вы обнаружили эту проблему в продакшн-коде без тестов?

**↳ Ответ:** Симптомы в продакшне: данные сохраняются хотя должны были откатиться (или наоборот), метрики транзакций не совпадают с ожидаемыми. Способы обнаружения: включить `spring.jpa.show-sql=true` + `logging.level.org.springframework.transaction=TRACE` — видно когда транзакция открывается и закрывается. P6Spy или Datasource Proxy — логируют SQL с транзакционным контекстом. Код-ревью: паттерн вызова `this.method()` внутри `@Transactional` — red flag. Статический анализ: SpotBugs/ArchUnit могут поймать такие паттерны.

---

## 19. Как решить проблему с self-invocation?

Четыре основных решения, каждое со своими trade-off:

**1. Вынести метод в отдельный бин** — лучшее решение с точки зрения дизайна. Если два метода нужно вызывать независимо с разными транзакционными настройками — они, вероятно, принадлежат разным сервисам.

**2. Self-injection через `@Autowired`** — внедрить прокси самого себя. Работает, но выглядит странно и намекает на архитектурную проблему.

**3. `AopContext.currentProxy()`** — получить текущий прокси. Требует `@EnableAspectJAutoProxy(exposeProxy = true)`. Антипаттерн: код знает о своём прокси.

**4. `TransactionTemplate`** — программное управление транзакциями. Явно, надёжно, не зависит от прокси.

```java
// Решение 1: вынести в отдельный бин (рекомендуется)
@Service
public class NotificationPersistenceService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void save(Notification n) { repository.save(n); }
}

@Service
public class NotificationService {
    private final NotificationPersistenceService persistenceService;

    @Transactional
    public void sendAndSave(Notification n) {
        persistenceService.save(n); // вызов через прокси ✅
    }
}

// Решение 2: self-injection
@Service
public class NotificationService {
    @Autowired
    private NotificationService self; // Spring внедрит прокси

    @Transactional
    public void sendAndSave(Notification n) {
        self.save(n); // вызов через прокси ✅
    }
}

// Решение 3: AopContext (нужно exposeProxy = true в @EnableAspectJAutoProxy)
@Transactional
public void sendAndSave(Notification n) {
    ((NotificationService) AopContext.currentProxy()).save(n);
}
```

❗ На собесе важно не просто назвать решения, но обосновать: вынесение в отдельный бин — это единственное решение, которое улучшает дизайн. Остальные — костыли. Self-injection может вызвать проблемы при circular dependency detection в Spring Boot 2.6+.

**→ Уточняющий вопрос:** Как TransactionTemplate отличается от @Transactional по поведению?

**↳ Ответ:** `@Transactional` — декларативное управление через AOP-прокси (не работает для self-invocation, private-методов, final-классов). `TransactionTemplate` — программное: явно вызываешь `template.execute(status -> { ... })`, прокси не нужен, работает из любого контекста. Плюс `TransactionTemplate`: полный контроль, нет hidden-магии, работает там где AOP не может. Минус: более verbose. Третий вариант: `PlatformTransactionManager` напрямую — ещё ниже уровень, для сложных сценариев.

---

## 20. Что такое автоконфигурация в Spring Boot?

Автоконфигурация — механизм, позволяющий Spring Boot автоматически настраивать приложение на основе того, что находится в classpath и что уже сконфигурировано. Добавили `spring-boot-starter-data-jpa` — Spring Boot автоматически настроит DataSource, EntityManagerFactory, TransactionManager. Разработчик может переопределить любой из этих бинов.

Технически это работает через `@EnableAutoConfiguration`, которая читает файлы `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 2.7+) из всех jar-файлов в classpath. Каждая автоконфигурация — это `@Configuration`-класс с условиями `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`. Условия гарантируют, что автоконфигурация применяется только когда нужно и не переопределяет явные настройки.

```java
// Пример, как выглядит автоконфигурация внутри Spring Boot:
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, JdbcTemplate.class })
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(JdbcProperties.class)
public class JdbcTemplateAutoConfiguration {

    @Bean
    @Primary
    @ConditionalOnMissingBean(JdbcOperations.class) // создаст только если нет своего
    public JdbcTemplate jdbcTemplate(DataSource dataSource, JdbcProperties properties) {
        JdbcTemplate template = new JdbcTemplate(dataSource);
        // настройка из application.properties
        return template;
    }
}

// В вашем проекте: если объявить свой JdbcTemplate — автоконфигурация отступит
@Bean
public JdbcTemplate customJdbcTemplate(DataSource dataSource) {
    JdbcTemplate template = new JdbcTemplate(dataSource);
    template.setQueryTimeout(30);
    return template;
}
```

❗ Часто спрашивают: "Как понять, какая автоконфигурация применилась?" — запустить с `--debug` или добавить `logging.level.org.springframework.boot.autoconfigure=DEBUG`. В логах будет `CONDITIONS EVALUATION REPORT` с причинами, почему каждая автоконфигурация применилась или была пропущена.

**→ Уточняющий вопрос:** Как написать свою автоконфигурацию для переиспользования в нескольких проектах?

**↳ Ответ:** Создай `@AutoConfiguration`-класс с `@ConditionalOnClass`, `@ConditionalOnMissingBean` и т.д. Добавь его имя в `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 3) или `META-INF/spring.factories` (Boot 2). Упакуй в jar. Потребители добавляют jar в зависимости — автоконфигурация подхватится автоматически. Конвенция имени: `my-spring-boot-starter` (зависимости) + `my-spring-boot-autoconfigure` (конфигурационный код) — две раздельных артефакта.

---

## 21. Как работает @SpringBootApplication?

`@SpringBootApplication` — это мета-аннотация, объединяющая три аннотации: `@SpringBootConfiguration` (специализация `@Configuration` — помечает главный класс как источник бинов), `@EnableAutoConfiguration` (включает механизм автоконфигурации), `@ComponentScan` (сканирует пакет главного класса и все подпакеты).

При запуске `SpringApplication.run(MyApp.class, args)` Spring Boot: создаёт `ApplicationContext`, запускает `@ComponentScan` для обнаружения компонентов, загружает автоконфигурации, публикует события (ApplicationStartingEvent, ApplicationEnvironmentPreparedEvent и т.д.), поднимает встроенный сервер (Tomcat/Jetty).

```java
@SpringBootApplication
// эквивалентно:
// @SpringBootConfiguration
// @EnableAutoConfiguration
// @ComponentScan("com.example") // пакет главного класса
public class EcommerceApplication {

    public static void main(String[] args) {
        SpringApplication.run(EcommerceApplication.class, args);
    }
}

// Можно кастомизировать ComponentScan:
@SpringBootApplication(scanBasePackages = {"com.example.app", "com.example.shared"})
public class EcommerceApplication { ... }

// Или исключить конкретные автоконфигурации:
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class EcommerceApplication { ... }
```

❗ Важный момент: `@ComponentScan` сканирует только пакет класса с `@SpringBootApplication` и его подпакеты. Если поместить главный класс не в корневой пакет, а в подпакет — часть компонентов не найдётся. Главный класс должен быть в корневом пакете приложения.

**→ Уточняющий вопрос:** Что произойдёт, если поместить @SpringBootApplication-класс внутрь подпакета, а не в корень?

**↳ Ответ:** `@ComponentScan` сканирует пакет класса и его подпакеты. Если главный класс в `com.example.app.config` — сканируется только `com.example.app.config.*`. Компоненты в `com.example.app.service`, `com.example.app.controller` не найдутся, их бины не создадутся. Приложение запустится, но эндпоинты не будут работать (404 или NullPointerException при попытке найти сервис). Решение: главный класс в корне пакета (`com.example`) или явный `scanBasePackages`.

---

## 22. Что такое starter в Spring Boot?

Starter — это Maven/Gradle зависимость, которая тянет за собой набор согласованных библиотек для определённой задачи. Вместо того чтобы подбирать совместимые версии Spring Web, Jackson, Tomcat, достаточно добавить `spring-boot-starter-web` — и всё придёт в правильных версиях.

Стартер — это просто `pom.xml` (или `build.gradle`) без кода. Он объявляет зависимости и BOM (Bill of Materials) для управления версиями. Spring Boot BOM гарантирует, что все компоненты совместимы между собой. К стартеру обычно прилагается автоконфигурация, которая настраивает библиотеки из classpath автоматически.

```xml
<!-- Одна строка заменяет десятки транзитивных зависимостей -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <!-- подтягивает: spring-data-jpa, hibernate-core, spring-jdbc, 
         spring-tx, HikariCP и другие в нужных версиях -->
</dependency>

<!-- Популярные стартеры:
     spring-boot-starter-web        — REST API (Tomcat + Spring MVC + Jackson)
     spring-boot-starter-data-jpa   — JPA/Hibernate
     spring-boot-starter-security   — Spring Security
     spring-boot-starter-test       — JUnit 5 + Mockito + AssertJ
     spring-boot-starter-validation — Jakarta Validation (Hibernate Validator)
     spring-boot-starter-actuator   — метрики, health checks
-->
```

❗ Стартер ≠ автоконфигурация. Стартер — зависимость (управление classpath), автоконфигурация — код, который настраивает бины. Одна библиотека может иметь автоконфигурацию без стартера. Вопрос "чем стартер отличается от автоконфигурации" — типичная проверка глубины понимания.

**→ Уточняющий вопрос:** Как создать собственный starter для внутренней библиотеки компании?

**↳ Ответ:** Два модуля: `company-feature-spring-boot-autoconfigure` (содержит `@AutoConfiguration`-класс с `@ConditionalOn*` и регистрацию в `AutoConfiguration.imports`) и `company-feature-spring-boot-starter` (только `pom.xml` с зависимостями: autoconfigure-модуль + сама библиотека). Потребитель добавляет только starter — получает и библиотеку, и автоконфигурацию. Условия `@ConditionalOnMissingBean` позволяют переопределять дефолты. Публикуй в Nexus/Artifactory.

---

## 23. Что делает аннотация @ComponentScan?

`@ComponentScan` говорит Spring, в каких пакетах искать классы с аннотациями `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`, `@Configuration` и регистрировать их как бины. Без `@ComponentScan` Spring не знает, где искать компоненты.

По умолчанию (без параметров) сканирует пакет класса, на котором стоит аннотация, и все его подпакеты. Можно указать явные пакеты, исключить или включить по фильтрам. `@SpringBootApplication` включает `@ComponentScan` — поэтому главный класс нужно размещать в корневом пакете.

```java
// Явное указание пакетов
@SpringBootApplication
@ComponentScan(basePackages = {
    "com.example.api",
    "com.example.service",
    "com.shared.components" // внешний модуль
})
public class Application { ... }

// Исключение классов из сканирования (например, тестовые конфиги)
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = TestConfiguration.class
    )
)

// Включить только классы с определённой аннотацией
@ComponentScan(
    basePackages = "com.example",
    useDefaultFilters = false,
    includeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION, classes = Service.class
    )
)
```

❗ Типичная ошибка на проектах с модульной структурой: компоненты из другого Maven-модуля не находятся, потому что их пакет не входит в зону сканирования. Решение: явно добавить пакет или создать `@SpringBootApplication` в общем корневом пакете.

**→ Уточняющий вопрос:** Как @ComponentScan работает с @Import и чем они отличаются?

**↳ Ответ:** `@ComponentScan` — сканирует пакеты, находит классы с аннотациями-стереотипами. `@Import` — регистрирует конкретные классы как бины/конфигурации явно, без сканирования. `@Import(MyConfig.class)` добавит бины из `MyConfig` даже если она в другом пакете вне сканирования. `@Import` может импортировать `@Configuration`, обычный класс (регистрируется как бин), `ImportSelector` (динамический выбор классов) или `ImportBeanDefinitionRegistrar` (программная регистрация). Autoconfiguration использует именно эти механизмы.

---

## 24. Что такое @Configuration класс?

`@Configuration` — аннотация, помечающая класс как источник определений бинов (фабрика бинов). Методы с `@Bean` внутри него регистрируют бины в ApplicationContext. Это Java-альтернатива XML-конфигурации.

Ключевая особенность: `@Configuration`-классы обрабатываются через CGLIB-прокси. Spring создаёт подкласс этого класса, и вызовы `@Bean`-методов перехватываются: при повторном вызове возвращается уже существующий singleton-бин вместо создания нового. Это называется "full" конфигурация (в отличие от `@Component` с `@Bean`, где прокси нет — "lite" конфигурация).

```java
@Configuration
public class DataConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost/mydb");
        ds.setMaximumPoolSize(20);
        return ds;
    }

    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource()); // вызов dataSource() вернёт тот же singleton!
    }

    @Bean
    public NamedParameterJdbcTemplate namedJdbcTemplate() {
        return new NamedParameterJdbcTemplate(dataSource()); // тот же DataSource бин
    }
}

// Если бы это был @Component — dataSource() создавал бы новый объект каждый раз
// В @Configuration — CGLIB перехватит и вернёт тот же бин из контейнера
```

❗ Разница между `@Configuration` (full) и `@Component` с `@Bean` (lite) — частый вопрос. В lite-режиме повторный вызов `@Bean`-метода создаёт новый объект. В full-режиме — возвращает singleton. Если конфигурационный класс помечен только `@Component` — можно получить несколько экземпляров одного бина.

**→ Уточняющий вопрос:** Что произойдёт, если сделать @Configuration-класс final?

**↳ Ответ:** CGLIB не сможет создать подкласс — Spring выбросит исключение при старте (в Boot 3) или переключится в lite-режим (в Boot 2 с предупреждением). В lite-режиме вызовы `@Bean`-методов между собой создают новые объекты (нет перехвата), что нарушает ожидание singleton-поведения. Правило: никогда не делай `@Configuration`-классы final. Это та же причина, почему не делают final методы с AOP-аннотациями.

---

## 25. В чём разница между @Component, @Service, @Repository, @Controller?

Технически все четыре аннотации являются вариантами `@Component`: `@Service`, `@Repository`, `@Controller` объявлены с `@Component` как мета-аннотацией. С точки зрения обнаружения компонентов через `@ComponentScan` — они идентичны.

Различия семантические и функциональные. `@Repository` имеет реальное дополнительное поведение: Spring применяет `PersistenceExceptionTranslationPostProcessor`, который перехватывает нативные исключения persistence (Hibernate, JDBC) и переводит их в иерархию Spring `DataAccessException`. `@Controller` распознаётся Spring MVC для маппинга запросов. `@Service` — чисто семантический, документирует бизнес-логику.

```java
@Repository // DataAccessException translation + маркер слоя данных
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}

@Service // бизнес-логика, транзакции
public class UserService {
    private final UserRepository userRepository;

    @Transactional
    public User register(RegisterRequest request) {
        // HibernateException будет перехвачено @Repository и брошено как DataAccessException
        return userRepository.save(new User(request));
    }
}

@RestController // @Controller + @ResponseBody на всех методах
@RequestMapping("/users")
public class UserController {
    private final UserService userService;

    @PostMapping
    public ResponseEntity<User> register(@RequestBody @Valid RegisterRequest request) {
        return ResponseEntity.ok(userService.register(request));
    }
}
```

❗ На собесе спрашивают: "Можно ли использовать `@Component` вместо `@Service`?" — технически да, и Spring найдёт бин. Но потеряете семантику, инструменты анализа (например, архитектурные правила ArchUnit) не смогут проверить правильность слоёв, и `@Repository` потеряет translation исключений.

**→ Уточняющий вопрос:** Что такое PersistenceExceptionTranslationPostProcessor и как он связан с @Repository?

**↳ Ответ:** `PersistenceExceptionTranslationPostProcessor` — BeanPostProcessor, который оборачивает все бины с `@Repository` в прокси. Этот прокси перехватывает нативные persistence-исключения (`HibernateException`, `PersistenceException`, `DataIntegrityViolationException` от JDBC) и транслирует их в иерархию `DataAccessException`. Зачем: сервисный слой не должен зависеть от деталей persistence-технологии (Hibernate vs JDBC vs MyBatis). При смене ORM код сервиса не меняется — только `@Repository` меняет реализацию.

---

## 26. Что делает аннотация @Autowired?

`@Autowired` указывает Spring внедрить зависимость автоматически, находя подходящий бин в ApplicationContext по типу. Применяется к конструктору, полю или сеттеру. При нескольких бинах одного типа — ошибка (NoUniqueBeanDefinitionException), если не уточнить через `@Qualifier` или `@Primary`.

Обрабатывается `AutowiredAnnotationBeanPostProcessor` в фазе populate properties. На конструкторе с одним аргументом в Spring 4.3+ `@Autowired` не обязателен. По умолчанию `@Autowired(required = true)` — если бин не найден, приложение падает на старте. `required = false` делает зависимость опциональной.

```java
@Service
public class ReportService {

    // На конструкторе — Spring 4.3+ не требует @Autowired явно
    private final ReportRepository reportRepository;
    private final Optional<EmailService> emailService; // Optional для опциональных зависимостей

    @Autowired // для ясности, можно опустить
    public ReportService(ReportRepository reportRepository,
                         Optional<EmailService> emailService) {
        this.reportRepository = reportRepository;
        this.emailService = emailService;
    }

    // @Autowired на сеттере — для опциональных зависимостей
    private MetricsService metricsService;

    @Autowired(required = false)
    public void setMetricsService(MetricsService metricsService) {
        this.metricsService = metricsService;
    }
}
```

❗ Если есть несколько бинов одного типа и ни один не помечен `@Primary`, `@Autowired` выбросит `NoUniqueBeanDefinitionException`. Исключение: если имя поля/параметра совпадает с именем бина — Spring выберет по имени (fallback). Но полагаться на это не стоит — лучше явный `@Qualifier`.

**→ Уточняющий вопрос:** Как Spring разрешает @Autowired, если есть несколько бинов одного типа?

**↳ Ответ:** Алгоритм: 1) найти все бины нужного типа, 2) если один — внедрить, 3) если несколько: сначала смотрит на `@Primary`, затем пытается сопоставить по имени (имя поля/параметра = имя бина), затем смотрит `@Qualifier`. Если ни один механизм не выбрал — `NoUniqueBeanDefinitionException`. Правило: при нескольких кандидатах всегда используй явный `@Qualifier`, не полагайся на совпадение имён — это хрупко при рефакторинге.

---

## 27. Что делать, если есть несколько бинов одного типа?

Spring предоставляет несколько механизмов разрешения неоднозначности. **`@Primary`** — пометить один бин как предпочтительный; он будет выбран по умолчанию, если не указано иное. **`@Qualifier("name")`** — явно указать имя бина для внедрения. **Именование через имя параметра/поля** — если имя поля совпадает с именем бина, Spring выберет его (ненадёжно, лучше явный qualifier). **Внедрение коллекции** — `List<SomeInterface>` получит все бины этого типа.

```java
// Два бина одного типа
@Bean
@Primary // выбирается по умолчанию
public NotificationSender emailSender() {
    return new EmailNotificationSender();
}

@Bean
public NotificationSender smsSender() {
    return new SmsNotificationSender();
}

// Внедрение с @Qualifier
@Service
public class AlertService {
    private final NotificationSender emailSender;
    private final NotificationSender smsSender;

    public AlertService(
            @Qualifier("emailSender") NotificationSender emailSender,
            @Qualifier("smsSender") NotificationSender smsSender) {
        this.emailSender = emailSender;
        this.smsSender = smsSender;
    }
}

// Внедрение всех бинов одного типа
@Service
public class BroadcastService {
    private final List<NotificationSender> senders; // получит email + sms

    public BroadcastService(List<NotificationSender> senders) {
        this.senders = senders;
    }

    public void broadcast(String message) {
        senders.forEach(s -> s.send(message));
    }
}
```

❗ `@Primary` влияет на все места, где внедряется этот тип без qualifier. Это глобальный дефолт. Если вам нужна разная реализация в разных местах — `@Primary` не подходит, используйте `@Qualifier` явно везде.

**→ Уточняющий вопрос:** Как работает @ConditionalOnMissingBean в связке с @Primary при автоконфигурации?

**↳ Ответ:** `@ConditionalOnMissingBean(SomeService.class)` — автоконфигурация создаёт бин только если в контексте ещё нет бина этого типа. Если пользователь объявил свой `SomeService` (с `@Primary` или без) — автоконфигурация отступает. Если пользователь объявил без `@Primary` а автоконфигурация всё же создаёт второй бин — возникнет `NoUniqueBeanDefinitionException`. Порядок важен: автоконфигурация загружается после пользовательских бинов (`@AutoConfiguration(after = ...)` управляет порядком).

---

## 28. Что такое @Qualifier?

`@Qualifier` — аннотация для уточнения, какой именно бин Spring должен внедрить, когда в контексте несколько кандидатов одного типа. По умолчанию значение — имя бина (имя метода `@Bean` или имя класса с маленькой буквы для `@Component`).

Более мощное использование: создать собственную аннотацию на основе `@Qualifier`. Это позволяет уйти от строковых имён (которые легко опечатать) к типобезопасным аннотациям. Такой подход популярен в enterprise-проектах.

```java
// Стандартное использование — по имени бина
@Service
public class PaymentService {

    @Autowired
    @Qualifier("stripeGateway") // имя бина
    private PaymentGateway gateway;
}

// Типобезопасный кастомный qualifier — лучшая практика
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface StripeGateway {}

@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface PayPalGateway {}

@Bean
@StripeGateway
public PaymentGateway stripeGateway() { return new StripePaymentGateway(); }

@Bean
@PayPalGateway
public PaymentGateway paypalGateway() { return new PaypalPaymentGateway(); }

// Использование — типобезопасно, рефакторинг не сломает
@Service
public class CheckoutService {
    public CheckoutService(@StripeGateway PaymentGateway gateway) { ... }
}
```

❗ Строковое значение `@Qualifier("stripeGateway")` — хрупко: опечатка в строке не обнаруживается компилятором, а вылезет только в рантайме. Кастомные аннотации-qualifier — более надёжный подход. На собесе стоит упомянуть оба варианта и объяснить разницу.

**→ Уточняющий вопрос:** В чём разница между @Qualifier и @Primary при конфликте кандидатов?

**↳ Ответ:** `@Primary` — глобальный дефолт: побеждает во всех точках внедрения без qualifier. `@Qualifier` — локальное уточнение: работает только там, где явно написано. Если нужно одну реализацию везде по умолчанию и иногда другую — используй `@Primary` + `@Qualifier` там где нужна альтернатива. Если в разных местах нужны разные реализации и нет явного дефолта — `@Qualifier` везде без `@Primary`.

---

## 29. Что такое profiles в Spring?

Profiles — механизм условной активации бинов и конфигурации в зависимости от среды запуска. Бин или конфигурационный класс с `@Profile("dev")` будет создан только когда активен профиль `dev`. Это позволяет иметь разные реализации для dev/test/prod без изменения кода.

Активировать профиль можно через: `spring.profiles.active=prod` в `application.properties`, переменную окружения `SPRING_PROFILES_ACTIVE=prod`, аргумент JVM `-Dspring.profiles.active=prod`, или программно `SpringApplication.setAdditionalProfiles()`. Можно активировать несколько профилей одновременно.

Profiles применяются и к `application.properties`: `application-dev.properties`, `application-prod.properties` автоматически загружаются при активном соответствующем профиле и переопределяют базовый `application.properties`.

```java
// Разные DataSource для разных сред
@Configuration
@Profile("dev")
public class DevDataConfig {
    @Bean
    public DataSource dataSource() {
        // H2 in-memory для разработки
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2).build();
    }
}

@Configuration
@Profile("prod")
public class ProdDataConfig {
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(System.getenv("DB_URL"));
        return ds;
    }
}

// Условная логика в коде
@Service
public class FeatureFlagService {

    @Value("${spring.profiles.active:default}")
    private String activeProfile;

    // Или через Environment
    @Autowired
    private Environment env;

    public boolean isDebugEnabled() {
        return env.acceptsProfiles(Profiles.of("dev", "local"));
    }
}

// application-dev.properties:
// spring.jpa.show-sql=true
// logging.level.com.example=DEBUG

// application-prod.properties:
// spring.jpa.show-sql=false
// logging.level.com.example=WARN
```

❗ Распространённая ошибка: бин без `@Profile` существует во всех профилях. Если бин с `@Profile("dev")` переопределяет бин без профиля — может возникнуть конфликт (два бина одного типа). Нужно либо `@Primary`, либо правильная структура — один базовый бин и профильные переопределения через `@Profile("!prod")` (все кроме prod).

**→ Уточняющий вопрос:** Как протестировать приложение с конкретным профилем в JUnit 5?

**↳ Ответ:** `@ActiveProfiles("test")` на тест-классе — активирует профиль для всех тестов в классе. Можно комбинировать: `@ActiveProfiles({"test", "kafka-mock"})`. Для Spring Boot: `@SpringBootTest` поднимает полный контекст, `@ActiveProfiles` выбирает конфигурацию. Альтернатива: `@TestPropertySource(properties = "spring.profiles.active=test")`. В `@DataJpaTest` / `@WebMvcTest` — работает аналогично. Правило: test-профиль обычно активирует H2 вместо реальной БД и mock-и внешних сервисов.`
