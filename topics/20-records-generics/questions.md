# Records и Дженерики

> 📇 Справочник уровня middle. Формат: ответ по сути → пример при необходимости → ❗ на чём ловят на собесе.

**Всего вопросов: 27**

---

## 1. Что такое Record в Java и с какой версии они доступны?

Record — специальный вид класса, предназначенный для хранения неизменяемых данных. Компилятор автоматически генерирует канонический конструктор, аксессоры (по имени компонента, не `getXxx()`), а также `equals()`, `hashCode()` и `toString()`. Record неявно `final` и наследует `java.lang.Record`. Это прямая замена boilerplate-heavy POJO в роли DTO, value object или tuple.

Стабильно доступны с Java 16 (JEP 395). Preview-режим — с Java 14. До records команды использовали Lombok `@Value`, но records — это языковой примитив без дополнительных зависимостей.

```java
// Вместо 50 строк POJO + Lombok
record Point(double x, double y) {}

// Использование
Point p = new Point(1.0, 2.0);
System.out.println(p.x());   // аксессор: x(), а не getX()
System.out.println(p);       // Point[x=1.0, y=2.0]
```

❗ Ловушка на собесе: перепутать версию — говорят «с Java 14» (это preview). Правильный ответ: стабильно с Java 16. Также: аксессоры называются `name()`, а не `getName()` — классическая ошибка в коде.

**→ Уточняющий вопрос:** Чем record отличается от `@Value` Lombok, и почему record предпочтительнее в новых проектах?

---

## 2. В чём основные отличия Record от обычного класса?

Record имеет жёсткие ограничения относительно обычного класса: все компоненты (поля из заголовка) неявно `private final`; нельзя добавить нестатические поля вне заголовка; нет сеттеров; сам record неявно `final`, от него нельзя наследоваться; он уже наследует `java.lang.Record` и не может наследовать другой класс. Взамен — автогенерация всего boilerplate.

В Spring-приложениях record идеально подходит как DTO для слоя API (входящие/исходящие данные контроллера), как внутренние value objects (деньги, координаты, диапазоны дат), как ключи в Map. Не подходит для JPA-сущностей (нужны изменяемые поля и конструктор без аргументов).

```java
// Record как DTO в Spring REST
record CreateOrderRequest(String userId, List<String> itemIds, BigDecimal budget) {}

record OrderResponse(String orderId, String status, BigDecimal total) {}

@PostMapping("/orders")
public OrderResponse create(@RequestBody CreateOrderRequest request) {
    Order order = orderService.create(request.userId(), request.itemIds(), request.budget());
    return new OrderResponse(order.getId(), order.getStatus().name(), order.getTotal());
}
```

❗ Ловушка: попытка использовать record как JPA entity упадёт на старте (`@Entity` требует non-final класс с no-arg конструктором). Это частый вопрос — «почему record не работает с Hibernate?».

**→ Уточняющий вопрос:** Почему record нельзя использовать как JPA entity, и что использовать вместо него для маппинга из БД?

---

## 3. Можно ли наследоваться от Record или наследовать Record от другого класса?

Record неявно `final` — от него наследоваться нельзя. Компилятор запрещает это на уровне синтаксиса. Record также не может расширять другой класс (`extends`), потому что он уже неявно наследует `java.lang.Record`. Это принципиальное дизайн-решение: record — это «прозрачный носитель данных», без иерархий наследования.

Однако record **может** реализовывать интерфейсы (`implements`). Это единственный механизм расширяемости — через интерфейсы. Это позволяет использовать record в паттернах с полиморфизмом.

```java
interface Printable {
    String formatted();
}

interface Validatable {
    boolean isValid();
}

record Money(BigDecimal amount, String currency)
    implements Printable, Validatable {

    // компактный конструктор — валидация
    Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount cannot be negative");
    }

    @Override
    public String formatted() {
        return amount.toPlainString() + " " + currency;
    }

    @Override
    public boolean isValid() {
        return amount != null && currency != null && currency.length() == 3;
    }
}
```

❗ Ловушка: «запретить наследование — это ограничение». На собесе правильный ответ — это осознанное решение. Иерархия наследования для носителей данных создаёт хрупкие конструкции (проблема Rectangle/Square). Record поощряет использование интерфейсов вместо классовой иерархии.

**→ Уточняющий вопрос:** Как sealed interfaces + records используются вместе для моделирования алгебраических типов данных (ADT) в Java?

---

## 4. Можно ли добавлять дополнительные методы в Record?

Да — record поддерживает дополнительные методы. Можно добавлять: обычные методы экземпляра (instance methods), статические методы, статические поля, реализации методов интерфейсов. Нельзя: добавлять нестатические поля экземпляра вне компонентов заголовка (это нарушило бы прозрачность record).

Можно также переопределить автогенерированные методы — `equals`, `hashCode`, `toString` и аксессоры. Например, `toString` может быть переопределён для скрытия чувствительных данных.

```java
record Product(String sku, String name, BigDecimal price, boolean active) {

    // Статическая фабрика
    static Product create(String sku, String name, BigDecimal price) {
        return new Product(sku, name, price, true);
    }

    // Бизнес-метод
    boolean isExpensive() {
        return price.compareTo(new BigDecimal("10000")) > 0;
    }

    // Переопределение toString: скрываем цену в логах
    @Override
    public String toString() {
        return "Product[sku=%s, name=%s, active=%s]".formatted(sku, name, active);
    }

    // Статическое поле — OK
    static final BigDecimal VAT_RATE = new BigDecimal("0.20");

    BigDecimal priceWithVat() {
        return price.multiply(BigDecimal.ONE.add(VAT_RATE));
    }
}
```

❗ Ловушка: попытка объявить `private String fullDescription;` внутри record (нестатическое поле вне заголовка) — ошибка компиляции. Только компоненты в заголовке могут быть полями экземпляра.

**→ Уточняющий вопрос:** Когда стоит переопределить аксессор в record — приведите пример использования?

