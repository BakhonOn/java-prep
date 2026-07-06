# ООП и SOLID

> 📇 Справочник уровня middle. Формат: ответ по сути → пример при необходимости → ❗ на чём ловят на собесе.

**Всего вопросов: 22**

---

## 1. Что такое принцип Single Responsibility и как его применять?

Single Responsibility Principle (SRP) — у класса должна быть ровно одна причина для изменения, то есть одна ответственность. Роберт Мартин определяет «причину для изменения» как конкретного стейкхолдера или бизнес-роль, которая инициирует правку кода. Если класс правится и из-за изменения бизнес-логики, и из-за смены формата отчёта, и из-за нового способа нотификации — это три разные причины.

На практике применяется вертикальным расслоением: каждый класс отражает одну концепцию домена или один технический аспект. В Spring-приложении `UserService` содержит бизнес-правила создания пользователя, `UserRepository` — работу с БД, `UserNotificationService` — отправку писем. Изменение шаблона письма затрагивает только `UserNotificationService`, не трогая логику в `UserService`.

Практический тест: попробуйте описать назначение класса одним предложением без союза «и». «Класс сохраняет пользователя **и** шлёт письмо **и** логирует» — три ответственности.

❗ **Ловушка собеса:** путают SRP с «делать одно действие» — тогда получаются сотни крошечных классов. SRP — про одну *причину для изменения*, а не про один метод. Также спрашивают: «А UserService не слишком большой, если в нём 10 методов?» — правильный ответ: это нормально, если все 10 методов обслуживают одну бизнес-ответственность.

**→ Уточняющий вопрос:** Как вы разграничиваете ответственности между `UserService` и `UserDomainService` — когда появляется второй?

**↳ Ответ:** `UserService` (Application Service) — координирует use case: валидация входа, вызов репозитория, нотификации, транзакции. `UserDomainService` появляется, когда бизнес-правило вовлекает несколько агрегатов или не принадлежит ни одному из них — например, «проверить уникальность email среди всех пользователей» требует обращения к репозиторию, что не должно делать само доменное Entity. Правило: если операция работает только с данными одного User — метод в User; если нужны данные других User — UserDomainService.

---

## 2. Приведите пример нарушения принципа Single Responsibility

Классическое нарушение — «толстый» сервис, куда свалено всё:

```java
// Нарушение SRP: один класс — несколько причин для изменения
public class UserService {
    public void registerUser(UserDto dto) {
        // 1. Валидация — причина 1: смена правил валидации
        if (dto.getEmail() == null || !dto.getEmail().contains("@")) {
            throw new IllegalArgumentException("Bad email");
        }
        // 2. Бизнес-логика — причина 2: бизнес-правила регистрации
        User user = new User(dto.getEmail(), dto.getName());
        user.setCreatedAt(LocalDateTime.now());
        // 3. Персистентность — причина 3: смена ORM / БД
        entityManager.persist(user);
        // 4. Нотификация — причина 4: смена почтового провайдера
        emailClient.send(user.getEmail(), "Welcome!", "Hi " + user.getName());
        // 5. Аудит — причина 5: смена формата лога
        auditLog.write("User registered: " + user.getEmail());
    }
}
```

Правильное разделение:

```java
// SRP: каждый класс — одна ответственность
@Service
public class UserRegistrationService {
    private final UserValidator validator;
    private final UserRepository repository;
    private final UserNotificationService notifier;
    private final AuditService auditService;

    public User register(UserDto dto) {
        validator.validate(dto);                        // делегируем валидацию
        User user = User.create(dto);                   // фабричный метод домена
        repository.save(user);                          // персистентность
        notifier.sendWelcomeEmail(user);                // нотификация
        auditService.logRegistration(user);             // аудит
        return user;
    }
}
```

Теперь изменение шаблона письма затрагивает только `UserNotificationService`. Переход с JPA на JDBC — только `UserRepository`. `UserRegistrationService` остаётся стабильным.

❗ **Ловушка собеса:** на практике «а где граница?» — интервьюер будет давить. Ответ: граница там, где разные команды/бизнес-роли инициируют изменения. Email-маркетинг меняет шаблон письма, DBA меняет схему — это разные причины.

**→ Уточняющий вопрос:** Где держать логику маппинга `UserDto → User` — в `UserRegistrationService`, в самом `User` или в отдельном `UserMapper`?

**↳ Ответ:** В `UserMapper` (отдельный класс или MapStruct-интерфейс) — маппинг это отдельная ответственность: преобразование форматов слоя API в доменные объекты. Если в `UserRegistrationService` — нарушение SRP (он и бизнес-логику выполняет, и форматы конвертирует). Если в самом `User` — нарушение DIP (доменный объект знает о DTO из слоя API). Правило: маппинг живёт на границе слоёв, в классе без бизнес-логики.

---

## 3. Что такое принцип Open/Closed?

Open/Closed Principle (OCP) — программные сущности должны быть открыты для расширения, но закрыты для модификации. Новое поведение добавляется через новые классы/реализации, а не через правку существующего кода. Механизм достижения — полиморфизм и абстракции (интерфейсы, абстрактные классы).

Мотивация: чем меньше существующий код правится, тем меньше регрессий. Каждое изменение старого кода — риск. Если добавление нового типа платежа не трогает `OrderService`, то этот сервис «закрыт для модификации». Он «открыт для расширения» через новую реализацию `PaymentProcessor`.

В Spring это реализуется через `@Service`-бины с общим интерфейсом. Spring сам через DI подставит нужную реализацию.

```java
// Нарушение OCP: каждый новый тип платежа требует правки метода
public class PaymentService {
    public void processPayment(Order order, String type) {
        if ("CARD".equals(type)) {
            cardGateway.charge(order.getAmount());
        } else if ("PAYPAL".equals(type)) {
            paypalClient.pay(order.getAmount());
        } else if ("CRYPTO".equals(type)) {  // новый тип — правим существующий код
            cryptoGateway.transfer(order.getAmount());
        }
    }
}

// Правильно: новый тип платежа — новый класс, старый код не трогаем
public interface PaymentProcessor {
    void process(Order order);
    boolean supports(PaymentType type);
}

@Component
public class CardPaymentProcessor implements PaymentProcessor {
    public void process(Order order) { cardGateway.charge(order.getAmount()); }
    public boolean supports(PaymentType type) { return type == PaymentType.CARD; }
}

@Component
public class CryptoPaymentProcessor implements PaymentProcessor {
    public void process(Order order) { cryptoGateway.transfer(order.getAmount()); }
    public boolean supports(PaymentType type) { return type == PaymentType.CRYPTO; }
}

@Service
public class PaymentService {
    private final List<PaymentProcessor> processors;  // Spring инжектирует все реализации

    public void processPayment(Order order, PaymentType type) {
        processors.stream()
            .filter(p -> p.supports(type))
            .findFirst()
            .orElseThrow(() -> new UnsupportedPaymentTypeException(type))
            .process(order);
    }
}
```

