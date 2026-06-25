# Stream API

> 📇 Справочник уровня middle. Формат: ответ по сути → как работает внутри → пример из реального кода → ❗ ловушка на собесе → уточняющий вопрос.

**Всего вопросов: 29**

---

## 1. Какие преимущества даёт использование Stream API?

Stream API переводит обработку коллекций в декларативный стиль: ты описываешь *что* нужно сделать, а не *как* итерироваться. Это резко снижает количество boilerplate-кода и делает намерение очевидным — `orders.stream().filter(Order::isActive).map(Order::getTotal).reduce(BigDecimal.ZERO, BigDecimal::add)` читается как задача, а не как алгоритм.

Ключевые технические преимущества: ленивость (intermediate-операции не выполняются до terminal), композиция (операции выстраиваются в pipeline, JIT оптимизирует их вместе), и тривиальное переключение на параллелизм через `.parallel()` без ручной работы с потоками. Кроме того, стримы работают не только с коллекциями — они абстрагируют источник данных: файлы, генераторы, I/O.

```java
// До Stream API
List<String> result = new ArrayList<>();
for (User user : users) {
    if (user.isActive()) {
        result.add(user.getEmail().toLowerCase());
    }
}

// С Stream API
List<String> result = users.stream()
    .filter(User::isActive)
    .map(user -> user.getEmail().toLowerCase())
    .collect(Collectors.toList());
```

❗ Ловушка собеса: кандидаты часто называют только «краткость кода», забывая про ленивость и параллелизм. Интервьюер ждёт упоминания lazy evaluation и того, что без terminal-операции конвейер вообще не запустится.

**→ Уточняющий вопрос:** Если Stream API такой хороший, почему для маленьких коллекций иногда цикл предпочтительнее?

---

## 2. В чём разница между intermediate и terminal операциями?

Intermediate-операции (`filter`, `map`, `flatMap`, `sorted`, `distinct`, `peek`, `limit`, `skip`) — ленивые: они возвращают новый `Stream<T>` и не выполняются немедленно. Каждый вызов просто добавляет шаг в pipeline. Terminal-операции (`collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch` и др.) — потребляют стрим, запускают весь накопленный конвейер и возвращают результат или порождают побочный эффект.

Внутри это реализовано через цепочку объектов `Sink`. Каждая intermediate-операция создаёт `Sink`, который ссылается на следующий. Terminal-операция создаёт «финальный» `Sink` и запускает `spliterator.forEachRemaining(sink)`. Весь pipeline проходится ровно один раз — элементы не материализуются между шагами (за исключением stateful-операций вроде `sorted`).

```java
Stream<String> pipeline = orders.stream()           // источник
    .filter(o -> o.getStatus() == PENDING)           // intermediate #1
    .map(Order::getCustomerEmail);                   // intermediate #2
// Пока ничего не выполнялось!

long count = pipeline.count();                       // terminal — запускает всё
```

❗ Ловушка собеса: классический вопрос — «что выведет этот код?» где стрим собирают, но terminal-операция отсутствует. Правильный ответ — ничего не произойдёт, `forEach` не был вызван.

**→ Уточняющий вопрос:** Какие intermediate-операции являются stateful, и почему это важно для parallel streams?

---

## 3. Что делает операция filter()?

`filter(Predicate<T> predicate)` — intermediate-операция, которая пропускает дальше по конвейеру только те элементы, для которых предикат возвращает `true`. Остальные элементы просто «проваливаются» и не передаются следующей операции.

Реализация: при каждом вызове `sink.accept(element)` сначала проверяется предикат, и только в случае `true` вызывается `downstream.accept(element)`. Операция stateless — она не хранит состояние между элементами, что делает её безопасной для параллельных стримов.

```java
// Фильтруем только активные заказы на сумму > 1000
List<Order> highValueOrders = orders.stream()
    .filter(Order::isActive)
    .filter(o -> o.getTotal().compareTo(BigDecimal.valueOf(1000)) > 0)
    .collect(Collectors.toList());

// Эквивалентно одному filter с &&, но два filter читаемее
```

❗ Ловушка собеса: `filter` с side effect в предикате (например, удаляющим элемент из внешней коллекции) — антипаттерн, который сломается при parallel stream. Также: передача `null` в предикат вызовет `NullPointerException`, не пропустит элемент.

**→ Уточняющий вопрос:** Имеет ли значение порядок операций filter и map с точки зрения производительности?

---

## 4. Что делает операция map()?

`map(Function<T, R> mapper)` — intermediate-операция преобразования: берёт каждый элемент типа `T` и возвращает элемент типа `R`. Это трансформация один-к-одному — количество элементов не меняется, меняется только их тип или значение.

Операция stateless. Существуют специализированные версии: `mapToInt`, `mapToLong`, `mapToDouble` — они возвращают примитивные стримы (`IntStream`, `LongStream`, `DoubleStream`) и позволяют избежать boxing/unboxing. Для обратного преобразования — `mapToObj`.

```java
// Получаем список email-адресов активных пользователей
List<String> emails = users.stream()
    .filter(User::isActive)
    .map(User::getEmail)           // User -> String
    .map(String::toLowerCase)      // String -> String
    .collect(Collectors.toList());

// Считаем сумму цен заказов без boxing
double totalRevenue = orders.stream()
    .mapToDouble(Order::getPrice)  // Order -> double (без autoboxing)
    .sum();
```

❗ Ловушка собеса: использование `map` вместо `mapToInt` для числовых операций вызывает ненужный boxing. На больших коллекциях это заметно по GC-давлению. Также путают `map` и `flatMap` — если функция возвращает `Optional` или `List`, нужен `flatMap`.

**→ Уточняющий вопрос:** Когда предпочесть mapToInt()/mapToDouble() вместо map()? Что даёт IntStream дополнительно?

---

## 5. Что делает операция collect()?

`collect(Collector<T, A, R> collector)` — terminal-операция, которая «собирает» элементы стрима в результирующий контейнер: коллекцию, строку, число, или любую другую структуру. Это самая гибкая terminal-операция, поскольку поведение полностью определяется переданным `Collector`.

Внутри `Collector` определяет три компонента: `supplier()` — создаёт пустой контейнер-накопитель, `accumulator()` — добавляет элемент в контейнер, `combiner()` — объединяет два контейнера (используется при parallel), и опционально `finisher()` — финальное преобразование контейнера. При последовательном стриме `combiner` не вызывается.