---

## 5. Какие методы автоматически генерируются для Record?

Компилятор генерирует: **канонический конструктор** со всеми компонентами в качестве параметров; **аксессоры** — по одному на каждый компонент, с тем же именем (не `getXxx()`); **equals(Object)** — сравнивает все компоненты через их собственный `equals`; **hashCode()** — вычисляется на основе всех компонентов; **toString()** — возвращает имя record + все компоненты в формате `ClassName[field1=val1, field2=val2]`.

Важно: `equals` и `hashCode` основаны на **всех** компонентах без исключения. Если компонент — изменяемый объект (например, `List`), его изменение после создания record не повлияет на `hashCode` (record хранит ссылку), но может нарушить корректность семантики равенства.

```java
record User(Long id, String email, List<String> roles) {}

User u1 = new User(1L, "alice@example.com", List.of("ADMIN"));
User u2 = new User(1L, "alice@example.com", List.of("ADMIN"));

System.out.println(u1.equals(u2));     // true — все компоненты равны
System.out.println(u1.id());           // 1 — аксессор, не getId()
System.out.println(u1.email());        // alice@example.com
System.out.println(u1);
// User[id=1, email=alice@example.com, roles=[ADMIN]]
```

❗ Ловушка: аксессоры называются `id()`, `email()` — не `getId()`, `getEmail()`. Фреймворки (Jackson по умолчанию), ожидающие JavaBeans-конвенцию, могут не увидеть поля record. Jackson с Java 16+ поддерживает record нативно через `ConstructorPropertiesAnnotationIntrospector`.

**→ Уточняющий вопрос:** Как Jackson сериализует/десериализует Record, и нужна ли какая-то конфигурация?

---

## 6. Можно ли переопределить конструктор в Record?

Да, record поддерживает несколько форм конструкторов. **Канонический конструктор** — полная форма с явным присваиванием всех компонентов; подходит для валидации с доступом к исходным значениям. **Компактный конструктор** — сокращённая форма без списка параметров, присваивание делается автоматически компилятором после тела (идеален для валидации и нормализации). **Дополнительные конструкторы** — должны делегировать в канонический через `this(...)`.

```java
record Email(String value) {

    // Компактный конструктор — нормализация + валидация
    Email {
        Objects.requireNonNull(value, "Email cannot be null");
        value = value.trim().toLowerCase();  // нормализация до присваивания
        if (!value.contains("@"))
            throw new IllegalArgumentException("Invalid email: " + value);
    }

    // Дополнительный конструктор — должен вызвать канонический
    Email(String local, String domain) {
        this(local + "@" + domain);  // делегирует в канонический
    }
}

// Использование
Email e = new Email("  Alice@EXAMPLE.COM  ");
System.out.println(e.value()); // alice@example.com — нормализовано
```

❗ Ловушка: в компактном конструкторе нельзя вернуть значение через `return` и нельзя присваивать `this.field = ...` — присваивание делает компилятор автоматически после тела. Попытка присвоить `this.value = ...` — ошибка компиляции.

**→ Уточняющий вопрос:** Зачем использовать компактный конструктор вместо канонического — в чём практическая разница?

---

## 7. Что такое компактный конструктор (compact constructor) в Record?

Компактный конструктор — синтаксическое сокращение для канонического конструктора record, предназначенное для валидации и нормализации компонентов. Он объявляется без круглых скобок с параметрами — только имя record и тело в фигурных скобках. Все параметры доступны внутри тела как мутабельные переменные. Компилятор автоматически добавляет присваивание `this.x = x` после тела для всех компонентов.

Ключевая особенность: внутри тела компактного конструктора параметры ещё **мутабельны** — их можно переприсвоить для нормализации. Компилятор берёт финальные значения переменных и присваивает их полям.

```java
record DateRange(LocalDate start, LocalDate end) {

    // Компактный конструктор: валидация + нормализация
    DateRange {
        Objects.requireNonNull(start, "start");
        Objects.requireNonNull(end, "end");
        if (start.isAfter(end)) {
            // нормализация: если перепутали порядок — исправляем
            LocalDate tmp = start;
            start = end;  // переприсваиваем параметр (не this.start!)
            end = tmp;
        }
        // компилятор сгенерирует: this.start = start; this.end = end;
    }

    long days() {
        return ChronoUnit.DAYS.between(start, end);
    }
}

DateRange r = new DateRange(LocalDate.of(2024, 3, 1), LocalDate.of(2024, 1, 1));
System.out.println(r.start()); // 2024-01-01 — нормализовано
System.out.println(r.days());  // 60
```

❗ Ловушка: попытка написать `this.start = end` внутри компактного конструктора — ошибка компиляции. Нужно менять именно параметр (`start = end`), а не поле. Компилятор генерирует финальное присваивание полей после всего тела конструктора.

**→ Уточняющий вопрос:** Как обеспечить глубокую иммутабельность record, если один из компонентов — изменяемый `List`?

---

## 8. Можно ли объявлять статические поля и методы в Record?

Да, статические поля и методы в record разрешены без ограничений — ограничение касается только нестатических (instance) полей вне компонентов. Статические поля часто используются для констант или кэша. Статические фабричные методы — популярный паттерн для record, особенно когда нужна нормализация или множество «конструкторов» с разными именами.

```java
record Money(long amountInCents, String currency) {

    // Статические константы
    static final Money ZERO_USD = new Money(0L, "USD");
    static final Money ZERO_EUR = new Money(0L, "EUR");

    // Статическая фабрика — именованные конструкторы
    static Money ofDollars(double dollars) {
        return new Money(Math.round(dollars * 100), "USD");
    }

    static Money ofEuros(double euros) {
        return new Money(Math.round(euros * 100), "EUR");
    }

    // Статический метод-хелпер
    static Money sum(List<Money> amounts) {
        String currency = amounts.get(0).currency();
        long total = amounts.stream().mapToLong(Money::amountInCents).sum();
        return new Money(total, currency);
    }

    // Инстанс-метод
    Money add(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
        return new Money(amountInCents + other.amountInCents, currency);
    }
}

Money price = Money.ofDollars(19.99);
Money tax   = Money.ofDollars(1.60);
Money total = price.add(tax);
```