❗ **Ловушка собеса:** OCP не означает «никогда не трогать старый код» — рефакторинг и исправление багов это нормально. OCP защищает от изменений *из-за добавления новых вариантов поведения*. Также спрашивают про стоимость: заранее проектировать OCP для всего — over-engineering, точки расширения нужно выявлять по реальным требованиям.

**→ Уточняющий вопрос:** Какой паттерн проектирования лежит в основе этого примера с `PaymentProcessor`?

**↳ Ответ:** Паттерн Strategy: интерфейс `PaymentProcessor` определяет алгоритм, каждая реализация (`CardPaymentProcessor`, `CryptoPaymentProcessor`) — отдельная стратегия. `PaymentService` выбирает нужную стратегию через `supports()` — это Chain of Responsibility в миниатюре. OCP здесь достигается именно через Strategy: новый способ оплаты = новый класс-стратегия, никакого изменения существующего кода.

---

## 4. Как рефакторить код, нарушающий принцип Open/Closed?

Нарушение OCP легче всего узнать по `if/else` или `switch` на типе/роли: каждый новый «тип» требует добавления ветки в существующий метод. Рефакторинг: заменить условную диспетчеризацию полиморфизмом через паттерн Strategy или Command.

Шаги рефакторинга:
1. Выделить интерфейс с методом, который делает «разное для разных типов».
2. Каждую ветку `if/else` превратить в отдельную реализацию интерфейса.
3. Заменить `switch` в вызывающем коде на коллекцию стратегий или фабрику.
4. Добавить новый тип → создать новый класс, старый код не трогаем.

```java
// До рефакторинга — нарушение OCP
public class OrderDiscountService {
    public BigDecimal calculateDiscount(Order order) {
        switch (order.getCustomerType()) {
            case VIP:     return order.getTotal().multiply(new BigDecimal("0.20"));
            case REGULAR: return order.getTotal().multiply(new BigDecimal("0.05"));
            case NEW:     return BigDecimal.ZERO;
            default:      throw new IllegalArgumentException("Unknown type");
        }
    }
}

// После рефакторинга — OCP соблюдён
public interface DiscountPolicy {
    BigDecimal calculate(Order order);
    boolean appliesTo(CustomerType type);
}

@Component
public class VipDiscountPolicy implements DiscountPolicy {
    public BigDecimal calculate(Order order) {
        return order.getTotal().multiply(new BigDecimal("0.20"));
    }
    public boolean appliesTo(CustomerType type) { return type == CustomerType.VIP; }
}

// Новая скидка для партнёров: просто новый класс, ничего не трогаем
@Component
public class PartnerDiscountPolicy implements DiscountPolicy {
    public BigDecimal calculate(Order order) {
        return order.getTotal().multiply(new BigDecimal("0.15"));
    }
    public boolean appliesTo(CustomerType type) { return type == CustomerType.PARTNER; }
}
```

❗ **Ловушка собеса:** интервьюер попросит объяснить, где «закрытость» реализована. Правильный ответ: `OrderDiscountService` / `PaymentService` — закрытый класс, его код не меняется при добавлении нового типа. Реализации интерфейса — точки расширения.

**→ Уточняющий вопрос:** Как Spring узнаёт, какую реализацию `DiscountPolicy` использовать для конкретного заказа?

**↳ Ответ:** Spring инжектирует `List<DiscountPolicy>` со всеми бинами, реализующими интерфейс, — это называется collection injection. Далее код сам итерирует и вызывает `appliesTo(customerType)`: найденная реализация и применяется. Spring не выбирает — выбирает бизнес-логика через метод `supports()`/`appliesTo()`. Это и есть суть OCP в связке со Spring DI: новый бин автоматически появится в списке без правки `OrderDiscountService`.

---

## 5. Что такое принцип Liskov Substitution?

Liskov Substitution Principle (LSP) — если S является подтипом T, то объекты типа T в программе можно заменить объектами типа S без изменения корректности программы. Барбара Лисков, 1987. Это не просто «подкласс наследует методы» — это требование к *контракту поведения*.

Контракт включает три требования: подкласс не должен *ужесточать предусловия* (принимать меньше, чем родитель), не должен *ослаблять постусловия* (возвращать что-то неожиданное), и не должен нарушать *инварианты* родительского класса. Если клиентский код работает с `UserRepository`, он должен корректно работать с любой реализацией: `JpaUserRepository`, `InMemoryUserRepository`, `CachingUserRepository` — без специальных проверок.

```java
// Контракт: findById никогда не возвращает null — бросает исключение
public interface UserRepository {
    User findById(Long id);  // постусловие: не null, или UserNotFoundException
}

// Правильная реализация — соблюдает контракт
@Repository
public class JpaUserRepository implements UserRepository {
    public User findById(Long id) {
        return em.find(User.class, id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}

// Нарушение LSP — возвращает null вместо исключения
@Repository
public class LegacyUserRepository implements UserRepository {
    public User findById(Long id) {
        return legacyDao.get(id);  // может вернуть null — клиент упадёт с NPE
    }
}
```

❗ **Ловушка собеса:** LSP часто проверяют именно на примере нотификации `Optional` vs исключений в репозиториях — одна реализация бросает исключение, другая возвращает null, это нарушение LSP. Ещё спрашивают: «Нарушает ли LSP переопределение метода?» — нет, если контракт сохранён.

**→ Уточняющий вопрос:** Как связан LSP с принципом Design by Contract?

**↳ Ответ:** Design by Contract (DbC, Бертран Мейер) формализует то, что LSP требует неформально: каждый метод имеет предусловия (что должно быть истинным при вызове), постусловия (что будет истинным после), и инварианты класса (что всегда истинно). LSP в терминах DbC: подкласс не должен усиливать предусловия и не должен ослаблять постусловия. Практически — если интерфейс говорит «возвращает User или бросает UserNotFoundException», то реализация, возвращающая null, нарушает постусловие, а значит LSP.

---

## 6. Приведите пример нарушения принципа Liskov Substitution

Каноничный пример — `Square extends Rectangle`. В математике квадрат — частный случай прямоугольника, но в объектной модели это нарушение LSP:

```java
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width)   { this.width = width; }
    public void setHeight(int height) { this.height = height; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;  // квадрат: принудительно ставим оба
    }
    @Override
    public void setHeight(int height) {
        this.height = height;
        this.width = height;  // и здесь тоже
    }
}

// Клиентский код, корректный для Rectangle
void stretch(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    assert r.area() == 50;  // для Rectangle — true, для Square — FALSE (100)
}
```

Вызов `stretch(new Square())` ломает assertion — Square не является корректной заменой Rectangle. Решение: сделать оба класса независимыми (без наследования) или ввести общий интерфейс `Shape` только с `area()`, убрав сеттеры ширины/высоты из интерфейса.

Реальный аналог в доменных моделях: `PremiumOrder extends Order` и переопределение `cancel()` с выбросом исключения (Premium-заказ нельзя отменить). Весь код, работающий с `Order` и вызывающий `cancel()`, сломается на `PremiumOrder`.