```java
// Собираем в список
List<Order> pending = orders.stream()
    .filter(o -> o.getStatus() == PENDING)
    .collect(Collectors.toList());

// Собираем в строку для логирования
String orderIds = orders.stream()
    .map(o -> String.valueOf(o.getId()))
    .collect(Collectors.joining(", ", "[", "]")); // "[1, 2, 3]"

// Группируем заказы по статусу
Map<OrderStatus, List<Order>> byStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus));
```

❗ Ловушка собеса: `Collectors.toList()` не гарантирует конкретную реализацию и изменяемость (хотя на практике возвращает `ArrayList`). В Java 16+ появился `Stream.toList()` — но он возвращает **неизменяемый** список. Путаница между ними — частая ошибка.

**→ Уточняющий вопрос:** Чем отличается Collectors.toList() от Stream.toList() в Java 16+?

---

## 6. Что такое Collector и какие есть встроенные Collectors?

`Collector<T, A, R>` — интерфейс стратегии накопления результата для `collect()`. Параметры: `T` — тип элемента стрима, `A` — тип внутреннего аккумулятора, `R` — тип финального результата. Класс `Collectors` предоставляет фабричные методы для стандартных случаев.

Встроенные коллекторы: `toList()`, `toSet()`, `toUnmodifiableList()` — базовые; `toMap(keyFn, valueFn)` — в Map; `joining(delimiter, prefix, suffix)` — конкатенация строк; `groupingBy(classifier)` — группировка в `Map<K, List<T>>`; `partitioningBy(predicate)` — деление на два списка; `counting()`, `summingInt()`, `averagingDouble()`, `summarizingInt()` — агрегаты; `toCollection(Supplier)` — в произвольную коллекцию.

```java
// groupingBy с downstream-коллектором — группируем и считаем
Map<String, Long> orderCountByCity = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getCity,
        Collectors.counting()
    ));

// partitioningBy — делим пользователей на VIP и обычных
Map<Boolean, List<User>> partitioned = users.stream()
    .collect(Collectors.partitioningBy(u -> u.getTotalSpend() > 10_000));
List<User> vip = partitioned.get(true);

// Собираем в конкретную реализацию
TreeSet<String> sorted = names.stream()
    .collect(Collectors.toCollection(TreeSet::new));
```

❗ Ловушка собеса: `groupingBy` без downstream-коллектора возвращает `Map<K, List<T>>`. Если нужна сумма, количество или другая агрегация — надо передать downstream. Путаница между `groupingBy` и `toMap` — тоже частая ошибка: `toMap` падает на дубликатах ключей, `groupingBy` — нет.

**→ Уточняющий вопрос:** Как написать собственный Collector? Какие методы нужно реализовать?

---

## 7. Что делает операция flatMap()?

`flatMap(Function<T, Stream<R>> mapper)` — intermediate-операция, которая для каждого элемента создаёт стрим дочерних элементов, а затем «расплющивает» (flatten) все эти стримы в один. Это операция один-ко-многим: один входящий элемент порождает ноль, один или несколько выходящих.

Внутри: для каждого элемента `T` вызывается `mapper.apply(t)`, что возвращает `Stream<R>`. Затем этот стрим «инлайнится» в общий конвейер — его элементы передаются в следующую операцию по одному, без промежуточной материализации. Если `mapper` вернул `null` — `NullPointerException`.

```java
// У каждого заказа есть список товаров — получаем все товары всех заказов
List<Product> allProducts = orders.stream()
    .flatMap(order -> order.getItems().stream())  // Order -> Stream<OrderItem>
    .map(OrderItem::getProduct)                   // OrderItem -> Product
    .distinct()
    .collect(Collectors.toList());

// Разбиваем строки на слова (каждая строка -> массив слов -> Stream<String>)
List<String> words = sentences.stream()
    .flatMap(s -> Arrays.stream(s.split("\\s+")))
    .collect(Collectors.toList());

// flatMap с Optional (Java 9+) — убираем пустые Optional из стрима
List<String> emails = users.stream()
    .flatMap(u -> u.getOptionalEmail().stream()) // Optional<String> -> Stream<String>
    .collect(Collectors.toList());
```

❗ Ловушка собеса: `flatMap` не работает с `null` — mapper должен вернуть пустой стрим (`Stream.empty()`), а не `null`. Также: `flatMap` в отличие от `map` не может быть оптимизирован JIT так же агрессивно — промежуточные стримы всё же создаются.

**→ Уточняющий вопрос:** Как работает flatMap с Optional::stream в Java 9+? Почему это лучше, чем filter + map?

---

## 8. В чём разница между map() и flatMap()?

`map` — преобразование один-к-одному: каждый входящий элемент порождает ровно один выходящий элемент, тип может измениться. Если функция возвращает `List` или `Optional`, результатом будет `Stream<List<T>>` или `Stream<Optional<T>>` — со вложенностью.

`flatMap` — преобразование один-ко-многим с уплощением: функция возвращает `Stream<R>`, и этот внутренний стрим «разворачивается» в общий конвейер. Если результат функции — `Optional<String>`, то `map` даст `Stream<Optional<String>>`, а `flatMap(Optional::stream)` (Java 9+) даст `Stream<String>` без пустых элементов.

```java
// map -> Stream<List<String>>: вложенность сохраняется
Stream<List<String>> nested = departments.stream()
    .map(Department::getEmployeeNames);

// flatMap -> Stream<String>: вложенность убрана
Stream<String> flat = departments.stream()
    .flatMap(d -> d.getEmployeeNames().stream());

// Критичный кейс: Optional
// map даёт Stream<Optional<Address>>
Stream<Optional<Address>> withOptionals = users.stream()
    .map(User::getOptionalAddress);

// flatMap даёт Stream<Address> — только непустые
Stream<Address> addresses = users.stream()
    .flatMap(u -> u.getOptionalAddress().stream()); // Java 9+
```

❗ Ловушка собеса: интервьюер часто показывает код с `map(collection -> collection.stream())` и спрашивает, что получится. Правильный ответ — `Stream<Stream<T>>`, и это почти никогда не то, что нужно. Нужен `flatMap`.

**→ Уточняющий вопрос:** Есть ли у flatMap аналоги в других языках, которые ты знаешь? Как это называется в функциональном программировании?