❗ Ловушка: `static final` поля инициализируются **до** создания экземпляров, поэтому цикличные ссылки (статическое поле record ссылается на сам record через `new`) работают корректно — это стандартный Java порядок инициализации классов.

**→ Уточняющий вопрос:** Как статические фабричные методы в record помогают избежать телескопических конструкторов?

---

## 9. Являются ли поля Record финальными?

Да, все компоненты record неявно `private final`. После создания экземпляра изменить их значение невозможно ни через рефлексию без `setAccessible`, ни через стандартный API. Это гарантирует shallow immutability — поля ссылки не изменяются, но объекты, на которые они ссылаются, могут быть изменяемыми.

Для deep immutability нужно дополнительно использовать неизменяемые коллекции. В компактном конструкторе можно превратить изменяемый `List` в неизменяемый через `List.copyOf()`.

```java
record UserProfile(String name, List<String> permissions) {
    // Компактный конструктор для deep immutability
    UserProfile {
        Objects.requireNonNull(name);
        permissions = List.copyOf(permissions); // defensive copy → неизменяемый список
    }
}

UserProfile profile = new UserProfile("Alice", new ArrayList<>(List.of("READ", "WRITE")));
// profile.permissions() — List.of, нельзя add/remove
// profile.name() — String, immutable by nature

// Попытка изменить через рефлексию
Field f = UserProfile.class.getDeclaredField("name");
f.setAccessible(true);
f.set(profile, "Bob"); // IllegalAccessException или работает, но нарушает семантику
```

❗ Ловушка: «record иммутабельный» — это правда только для shallow immutability. Если компонент — `ArrayList`, его содержимое можно изменить. На собесе часто спрашивают, как сделать record по-настоящему неизменяемым — ответ: `List.copyOf()` в компактном конструкторе.

**→ Уточняющий вопрос:** Чем `List.copyOf()` отличается от `Collections.unmodifiableList()` с точки зрения защиты данных?

---

## 10. Можно ли использовать Record как ключ в HashMap?

Да, и record является **идеальным** ключом в HashMap. Три условия для корректного ключа: иммутабельность (нельзя изменить объект после помещения в Map), корректный `equals()` (проверяет равенство по значению), корректный `hashCode()` согласованный с `equals`. Record выполняет все три автоматически.

Проблема с обычными POJO-ключами: если `equals`/`hashCode` генерируются только по id, а объект мутируется — Map теряет запись. С record это невозможно: поля `final`, `equals`/`hashCode` по всем компонентам.

```java
record Coordinate(double lat, double lon) {}

// Record как ключ в Map — безопасно и эффективно
Map<Coordinate, String> cityMap = new HashMap<>();
cityMap.put(new Coordinate(55.75, 37.62), "Moscow");
cityMap.put(new Coordinate(59.93, 30.32), "St. Petersburg");

// Поиск по значению — работает корректно, hashCode совпадает
String city = cityMap.get(new Coordinate(55.75, 37.62));
System.out.println(city); // Moscow — equals по значению

// Составной ключ в кэше сервиса
record CacheKey(String userId, String productId, String currency) {}
Map<CacheKey, Price> priceCache = new ConcurrentHashMap<>();
priceCache.put(new CacheKey("u1", "p1", "USD"), new Price(99.99));
```

❗ Ловушка: если компонент record — изменяемый объект (например, `Date`, `ArrayList`), и он изменяется после помещения в Map — `hashCode` может измениться, и запись «потеряется». Правило: используй только иммутабельные компоненты в record-ключах (String, Long, LocalDate, другие record).

**→ Уточняющий вопрос:** Почему record с изменяемым `List` компонентом опасно использовать как ключ HashMap, даже если `hashCode` вычислен при создании?

---

## 11. Что такое дженерики (Generics) в Java?

Дженерики — механизм параметрической полиморфизации типов: классы, интерфейсы и методы объявляются с типом-параметром (`<T>`), конкретный тип подставляется при использовании. Цель — типобезопасность на этапе компиляции без приведений типов в рантайме.

До дженериков (Java < 5) коллекции работали через `Object`, что требовало явных кастов и позволяло класть разные типы в одну коллекцию — ошибки обнаруживались только в рантайме. С дженериками компилятор знает тип элементов и выдаёт ошибку при попытке добавить некорректный тип.

```java
// До Java 5 — небезопасно
List list = new ArrayList();
list.add("hello");
list.add(42);           // компилятор не возражает
String s = (String) list.get(1); // ClassCastException в рантайме

// С дженериками — типобезопасно
List<String> strings = new ArrayList<>();
strings.add("hello");
// strings.add(42);     // ошибка компиляции — невозможно добавить Integer
String s2 = strings.get(0); // не нужен каст

// Дженерик-класс
class Box<T> {
    private T value;
    Box(T value) { this.value = value; }
    T get() { return value; }
}

Box<String> strBox = new Box<>("hello");
Box<Integer> intBox = new Box<>(42);
String v = strBox.get(); // нет каста
```

❗ Ловушка: дженерики существуют только на уровне компилятора — в рантайме информация о типе-параметре стирается (type erasure). `List<String>` и `List<Integer>` в байткоде — один и тот же класс `List`.

**→ Уточняющий вопрос:** Что такое type erasure, и почему именно он, а не сохранение типов, было выбрано для реализации дженериков в Java?

---

## 12. В чём преимущества использования дженериков?

Три главных преимущества. **Типобезопасность** — ошибки несовместимости типов обнаруживаются компилятором, а не в рантайме. **Отсутствие явных кастов** — вместо `(String) list.get(0)` пишем `list.get(0)`, код чище и безопаснее. **Переиспользуемость** — один класс или метод работает с любым типом, не нужно писать `StringList`, `IntegerList`, `OrderList`.