```java
// Нарушение: PremiumOrder не является полноценным Order
public class PremiumOrder extends Order {
    @Override
    public void cancel() {
        throw new UnsupportedOperationException("Premium orders cannot be cancelled");
    }
}

// Решение: разделить на интерфейсы
public interface Cancellable {
    void cancel();
}
public class RegularOrder extends Order implements Cancellable { ... }
public class PremiumOrder extends Order { /* cancel() отсутствует */ }
```

❗ **Ловушка собеса:** «Square — это математически прямоугольник, почему нарушение?» — ответ: математическое отношение подмножества ≠ объектное отношение подтипа. LSP требует поведенческой совместимости, а не только структурной. Ключевое слово — *контракт*, а не *тип*.

**→ Уточняющий вопрос:** Как исправить иерархию Rectangle/Square, не нарушая LSP?

**↳ Ответ:** Три варианта: (1) сделать оба класса независимыми, не наследовать Square от Rectangle вовсе; (2) ввести общий интерфейс `Shape` только с `area()` — без сеттеров ширины/высоты, которые нарушают контракт; (3) сделать Rectangle иммутабельным (только getters) — тогда и Square и Rectangle реализуют `area()` без мутирующих методов, нарушение исчезает. Правило: если подкласс вынужден нарушить контракт метода родителя — наследование неправильное.

---

## 7. Что такое принцип Interface Segregation?

Interface Segregation Principle (ISP) — клиент не должен зависеть от методов, которые он не использует. Лучше несколько узких интерфейсов, чем один «жирный». Проблема «жирного» интерфейса: изменение одного метода вынуждает перекомпилировать/переписывать всех клиентов, даже тех, кто этот метод не использует.

```java
// Нарушение ISP: один интерфейс, но OrderProcessor использует только 2 из 5 методов
public interface UserManager {
    User findById(Long id);
    void save(User user);
    void delete(User user);
    void sendWelcomeEmail(User user);
    void generateReport(User user);
}

// OrderProcessor вынужден зависеть от sendWelcomeEmail и generateReport — не нужных ему
@Service
public class OrderProcessor {
    private final UserManager userManager;
    public void process(Order order) {
        User user = userManager.findById(order.getUserId());  // нужен только этот метод
        // ...
    }
}
```

Решение — разделить на интерфейсы по роли:

```java
public interface UserReader {
    User findById(Long id);
    Optional<User> findByEmail(String email);
}

public interface UserWriter {
    void save(User user);
    void delete(User user);
}

public interface UserNotifier {
    void sendWelcomeEmail(User user);
    void sendPasswordReset(User user);
}

// OrderProcessor зависит только от нужного интерфейса
@Service
public class OrderProcessor {
    private final UserReader userReader;  // узкая зависимость
    // ...
}

// JpaUserRepository реализует несколько интерфейсов
@Repository
public class JpaUserRepository implements UserReader, UserWriter {
    // ...
}
```

❗ **Ловушка собеса:** ISP не требует «один метод — один интерфейс». Интерфейс группирует методы, которые меняются по одной причине и используются одним типом клиента. Также спрашивают: «Не противоречит ли ISP принципу DRY?» — нет, DRY про дублирование реализации, ISP про зависимости.

**→ Уточняющий вопрос:** Как ISP соотносится с гранулярностью интерфейсов в Spring — почему `JpaRepository` такой «жирный»?

**↳ Ответ:** `JpaRepository` нарушает ISP намеренно ради удобства: Spring Data хочет предоставить «всё в одном месте» из соображений developer experience, а не архитектурной чистоты. Решение для production-кода: не инжектировать `JpaRepository` напрямую в сервисы — создать собственный узкий интерфейс `OrderRepository` с нужными методами и реализовать его через `JpaOrderRepository extends JpaRepository`. Сервис зависит от узкого интерфейса, JPA-детали скрыты.

---

## 8. Что такое принцип Dependency Inversion?

Dependency Inversion Principle (DIP) — два правила: (1) высокоуровневые модули не должны зависеть от низкоуровневых, оба зависят от абстракций; (2) абстракции не зависят от деталей, детали зависят от абстракций. «Инверсия» означает разворот привычной зависимости: вместо того, чтобы `OrderService` зависел от конкретного `MySqlOrderRepository`, оба зависят от `OrderRepository`.

Механизм: вводим интерфейс на «стыке» высокоуровневого и низкоуровневого модуля. Высокоуровневый модуль определяет интерфейс (или он живёт в том же слое домена), низкоуровневый модуль реализует его. Направление зависимости инвертируется: инфраструктура зависит от домена, а не наоборот.

```java
// Нарушение DIP: OrderService жёстко зависит от конкретной реализации
@Service
public class OrderService {
    private final MySqlOrderRepository repository;  // конкретный класс, не интерфейс
    private final SmtpEmailSender emailSender;      // привязка к конкретной реализации

    public OrderService() {
        this.repository = new MySqlOrderRepository();  // создаёт сам — полный антипаттерн
        this.emailSender = new SmtpEmailSender("smtp.gmail.com");
    }
}

// Правильно: DIP — зависим от абстракций
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(Long id);
}

public interface NotificationService {
    void notifyOrderCreated(Order order);
}

@Service
public class OrderService {
    private final OrderRepository orderRepository;       // интерфейс
    private final NotificationService notificationService;  // интерфейс

    // Зависимости внедряются извне (DI)
    public OrderService(OrderRepository orderRepository,
                        NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.notificationService = notificationService;
    }

    public Order createOrder(CreateOrderCommand cmd) {
        Order order = Order.create(cmd);
        orderRepository.save(order);
        notificationService.notifyOrderCreated(order);
        return order;
    }
}
```

❗ **Ловушка собеса:** DIP ≠ DI (Dependency Injection). DIP — архитектурный принцип «зависеть от абстракций». DI — паттерн передачи зависимостей извне. Можно использовать DI, передавая конкретные классы (DI без DIP). DIP без DI — теоретически красиво, но практически неудобно. Вместе — максимальный эффект.

**→ Уточняющий вопрос:** Где в Hexagonal Architecture (Ports & Adapters) явно прослеживается DIP?

**↳ Ответ:** В Hexagonal Architecture Ports — это интерфейсы, Adapters — реализации. Домен (центр) определяет Port интерфейсы (`OrderRepository`, `PaymentGateway`) — это «розетки». Инфраструктурные Adapters (`JpaOrderRepository`, `StripePaymentAdapter`) реализуют эти интерфейсы — это «вилки». Домен не знает о конкретных реализациях — чистый DIP: высокоуровневый домен задаёт абстракцию, низкоуровневая инфраструктура её реализует.

---

## 9. Зачем вообще нужны принципы SOLID?