---

## 9. Что такое параллельные стримы?

Параллельный стрим — это стрим, элементы которого обрабатываются в нескольких потоках одновременно. Под капотом используется `ForkJoinPool.commonPool()`: данные рекурсивно делятся на части через `Spliterator`, каждая часть обрабатывается в отдельной задаче `ForkJoinTask`, затем результаты объединяются.

Параллелизация прозрачна для API: тот же набор операций (`filter`, `map`, `collect`) работает и в последовательном, и в параллельном режиме. Но семантика может отличаться: порядок встречи элементов становится недетерминированным, что важно для операций типа `findFirst` (он всё равно вернёт первый по encounter-order) и `forEach` (порядок вызовов не гарантирован).

```java
// Последовательный
List<Report> reports = largeDataset.stream()
    .filter(Record::isValid)
    .map(this::generateReport)  // тяжёлая CPU-операция
    .collect(Collectors.toList());

// Параллельный — то же самое, но быстрее на большом объёме
List<Report> reports = largeDataset.parallelStream()
    .filter(Record::isValid)
    .map(this::generateReport)
    .collect(Collectors.toList());
```

❗ Ловушка собеса: параллельный стрим — не серебряная пуля. На коллекции из 100 элементов с простой операцией параллельный стрим будет **медленнее** из-за overhead на разбиение, объединение результатов и координацию потоков.

**→ Уточняющий вопрос:** Как параллельный стрим разделяет данные? Что такое Spliterator и какие его характеристики влияют на эффективность параллелизма?

---

## 10. Когда использовать parallel streams?

Параллельные стримы выгодны при одновременном выполнении трёх условий: большой объём данных (тысячи и более элементов), тяжёлые CPU-bound операции над каждым элементом, и источник данных легко делится (`ArrayList`, массив, `IntStream.range` — да; `LinkedList`, `HashSet` — хуже). При соблюдении этих условий ускорение кратно числу ядер.

Когда НЕ использовать: маленькие коллекции (overhead больше выгоды), I/O-bound операции (потоки блокируются — лучше async), операции с порядком (неупорядоченные результаты могут нарушить логику), при наличии синхронизации внутри функций (снижает параллелизм до нуля), и при чувствительном окружении (общий `ForkJoinPool` — заблокируешь другие части приложения).

```java
// Хороший кейс: 100k записей, тяжёлый расчёт риска
double totalRisk = transactions.parallelStream()   // 100_000 элементов
    .filter(Transaction::isHighValue)
    .mapToDouble(this::calculateRiskScore)          // CPU-интенсивно
    .sum();

// Плохой кейс: 10 пользователей — overhead не окупится
users.parallelStream()  // не нужно!
    .filter(User::isActive)
    .collect(Collectors.toList());
```

❗ Ловушка собеса: часто спрашивают про `LinkedList` — у него нет эффективного `Spliterator` (последовательный доступ), поэтому параллелизм даже на больших данных будет неэффективен.

**→ Уточняющий вопрос:** Как можно замерить, действительно ли parallel stream быстрее для конкретной задачи?

---

## 11. Как создать parallel stream?

Два способа: `collection.parallelStream()` — создаёт параллельный стрим сразу из коллекции; или `stream.parallel()` — преобразует существующий последовательный стрим в параллельный. Обратно: `stream.sequential()` — переключает обратно в последовательный режим.

Важный нюанс: вызов `.parallel()` или `.sequential()` влияет на весь pipeline, а не только на часть после вызова. Последний вызов одного из двух в цепочке определяет режим всего стрима. Смешивание `.parallel().sequential()` в одном pipeline не создаёт «частично параллельного» выполнения.

```java
// Способ 1: прямо из коллекции
Stream<Order> parallel1 = orders.parallelStream();

// Способ 2: преобразование
Stream<Order> parallel2 = orders.stream().parallel();

// Способ 3: с кастомным ForkJoinPool (чтобы не блокировать commonPool)
ForkJoinPool customPool = new ForkJoinPool(4);
List<Result> results = customPool.submit(() ->
    orders.parallelStream()
        .map(this::process)
        .collect(Collectors.toList())
).get();
customPool.shutdown();

// Последний вызов определяет режим всего pipeline
orders.parallelStream()
    .filter(...)
    .sequential()   // весь pipeline будет последовательным!
    .collect(...);
```

❗ Ловушка собеса: кандидаты не знают о возможности передать кастомный `ForkJoinPool` через `submit`. Это важно в production, где нельзя блокировать общий пул. Также: `sequential()` отменяет `parallelStream()` для всего конвейера.

**→ Уточняющий вопрос:** Что будет, если в одном pipeline вызвать и parallel(), и sequential()? Какой режим победит?

---

## 12. Какие потенциальные проблемы могут быть с параллельными стримами?

Главная проблема — гонки данных при изменении разделяемого изменяемого состояния из лямбд. Если несколько потоков пишут в одну `ArrayList` без синхронизации — получаем потерю данных или `ConcurrentModificationException`. Вторая проблема — недетерминированный порядок: `forEach` в параллельном стриме не гарантирует порядок обработки.

Третья проблема — засорение общего `ForkJoinPool.commonPool()`. Все `parallelStream()` в JVM делят один пул. Если одна тяжёлая операция занимает все потоки — другие параллельные стримы встают в очередь. Четвёртая — нарушение stateful-операциями: `sorted()` и `distinct()` в параллельном стриме требуют объединения результатов, это дорого и снижает выгоду от параллелизма.

```java
// НЕПРАВИЛЬНО: гонка при записи в ArrayList
List<String> results = new ArrayList<>();
orders.parallelStream()
    .map(Order::getId)
    .forEach(results::add);  // НЕ потокобезопасно!

// ПРАВИЛЬНО: используем collect
List<String> results = orders.parallelStream()
    .map(Order::getId)
    .collect(Collectors.toList());  // collector потокобезопасен

// ПРАВИЛЬНО при необходимости accumulate: потокобезопасная структура
List<String> safe = Collections.synchronizedList(new ArrayList<>());
// или CopyOnWriteArrayList, или через .collect()
```

❗ Ловушка собеса: классический вопрос — «что не так с этим кодом?» где `forEach` пишет в `ArrayList`. Ожидаемый ответ: non-thread-safe, нужен `collect` или `ConcurrentHashMap`/`CopyOnWriteArrayList`.