Дополнительно: дженерики документируют намерение. `List<User>` говорит: здесь только `User`. `Map<String, List<Order>>` — понятная структура без комментариев. Это живая документация в типах.

```java
// Дженерик-утилита: работает с любым типом Comparable
public static <T extends Comparable<T>> T max(List<T> list) {
    if (list.isEmpty()) throw new NoSuchElementException();
    T result = list.get(0);
    for (T item : list) {
        if (item.compareTo(result) > 0) result = item;
    }
    return result;
}

// Работает с любым типом, реализующим Comparable
Integer maxInt = max(List.of(1, 5, 3, 2));         // 5
String maxStr  = max(List.of("banana", "apple", "cherry")); // cherry
LocalDate maxDate = max(List.of(LocalDate.now(), LocalDate.of(2020, 1, 1)));
```

❗ Ловушка: кандидаты часто называют «производительность» как преимущество — это неверно. Дженерики работают через erasure, никаких оптимизаций в рантайме нет. Преимущества — исключительно в безопасности типов и читаемости кода на уровне компиляции.

**→ Уточняющий вопрос:** Какие ограничения накладывает type erasure на использование дженериков в рантайме?

---

## 13. Что такое type erasure (стирание типов)?

Type erasure — механизм реализации дженериков в Java: вся информация о параметрах типа существует только во время компиляции и **стирается** при генерации байткода. В байткоде `List<String>`, `List<Integer>`, `List<User>` — это просто `List`. Параметр типа заменяется на `Object` (unbounded) или на первый бound (для `<T extends Comparable<T>>` → `Comparable`).

Это решение было принято для **обратной совместимости** с кодом до Java 5: существующие скомпилированные библиотеки, работающие с raw `List`, должны были продолжать работать с новым кодом, использующим `List<String>`.

```java
// После компиляции — одно и то же в байткоде
List<String>  list1 = new ArrayList<>();
List<Integer> list2 = new ArrayList<>();

System.out.println(list1.getClass() == list2.getClass()); // true — оба ArrayList

// Что НЕЛЬЗЯ делать из-за erasure:
// 1. instanceof T
// 2. new T()
// 3. new T[]
// 4. List<String>.class  — только List.class

// Проверка в рантайме невозможна:
Object obj = list1;
if (obj instanceof List<String> ls) { // Java 16+ pattern matching
    // корректно — но только для raw type check!
}

// Обход erasure через токены типа
public <T> T fromJson(String json, Class<T> type) {
    return objectMapper.readValue(json, type); // Class<T> хранит тип явно
}
User user = fromJson(json, User.class); // OK
```

❗ Ловушка: нельзя написать `new T()` — компилятор не знает, какой конструктор вызвать, потому что в рантайме T не существует. Решение: передавать `Class<T>` или `Supplier<T>`. Это классический вопрос на собесах.

**→ Уточняющий вопрос:** Как обойти erasure при необходимости создать экземпляр типа T в дженерик-методе?

---

## 14. Можно ли создать массив дженерик-типа?

Создать массив с дженерик-параметром напрямую нельзя: `new T[]` — ошибка компиляции, `new List<String>[]` — тоже ошибка. Причина — двойное ограничение: из-за type erasure компилятор не может гарантировать типобезопасность массива в рантайме, а массивы в Java ковариантны (`String[]` является подтипом `Object[]`), что создаёт heap pollution.

Доказательство проблемы: если бы `new List<String>[]` было разрешено, то через ковариантность массивов можно было бы подложить `List<Integer>` в массив `List<String>[]` — классический `ClassCastException` без предупреждения.

```java
// Ошибка компиляции:
// T[] arr = new T[10];
// List<String>[] arr2 = new List<String>[5];

// Обходы:
// 1. Массив Object[] с кастом (unchecked, но часто используется в коллекциях)
@SuppressWarnings("unchecked")
T[] createArray(int size) {
    return (T[]) new Object[size]; // работает, но только если не экспортируется наружу
}

// 2. Через Class<T>
<T> T[] createTypedArray(Class<T> type, int size) {
    return (T[]) Array.newInstance(type, size); // создаёт реальный T[] в рантайме
}

String[] arr = createTypedArray(String.class, 5);

// 3. Использовать List вместо массива — предпочтительный вариант
List<T> createList(int initialCapacity) {
    return new ArrayList<>(initialCapacity); // проблем нет
}
```

❗ Ловушка: `ArrayList` внутри хранит `Object[]` и делает unchecked-каст — именно поэтому там `@SuppressWarnings("unchecked")`. Это internal implementation detail, который скрыт от пользователя за типобезопасным API.

**→ Уточняющий вопрос:** Почему массивы в Java ковариантны, а дженерики инвариантны — в чём разница по безопасности?

---

## 15. Что такое bounded type parameters?

Bounded type parameters — ограничения на параметр типа через `extends` или `super`. `<T extends Comparable<T>>` означает: T должен быть Comparable. Без ограничения T — это просто `Object`, и к нему нельзя вызвать ничего, кроме методов `Object`. С ограничением компилятор знает, что у T есть методы границы, и позволяет их вызывать.

Multiple bounds: `<T extends Serializable & Comparable<T>>` — T реализует оба интерфейса. Класс в bounds должен идти первым: `<T extends SomeClass & InterfaceA & InterfaceB>`.

```java
// Без bounds — только Object методы
public <T> T max(List<T> list) {
    // нельзя: list.get(0).compareTo(...)
    // нельзя сравнивать — T неизвестен
    return list.get(0);
}

// С bounds — компилятор знает, что T сравниваем
public <T extends Comparable<T>> T max(List<T> list) {
    T result = list.get(0);
    for (T item : list) {
        if (item.compareTo(result) > 0) result = item; // OK!
    }
    return result;
}

// Bounds на уровне класса
class SortedContainer<T extends Comparable<T> & Serializable> {
    private final TreeSet<T> data = new TreeSet<>();

    void add(T item) { data.add(item); }
    T first() { return data.first(); }
}

SortedContainer<Integer> container = new SortedContainer<>();
container.add(5); container.add(1); container.add(3);
System.out.println(container.first()); // 1
```