SOLID — это набор эвристик, которые помогают управлять сложностью программного обеспечения при его росте. Без них кодовая база быстро деградирует: появляются хрупкие зависимости (изменение в одном месте ломает другое), «лишние знания» (класс знает слишком много о деталях других), трудности с тестированием (невозможно подменить зависимость моком).

На практике SOLID напрямую снижает стоимость изменений. Когда добавляют нового провайдера платежей — не трогают `OrderService` (OCP). Когда меняют базу данных — не переписывают бизнес-логику (DIP). Когда пишут юнит-тест на `OrderService` — мокают `PaymentProcessor` интерфейс без поднятия базы (DIP + SRP). Каждый принцип атакует конкретный вид связанности.

SOLID — не религия, а инструменты. «Правильное» применение зависит от контекста: небольшой скрипт не требует пяти уровней абстракции. Ценность принципов проявляется в командных проектах с долгосрочной поддержкой.

❗ **Ловушка собеса:** «SOLID — это теория, на практике мы не всегда следуем». Правильный ответ показывает прагматизм: принципы — компромисс между читаемостью, расширяемостью и сложностью. Знание *когда нарушить* — признак senior-мышления, но для middle важно уметь объяснить *зачем* каждый принцип.

**→ Уточняющий вопрос:** Какой принцип из SOLID вы считаете наиболее часто нарушаемым в реальных проектах и почему?

**↳ Ответ:** SRP — нарушается чаще всего, потому что его легче всего игнорировать в краткосрочной перспективе: проще добавить метод в существующий сервис, чем создать новый класс. Симптом: «жирные» сервисы с 10+ инжекциями и 500+ строками кода — это каждый второй production-проект. DIP нарушают реже, потому что Spring DI само подталкивает к использованию интерфейсов. LSP нарушают редко осознанно, но часто незаметно через `UnsupportedOperationException`.

---

## 10. Что такое композиция и наследование?

Наследование (`extends`) — отношение «is-a»: подкласс полностью является разновидностью родительского класса и наследует его реализацию. Композиция — отношение «has-a»: класс содержит экземпляр другого класса и делегирует ему часть работы. Это два принципиально разных механизма повторного использования кода.

Наследование создаёт жёсткую статическую связь: подкласс знает о внутренней реализации родителя (проблема хрупкого базового класса), иерархия фиксируется на этапе компиляции. Композиция гибче: можно менять поведение в рантайме, подменяя вложенный объект; нет утечки внутренних деталей.

```java
// Наследование: EmailNotificationService — это NotificationService
public abstract class NotificationService {
    public void notify(User user, String message) {
        String formatted = format(message);  // шаблонный метод
        send(user, formatted);
    }
    protected abstract void send(User user, String msg);
    protected String format(String msg) { return "[APP] " + msg; }
}

public class EmailNotificationService extends NotificationService {
    protected void send(User user, String msg) {
        emailClient.send(user.getEmail(), msg);
    }
}

// Композиция: NotificationService имеет MessageSender
public interface MessageSender {
    void send(String destination, String message);
}

@Component
public class NotificationService {
    private final MessageSender sender;  // внедряется, можно заменить в тестах

    public NotificationService(MessageSender sender) { this.sender = sender; }

    public void notify(User user, String message) {
        sender.send(user.getEmail(), "[APP] " + message);
    }
}
```

❗ **Ловушка собеса:** спрашивают «всегда ли композиция лучше?». Наследование оправдано, когда подтип действительно является более специализированной версией родителя с соблюдением LSP — например, `ArrayList extends AbstractList`. Чаще стоит выбрать композицию.

**→ Уточняющий вопрос:** Как паттерн Decorator реализует композицию, и чем он лучше наследования в этом случае?

**↳ Ответ:** Decorator содержит ссылку на тот же интерфейс, что и оборачиваемый объект, и делегирует ему вызовы, добавляя поведение до или после. В отличие от наследования, можно динамически создать `LoggingOrderRepository(CachingOrderRepository(JpaOrderRepository()))` — и порядок можно менять в рантайме. Наследование зафиксирует иерархию на этапе компиляции, а с Decorator можно комбинировать любое количество поведений без создания комбинаторного взрыва подклассов.

---

## 11. В каких случаях лучше использовать композицию вместо наследования?

Правило «Prefer composition over inheritance» (GoF) отражает практический опыт: наследование создаёт сильную статическую связь, нарушает инкапсуляцию (подкласс видит protected-внутренности), и ведёт к проблеме «хрупкого базового класса» — изменение родителя незаметно ломает всех потомков.

Композиция предпочтительна когда: (1) хочется переиспользовать поведение, не становясь подтипом; (2) нужно менять поведение в рантайме; (3) иерархия стала слишком глубокой; (4) класс использует только часть функциональности родителя.

```java
// Антипример: наследование ради переиспользования кода (не для is-a)
// OrderService РАСШИРЯЕТ BaseService только чтобы получить метод log()
public class OrderService extends BaseService {
    public void createOrder(Order order) {
        log("Creating order");  // взято из BaseService
        // ...
    }
}

// Правильно: OrderService использует логгер через композицию
@Service
public class OrderService {
    private final Logger log = LoggerFactory.getLogger(OrderService.class);
    private final OrderRepository orderRepository;
    private final PaymentProcessor paymentProcessor;

    public void createOrder(CreateOrderCommand cmd) {
        log.info("Creating order for user {}", cmd.getUserId());
        Order order = Order.create(cmd);
        orderRepository.save(order);
        paymentProcessor.charge(order);
    }
}
```

Наследование оправдано для настоящих «is-a» отношений с устойчивой иерархией: `JpaRepository` реализации, иерархии исключений (`PaymentException extends BusinessException`), Template Method паттерн при расширении фреймворка.

❗ **Ловушка собеса:** «Hibernate entity с наследованием — это хорошо?» Ответ: Hibernate поддерживает наследование сущностей (`@Inheritance`), но на практике это часто ведёт к сложным запросам (JOINED стратегия) или nullable полям (SINGLE_TABLE). Стоит осознанно выбирать.

**→ Уточняющий вопрос:** Как паттерны Strategy и Decorator используют композицию, и чем они отличаются?

**↳ Ответ:** Strategy — заменяет алгоритм целиком: контекст хранит одну стратегию и делегирует ей весь алгоритм (либо сортировка по имени, либо по дате). Decorator — оборачивает объект и добавляет поведение вокруг существующей логики, сохраняя тот же интерфейс: и логирование, и кэширование, и основная логика — все присутствуют в цепочке. Правило: если нужно выбрать один из алгоритмов — Strategy; если нужно добавить несколько слоёв поведения вокруг одного объекта — Decorator.

---

## 12. Что такое делегирование в ООП?

Делегирование — паттерн, при котором объект передаёт выполнение запроса другому объекту (делегату) вместо того, чтобы обрабатывать его самостоятельно. Это основной механизм реализации композиции: класс «имеет» делегата и перенаправляет вызовы к нему. Делегирование — альтернатива наследованию для переиспользования поведения.