**→ Уточняющий вопрос:** Почему Collectors.toList() безопасен для parallel stream, если ArrayList не потокобезопасен?

---

## 13. Что такое ForkJoinPool и как он связан с parallel streams?

`ForkJoinPool` — специализированный пул потоков с work-stealing алгоритмом. Каждый рабочий поток имеет собственную двустороннюю очередь задач (deque). Когда поток заканчивает свои задачи, он «крадёт» задачи из хвоста очереди другого потока, обеспечивая балансировку нагрузки без централизованной синхронизации.

`ForkJoinPool.commonPool()` — статический пул, который `parallelStream()` использует по умолчанию. Его размер = `Runtime.getRuntime().availableProcessors() - 1` (можно изменить системным свойством `java.util.concurrent.ForkJoinPool.common.parallelism`). Все `parallelStream()` в одном JVM-процессе разделяют этот пул, что является критическим ограничением в серверных приложениях.

```java
// Проверяем размер общего пула
System.out.println(ForkJoinPool.commonPool().getParallelism()); // обычно cores - 1

// Параллельный стрим в своём пуле (изоляция от commonPool)
ForkJoinPool pool = new ForkJoinPool(8);
try {
    Map<String, Long> result = pool.submit(() ->
        transactions.parallelStream()
            .collect(Collectors.groupingBy(
                Transaction::getCategory,
                Collectors.counting()
            ))
    ).get();
} finally {
    pool.shutdown();
}

// Пример work-stealing: задача A разбивается на A1 и A2,
// поток 2 "крадёт" A2 у потока 1, если поток 1 занят A1
```

❗ Ловушка собеса: в Spring Boot приложении `parallelStream()` разделяет `commonPool` с другими задачами (включая `@Async`, CompletableFuture). Блокирующая операция в `parallelStream` может заморозить весь пул. Это частая причина дедлоков в production.

**→ Уточняющий вопрос:** Как ForkJoinPool.commonPool() влияет на другие части приложения, использующие CompletableFuture.supplyAsync() без явного Executor?

---

## 14. Можно ли изменять состояние внешних переменных в Stream операциях?

Технически — иногда можно (изменение полей объектов, к которым есть ссылка), но **нельзя** изменять локальные переменные, захваченные лямбдой — они должны быть effectively final. Изменять объекты, на которые указывают effectively final ссылки, формально разрешено, но является антипаттерном.

Причин запрета две. Первая: функциональный стиль требует чистых функций без побочных эффектов — это делает код предсказуемым, тестируемым и легко читаемым. Вторая: в `parallelStream()` изменение разделяемого состояния из нескольких потоков создаёт гонки данных. Javadoc Stream API прямо говорит, что поведение стрима непредсказуемо при наличии side effects в non-terminal операциях.

```java
// ЗАПРЕЩЕНО: переменная не effectively final
int count = 0;
orders.stream()
    .filter(Order::isActive)
    .forEach(o -> count++);  // Ошибка компиляции!

// АНТИПАТТЕРН: изменение объекта — компилируется, но опасно
List<Order> collected = new ArrayList<>();
orders.stream().forEach(collected::add);  // Работает, но плохо

// ПРАВИЛЬНО: используем reduce/collect
long count = orders.stream().filter(Order::isActive).count();
List<Order> collected = orders.stream().collect(Collectors.toList());

// КРИТИЧНО для parallel: это race condition!
AtomicInteger counter = new AtomicInteger();
orders.parallelStream()
    .filter(Order::isActive)
    .forEach(o -> counter.incrementAndGet());  // AtomicInteger — ОК,
    // но лучше: orders.parallelStream().filter(Order::isActive).count()
```

❗ Ловушка собеса: кандидаты часто говорят «нельзя изменять переменные», но на самом деле `AtomicInteger` можно — вопрос в том, **стоит ли**, когда есть `count()`. Интервьюер ищет понимание разницы между «можно технически» и «нужно ли».

**→ Уточняющий вопрос:** Что такое effectively final? Почему Java требует это от переменных, захваченных лямбдами?

---

## 15. Что такое побочные эффекты (side effects) в Stream?

Побочный эффект (side effect) — это любое изменение состояния, которое происходит снаружи от самой функции: запись в внешнюю коллекцию, изменение поля объекта, логирование в базу, вызов сервиса, инкремент счётчика. В контексте Stream — это действие, которое происходит «мимо» результата операции.

Одна terminal-операция явно рассчитана на side effects — `forEach`. Она как раз предназначена для «применить действие к каждому элементу». Но использование side effects в intermediate-операциях (`filter`, `map`, `peek`) — антипаттерн: они должны быть чистыми функциями. `peek()` технически intermediate, но и он задуман только для отладки, не для бизнес-логики.

```java
// ПЛОХО: side effect в map (запись в БД внутри map)
List<Order> processed = orders.stream()
    .filter(Order::isPending)
    .map(order -> {
        order.setStatus(PROCESSING);     // side effect!
        orderRepository.save(order);     // side effect + I/O!
        return order;
    })
    .collect(Collectors.toList());

// ЛУЧШЕ: разделить трансформацию и действие
List<Order> toProcess = orders.stream()
    .filter(Order::isPending)
    .collect(Collectors.toList());

toProcess.forEach(order -> {
    order.setStatus(PROCESSING);
    orderRepository.save(order);
});
```

❗ Ловушка собеса: кандидаты показывают код с `map`, который вызывает `repository.save()` — это классический антипаттерн. `map` должен быть трансформацией, а не командой. Смешение — нарушение принципа CQS (Command Query Separation).

**→ Уточняющий вопрос:** Есть ли случаи, когда side effects в Stream оправданы? Что насчёт forEach?

---

## 16. Почему следует избегать побочных эффектов в Stream?

Три основных причины. Первая — непредсказуемость при параллелизме: в `parallelStream()` несколько потоков одновременно вызывают лямбду, и если она пишет в разделяемое состояние — гарантированная гонка данных. Код будет работать «иногда» — худший вид бага. Вторая — нарушение ленивости: если intermediate-операция с side effect попадает в ленивый pipeline, трудно сказать, когда именно произойдёт side effect — или произойдёт ли вообще.

Третья — тестируемость и читаемость: функция с side effect сложнее тестировать (нужен mock), сложнее переиспользовать, и её поведение неочевидно из сигнатуры. Чистая функция `Function<T, R>` гарантирует, что результат зависит только от входа. Функция с side effect — «чёрный ящик» с внешними зависимостями.