❗ Ловушка: `extends` в bounds — не то же самое, что `extends` в наследовании. `<T extends Number>` работает и для классов (Integer, Double) и для интерфейсов. В bounds не используется `implements` — только `extends` для обоих случаев.

**→ Уточняющий вопрос:** Как bounds стираются при компиляции — во что превращается `<T extends Comparable<T>>` в байткоде?

---

## 16. В чём разница между `<? extends T>` и `<? super T>`?

`<? extends T>` — wildcard «T или любой подтип T». Это **producer** позиция: из такой структуры можно **читать** данные как T, но **нельзя добавлять** (кроме null) — компилятор не знает, какой именно подтип. Аналогия: «тарелка с фруктами, но неизвестно с какими конкретно».

`<? super T>` — wildcard «T или любой супертип T». Это **consumer** позиция: в такую структуру можно **записывать** T (и его подтипы), но читать можно только как `Object` — компилятор знает, что там точно есть место для T, но не знает точный тип для чтения.

```java
// <? extends T> — Producer (чтение)
public double sumNumbers(List<? extends Number> list) {
    double sum = 0;
    for (Number n : list) sum += n.doubleValue(); // читаем как Number — OK
    // list.add(1.5);   // НЕЛЬЗЯ — не знаем точный тип списка
    return sum;
}
sumNumbers(List.of(1, 2, 3));    // List<Integer> — OK
sumNumbers(List.of(1.0, 2.0));   // List<Double> — OK

// <? super T> — Consumer (запись)
public void fillWithDefaults(List<? super Integer> list, int count) {
    for (int i = 0; i < count; i++) list.add(0); // записываем Integer — OK
    // Integer x = list.get(0); // НЕЛЬЗЯ — тип неизвестен (может быть Object)
    Object o = list.get(0); // только как Object
}
List<Number> numbers = new ArrayList<>();
fillWithDefaults(numbers, 5); // OK — Number супертип Integer
```

❗ Ловушка: частая путаница с направлением. Запомни: `extends` — читаем (но не пишем), `super` — пишем (но читаем только как Object). Или через PECS: Producer Extends, Consumer Super.

**→ Уточняющий вопрос:** Почему нельзя добавить элемент в `List<? extends Number>`, если там точно хранятся Number-совместимые объекты?

---

## 17. Что такое PECS (Producer Extends Consumer Super)?

PECS — мнемоника из книги Effective Java Джошуа Блоха для выбора wildcard: если параметризованный тип **производит** данные (вы читаете из него) — используй `<? extends T>`. Если **потребляет** данные (вы записываете в него) — используй `<? super T>`. Если и то и другое — обычный `<T>`.

Классический пример — `Collections.copy(destination, source)`: source — producer (из него читаем), destination — consumer (в него пишем). Именно поэтому сигнатура: `copy(List<? super T> dest, List<? extends T> src)`.

```java
// PECS в собственном коде
public <T> void transferElements(
        List<? extends T> source,       // Producer — читаем
        List<? super T>   destination) { // Consumer — пишем
    for (T element : source) {
        destination.add(element);
    }
}

// Реальный пример: Stack
class Stack<E> {
    // Добавить все из producer
    public void pushAll(Iterable<? extends E> source) {
        for (E e : source) push(e);
    }

    // Переложить всё в consumer
    public void popAll(Collection<? super E> destination) {
        while (!isEmpty()) destination.add(pop());
    }
}

Stack<Number> numStack = new Stack<>();
numStack.pushAll(List.of(1, 2, 3));           // List<Integer> extends Number — OK
List<Object> dest = new ArrayList<>();
numStack.popAll(dest);                         // List<Object> super Number — OK
```

❗ Ловушка: PECS применяется к **параметрам методов**, не к возвращаемым типам. Возвращаемый тип с wildcard (`List<? extends T>`) — плохая практика, вынуждает клиента думать о wildcard. Используй конкретный тип или `<T>` для возвращаемых значений.

**→ Уточняющий вопрос:** Почему методы с `<? extends T>` параметром безопаснее, чем с `<T>`, при работе с разными подтипами?

---

## 18. Можно ли использовать примитивные типы как параметры дженериков?

Нет — параметры типа дженериков в Java могут быть только ссылочными типами (Reference Types). `List<int>`, `Map<long, double>` — ошибки компиляции. Причина: дженерики после erasure оперируют `Object`, а примитивы не наследуют `Object`.

Для работы с примитивами используются wrapper-классы (`Integer`, `Long`, `Double` и т.д.) в сочетании с автобоксингом/автораспаковкой. Автобоксинг прозрачен для кода, но создаёт overhead — allocation объекта и unboxing при каждой операции.

```java
// Примитивы — нельзя
// List<int> ints = new ArrayList<>(); // ошибка компиляции

// Обёртки + автобоксинг — работает
List<Integer> ints = new ArrayList<>();
ints.add(42);           // автобоксинг: 42 → Integer.valueOf(42)
int value = ints.get(0); // автораспаковка: Integer → int

// Для примитивов в production используют специализированные коллекции:
// Eclipse Collections, Trove, koloboke — без бокинга
IntArrayList primitiveList = new IntArrayList();
primitiveList.add(1); primitiveList.add(2); // нет boxing!

// Java 21+: начало работы над Project Valhalla — дженерики над примитивами
// В будущем: List<int>, List<double> — но пока не готово в mainline
```

❗ Ловушка: автобоксинг в горячих путях — источник GC pressure. `List<Integer>` с миллионом элементов создаёт миллион Integer-объектов в heap. В production-коде с большими объёмами числовых данных это критично — используют `int[]`, `double[]` или специализированные библиотеки.