Ключевое отличие от простой агрегации: при делегировании объект явно передаёт себя (`this`) делегату или делегат знает об исходном контексте. В Java это часто просто цепочка вызовов через поле.

```java
// OrderService делегирует специализированные операции конкретным классам
@Service
public class OrderService {
    private final OrderRepository repository;          // делегат персистентности
    private final PaymentProcessor paymentProcessor;   // делегат оплаты
    private final InventoryService inventoryService;   // делегат склада
    private final OrderNotifier notifier;              // делегат нотификации

    public Order placeOrder(PlaceOrderCommand cmd) {
        inventoryService.reserve(cmd.getItems());      // делегируем резервирование
        Order order = Order.create(cmd);
        paymentProcessor.charge(order);                // делегируем оплату
        repository.save(order);                        // делегируем сохранение
        notifier.onOrderPlaced(order);                 // делегируем нотификацию
        return order;
    }
}
```

В Spring делегирование встречается в `@Transactional` прокси: Spring создаёт прокси, который делегирует вызов оригинальному бину, обернув его в транзакцию. `TransactionInterceptor` — делегат управления транзакцией.

❗ **Ловушка собеса:** «Чем делегирование отличается от простого вызова метода?» — принципиально механизм тот же, термин подчёркивает архитектурное намерение: класс сознательно не берёт ответственность на себя, а передаёт её специализированному объекту.

**→ Уточняющий вопрос:** Как паттерн Proxy использует делегирование, и как это связано с `@Transactional` в Spring?

**↳ Ответ:** Spring создаёт CGLIB-прокси для бинов с `@Transactional`: прокси-объект оборачивает оригинальный бин, перехватывает вызов метода, открывает транзакцию, делегирует вызов оригинальному объекту через `target.method()`, затем коммитит или роллбэчит. Это и есть Proxy паттерн с делегированием. Отсюда знаменитая ловушка: вызов `@Transactional` метода из другого метода того же класса — прокси обходится, транзакция не создаётся, потому что `this.method()` идёт напрямую к оригиналу, минуя прокси.

---

## 13. Как принцип Single Responsibility связан с cohesion?

Cohesion (связность) — степень, в которой методы и поля класса связаны между собой и работают над общей задачей. Высокая связность означает, что все элементы класса вносят вклад в одну цель. SRP — это практическая формулировка высокой связности: если у класса одна причина для изменения, значит все его методы обслуживают одну ответственность — высокая связность достигается автоматически.

Метрика LCOM (Lack of Cohesion of Methods) формально измеряет связность: если методы класса обращаются к непересекающимся наборам полей — низкая связность, класс нужно делить.

```java
// Низкая связность — UserService с несвязанными группами методов
public class UserService {
    // Группа 1: работа с профилем (поля: name, email, avatar)
    public void updateProfile(Long id, String name) { ... }
    public void uploadAvatar(Long id, byte[] image) { ... }

    // Группа 2: аутентификация (поля: passwordHash, lastLogin, sessions)
    public boolean authenticate(String email, String password) { ... }
    public void logout(Long id) { ... }

    // Группа 3: биллинг (поля: subscriptionPlan, paymentMethod)
    public void upgradePlan(Long id, Plan plan) { ... }
    public void processPayment(Long id, BigDecimal amount) { ... }
}
```

Три группы методов с непересекающимися полями → три класса: `UserProfileService`, `AuthenticationService`, `BillingService`. У каждого высокая связность и одна причина для изменения.

❗ **Ловушка собеса:** путают high cohesion с tight coupling. Высокая связность (cohesion) — хорошо, внутри класса. Слабая связанность (low coupling) — хорошо, между классами. Это разные метрики, оба важны.

**→ Уточняющий вопрос:** Что такое LCOM и как он помогает обнаружить нарушения SRP автоматически?

**↳ Ответ:** LCOM (Lack of Cohesion of Methods) — метрика, показывающая, насколько методы класса «разрознены»: высокий LCOM означает, что методы обращаются к непересекающимся наборам полей — признак класса с несколькими ответственностями. Инструменты: SonarQube, IntelliJ IDEA (Metrics plugin), SpotBugs измеряют LCOM автоматически. Правило: если LCOM > 1 и в классе чётко видны несвязанные группы полей — Extract Class по Фаулеру.

---

## 14. Что произойдёт, если класс имеет несколько причин для изменения?

При нескольких причинах для изменения один класс становится точкой пересечения несвязанных требований. Это ведёт к нескольким практическим проблемам: параллельные изменения от разных команд создают конфликты при мёрже; изменение для одной цели случайно ломает другую функциональность («регрессия»); тесты разрастаются и теряют фокус.

Со временем такой класс превращается в God Object: сотни строк, десятки методов, нетривиальные взаимозависимости между ними. Новый разработчик не может понять класс без изучения всего контекста. Любое изменение требует осторожности — неизвестно, что ещё может сломаться.

```java
// Проблема: команда A меняет правила скидок, команда B меняет формат отчёта
// Оба изменения идут в один класс — конфликты, риск регрессий
public class OrderProcessor {
    public void process(Order order) { /* бизнес-логика */ }
    public BigDecimal calculateDiscount(Order order) { /* финансы */ }
    public String generateReport(Order order) { /* отчётность */ }
    public void sendConfirmation(Order order) { /* нотификации */ }
    public void updateInventory(Order order) { /* склад */ }
}

// После SRP-рефакторинга каждая команда работает в своём классе
// OrderProcessor → делегирует → DiscountService, ReportService, NotificationService, InventoryService
```

Дополнительный эффект: тесты превращаются в интеграционные без намерения. Чтобы протестировать расчёт скидки, нужно создать заглушки для email-отправки и отчётов — всё в одном классе.

❗ **Ловушка собеса:** «Как вы обнаруживаете нарушение SRP на ревью?» — хорошие индикаторы: класс длиннее 200–300 строк, метод длиннее 20–30 строк, большое количество инжектируемых зависимостей (>5), тест требует создания сложного fixture.

**→ Уточняющий вопрос:** Как количество инжектируемых зависимостей в конструкторе служит индикатором нарушения SRP?

**↳ Ответ:** Если конструктор принимает более 5–6 зависимостей — высокая вероятность нарушения SRP: класс координирует слишком много разных областей. Более точный индикатор: если разные методы класса используют непересекающиеся подмножества зависимостей (`methodA` использует `repo + validator`, `methodB` использует `emailSender + formatter`) — это два класса в одном. Правило: «конструктор как запах кода» — больше 5 параметров означает стоп и анализ.

---

## 15. Как SOLID помогает в тестировании кода?

SOLID и тестируемость взаимосвязаны напрямую: принципы проектирования, снижающие связанность, автоматически делают код удобным для unit-тестирования. SRP гарантирует, что класс делает одно — его тест проверяет одно, без необходимости настраивать несвязанные зависимости.

DIP — ключевой принцип для тестируемости: зависимость от интерфейсов позволяет подменять реализации моками. Без DIP тест `OrderService` потребует реальной базы данных и SMTP-сервера. С DIP — `OrderRepository` и `NotificationService` мокируются за секунды.