```java
// ОПАСНО в parallelStream: состояние гонки
Map<String, Integer> stats = new HashMap<>();
transactions.parallelStream()
    .forEach(t -> stats.merge(t.getCategory(), 1, Integer::sum)); // race condition!

// ПРАВИЛЬНО: использовать потокобезопасный сбор
Map<String, Long> stats = transactions.parallelStream()
    .collect(Collectors.groupingBy(
        Transaction::getCategory,
        Collectors.counting()
    ));

// ПРАВИЛЬНО: если нужен ConcurrentHashMap
Map<String, Long> stats = transactions.parallelStream()
    .collect(Collectors.groupingByConcurrent(
        Transaction::getCategory,
        Collectors.counting()
    ));
```

❗ Ловушка собеса: `groupingBy` vs `groupingByConcurrent` — вторая специально для parallel streams, она использует `ConcurrentHashMap` и не требует финального объединения. Большинство кандидатов не знают о её существовании.

**→ Уточняющий вопрос:** Чем Collectors.groupingByConcurrent() отличается от groupingBy() в контексте parallel streams?

---

## 17. Что делает операция reduce()?

`reduce` — terminal-операция «свёртки»: последовательно применяет бинарную функцию к элементам, накапливая единственное итоговое значение. Три сигнатуры: `reduce(BinaryOperator<T>)` → `Optional<T>` (для пустого стрима); `reduce(T identity, BinaryOperator<T>)` → `T` (identity — начальное значение, возвращается при пустом стриме); `reduce(U identity, BiFunction<U,T,U>, BinaryOperator<U>)` → `U` (для смены типа, используется в параллельных стримах).

Функция должна быть ассоциативной (для параллельного reduce) и не иметь side effects. В параллельном стриме `reduce` разбивает данные на части, сворачивает каждую часть, затем объединяет через combiner (третий аргумент или та же функция).

```java
// Сумма заказов
BigDecimal total = orders.stream()
    .map(Order::getTotal)
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// Максимальный заказ
Optional<Order> maxOrder = orders.stream()
    .reduce((o1, o2) -> o1.getTotal().compareTo(o2.getTotal()) >= 0 ? o1 : o2);

// Трёхаргументная форма: аккумулируем Long из stream<Order>
Long totalItems = orders.parallelStream()
    .reduce(
        0L,                                                  // identity
        (acc, order) -> acc + order.getItemCount(),          // accumulator
        Long::sum                                            // combiner (для parallel)
    );
```

❗ Ловушка собеса: в трёхаргументной форме `reduce` для параллельного стрима combiner должен быть совместим с accumulator: `combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)`. Нарушение этого — тихий баг, проявляющийся только в parallel режиме.

**→ Уточняющий вопрос:** Почему reduce() с мутабельным аккумулятором хуже collect()? В чём разница с точки зрения производительности?

---

## 18. В чём разница между reduce() и collect()?

`reduce` предназначен для иммутабельного свёртывания: на каждом шаге создаётся новое значение. Например, `BigDecimal::add` создаёт новый `BigDecimal` при каждом сложении. Для примитивных типов это эффективно, но для коллекций `reduce` будет создавать новый `List` на каждом шаге — `O(n²)` по памяти и времени.

`collect` предназначен для мутабельной редукции: один контейнер (например, `ArrayList`) создаётся единожды через `supplier()`, и элементы добавляются в него через `accumulator()`. При параллельном стриме создаётся несколько частичных контейнеров, которые объединяются через `combiner()`. Поэтому `collect(Collectors.toList())` — `O(n)` по памяти, а эквивалентный `reduce` с конкатенацией списков — `O(n²)`.

```java
// ПРАВИЛЬНО: collect для сборки коллекции — O(n)
List<String> emails = users.stream()
    .map(User::getEmail)
    .collect(Collectors.toList());

// НЕПРАВИЛЬНО: reduce для сборки коллекции — O(n²)!
List<String> emails = users.stream()
    .map(User::getEmail)
    .reduce(
        new ArrayList<>(),
        (list, email) -> { list.add(email); return list; },  // мутируем identity!
        (l1, l2) -> { l1.addAll(l2); return l1; }
    );
// Плюс здесь мутируется identity — это баг в parallel режиме

// ПРАВИЛЬНО: reduce для скаляров
OptionalDouble max = orders.stream()
    .mapToDouble(Order::getTotal)
    .reduce(Double::max);
```

❗ Ловушка собеса: попытка использовать `reduce` для сборки `List` с мутабельным identity-аккумулятором — классический баг. Javadoc явно предупреждает: identity не должен мутироваться в accumulator.

**→ Уточняющий вопрос:** Когда бы ты предпочёл reduce() вместо collect()? Есть ли ситуации, где reduce() принципиально правильнее?

---

## 19. Что такое операция peek() и когда её использовать?

`peek(Consumer<T> action)` — intermediate-операция, которая выполняет действие над каждым элементом и передаёт его дальше неизменённым. Возвращает `Stream<T>` с теми же элементами. Предназначена исключительно для отладки и диагностики: чтобы «подсмотреть», что проходит через конвейер на данном этапе.

Ключевые ограничения: как любая intermediate-операция, `peek` ленива — без terminal-операции вызова `Consumer` не будет. В parallel stream вызовы `peek` в разных потоках могут перемежаться в непредсказуемом порядке. Оптимизатор JVM вправе пропустить вызов `peek`, если считает, что результат операции не используется (это происходит редко, но документировано).

```java
// Правильное использование: отладка pipeline
List<Order> result = orders.stream()
    .filter(Order::isPending)
    .peek(o -> log.debug("After filter: {}", o.getId()))
    .map(this::enrichOrder)
    .peek(o -> log.debug("After enrich: {}", o.getId()))
    .collect(Collectors.toList());

// АНТИПАТТЕРН: бизнес-логика в peek
orders.stream()
    .filter(Order::isPending)
    .peek(orderService::process)  // НЕЛЬЗЯ! Это бизнес-логика, не отладка
    .collect(Collectors.toList());
// Вдруг завтра кто-то изменит terminal на count() — process() не вызовется!
```