**→ Уточняющий вопрос:** Что такое Project Valhalla, и как он изменит ограничение на примитивы в дженериках?

---

## 19. Что такое raw types и почему их следует избегать?

Raw type — использование дженерик-класса без указания параметра типа: `List` вместо `List<String>`, `Map` вместо `Map<String, Integer>`. Это сохранено исключительно для обратной совместимости с кодом, написанным до Java 5. Raw types отключают все проверки типов компилятора.

При использовании raw type компилятор выдаёт предупреждение `unchecked` и не проверяет типы элементов. Результат — `ClassCastException` в рантайме там, где всё выглядит безобидно. Отладка таких ошибок крайне сложна.

```java
// Raw type — теряем типобезопасность
List rawList = new ArrayList();
rawList.add("hello");
rawList.add(42);         // компилятор молчит
String s = (String) rawList.get(1); // ClassCastException в рантайме!

// Правильно
List<String> stringList = new ArrayList<>();
stringList.add("hello");
// stringList.add(42);  // ошибка компиляции — защита!

// Опасный паттерн: передача raw в дженерик-метод
static void printList(List list) {           // raw parameter
    for (Object o : list) System.out.println(o);
}

List<Integer> numbers = List.of(1, 2, 3);
printList(numbers); // работает, но теряем типобезопасность
// правильно: List<?> или <T> List<T>

// Raw type в legacy API (приходится работать)
@SuppressWarnings("unchecked")
public <T> T getBean(String name) {
    return (T) applicationContext.getBean(name); // внутри Spring
}
```

❗ Ловушка: raw types «заражают» весь вызывающий код — если метод принимает `List`, компилятор снимает проверки и для всех вызовов, связанных с этим списком. Один raw type может подавить предупреждения компилятора для большого участка кода.

**→ Уточняющий вопрос:** Чем `List<?>` (wildcard) отличается от `List` (raw type) с точки зрения типобезопасности?

---

## 20. Что произойдёт при попытке создать экземпляр дженерик-типа через new T()?

`new T()` — ошибка компиляции. Из-за type erasure тип `T` в рантайме неизвестен, поэтому компилятор не знает, какой конструктор вызывать и какой класс инстанциировать. После erasure T превращается в `Object` (или в bound), вызов `new Object()` явно не то, что нужно.

Три способа обойти это ограничение:

```java
// 1. Через Class<T> — рефлексия
public class Factory<T> {
    private final Class<T> type;

    Factory(Class<T> type) { this.type = type; }

    T create() {
        try {
            return type.getDeclaredConstructor().newInstance();
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException("Cannot instantiate " + type, e);
        }
    }
}

Factory<ArrayList> factory = new Factory<>(ArrayList.class);
ArrayList list = factory.create(); // OK

// 2. Через Supplier<T> — предпочтительно, без рефлексии
public class Repository<T> {
    private final Supplier<T> constructor;

    Repository(Supplier<T> constructor) { this.constructor = constructor; }

    T newInstance() { return constructor.get(); }
}

Repository<User> repo = new Repository<>(User::new);
User user = repo.newInstance();

// 3. Через абстрактный метод (фабричный метод)
abstract class Cache<T> {
    protected abstract T createDefault();

    T getOrDefault(String key) {
        T cached = get(key);
        return cached != null ? cached : createDefault();
    }
}

class UserCache extends Cache<User> {
    protected User createDefault() { return new User(); }
}
```

❗ Ловушка: `Class.newInstance()` (без аргументов, старый API) устарел с Java 9 и проглатывает checked exceptions. Правильно: `getDeclaredConstructor().newInstance()` — явный и безопасный.

**→ Уточняющий вопрос:** Почему `Supplier<T>` предпочтительнее `Class<T>` для фабричного создания объектов в дженерик-коде?

---

## 21. В чём разница между `List<?>` и `List<Object>`?

`List<Object>` — список, параметризованный именно `Object`. Можно добавлять в него любые объекты (любой объект совместим с Object). Но `List<String>` **не** является подтипом `List<Object>` — это инвариантность дженериков. Метод, принимающий `List<Object>`, не примет `List<String>`.

`List<?>` — список неизвестного типа (wildcard). Из него можно читать только как `Object`, записывать нельзя (кроме null). Однако `List<String>`, `List<Integer>`, `List<User>` — все являются подтипами `List<?>`. Это делает `List<?>` полезным для read-only работы с любым списком.

```java
// List<Object> — НЕ принимает List<String>
static void printAll(List<Object> list) {
    list.forEach(System.out::println);
}
List<String> strings = List.of("a", "b");
// printAll(strings); // ОШИБКА компиляции — String[] не Object[]

// List<?> — принимает любой List (read-only)
static void printAll(List<?> list) {
    for (Object o : list) System.out.println(o); // читаем как Object — OK
    // list.add("hello"); // НЕЛЬЗЯ — неизвестный тип
    list.add(null);       // null можно — нет типа
}
printAll(strings);  // OK!
printAll(List.of(1, 2, 3)); // OK!

// List<Object> — можно добавлять
List<Object> objects = new ArrayList<>();
objects.add("string");
objects.add(42);
objects.add(new User());
```

❗ Ловушка: «раз `List<String>` нельзя передать в `List<Object>` — дженерики неудобны». Правильный ответ: это принципиальное решение безопасности. Если бы `List<String>` был подтипом `List<Object>`, можно было бы добавить Integer в List<String> через ссылку List<Object> — heap pollution. Wildcards решают задачу гибкости.

**→ Уточняющий вопрос:** Если `List<String>` не подтип `List<Object>`, то как объяснить, что `String[]` является подтипом `Object[]`?

---

## 22. Можно ли перегружать методы, отличающиеся только дженерик-параметрами?

Нет — перегрузка по дженерик-параметру невозможна, потому что после стирания типов оба метода имеют одинаковую сигнатуру. `process(List<String>)` и `process(List<Integer>)` после erasure становятся одинаковыми `process(List)` — компилятор выдаёт ошибку «method clash».