```java
// DIP + SRP = лёгкое тестирование
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock private OrderRepository orderRepository;    // мок интерфейса (DIP)
    @Mock private PaymentProcessor paymentProcessor; // мок интерфейса (DIP)
    @Mock private OrderNotifier notifier;            // мок интерфейса (DIP)

    @InjectMocks private OrderService orderService;  // тестируем один класс (SRP)

    @Test
    void shouldSaveOrderAfterSuccessfulPayment() {
        // Arrange
        CreateOrderCommand cmd = CreateOrderCommand.of(1L, List.of(item()));
        // Act
        orderService.placeOrder(cmd);
        // Assert — проверяем только одну ответственность
        verify(orderRepository).save(any(Order.class));
    }
}
```

ISP делает моки компактными: мок реализует только нужный интерфейс, а не жирный. LSP гарантирует, что мок-замена не нарушает контракт — тест остаётся релевантным. OCP позволяет тестировать каждую стратегию/политику отдельно.

❗ **Ловушка собеса:** «Если код плохо тестируется — это признак плохой архитектуры». Это верно: трудность написания unit-теста — прямой симптом нарушения SOLID (особенно DIP и SRP). Это называют «тестируемость как дизайн-ограничение».

**→ Уточняющий вопрос:** Что такое Test-Driven Development и как он *автоматически* приводит к SOLID-дизайну?

**↳ Ответ:** TDD — цикл «Red → Green → Refactor»: сначала пишешь failing тест, потом минимальный код для его прохождения, потом рефакторишь. Автоматически приводит к SOLID: чтобы написать тест до кода, нужно думать об интерфейсах (DIP), иначе тест невозможно написать без поднятия базы; один тест = одно поведение (SRP); трудность написания теста — немедленный feedback о нарушении принципов. TDD — «дизайн-инструмент», а не только способ получить тестовое покрытие.

---

## 16. Как принцип Dependency Inversion связан с Dependency Injection?

DIP (Dependency Inversion Principle) — архитектурный принцип: зависеть от абстракций, а не от конкретных реализаций. DI (Dependency Injection) — паттерн реализации: зависимости передаются объекту извне (через конструктор, setter или поле), а не создаются внутри. Это разные уровни абстракции: DIP — «что», DI — «как».

Связь: DI — наиболее удобный способ реализовать DIP. Если внедрять зависимости через конструктор типами интерфейсов — DIP соблюдён. Если внедрять конкретные классы — DI есть, DIP нарушен. Spring IoC Container реализует DI, но именно разработчик отвечает за то, чтобы типы зависимостей были интерфейсами (DIP).

```java
// DI без DIP: зависимость передаётся извне, но это конкретный класс
@Service
public class OrderService {
    private final MySqlOrderRepository repository;  // конкретный класс — DIP нарушен

    public OrderService(MySqlOrderRepository repository) {  // DI есть
        this.repository = repository;
    }
}

// DI + DIP: зависимость — интерфейс, подменяется в тестах и при смене реализации
@Service
public class OrderService {
    private final OrderRepository repository;  // интерфейс — DIP соблюдён

    public OrderService(OrderRepository repository) {  // DI
        this.repository = repository;
    }
}

// Spring context
@Configuration
public class AppConfig {
    @Bean
    public OrderRepository orderRepository(JpaOrderRepository impl) {
        return impl;  // Spring подставит конкретную реализацию
    }
}
```

В Spring: `@Autowired` реализует DI. DIP соблюдается, когда тип поля/параметра конструктора — интерфейс. Spring поощряет DIP через Convention over Configuration: `@Repository`, `@Service` — обычно бины, скрытые за интерфейсами.

❗ **Ловушка собеса:** «Spring сам делает DIP?» — нет. Spring даёт DI-инфраструктуру. DIP — ваш выбор типа зависимости. Можно написать `@Autowired MySqlUserRepository repo` — Spring инжектирует, но DIP нарушен.

**→ Уточняющий вопрос:** Чем constructor injection предпочтительнее field injection (`@Autowired` на поле) с точки зрения SOLID и тестируемости?

**↳ Ответ:** Constructor injection делает зависимости явными и неизменяемыми (`final`): класс нельзя создать без зависимостей — это гарантия корректности. Field injection скрывает зависимости (не видно из конструктора), позволяет создать объект в невалидном состоянии, и без Spring-контейнера поле останется null — тест без Mockito runner сломается. С constructor injection тест пишется как `new OrderService(mockRepo, mockNotifier)` — никакого фреймворка не нужно. Это прямое следствие DIP и SRP.

---

## 17. Что такое Law of Demeter (принцип наименьшего знания)?

Law of Demeter (LoD) — объект должен взаимодействовать только с «ближайшими друзьями»: своими полями, параметрами методов, объектами, которые он сам создал, и глобальными объектами. Нельзя «путешествовать» по цепочке объектов: `order.getCustomer().getAddress().getCity()` — это нарушение. LoD иногда называют «не разговаривай с незнакомцами».

Нарушение LoD создаёт скрытые зависимости: `OrderService` де-факто знает о внутренней структуре `Customer` и `Address`. Изменение `Customer` (например, переименование `getAddress()` в `getShippingAddress()`) сломает `OrderService`, хотя он якобы работает только с `Order`.

```java
// Нарушение Law of Demeter — цепочка вызовов раскрывает внутренности
@Service
public class ShippingService {
    public void ship(Order order) {
        // OrderService знает о структуре Customer, Address — скрытая зависимость
        String city = order.getCustomer().getAddress().getCity();
        String zip  = order.getCustomer().getAddress().getZipCode();
        courier.schedule(city, zip, order.getId());
    }
}

// Соблюдение LoD: Order предоставляет нужную информацию напрямую
public class Order {
    private Customer customer;

    public ShippingAddress getShippingAddress() {  // Order сам извлекает нужное
        return customer.getAddress().toShippingAddress();
    }
}

@Service
public class ShippingService {
    public void ship(Order order) {
        ShippingAddress address = order.getShippingAddress();  // один вызов
        courier.schedule(address.getCity(), address.getZipCode(), order.getId());
    }
}
```

LoD тесно связан с DIP и инкапсуляцией: нарушение LoD часто означает, что объект знает слишком много о деталях других объектов, что создаёт хрупкую связанность.

❗ **Ловушка собеса:** «А builder-паттерн не нарушает LoD — там же `user.builder().name("x").email("y").build()`?» — нет, это fluent interface на одном объекте-строителе, не разные объекты предметной области.

**→ Уточняющий вопрос:** Как нарушение Law of Demeter влияет на покрытие тестами и сложность моков?