❗ Ловушка собеса: кандидаты иногда используют `peek` для «производительного» логирования или сохранения в БД. Это неправильно — зависимость от side effect в промежуточной операции ненадёжна. Правило: `peek` только для отладочного `log.debug`, никакой бизнес-логики.

**→ Уточняющий вопрос:** Что произойдёт с peek, если заменить terminal-операцию collect() на count()? Вызовется ли Consumer?

---

## 20. Можно ли повторно использовать Stream?

Нет. `Stream` — это «одноразовый» объект. После вызова terminal-операции стрим переходит в состояние «consumed/closed». Любая последующая попытка использовать тот же объект `Stream` вызовет `IllegalStateException: stream has already been operated upon or closed`.

Это архитектурное решение: `Stream` моделирует однократный проход по данным. Если нужно выполнить несколько операций — нужно создавать новый стрим из источника. `Collection` как источник бесконечно переиспользуема: каждый вызов `.stream()` создаёт новый независимый стрим. Чтобы «сохранить» промежуточный результат — нужно его материализовать через `collect`.

```java
Stream<Order> stream = orders.stream().filter(Order::isActive);

long count = stream.count();           // OK, стрим используется
List<Order> list = stream.collect(...); // IllegalStateException!

// ПРАВИЛЬНО: создавать стрим заново
long count = orders.stream().filter(Order::isActive).count();
List<Order> list = orders.stream().filter(Order::isActive).collect(...);

// Или материализовать промежуточный результат
List<Order> active = orders.stream()
    .filter(Order::isActive)
    .collect(Collectors.toList()); // сохраняем в List

long count = active.size();
String emails = active.stream()...  // создаём новый стрим из List
```

❗ Ловушка собеса: кандидаты иногда пытаются «закешировать» стрим в поле класса и переиспользовать его. Это всегда баг. `Stream` нельзя хранить в полях — только `Supplier<Stream<T>>`.

**→ Уточняющий вопрос:** Можно ли хранить Stream в поле класса? Как правильно реализовать «ленивый поставщик стрима»?

---

## 21. Что такое lazy evaluation в Stream?

Ленивость (lazy evaluation) означает, что intermediate-операции не выполняются в момент вызова — они лишь регистрируются в pipeline. Вычисление откладывается до момента, когда terminal-операция «протягивает» данные через весь конвейер. Этим Stream отличается от `for` цикла, где каждый шаг выполняется немедленно.

Ленивость даёт две оптимизации. Первая — short-circuit: если terminal-операция нашла то, что искала (например, `findFirst` или `limit(5)`), выполнение прекращается без обработки оставшихся элементов. Вторая — loop fusion: JIT может объединить несколько операций в один проход, избежав создания промежуточных коллекций (это зависит от JVM и не гарантировано спецификацией, но происходит на практике).

```java
// Демонстрация ленивости: что напечатается?
Stream.of(1, 2, 3, 4, 5)
    .filter(n -> {
        System.out.println("filter: " + n);
        return n % 2 == 0;
    })
    .map(n -> {
        System.out.println("map: " + n);
        return n * 10;
    })
    .findFirst();

// Вывод:
// filter: 1
// filter: 2
// map: 2
// Элементы 3, 4, 5 не обрабатывались! findFirst нашёл ответ после элемента 2
```

❗ Ловушка собеса: кандидаты думают, что сначала `filter` обходит ВСЕ элементы, потом `map` обходит оставшиеся. Это неверно. Каждый элемент проходит весь pipeline до следующего шага, и при `findFirst` выполнение прекращается как только найден первый подходящий элемент.

**→ Уточняющий вопрос:** Как ленивость связана с возможностью работать с бесконечными стримами? Приведи пример бесконечного стрима.

---

## 22. Когда начинается выполнение операций в Stream?

Выполнение начинается ровно в момент вызова terminal-операции (`collect`, `count`, `forEach`, `findFirst`, `anyMatch` и т.д.). До этого момента все `filter`, `map`, `flatMap` — лишь «рецепт» (цепочка объектов `Sink`), ни один элемент источника данных не обрабатывается.

Это принципиально для понимания: если промежуточные операции дороги (например, обращение к БД внутри `map` — антипаттерн, но бывает) — они не будут вызваны «дёшево» при построении стрима. И наоборот: если terminal-операция не вызвана, никакая работа не будет проделана — это частая причина «почему мой `forEach` не работает» (забыли вызвать terminal).

```java
// Этот код ничего не делает: только строит pipeline
Stream<String> pipeline = users.stream()
    .filter(User::isActive)
    .map(User::getName)
    .sorted();
// Здесь выполнение ещё не началось

// Только здесь начинается реальная работа:
List<String> names = pipeline.collect(Collectors.toList()); // ЗАПУСК

// Практический пример: логгируем момент начала
log.info("Building pipeline...");
Stream<Order> stream = orders.stream()
    .filter(Order::isValid)
    .map(this::enrich); // enrich ещё НЕ вызван

log.info("Executing pipeline...");
List<Order> result = stream.collect(Collectors.toList()); // enrich вызывается ЗДЕСЬ
```

❗ Ловушка собеса: вопрос «в каком порядке выполняются операции» — популярный. Ответ: не построчно (сначала все filter, потом все map), а вертикально — каждый элемент проходит через все операции по очереди, один за другим.

**→ Уточняющий вопрос:** Как ленивость и момент выполнения влияют на обработку исключений в Stream?

---

## 23. Что делают операции distinct(), sorted(), limit(), skip()?

`distinct()` — убирает дубликаты, используя `equals()` и `hashCode()`. Stateful-операция: должна «помнить» все уже встреченные элементы (хранит в `HashSet`). В parallel режиме требует дополнительной синхронизации.

`sorted()` / `sorted(Comparator)` — сортирует элементы. Stateful и «барьерная» операция: для сортировки нужно увидеть все элементы, поэтому материализует весь стрим в массив. Это разрушает ленивость — после `sorted` следующие операции работают с уже накопленным массивом.

`limit(n)` — возвращает первые `n` элементов, затем прекращает обработку (short-circuit). `skip(n)` — пропускает первые `n` элементов. Оба stateless для последовательных стримов, но в parallel режиме `limit` и `skip` с encounter-order дороги.