Это называется erasure collision. JVM не поддерживает два метода с одинаковой сигнатурой в одном классе — это ограничение байткода.

```java
public class Processor {
    // ОШИБКА КОМПИЛЯЦИИ: оба метода имеют одинаковую erasure
    // public void process(List<String> list) { ... }
    // public void process(List<Integer> list) { ... }
    // После erasure: process(List) и process(List) — конфликт

    // Решения:
    // 1. Разные имена методов
    public void processStrings(List<String> list) { ... }
    public void processIntegers(List<Integer> list) { ... }

    // 2. Дженерик-метод с bounded parameter
    public <T extends Number> void process(List<T> list) { ... }

    // 3. Перегрузка по другому параметру (OK — разные erasure)
    public void process(List<String> list, String delimiter) { ... }
    public void process(List<Integer> list, int base) { ... }
    // Эти — разные сигнатуры и после erasure

    // 4. Метод с wildcard для обоих
    public void process(List<?> list) {
        for (Object item : list) handle(item);
    }
}
```

❗ Ловушка: иногда перегрузка по дженерикам компилируется (если добавляется дополнительный параметр другого типа), но семантически запутывает. Лучше давать методам осмысленные имена. Ещё ловушка: `void foo(List<String>)` и `void foo(List)` — одна erasure, компилятор откажет.

**→ Уточняющий вопрос:** Как bridge methods помогают сохранить перегрузку при стирании типов в случае наследования?

---

## 23. Что такое recursive type bound?

Recursive type bound (рекурсивное ограничение типа) — параметр типа, ограниченный через самого себя: `<T extends Comparable<T>>`. Это означает: T должен быть сравниваем с самим собой. Паттерн особенно полезен для методов сортировки, поиска максимума/минимума, бинарного поиска.

Почему нельзя просто написать `<T extends Comparable>`? Потому что raw type `Comparable` теряет информацию о том, с чем сравнивается T. `Comparable<T>` гарантирует: T.compareTo(T) — правильное сравнение, не T.compareTo(Object).

```java
// Метод максимума — классика recursive type bound
public static <T extends Comparable<T>> T max(Collection<T> c) {
    if (c.isEmpty()) throw new NoSuchElementException();
    Iterator<T> i = c.iterator();
    T candidate = i.next();
    while (i.hasNext()) {
        T next = i.next();
        if (next.compareTo(candidate) > 0)
            candidate = next;
    }
    return candidate;
}

// Работает для любого Comparable
Integer maxInt = max(List.of(3, 1, 4, 1, 5, 9));     // 9
String maxStr  = max(Set.of("banana", "cherry"));      // cherry

// Реальный пример: Builder с рекурсивным bounds (Curiously Recurring)
abstract class Builder<T, B extends Builder<T, B>> {
    protected String name;

    @SuppressWarnings("unchecked")
    public B withName(String name) {
        this.name = name;
        return (B) this;   // возвращает конкретный подтип Builder
    }

    public abstract T build();
}

class UserBuilder extends Builder<User, UserBuilder> {
    private String email;
    public UserBuilder withEmail(String email) { this.email = email; return this; }
    public User build() { return new User(name, email); }
}

User user = new UserBuilder().withName("Alice").withEmail("a@b.com").build();
```

❗ Ловушка: `<T extends Comparable<T>>` не гарантирует, что T сравним с другими типами — только сам с собой. `Integer` extends `Number`, но `Integer.compareTo(Long)` не существует. `<T extends Comparable<T>>` — правильное ограничение для однотипного сравнения.

**→ Уточняющий вопрос:** Что такое Curiously Recurring Template Pattern (CRTP) в Java и как он реализуется через recursive type bounds?

---

## 24. Как работают дженерики с наследованием (является ли `List<String>` подтипом `List<Object>`)?

Нет — `List<String>` **не** является подтипом `List<Object>`, хотя `String` является подтипом `Object`. Дженерики **инвариантны** по умолчанию. Это принципиальное решение безопасности.

Доказательство необходимости инвариантности: если бы `List<String>` был подтипом `List<Object>`, можно было бы написать `List<Object> lo = new ArrayList<String>(); lo.add(42);` — и получить `ClassCastException` при попытке прочитать элемент как `String`. Эта проблема реально существует с массивами (они ковариантны) и называется `ArrayStoreException`.

```java
// Массивы — ковариантны (небезопасно!)
Object[] objs = new String[3];  // OK — String[] подтип Object[]
objs[0] = 42;  // ArrayStoreException в рантайме!

// Дженерики — инвариантны (безопасно)
// List<Object> lo = new ArrayList<String>(); // ОШИБКА компиляции

// Правило подтипирования для дженериков:
// List<Integer> — подтип List<Integer> только
// List<Integer> — НЕ подтип List<Number>, List<Object>

// Для гибкости — wildcards:
List<? extends Number> numbers = new ArrayList<Integer>(); // OK
List<? super Integer> integers = new ArrayList<Number>(); // OK

// Пример с использованием:
public void processNumbers(List<? extends Number> list) {
    // list может быть List<Integer>, List<Double>, List<Float>
    double sum = list.stream().mapToDouble(Number::doubleValue).sum();
}
processNumbers(List.of(1, 2, 3));    // List<Integer> — OK
processNumbers(List.of(1.0, 2.0));   // List<Double> — OK
```

❗ Ловушка: «дженерики инвариантны, массивы ковариантны» — это должен знать каждый middle. Классическая ошибка: передать `List<Integer>` туда, где ожидается `List<Number>`, и получить неожиданную ошибку компиляции.

**→ Уточняющий вопрос:** Как wildcards делают дженерики «ковариантными» или «контравариантными» в зависимости от `extends`/`super`?

---

## 25. Что такое bridge methods и зачем они нужны?