**↳ Ответ:** При `order.getCustomer().getAddress().getCity()` тест должен настроить цепочку моков: `when(order.getCustomer()).thenReturn(mockCustomer)`, `when(mockCustomer.getAddress()).thenReturn(mockAddress)`, `when(mockAddress.getCity()).thenReturn("Moscow")` — три мока для одной строки кода. Это называют «train wreck mocking». Если LoD соблюдён и `order.getShippingCity()` — один мок. Правило: чем длиннее цепочка вызовов в коде — тем сложнее его тест. Сложность теста — индикатор нарушения LoD.

---

## 18. Как рефакторить God Object (божественный объект)?

God Object — класс, который знает слишком много и делает слишком много: содержит бизнес-логику, управляет состоянием, взаимодействует с БД, форматирует вывод, отправляет письма. Часто это исторически разросшийся сервис или контроллер. Признаки: 500+ строк, 10+ инжектируемых зависимостей, методы с несвязанными названиями.

Стратегия рефакторинга — «Extract Class» итеративно, с тестами на каждом шаге:

1. **Инвентаризация**: сгруппировать методы по тематике (работа с платежами, уведомления, отчёты, валидация).
2. **Extract Class**: для каждой группы создать отдельный класс, перенести методы.
3. **Делегирование**: в God Object оставить вызовы-делегаты (не ломать API).
4. **Замена делегатов**: постепенно переключить клиентов на новые классы.
5. **Удаление делегатов**: когда God Object опустел — удалить.

```java
// God Object: OrderManager делает всё
public class OrderManager {
    // 15 инжектируемых зависимостей...
    public void createOrder(...) { /* 50 строк */ }
    public void cancelOrder(...) { /* 30 строк */ }
    public void calculateTax(...) { /* 40 строк */ }
    public void generateInvoice(...) { /* 60 строк */ }
    public void sendConfirmationEmail(...) { /* 20 строк */ }
    public void updateInventory(...) { /* 30 строк */ }
    public void processRefund(...) { /* 45 строк */ }
    // ... ещё 10 методов
}

// После рефакторинга: OrderService — координатор, делегирует специалистам
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final TaxCalculationService taxService;       // выделен
    private final InvoiceService invoiceService;          // выделен
    private final OrderNotificationService notifier;      // выделен
    private final InventoryService inventoryService;      // выделен
    private final RefundService refundService;            // выделен

    public Order createOrder(CreateOrderCommand cmd) {
        BigDecimal tax = taxService.calculate(cmd);
        Order order = Order.create(cmd, tax);
        inventoryService.reserve(order);
        orderRepository.save(order);
        notifier.onOrderCreated(order);
        return order;
    }
}
```

❗ **Ловушка собеса:** «А если у OrderService 6 зависимостей — это тоже God Object?» — не обязательно, если все зависимости используются для одной бизнес-операции (создание заказа) и класс является *координатором*, а не исполнителем. Признак проблемы — когда методы класса используют непересекающиеся подмножества зависимостей.

**→ Уточняющий вопрос:** Какой рефакторинг из книги «Refactoring» Фаулера описывает выделение класса из God Object?

**↳ Ответ:** «Extract Class» — один из ключевых рефакторингов Фаулера: когда два класса «спрятаны» в одном, нужно выделить один из них в самостоятельный класс. Алгоритм: определить связанную группу полей и методов, создать новый класс, перенести поля и методы, оставить делегирующие методы в исходном классе, постепенно переключить клиентов. Также применимы «Move Method» и «Move Field» для перемещения отдельных элементов без полного Extract Class.

---

## 19. Как SOLID принципы помогают при расширении функционала?

SOLID создаёт «точки расширения» — места, где новый функционал добавляется без правки существующего кода. OCP напрямую формулирует это требование: расширение без модификации. На практике это означает, что добавление нового типа платежа, нового канала нотификации или нового правила скидки — это добавление нового класса, а не редактирование существующих.

DIP усиливает OCP: зависимость от интерфейса — это «розетка» для новой реализации. SRP обеспечивает, что каждая точка расширения изолирована. ISP гарантирует, что новая реализация реализует только нужные методы, а не весь жирный интерфейс.

```java
// Расширение: добавляем SMS-нотификацию без правки существующего кода
public interface OrderNotificationChannel {
    void notify(Order order, NotificationEvent event);
    boolean supports(NotificationEvent event);
}

@Component
public class EmailOrderNotifier implements OrderNotificationChannel {
    public void notify(Order order, NotificationEvent event) { /* email */ }
    public boolean supports(NotificationEvent event) { return true; }
}

// Новый канал: просто новый класс, ничего старое не трогаем
@Component
public class SmsOrderNotifier implements OrderNotificationChannel {
    public void notify(Order order, NotificationEvent event) { /* SMS */ }
    public boolean supports(NotificationEvent event) {
        return event == NotificationEvent.ORDER_SHIPPED;
    }
}

// NotificationService автоматически подберёт все реализации через Spring DI
@Service
public class OrderNotificationService {
    private final List<OrderNotificationChannel> channels;  // Spring соберёт все бины

    public void notifyAll(Order order, NotificationEvent event) {
        channels.stream()
            .filter(c -> c.supports(event))
            .forEach(c -> c.notify(order, event));
    }
}
```