```java
// Пагинация: 3-я страница по 10 элементов
List<Product> page = products.stream()
    .sorted(Comparator.comparing(Product::getName))
    .skip(20)    // пропустить первые 2 страницы
    .limit(10)   // взять 10
    .collect(Collectors.toList());

// distinct + sorted: порядок важен для производительности
// distinct ДО sorted — меньше элементов сортировать
List<String> uniqueSorted = transactions.stream()
    .map(Transaction::getCategory)
    .distinct()      // сначала убираем дубликаты
    .sorted()        // потом сортируем меньший набор
    .collect(Collectors.toList());
```

❗ Ловушка собеса: `sorted()` — это барьер. Если в pipeline стоит `sorted()`, до него всё ленивое, после него — уже материализованный массив. В parallel stream `sorted()` особенно дорог — нужен глобальный барьер синхронизации.

**→ Уточняющий вопрос:** Почему sorted() называют «stateful barrier operation»? Что это означает для parallel stream?

---

## 24. Как работает короткое замыкание (short-circuiting) в Stream?

Short-circuiting — оптимизация, при которой обработка элементов прекращается раньше, чем все элементы были обработаны. Применяется в двух контекстах: terminal-операции, способные завершиться без полного обхода (`findFirst`, `findAny`, `anyMatch`, `allMatch`, `noneMatch`, `limit`), и бесконечные стримы, которые без short-circuit никогда бы не завершились.

Механизм: при вызове terminal-операции `findFirst()`, как только `Sink` получает первый принятый элемент, он устанавливает флаг «cancellation requested», и spliterator прекращает подачу новых элементов. Операция `limit(n)` аналогично сигнализирует об остановке после `n` принятых элементов.

```java
// Работа с бесконечным стримом — только благодаря short-circuit
Optional<Integer> first = Stream.iterate(0, n -> n + 1) // 0, 1, 2, 3, ...
    .filter(n -> n % 17 == 0 && n % 13 == 0)
    .findFirst(); // находит 221, не зависает

// anyMatch тоже short-circuit: остановится на первом совпадении
boolean hasHighRisk = transactions.stream()
    .anyMatch(t -> t.getAmount().compareTo(RISK_THRESHOLD) > 0);
// Если первая транзакция уже высокорискована — остальные не проверяются

// allMatch: остановится на первом НЕ совпадении
boolean allVerified = users.stream()
    .allMatch(User::isEmailVerified);
// Если первый пользователь не верифицирован — проверка прекращается
```

❗ Ловушка собеса: `allMatch` на пустом стриме возвращает `true` (vacuous truth), `anyMatch` на пустом стриме — `false`, `noneMatch` на пустом — `true`. Это математически корректно, но интуитивно неожиданно.

**→ Уточняющий вопрос:** Что вернут anyMatch(), allMatch(), noneMatch() для пустого стрима? Почему именно эти значения?

---

## 25. Что такое операции anyMatch(), allMatch(), noneMatch()?

Три terminal short-circuit операции для проверки условия на элементах стрима. `anyMatch(predicate)` — возвращает `true`, если хотя бы один элемент удовлетворяет предикату (останавливается на первом совпадении). `allMatch(predicate)` — возвращает `true`, если все элементы удовлетворяют предикату (останавливается на первом НЕ совпадении). `noneMatch(predicate)` — возвращает `true`, если ни один элемент не удовлетворяет предикату (останавливается на первом совпадении).

Все три эффективнее `filter(...).count() > 0` или `filter(...).findFirst().isPresent()`, потому что явно передают намерение и могут быть лучше оптимизированы. В parallel stream они сигнализируют об отмене обработки, как только результат известен.

```java
// Проверяем бизнес-условия перед обработкой
if (orders.stream().noneMatch(Order::isActive)) {
    throw new IllegalStateException("No active orders to process");
}

// Проверка прав доступа
boolean canAccess = user.getRoles().stream()
    .anyMatch(role -> role == ADMIN || role == MANAGER);

// Валидация: все товары должны быть в наличии
boolean allAvailable = cart.getItems().stream()
    .allMatch(item -> inventoryService.isAvailable(item.getProductId()));

// Аналогия через filter (менее эффективно и менее читаемо)
boolean anyMatch = orders.stream().filter(Order::isActive).findFirst().isPresent();
// vs
boolean anyMatch = orders.stream().anyMatch(Order::isActive); // лучше
```

❗ Ловушка собеса: `anyMatch` vs `filter().findFirst().isPresent()` — семантически эквивалентны, но `anyMatch` явно выражает намерение и предпочтительнее. Интервьюер оценивает выбор идиоматичного кода.

**→ Уточняющий вопрос:** Как ведут себя anyMatch/allMatch/noneMatch в parallel stream? Гарантируется ли что предикат будет вызван минимальное число раз?

---

## 26. Что делают операции findFirst() и findAny()?

Обе возвращают `Optional<T>` с элементом из стрима. `findFirst()` возвращает первый элемент в encounter order (порядке встречи), то есть первый элемент источника, прошедший через все предыдущие операции. `findAny()` возвращает **любой** элемент — не обязательно первый.

Различие критично в parallel streams: `findFirst()` вынужден отслеживать encounter order, что требует дополнительной координации между потоками — потенциально менее эффективен. `findAny()` в parallel stream возвращает первый найденный любым потоком элемент — быстрее, но результат непредсказуем. В последовательном стриме `findAny()` на практике всегда возвращает первый элемент, но это деталь реализации, не контракт.

```java
// findFirst: гарантированно первый активный заказ
Optional<Order> firstPending = orders.stream()
    .filter(Order::isPending)
    .findFirst();

firstPending.ifPresent(order -> log.info("Processing first pending: {}", order.getId()));

// findAny: любой активный — для parallel stream быстрее
Optional<Order> anyPending = orders.parallelStream()
    .filter(Order::isPending)
    .findAny(); // в parallel — берёт первый найденный любым потоком

// Правильная обработка Optional-результата
Order order = orders.stream()
    .filter(o -> o.getId().equals(targetId))
    .findFirst()
    .orElseThrow(() -> new OrderNotFoundException(targetId));
```

❗ Ловушка собеса: кандидаты говорят «findAny в parallel быстрее» — верно, но нужно объяснить почему: `findFirst` в parallel требует соблюдения encounter order (какой элемент «первый» по исходной позиции в источнике), а `findAny` — нет. Если порядок не важен — всегда используй `findAny` в parallel.

**→ Уточняющий вопрос:** Что такое encounter order в Stream? Какие источники данных его имеют, а какие — нет?

---

## 27. Как собрать Stream в Map?