Bridge methods — синтетические методы, которые компилятор автоматически добавляет в байткод для сохранения корректности переопределения (override) при стирании типов. Без bridge methods полиморфизм ломался бы из-за erasure.

Классический случай: класс реализует `Comparable<MyClass>`. После erasure метод `compareTo(MyClass)` становится `compareTo(Object)`. Чтобы JVM могла вызвать нужный метод через ссылку `Comparable`, компилятор генерирует bridge метод `compareTo(Object)`, который делегирует в `compareTo(MyClass)` с кастом.

```java
class StringWrapper implements Comparable<StringWrapper> {
    private final String value;
    StringWrapper(String value) { this.value = value; }

    @Override
    public int compareTo(StringWrapper other) {  // конкретный метод
        return value.compareTo(other.value);
    }
    // Компилятор генерирует bridge:
    // public int compareTo(Object other) {
    //     return compareTo((StringWrapper) other);  // каст + делегирование
    // }
}

// Можно увидеть через javap -verbose StringWrapper.class
// Output includes:
//   public int compareTo(java.lang.StringWrapper);
//   public volatile int compareTo(java.lang.Object); // synthetic bridge

// Проверка через рефлексию
Method[] methods = StringWrapper.class.getDeclaredMethods();
for (Method m : methods) {
    System.out.println(m.getName() + " synthetic=" + m.isSynthetic() + " bridge=" + m.isBridge());
}
```

❗ Ловушка: bridge methods видны через рефлексию — `method.isBridge()` и `method.isSynthetic()` вернут `true`. Если вы перебираете методы через `getDeclaredMethods()` и не фильтруете bridge методы, можно случайно вызвать bridge вместо реального. Некоторые фреймворки (Spring, Hibernate) обрабатывают это.

**→ Уточняющий вопрос:** Почему Spring иногда «видит» лишние методы через рефлексию в проксированных классах, и как bridge methods к этому причастны?

---

## 26. Можно ли использовать несколько ограничений (bounds) для одного параметра типа?

Да — через `&`: `<T extends FirstClass & SecondInterface & ThirdInterface>`. Первый bound — класс (если есть), остальные — интерфейсы. Только один класс разрешён (Java не поддерживает множественное наследование классов). После erasure T заменяется на первый bound.

Multiple bounds используются, когда нужно гарантировать несколько контрактов от типа-параметра: например, что тип можно сериализовать И сравнивать, или что он имплементирует определённый бизнес-интерфейс И расширяет базовый класс.

```java
// Multiple bounds: Number + Comparable<T>
public static <T extends Number & Comparable<T>> T clamp(T value, T min, T max) {
    if (value.compareTo(min) < 0) return min;
    if (value.compareTo(max) > 0) return max;
    return value;
}

Integer clamped = clamp(150, 0, 100);  // 100
Double clamped2 = clamp(-5.0, 0.0, 10.0); // 0.0

// Реальный пример: generic DAO
public <T extends BaseEntity & Serializable> void saveAndCache(T entity) {
    repository.save(entity);          // BaseEntity метод — OK
    cacheManager.put(entity.getId(), entity); // Serializable для сериализации — OK
}

// Bounds с интерфейсом и несколькими интерфейсами
interface Auditable { Instant getCreatedAt(); }
interface Exportable { byte[] export(); }

public <T extends BaseEntity & Auditable & Exportable>
    void auditAndExport(T item) {
    auditLog.record(item.getId(), item.getCreatedAt()); // Auditable
    storage.store(item.export());                        // Exportable
}
```

❗ Ловушка: класс в bounds должен идти ПЕРВЫМ. `<T extends Serializable & Number>` — ошибка компиляции, если Number — класс. Правильно: `<T extends Number & Serializable>`. Также: всегда только один класс, остальные — интерфейсы.

**→ Уточняющий вопрос:** Во что компилируется `<T extends Number & Comparable<T>>` после erasure — что становится «стёртым» типом T?

---

## 27. Как реализовать шаблон Singleton с помощью Record?

Record не подходит для классического Singleton-паттерна по нескольким причинам: каждый вызов `new Record(...)` создаёт новый экземпляр; record не может иметь приватный конструктор (канонический конструктор всегда хотя бы package-private); record нельзя переопределить сериализацию так, чтобы гарантировать единственность экземпляра.

Лучший вариант для Singleton — `enum`, который гарантирует единственность, thread safety и устойчивость к сериализации. Record используется по своему прямому назначению — как Value Object или DTO.

```java
// НЕ делай так — это не Singleton:
record Config(String host, int port) {
    private static final Config INSTANCE = new Config("localhost", 8080);
    static Config getInstance() { return INSTANCE; }
    // Проблема: new Config("other", 9090) создаст второй экземпляр
}

// Правильный Singleton — enum
public enum AppConfig {
    INSTANCE;

    private final String dbUrl = System.getenv("DB_URL");
    private final int timeout = 30;

    public String getDbUrl() { return dbUrl; }
    public int getTimeout() { return timeout; }
}
// AppConfig.INSTANCE — единственный экземпляр, thread-safe, serializable

// Record используй для immutable value objects — его истинное назначение:
record DbConfig(String url, String username, int maxPoolSize) {
    static DbConfig fromEnv() {
        return new DbConfig(
            System.getenv("DB_URL"),
            System.getenv("DB_USER"),
            Integer.parseInt(System.getenv().getOrDefault("DB_POOL", "10"))
        );
    }
}

// В Spring Boot — record как @ConfigurationProperties (Spring Boot 3.x)
@ConfigurationProperties("app.db")
record DbProperties(String url, String username, String password, int poolSize) {}
```

❗ Ловушка: попытка сделать приватный канонический конструктор в record — ошибка компиляции. Компилятор требует, чтобы канонический конструктор имел как минимум такой же уровень доступа, как сам record. Для Singleton всегда выбирай enum.

**→ Уточняющий вопрос:** Почему enum является лучшим способом реализации Singleton в Java, и как он защищён от сериализации-атаки?