❗ **Ловушка собеса:** «Как заранее знать, где будут точки расширения?» — принципы применяются итеративно. YAGNI (You Aren't Gonna Need It): не делать расширяемость везде сразу. При первом нарушении OCP (второй `if` в `switch`) — рефакторить. Это называют «Open/Closed по факту», а не «по проекту».

**→ Уточняющий вопрос:** Как Spring использует `List<SomeInterface>` при инжекции для реализации OCP?

**↳ Ответ:** Spring при старте собирает все бины, реализующие `SomeInterface`, и инжектирует их как `List<SomeInterface>` — в порядке, определяемом `@Order` или `Ordered`. Это означает: добавить новый `@Component`-бин = автоматически попасть в список, без правки места, где он используется. Это прямая реализация OCP через Spring: «закрытый» сервис получает новые реализации через открытый механизм коллекции бинов. Контроль порядка: `@Order(1)`, `@Order(2)` или реализация `Ordered`.

---

## 20. Можно ли следовать всем принципам SOLID одновременно?

Да, в идеальном смысле принципы совместимы и дополняют друг друга. Но на практике строгое следование всем пяти принципам одновременно в каждом классе может привести к over-engineering: избыточному числу интерфейсов, классов и абстракций для простых задач. Это компромисс между «правильной» архитектурой и «прагматичной».

Напряжения между принципами: ISP толкает к мелким интерфейсам, но слишком много мелких интерфейсов усложняет навигацию по коду. OCP требует заготовить точки расширения заранее, но YAGNI говорит не усложнять. SRP дробит на мелкие классы, но слишком мелкие классы затрудняют понимание потока выполнения.

Прагматичный подход: применять принципы там, где они решают реальную проблему. Хорошая эвристика — «правило трёх»: когда третий раз меняешь одно и то же место по одной причине, вводи абстракцию. Для CRUD-сервиса на 3 метода пять уровней абстракции — вред.

```java
// Прагматичный CRUD — не нарушение SOLID, а здравый смысл
@RestController
@RequestMapping("/api/categories")
public class CategoryController {
    private final CategoryRepository categoryRepository;  // прямо JPA, без лишних слоёв

    public List<Category> getAll() { return categoryRepository.findAll(); }
    public Category getById(@PathVariable Long id) { return categoryRepository.findById(id)... }
    public Category create(@RequestBody CategoryDto dto) { ... }
}
```

❗ **Ловушка собеса:** «SOLID — это абсолютные правила?» — нет. Мартин сам говорит о принципах, а не о законах. Зрелый разработчик знает, *когда* применить принцип и *когда* осознанно нарушить. Ключевое слово: осознанно, с пониманием последствий.

**→ Уточняющий вопрос:** Что такое «архитектурные решения» (Architectural Decision Records) и как они помогают документировать осознанные компромиссы с SOLID?

**↳ Ответ:** ADR (Architectural Decision Record) — короткий документ (Markdown-файл в репо), фиксирующий: контекст, принятое решение, альтернативы, последствия. Например: «Решили не выносить логику маппинга в отдельный Mapper — нарушение SRP. Причина: слой очень прост, добавление класса усложнит навигацию без выгоды. Последствие: пересмотреть при росте маппинга.» Это делает компромисс явным и обоснованным, а не случайным. Инструменты: `adr-tools`, папка `docs/adr/` в репозитории.

---

## 21. Как определить, что класс имеет одну ответственность?

Несколько практических тестов: (1) **тест одного предложения** — опишите класс одним предложением без «и», «или», «а также». Если не получается — несколько ответственностей. (2) **тест изменений** — назовите все причины, по которым класс может измениться: если разные бизнес-роли или команды — нарушение SRP. (3) **тест полей** — разделяются ли поля класса на группы, которые используются в разных наборах методов (метрика LCOM)?

```java
// Тест одного предложения:
// "UserService управляет бизнес-логикой пользователей" — ОК
// "UserService управляет пользователями И шлёт письма И генерирует отчёты" — нарушение

// Тест изменений:
// Кто может попросить изменить класс?
// - Product Owner: изменить правила регистрации → бизнес-логика
// - Email-маркетолог: изменить шаблон письма → нотификация
// - DBA: изменить схему → персистентность
// Три разные роли → три ответственности

// Тест полей (LCOM):
public class OrderService {
    // Группа А — методы process/cancel используют эти поля
    private OrderRepository orderRepository;
    private PaymentProcessor paymentProcessor;

    // Группа Б — методы generateReport/exportCsv используют эти поля
    private ReportFormatter formatter;
    private CsvExporter csvExporter;

    public void processOrder(Order o) { orderRepository.save(o); paymentProcessor.charge(o); }
    public void cancelOrder(Long id) { /* использует orderRepository */ }

    public String generateReport(DateRange range) { /* использует formatter */ }
    public byte[] exportCsv(DateRange range) { /* использует csvExporter */ }
}
// Два несвязанных набора полей → класс нарушает SRP
```

На code review: более 5 инжектируемых зависимостей, тестовый класс требует моков для несвязанных вещей, класс импортирует и `javax.persistence`, и `javax.mail` — это сигналы.

❗ **Ловушка собеса:** «Всегда ли один публичный метод = одна ответственность?» — нет. `UserService` с методами `createUser`, `updateUser`, `findUserById`, `deactivateUser` — четыре метода, одна ответственность (управление жизненным циклом пользователя).

**→ Уточняющий вопрос:** Как принцип Single Responsibility соотносится с Domain-Driven Design — конкретно с понятием Aggregate?

**↳ Ответ:** Aggregate в DDD — это кластер связанных объектов с единой точкой входа (Aggregate Root), изменяемых только как единое целое. SRP на уровне агрегата: у него одна причина для изменения — изменение бизнес-правил его домена. Order Aggregate содержит OrderLines и ShippingAddress — это один домен, одна ответственность. Если агрегат начинает управлять и заказом, и оплатой, и нотификациями — нарушение и SRP, и границы агрегата одновременно. Границы агрегата = границы SRP на доменном уровне.

---

## 22. Какие антипаттерны противоречат SOLID принципам?

Каждый принцип SOLID имеет антипаттерны-нарушители:

**Нарушает SRP:**
- **God Object / God Class** — один класс делает всё: бизнес-логика, персистентность, нотификации.
- **Blob** — разросшийся контроллер или сервис, аналог God Object.

**Нарушает OCP:**
- **Switch Statements по типу** — `if (type == A) ... else if (type == B)` вместо полиморфизма.
- **Hardcoded Variants** — жёстко прошитые варианты поведения без точек расширения.

**Нарушает LSP:**
- **Refused Bequest** — подкласс наследует от родителя, но переопределяет методы через `throw new UnsupportedOperationException()` или игнорирует контракт.
- **Type Checking** — `instanceof` проверки перед вызовом метода (признак, что подтип не соответствует контракту).

**Нарушает ISP:**
- **Fat Interface / Header Interface** — один огромный интерфейс со всеми методами системы.
- **Polluted Interface** — реализации вынуждены делать `return null` или `throw` для ненужных методов.

**Нарушает DIP:**
- **Concrete Class Dependency** — зависимость от конкретных классов (`new MySqlRepository()`) вместо интерфейсов.
- **Service Locator** (антипаттерн DI) — `ServiceLocator.get(OrderRepository.class)` скрывает зависимости, затрудняет тестирование.

```java
// Refused Bequest — нарушение LSP
public class ReadOnlyOrderRepository extends JpaOrderRepository {
    @Override
    public void save(Order order) {
        throw new UnsupportedOperationException("Read-only!");  // нарушает контракт
    }
}

// Правильно: отдельный интерфейс
public interface ReadableOrderRepository {
    Optional<Order> findById(Long id);
    List<Order> findAll();
}
// ReadOnlyOrderRepository реализует ReadableOrderRepository, а не расширяет JpaOrderRepository
```

❗ **Ловушка собеса:** «Anemic Domain Model — это нарушение SOLID?» — косвенно да: анемичная модель нарушает SRP и DIP, концентрируя всю логику в сервисах, а объекты домена становятся просто структурами данных. Это противоречит принципу «логика должна быть там, где данные».

**→ Уточняющий вопрос:** Чем Anemic Domain Model отличается от Rich Domain Model и как это влияет на применение SRP в сервисном слое?

**↳ Ответ:** Anemic Domain Model — объекты домена содержат только поля и геттеры/сеттеры, вся бизнес-логика в Service-классах. Rich Domain Model — бизнес-логика инкапсулирована внутри самих объектов (`order.cancel()`, `money.add(other)`). При анемичной модели сервисный слой нарушает SRP: один `OrderService` выполняет и расчёт скидки, и валидацию, и управление статусом — это всё должно быть в Order. Правило: если видите много бизнес-логики в сервисах и только данные в entities — это анемия, и SRP нарушен на доменном уровне.