Через `Collectors.toMap(keyMapper, valueMapper)`. Первый аргумент — функция для извлечения ключа из элемента, второй — функция для значения. При отсутствии дубликатов ключей этого достаточно. Можно передать третий аргумент — merge-функцию для разрешения коллизий, и четвёртый — фабрику конкретной реализации `Map` (например, `LinkedHashMap::new` для сохранения порядка).

Альтернативы: `Collectors.groupingBy(classifier)` — для группировки, где несколько элементов имеют одинаковый ключ, результат `Map<K, List<V>>`. `Collectors.toUnmodifiableMap()` — неизменяемая карта. `Collectors.toMap` с `Function.identity()` — когда значением должен быть сам объект.

```java
// Простейший случай: ID -> Пользователь (уникальные ID)
Map<Long, User> usersById = users.stream()
    .collect(Collectors.toMap(User::getId, Function.identity()));

// Сохранение порядка вставки
Map<Long, User> orderedMap = users.stream()
    .collect(Collectors.toMap(
        User::getId,
        Function.identity(),
        (a, b) -> a,           // merge (не нужен, но требуется для 4-го аргумента)
        LinkedHashMap::new     // сохраняем порядок
    ));

// Если нужна группировка: один статус -> список заказов
Map<OrderStatus, List<Order>> byStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus));

// Для аналитики: город -> суммарная выручка
Map<String, BigDecimal> revenueByCity = orders.stream()
    .collect(Collectors.toMap(
        Order::getCity,
        Order::getTotal,
        BigDecimal::add  // merge: суммируем при дубликатах
    ));
```

❗ Ловушка собеса: если в данных есть два пользователя с одинаковым ID и merge-функция не передана — `toMap` бросит `IllegalStateException: Duplicate key`. В production-данных дубликаты почти всегда есть, поэтому merge-функция — правило, а не исключение.

**→ Уточняющий вопрос:** Чем Collectors.toMap() отличается от groupingBy()? Когда какой правильнее использовать?

---

## 28. Что делать при коллизиях ключей при сборке в Map?

Передать третий аргумент merge-функцию `BinaryOperator<V>` в `Collectors.toMap(keyFn, valueFn, mergeFunction)`. Merge-функция принимает два значения с одинаковым ключом и возвращает то, которое должно быть сохранено. Без неё при первом дубликате будет брошен `IllegalStateException: Duplicate key <key> (attempted merging values <v1> and <v2>)`.

Стратегии merge: `(existing, duplicate) -> existing` — взять первый, игнорировать дубликат; `(existing, duplicate) -> duplicate` — взять последний; `BigDecimal::add` — просуммировать числовые значения; `(a, b) -> a + "," + b` — конкатенировать строки; кастомная логика для сложных объектов. Для задач где один ключ = несколько значений — `groupingBy` предпочтительнее.

```java
// Данные: список транзакций, несколько транзакций от одного клиента
// Задача: суммарная сумма транзакций по clientId
Map<String, BigDecimal> totalByClient = transactions.stream()
    .collect(Collectors.toMap(
        Transaction::getClientId,
        Transaction::getAmount,
        BigDecimal::add           // при дубликате - складываем
    ));

// Взять первую запись (дедупликация по email)
Map<String, User> byEmail = users.stream()
    .collect(Collectors.toMap(
        User::getEmail,
        Function.identity(),
        (first, second) -> first   // при дубликате email — берём первого
    ));

// АЛЬТЕРНАТИВА через groupingBy (более явная семантика):
Map<String, List<Transaction>> byClient = transactions.stream()
    .collect(Collectors.groupingBy(Transaction::getClientId));
// Затем можно сделать map -> reduce по каждому списку
```

❗ Ловушка собеса: `IllegalStateException: Duplicate key` — одна из самых частых ошибок при работе со Stream в реальном коде. Интервьюер почти всегда проверяет, знает ли кандидат про merge-функцию. Не знать её — красный флаг.

**→ Уточняющий вопрос:** Когда лучше использовать groupingBy вместо toMap с merge-функцией?

---

## 29. Как работать с Optional в Stream?

`Optional` и `Stream` интегрированы в нескольких точках. В Java 9+ у `Optional` появился метод `.stream()`, который возвращает `Stream` из одного элемента (если значение есть) или пустой `Stream` (если `Optional.empty()`). Это ключевое для `flatMap` при работе с коллекциями `Optional`.

Паттерны: `flatMap(Optional::stream)` — самый чистый способ «распаковать» коллекцию `Optional` в стрим существующих значений (Java 9+). В Java 8 альтернатива: `filter(Optional::isPresent).map(Optional::get)` — рабочий, но менее читаемый. `Stream.of(optional).flatMap(Optional::stream)` — когда нужно работать с одним `Optional` в pipeline. `findFirst()` и `findAny()` возвращают `Optional<T>` — их нужно обрабатывать через `orElse`, `orElseGet`, `orElseThrow`, `ifPresent`.

```java
// Задача: получить email-адреса пользователей, у которых email опциональный
List<User> users = ...;

// Java 9+: flatMap(Optional::stream) — элегантно
List<String> emails = users.stream()
    .map(User::getOptionalEmail)        // Stream<Optional<String>>
    .flatMap(Optional::stream)          // Stream<String> (только непустые)
    .collect(Collectors.toList());

// Java 8: filter + map
List<String> emails = users.stream()
    .map(User::getOptionalEmail)
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());

// Работа с Optional-результатом поиска
String email = orders.stream()
    .filter(o -> o.getId().equals(targetId))
    .findFirst()
    .map(Order::getCustomerEmail)
    .orElse("no-reply@company.com");

// Цепочка Optional методов в pipeline
Optional<String> city = users.stream()
    .filter(u -> u.getId().equals(userId))
    .findFirst()
    .map(User::getAddress)          // Optional<Address>
    .map(Address::getCity);         // Optional<String>
```

❗ Ловушка собеса: в Java 8 нет `Optional.stream()`, поэтому `flatMap(Optional::stream)` не скомпилируется. В Java 9+ это идиома, и незнание этого метода — пробел. Также: вызов `Optional.get()` без `isPresent()` — антипаттерн, который сводит на нет смысл `Optional`.

**→ Уточняющий вопрос:** Чем orElse() отличается от orElseGet()? Когда orElse() может быть проблемой с точки зрения производительности?
