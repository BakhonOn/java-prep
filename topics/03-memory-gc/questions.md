# Память и Garbage Collection

> 📇 Справочник уровня middle. Формат: ответ по сути → как работает в JVM → реальный пример → ❗ ловушка на собесе → уточняющий вопрос.

**Всего вопросов: 28**

---

## 1. В чём разница между Heap и Stack?

Heap — общая для всех потоков куча, управляемая GC. Здесь живут все объекты и массивы. Stack — per-thread структура, организованная как стек фреймов: каждый вызов метода создаёт новый фрейм, который автоматически уничтожается при выходе из метода. Размер стека фиксирован (по умолчанию 512 КБ — 1 МБ на поток), а размер heap управляется через `-Xms`/`-Xmx`.

В JVM это принципиально разные регионы нативной памяти. Stack не подвержен GC — когда метод завершается, фрейм выталкивается и освобождается мгновенно. Heap требует сборки мусора, потому что время жизни объектов непредсказуемо и могут существовать циклические ссылки.

Практически: при рекурсии без выхода (`StackOverflowError`) заканчивается именно стек, а не heap. В Spring-сервисе большинство бизнес-объектов (DTO, Entity, список результатов из БД) живут в heap, а ссылки на них в конкретном методе сервиса — в stack.

```java
public void process() {
    int localPrimitive = 42;            // в stack (значение)
    String localRef = new String("hi"); // ссылка в stack, объект "hi" — в heap
    List<User> users = userRepo.findAll(); // ссылка в stack, список и объекты User — в heap
} // при выходе фрейм уничтожен, localRef/users — в stack больше нет,
  // но объекты в heap живут пока есть другие ссылки
```

❗ Классическая ловушка: "примитивы всегда в стеке" — неверно. Примитивное *поле* объекта (`int age` в классе `User`) живёт в heap вместе с объектом. В стеке лежат только локальные примитивы и ссылки.

**→ Уточняющий вопрос:** Где хранится примитивное поле класса — в heap или stack? А поле объекта внутри другого объекта?

---

## 2. Что хранится в Heap?

В heap хранятся все экземпляры объектов и массивы, созданные через `new`. Это включает поля экземпляров (как примитивные, так и ссылочные), содержимое массивов, а также String pool (с Java 7 он переехал из PermGen в heap). Heap делится на поколения: Young Generation (Eden + два Survivor) и Old Generation.

В JVM heap — единственная область, управляемая GC. Все объекты создаются в Eden (Young Gen). JVM аллоцирует память быстро через pointer bumping — просто сдвигает указатель на следующую свободную ячейку, никаких сложных поисков. Это гораздо быстрее, чем `malloc` в C. Когда Eden заполняется — срабатывает Minor GC.

В реальном Spring-приложении heap быстро заполняется при обработке больших списков из БД. Если сервис выгружает `SELECT * FROM orders` на миллион записей, все эти объекты окажутся в heap одновременно, что может вызвать OOM. Правильное решение — пагинация или стриминг (`Stream` + `ScrollableResults` в Hibernate).

❗ Ловушка: многие думают, что интернированные строки (`String.intern()`) хранятся вне heap — это было правдой до Java 7, когда пул был в PermGen. С Java 7+ String pool в heap, что важно для понимания GC-поведения строк.

**→ Уточняющий вопрос:** Где хранится строковый литерал `"hello"` в коде — и что происходит при вызове `new String("hello")`? Создаётся ли новый объект?

---

## 3. Что хранится в Stack?

Stack каждого потока хранит фреймы вызовов методов. Каждый фрейм содержит: локальные переменные (примитивы по значению, объекты — только ссылки), операндный стек (промежуточные результаты вычислений), ссылку на constant pool текущего класса и адрес возврата. Когда метод возвращает значение — фрейм выталкивается.

JVM управляет стеком без GC — это LIFO-структура. Каждый поток имеет собственный stack, поэтому параллельные вызовы одного метода не мешают друг другу (локальные переменные изолированы). Размер стека задаётся флагом `-Xss` (например, `-Xss512k`). При глубокой рекурсии или большом количестве локальных переменных во фрейме stack исчерпывается — `StackOverflowError`.

На практике: если Spring-сервис вызывает цепочку методов с делегированием глубиной 500+, это может вызвать SOE. Также типичная причина — бесконечная рекурсия в `equals()`/`hashCode()` при циклических объектных графах (например, двунаправленные связи Hibernate без правильной реализации).

❗ Ловушка: "в стеке лежат объекты" — неверно. В стеке лежат *ссылки* на объекты. Сами объекты всегда в heap (за исключением escape analysis, когда JIT может разместить объект на стеке — но это оптимизация, невидимая программисту).

**→ Уточняющий вопрос:** Что такое Escape Analysis и как JIT использует стек для оптимизации аллокации объектов?

---

## 4. Что такое Garbage Collection?

Garbage Collection — автоматический процесс поиска и освобождения памяти от объектов, до которых больше нет достижимых ссылок от GC roots. GC избавляет Java-разработчика от ручного `free()`/`delete`, устраняя целый класс ошибок: dangling pointers, double free, memory corruption. Взамен приходят паузы (stop-the-world) и накладные расходы на сам сборщик.

В JVM GC работает в несколько фаз: маркировка (marking) — обход графа объектов от GC roots, пометка живых; зачистка (sweep) — освобождение памяти мёртвых объектов; опционально уплотнение (compact) — дефрагментация heap для ускорения аллокации. Разные алгоритмы GC делают эти фазы по-разному: Serial делает всё за один проход с остановкой, G1 разбивает heap на регионы и работает инкрементально.

В реальном микросервисе на Spring Boot с G1 GC Minor GC происходит каждые секунды (очень быстро, 5–20 мс), а Full GC — редко, но может занять сотни миллисекунд. Если видите в логах частые Full GC — сигнал о проблеме с памятью или настройками.

```
# Включить логирование GC (Java 9+)
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
```

❗ Ловушка: GC не освобождает *всю* недостижимую память немедленно. Он собирает по регионам/поколениям по необходимости. После `System.gc()` тоже нет гарантии немедленного освобождения.

**→ Уточняющий вопрос:** Что такое фазы GC-цикла — marking, sweep, compact — и какие из них требуют stop-the-world?

---

## 5. Когда объект становится кандидатом на удаление GC?

Объект становится кандидатом на сборку, когда становится *недостижимым* от GC roots — то есть не существует ни одной цепочки сильных ссылок (strong references) от живых корней до этого объекта. Важно: наличие ссылок между самими объектами не спасает — если весь граф недостижим от roots, GC соберёт всё.

GC roots — это: локальные переменные в стеках живых потоков, статические поля классов, ссылки из JNI (нативный код), активные потоки. GC периодически обходит граф объектов с этих точек. Всё, до чего не добрался — мусор. Именно поэтому циклические ссылки (A → B → A) не являются утечкой — если от roots до A нет пути, оба объекта будут собраны.

Пример: метод вернул управление, локальная переменная `order` вышла из scope — если нет других ссылок на этот объект, он становится кандидатом. Но статический кэш `Cache.orders.put(id, order)` удержит объект в памяти до явного удаления из карты, независимо от того, нужен ли он.

```java
public Order createOrder() {
    Order o = new Order(); // o — кандидат на GC после return
    return o;              // но если вызывающий код сохранит ссылку — нет
}

// Статический кэш удерживает объект — он НЕ станет мусором
private static Map<Long, Order> cache = new HashMap<>();
cache.put(1L, new Order()); // объект живёт вечно пока cache жив
```

❗ Ловушка: "объект с `null`-полями будет собран" — нет. Обнуление полей внутри объекта не делает сам объект недостижимым. Недостижимой должна стать ссылка *на* объект.

**→ Уточняющий вопрос:** Если два объекта ссылаются друг на друга, но недостижимы от GC roots — соберёт ли их GC? Почему reference counting не используется в JVM?

---

## 6. Что такое утечка памяти в Java?

Утечка памяти в Java — ситуация, когда объекты остаются *достижимыми* (есть ссылки), но фактически больше не нужны приложению. GC не может их собрать, потому что технически они «живые». Память постепенно накапливается, и рано или поздно случается OOM. Это *логическая* утечка: в отличие от C/C++, здесь нет забытых `free()` — есть забытые ссылки.

В JVM утечки проявляются постепенным ростом heap — особенно Old Generation. После Full GC heap не возвращается к базовому уровню, а остаётся на повышенном. Инструменты диагностики: jstat (`jstat -gcutil <pid> 1s`) покажет рост Old Gen, JFR/Flight Recorder зафиксирует тренд, Eclipse MAT на heap dump найдёт «удержителей» памяти.

В реальном Spring-приложении типичная утечка: статический `HashMap` используется как кэш, но элементы никогда не удаляются. Или сервис регистрирует слушателей событий в `ApplicationEventPublisher`, но при рестарте контекста слушатели не удаляются и старые объекты висят в памяти.

```java
// Классическая утечка — статический кэш без ограничения размера
public class UserService {
    private static final Map<Long, User> cache = new HashMap<>();

    public User getUser(long id) {
        return cache.computeIfAbsent(id, userRepository::findById); // растёт бесконечно
    }
    // Решение: использовать Caffeine/Guava Cache с eviction policy
}
```

❗ Ловушка: в Java "нет утечек памяти" — это миф. Утечки есть, просто другой природы. GC не защищает от логических утечек через живые ссылки.

**→ Уточняющий вопрос:** Как отличить утечку памяти от просто высокого потребления? Что конкретно вы смотрите в heap dump, чтобы подтвердить утечку?

---

## 7. Как может возникнуть утечка памяти в Java?

Основные паттерны: (1) статические коллекции, куда добавляют, но не удаляют; (2) `ThreadLocal` без `remove()` в пуле потоков — каждый поток держит свою копию вечно; (3) слушатели/callbacks, зарегистрированные, но никогда не отписанные; (4) кэши без ограничения размера и eviction; (5) внутренние/анонимные классы, неявно держащие ссылку на внешний класс; (6) незакрытые ресурсы (JDBC Connection, InputStream).

Особенно опасен `ThreadLocal` в Spring с пулом потоков (Tomcat thread pool). При каждом HTTP-запросе создаётся запись в ThreadLocal текущего потока. Если не вызвать `remove()`, этот объект остаётся в потоке навсегда — поток переиспользуется, но старые данные накапливаются. В Hibernate `EntityManager` использует `ThreadLocal` внутри; неправильное управление транзакциями может оставить сессии открытыми.

```java
// Утечка через ThreadLocal в пуле потоков
private static final ThreadLocal<List<AuditEvent>> auditLog = new ThreadLocal<>();

@GetMapping("/process")
public void process() {
    auditLog.set(new ArrayList<>());
    // ... бизнес-логика ...
    // ЗАБЫЛИ вызвать auditLog.remove() — объект остаётся в потоке
}

// Правильно:
try {
    auditLog.set(new ArrayList<>());
    // логика
} finally {
    auditLog.remove(); // всегда в finally
}
```

```java
// Утечка через анонимный класс (держит ссылку на внешний)
public class EventService {
    private final List<Listener> listeners = new ArrayList<>();

    public void registerListener() {
        String bigData = loadHugeCacheFromDB(); // 100 MB
        listeners.add(new Listener() {
            @Override public void onEvent() {
                System.out.println(bigData.length()); // держит bigData
            }
        });
        // bigData не освобождается, пока жив listeners
    }
}
```

❗ Ловушка: `ThreadLocal` утечки особенно коварны в application server'ах (Tomcat, JBoss) — поток принадлежит серверу, а не приложению. При hot redeploy приложения поток остаётся, но старые classloader'ы с данными не выгружаются.

**→ Уточняющий вопрос:** Как WeakReference и WeakHashMap помогают избежать утечек? В каком конкретном сценарии вы бы их применили?

---

## 8. Что такое поколения в GC (young, old, metaspace)?

Generational GC основан на *гипотезе о поколениях*: большинство объектов умирают молодыми (short-lived DTO, промежуточные результаты вычислений), и лишь небольшая часть живёт долго (кэши, singleton-бины). Поэтому heap делится на регионы с разными стратегиями сборки: Young — частая быстрая сборка, Old — редкая и дорогая.

Young Generation: Eden (место создания объектов) + Survivor 0 и Survivor 1. После Minor GC выжившие переходят из Eden в Survivor, затем "прыгают" между S0 и S1, набирая возраст. После достижения `tenuring threshold` (по умолчанию 15 для G1) объект продвигается в Old. Old Generation (Tenured): долгоживущие объекты, собирается Major/Full GC. Metaspace (Java 8+): нативная память для метаданных классов — не в heap, отдельно.

В микросервисе на Spring Boot: бины (ApplicationContext, singleton-сервисы) сразу попадают в Old после первых Minor GC. HTTP-обработчики создают тысячи DTO — они быстро умирают в Young. Если видите, что Old Gen постоянно растёт — кандидат на утечку или накопление кэша.

❗ Ловушка: Metaspace не в heap, поэтому `-Xmx` его не ограничивает. Для контроля используйте `-XX:MaxMetaspaceSize=256m`. Если приложение динамически генерирует классы (CGLib в Spring, Groovy scripts) — Metaspace может исчерпаться независимо от heap.

**→ Уточняющий вопрос:** Что такое tenuring threshold и promotion failure? Как настроить баланс между Young и Old Gen для высоконагруженного REST-сервиса?

---

## 9. Что такое Young Generation?

Young Generation — область heap для вновь созданных объектов. Делится на три части: Eden (здесь рождаются все объекты через `new`), Survivor Space 0 и Survivor Space 1 (два буфера для выживших). Minor GC работает только с Young Gen и происходит очень часто (порой несколько раз в секунду при высокой нагрузке), но быстро — обычно 5–50 мс.

Алгоритм Minor GC: 1) Маркировка всего живого в Eden + активном Survivor. 2) Копирование живых объектов во второй Survivor (copying collector — нет фрагментации). 3) Eden и старый Survivor полностью очищаются. 4) Объектам увеличивается "возраст". 5) Объекты с возрастом ≥ threshold промотируются в Old Gen.

Размер Young Gen настраивается через `-XX:NewRatio=2` (Young:Old = 1:2) или явно `-XX:NewSize` / `-XX:MaxNewSize`. Для high-throughput сервисов с короткоживущими объектами увеличение Young Gen снижает частоту промоций в Old и уменьшает Full GC. Анализировать поведение можно через `jstat -gcnew <pid> 1s`.

```bash
# Мониторинг Young Gen через jstat
jstat -gcnew <pid> 1000   # каждые 1000 мс
# Колонки: S0C S1C S0U S1U TT MTT DSS EC EU YGC YGCT
# TT = tenuring threshold, YGC = count minor GC, YGCT = time
```

❗ Ловушка: если объект слишком большой для Eden (например, массив на 10 МБ), JVM может разместить его сразу в Old Gen (direct allocation) — это называется humongous allocation в G1. Такие объекты быстро засоряют Old Gen.

**→ Уточняющий вопрос:** Что такое TLAB (Thread-Local Allocation Buffer) и как он ускоряет аллокацию в Eden без синхронизации?

---

## 10. Что такое Old Generation (Tenured)?

Old Generation — область для долгоживущих объектов: тех, что пережили несколько Minor GC (достигли tenuring threshold) или были слишком большими для Eden (humongous objects в G1). Занимает большую часть heap (по умолчанию 2/3 при `NewRatio=2`). Собирается Major GC или Full GC — значительно реже, но гораздо дольше (секунды при больших кучах).

В JVM Old Gen работает иначе, чем Young: нет copying collector — вместо этого mark-sweep-compact (или concurrent marking в G1/ZGC). Это дороже. Full GC в Serial/Parallel полностью останавливает приложение. G1 делает concurrent marking параллельно с работой приложения, но stop-the-world фаза всё равно есть (initial mark, remark). Если Old Gen заполняется быстрее, чем GC успевает собирать — `OutOfMemoryError: Java heap space`.

Типичная проблема в проде: Spring-кэш (`@Cacheable` с неограниченным ConcurrentHashMap) накапливает объекты в Old Gen. После нескольких часов работы Old Gen заполнена на 90%, Full GC каждые 10 секунд, каждый занимает 3 секунды — сервис фактически недоступен 30% времени. Решение: ограничить кэш через Caffeine с `maximumSize` и `expireAfterWrite`.

❗ Ловушка: частые Full GC — симптом, не причина. Типичный ответ кандидата "нужно увеличить -Xmx" — неверен без анализа. Сначала нужно понять, почему Old Gen заполняется: реальная нагрузка или утечка.

**→ Уточняющий вопрос:** Что такое promotion failure и concurrent mode failure в CMS/G1? Как они приводят к деградации производительности?

---

## 11. Что такое Metaspace (или PermGen)?

Metaspace (Java 8+) — область нативной памяти (не heap) для хранения метаданных классов: структуры классов, методы, байткод, constant pool, аннотации. До Java 8 эту роль играл PermGen (Permanent Generation) — регион внутри heap с фиксированным размером. Замена PermGen на Metaspace была одним из ключевых изменений Java 8.

Отличие принципиально: PermGen был частью heap, имел фиксированный размер (`-XX:MaxPermSize=256m`), и при загрузке большого количества классов возникал `OutOfMemoryError: PermGen space`. Metaspace растёт динамически в нативной памяти — по умолчанию без верхнего предела (ограничен только доступной нативной памятью). Для контроля используется `-XX:MaxMetaspaceSize`. GC может собирать Metaspace при выгрузке classloader'ов.

Реальная проблема: Spring + CGLib создаёт прокси-классы для каждого `@Transactional`, `@Async`, `@Cacheable` бина. При динамической генерации классов (Groovy, JRuby, частые hot redeploy) Metaspace растёт. Без `-XX:MaxMetaspaceSize` это может исчерпать нативную память всего процесса JVM. Диагностика: `jcmd <pid> VM.metaspace` или JFR с профилем metaspace.

```bash
# Ограничить Metaspace и получить heap dump при OOM в Metaspace
-XX:MaxMetaspaceSize=256m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/

# Мониторинг через jstat
jstat -gcmetacapacity <pid>
```

❗ Ловушка: Metaspace не входит в `-Xmx`. Если контейнер Docker ограничен 512 МБ, а вы ставите `-Xmx400m` — JVM всё равно может упасть по нехватке памяти для Metaspace, стеков потоков и direct buffers. Реальное потребление памяти JVM = heap + metaspace + threads stacks + direct buffers + JIT code cache.

**→ Уточняющий вопрос:** Что вызывает утечку в Metaspace? Как понять, что classloader не выгружается при hot redeploy?

---

## 12. Какие алгоритмы GC существуют?

Основные сборщики в HotSpot JVM: **Serial GC** (`-XX:+UseSerialGC`) — однопоточный, stop-the-world, для маленьких приложений. **Parallel GC** (`-XX:+UseParallelGC`) — многопоточный, оптимизирует throughput за счёт пауз, был дефолтом до Java 8. **CMS** (Concurrent Mark Sweep, deprecated с Java 9, удалён в Java 14) — конкурентная маркировка, низкие паузы, но фрагментация. **G1 GC** (`-XX:+UseG1GC`) — дефолт с Java 9, балансирует throughput и latency. **ZGC** (`-XX:+UseZGC`) — сверхнизкие паузы, Java 15+ production-ready. **Shenandoah** (`-XX:+UseShenandoahGC`) — аналог ZGC от Red Hat.

Выбор зависит от приоритетов: максимальный throughput (batch-обработка) → Parallel GC; предсказуемые паузы ≤ 200 мс (веб-сервисы) → G1; паузы ≤ 10 мс (финтех, real-time) → ZGC или Shenandoah; маленькое приложение или CLI → Serial. Java Flight Recorder поможет сравнить реальные паузы в проде.

Для типичного Spring Boot микросервиса G1 с настройками `–XX:MaxGCPauseMillis=200` — разумный старт. Если SLA требует p99 latency ≤ 50 мс — смотрите на ZGC (Java 17+).

❗ Ловушка: CMS не имеет фазы compact — со временем Old Gen фрагментируется, и JVM вынуждена делать costly Serial Full GC для дефрагментации. Именно поэтому CMS deprecated. Кандидаты часто его называют как "хороший concurrent сборщик", не зная о проблеме фрагментации.

**→ Уточняющий вопрос:** Что значит "concurrent" в контексте GC? Какие фазы G1 выполняются конкурентно, а какие требуют stop-the-world?

---

## 13. Что такое G1 GC?

G1 (Garbage-First) — региональный сборщик, дефолтный с Java 9. Вместо монолитных Young/Old областей он делит heap на тысячи равных *регионов* (Region, обычно 1–32 МБ). Каждый регион динамически назначается как Eden, Survivor, Old или Humongous (для больших объектов > 50% размера региона). G1 приоритизирует сборку регионов с наибольшим количеством мусора — отсюда название "Garbage-First".

Цикл G1: Minor GC (evacuation pause) — собирает Eden и Survivor регионы, stop-the-world. Concurrent Marking — параллельно с приложением обходит граф живых объектов в Old регионах. Mixed GC — собирает и Young, и часть Old регионов одновременно. G1 пытается соблюдать паузу `–XX:MaxGCPauseMillis=200` (мягкий target, не гарантия). Full GC в G1 — fallback, однопоточный, очень дорогой, сигнализирует о проблемах.

Для настройки G1 в Spring микросервисе: установите `-XX:MaxGCPauseMillis=100–200`, `-XX:G1HeapRegionSize` (если heap небольшой, уменьшите до 1–2 МБ), включите `-XX:+G1UseAdaptiveIHOP` (адаптивный порог запуска concurrent marking). Анализируйте через `jcmd <pid> GC.heap_info` или JFR.

```bash
# Типовые флаги G1 для Spring Boot микросервиса
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=4m
-XX:+G1UseAdaptiveIHOP
-XX:InitiatingHeapOccupancyPercent=45
-Xlog:gc*:file=/app/logs/gc.log:time,uptime:filecount=5,filesize=20m
```

❗ Ловушка: `–XX:MaxGCPauseMillis` — это цель, не гарантия. G1 может её нарушить при высокой нагрузке. И Full GC в G1 — всегда сигнал тревоги: значит, concurrent marking не успевает за аллокацией (concurrent mode failure аналог CMS).

**→ Уточняющий вопрос:** Что такое humongous allocation в G1 и почему большие объекты (массивы > 50% региона) могут негативно влиять на GC?

---

## 14. Что такое ZGC?

ZGC (Z Garbage Collector) — сверхнизколатентный сборщик, стабильный с Java 15, значительно улучшен в Java 17+. Главная цель: паузы stop-the-world не более нескольких миллисекунд (≤ 10 мс) независимо от размера heap — даже при терабайтах данных. ZGC делает практически всю работу конкурентно с работающим приложением.

Технически ZGC использует colored pointers (цветные указатели в 64-битных ссылках) и load barriers для отслеживания состояния объектов без остановки приложения. Фазы: concurrent mark, concurrent relocate, concurrent remap — всё параллельно. Stop-the-world только для initial mark pause и final mark pause — буквально миллисекунды. ZGC не имеет generational model в оригинальной версии (generational ZGC появился в Java 21 как experimental, стабилен в Java 23+).

Когда использовать: финтех, trading systems, gaming сервисы, где p99 latency критична. Где не стоит: batch-jobs с максимальным throughput — ZGC тратит CPU на конкурентную работу, что снижает throughput по сравнению с Parallel GC. Мониторинг: `-Xlog:gc*` или JFR.

```bash
# Включить ZGC с базовыми настройками
-XX:+UseZGC
-Xms4g -Xmx4g            # рекомендуется Xms=Xmx для ZGC
-XX:ConcGCThreads=4       # потоки для конкурентной работы GC
-Xlog:gc*:file=/app/logs/gc.log
```

❗ Ловушка: ZGC требует больше CPU для конкурентной работы, поэтому на CPU-ограниченном окружении (маленький контейнер в Kubernetes) паузы могут быть хуже, чем ожидается. Всегда проверяйте в условиях, близких к production.

**→ Уточняющий вопрос:** Что такое colored pointers и load barriers в ZGC? Как они позволяют перемещать объекты без stop-the-world?

---

## 15. Что такое Shenandoah GC?

Shenandoah GC — низколатентный сборщик от Red Hat, доступный в OpenJDK 12+ (backport в 8 и 11). Как и ZGC, целится в паузы порядка миллисекунд независимо от размера heap. Ключевое отличие от ZGC: Shenandoah делает *конкурентную компактизацию* (concurrent compaction) — перемещает живые объекты, пока приложение работает, используя Brooks pointers (дополнительный указатель-forward в каждом объекте).

Архитектурно Shenandoah использует Brooks pointers: каждый объект имеет дополнительный указатель на "себя" (или на новое место если перемещён). При чтении/записи load/store barrier проверяет этот указатель. Это добавляет ~2–3% overhead на каждое обращение к объекту. Shenandoah имеет generational model (Shenandoah 3.0 / Generational Shenandoah добавлен экспериментально в Java 21).

Выбор между ZGC и Shenandoah: оба хороши для latency-critical сервисов. ZGC в Java 17+ часто показывает лучший throughput. Shenandoah может быть предпочтительнее при ограниченной памяти (ZGC требует больше RAM-overhead для colored pointers). Попробуйте оба с нагрузочным тестированием (Gatling, JMeter) и сравните через JFR.

```bash
-XX:+UseShenandoahGC
-Xms2g -Xmx2g
-XX:ShenandoahGCHeuristics=adaptive  # или compact, aggressive, static
-Xlog:gc*:file=/app/logs/gc.log
```

❗ Ловушка: Shenandoah не доступен в Oracle JDK (только OpenJDK). На собесе в компании, использующей Oracle JDK, упоминание Shenandoah может вызвать вопросы о практическом опыте.

**→ Уточняющий вопрос:** Чем Brooks pointers в Shenandoah отличаются от colored pointers в ZGC по overhead и механизму конкурентного перемещения?

---

## 16. Что такое stop-the-world?

Stop-the-world (STW) — пауза, во время которой JVM приостанавливает все потоки приложения для выполнения операции GC. В момент STW никакой бизнес-код не выполняется: нет обработки запросов, нет вычислений. Для пользователя это проявляется как "заморозка" или задержка ответа. Длительность пауз варьируется от долей миллисекунды (ZGC) до нескольких секунд (Serial/Parallel Full GC на большой куче).

Зачем нужен STW? Потому что GC должен работать с consistent snapshot — если объекты перемещаются, а приложение одновременно меняет ссылки, GC может "упустить" живые объекты или сломать граф. Чтобы этого не было, применяется safepoint mechanism: JVM ждёт, пока все потоки достигнут безопасной точки (safepoint), затем начинает GC. Safepoints расставлены в байткоде JVM (при вызовах методов, в циклах).

В высоконагруженном сервисе STW в 500 мс = все запросы за эти 500 мс зависли. Если у вас SLA 1 секунда, а Full GC занимает 800 мс — SLA нарушается. Диагностика: `-Xlog:safepoint` покажет все паузы, JFR → GC → Pause Events.

```bash
# Логировать STW паузы
-Xlog:gc+pause=info:file=/app/logs/gc-pause.log
-Xlog:safepoint=info

# Или через jcmd
jcmd <pid> VM.log what=gc+pause=info output=file=/tmp/gc.log
```

❗ Ловушка: "G1 не делает stop-the-world" — неверно. G1 делает STW для initial mark и final mark (remark) фаз. Только ZGC/Shenandoah свели STW до минимума. Путаница конкурентный ≠ без STW.

**→ Уточняющий вопрос:** Что такое safepoint и time-to-safepoint? Почему длинные циклы без safepoints могут задерживать начало GC-паузы?

---

## 17. Какие GC минимизируют stop-the-world паузы?

ZGC и Shenandoah — лидеры по минимизации STW: паузы 1–10 мс независимо от размера heap. Оба делают маркировку, эвакуацию и компактизацию конкурентно с работой приложения. G1 занимает промежуточную позицию: STW для Minor GC и mark phases (~10–100 мс), но Full GC всё ещё дорогой. Parallel GC — наименьшие паузы из "классических" (многопоточный STW), но это всё равно полные остановки.

Практическое сравнение для сервиса с heap 8 GB: Serial Full GC ~8–15 сек, Parallel Full GC ~2–4 сек, G1 Mixed GC ~50–200 мс, ZGC ~1–5 мс, Shenandoah ~1–5 мс. Разница в throughput: ZGC/Shenandoah тратят 10–15% CPU на конкурентную работу GC, Parallel GC не тратит CPU между GC-циклами, зато сами паузы дольше.

Выбор для микросервиса в Kubernetes: если подов много и heap небольшой (1–2 ГБ) — G1 с `MaxGCPauseMillis=100` может быть достаточно и дешевле по CPU. Если heap 4–16 ГБ или SLA по latency жёсткий — ZGC (Java 17+). Всегда нагрузочное тестирование с JFR.

❗ Ловушка: низкие паузы не равны высокому throughput. ZGC/Shenandoah могут иметь на 10–20% меньший throughput по сравнению с Parallel GC для batch-обработки. Нужно понимать, что приоритизировать.

**→ Уточняющий вопрос:** Как измерить реальные GC-паузы в production без остановки сервиса? Какой инструмент вы используете и что конкретно смотрите?

---

## 18. Что такое параметры -Xms и -Xmx?

`-Xms` задаёт начальный (минимальный) размер heap при старте JVM. `-Xmx` задаёт максимальный размер heap — JVM никогда не превысит это значение. При старте JVM выделяет `-Xms` памяти у ОС, и по мере роста нагрузки расширяет heap до `-Xmx`. Если аллоцировать больше `-Xmx` не получается и GC не может освободить место — `OutOfMemoryError`.

Когда `-Xms < -Xmx`, JVM динамически расширяет heap — это создаёт дополнительные паузы на расширение и может приводить к фрагментации. Поэтому в production часто рекомендуют устанавливать `-Xms = -Xmx`: heap сразу выделяется полностью, нет накладных расходов на расширение, поведение предсказуемо. Особенно важно для G1 и ZGC.

В Kubernetes/Docker важно учитывать, что JVM видит всю RAM хоста без UseContainerSupport (до Java 10). С Java 10+ `-XX:+UseContainerSupport` включён по умолчанию и JVM корректно читает cgroup limits. Флаг `-XX:MaxRAMPercentage=75.0` — удобная альтернатива явному `-Xmx` в контейнерах: берёт 75% от лимита памяти контейнера.

```bash
# Классический подход
java -Xms512m -Xmx512m -jar app.jar

# Для контейнеров (Java 11+)
java -XX:+UseContainerSupport \
     -XX:InitialRAMPercentage=50.0 \
     -XX:MaxRAMPercentage=75.0 \
     -jar app.jar

# Посмотреть текущие настройки heap живого процесса
jcmd <pid> VM.flags | grep -E "Xms|Xmx|HeapSize"
```

❗ Ловушка: в Docker-контейнере без явного `-Xmx` JVM может попытаться взять 25% RAM хоста (дефолт), что значительно превышает лимит контейнера. Результат: OOMKiller от ядра Linux убивает процесс JVM без `OutOfMemoryError` в логах. Это выглядит как внезапный рестарт контейнера.

**→ Уточняющий вопрос:** Как JVM определяет размер heap в Docker-контейнере? Что такое UseContainerSupport и когда его нужно явно включать?

---

## 19. Что произойдёт при OutOfMemoryError?

При OOM JVM не может выделить память для нового объекта, после того как исчерпала все возможности освободить её через GC. JVM бросает `java.lang.OutOfMemoryError` (наследник `Error`, не `Exception`). Обычно это фатально — приложение продолжить нормальную работу не может, хотя технически поймать `Error` можно. В Spring Boot этот `Error` пропагируется вверх и чаще всего приводит к завершению потока или всего процесса.

Последовательность событий: JVM пытается аллоцировать объект → Eden заполнен → Minor GC → не хватает → Old Gen заполнен → Full GC → всё равно не хватает → OOM. Флаг `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/` автоматически создаст heap dump в момент OOM — это критически важно для post-mortem анализа в проде. Также полезен `-XX:OnOutOfMemoryError="kill -9 %p"` для автоматического рестарта.

В Kubernetes: OOM часто проявляется как `CrashLoopBackOff` с exit code 137 (killed by OOMKiller) или с `java.lang.OutOfMemoryError` в логах. Правильная реакция: собрать heap dump (если настроен), проанализировать в Eclipse MAT, найти доминирующие объекты (Dominator Tree), выявить retention path.

```bash
# Критические флаги для production
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/crash/
-XX:+ExitOnOutOfMemoryError     # убить JVM при OOM (для контейнеров)
# или
-XX:OnOutOfMemoryError="sh /scripts/oom-handler.sh %p"
```

❗ Ловушка: ловить `OutOfMemoryError` в `catch (OutOfMemoryError e)` и "восстанавливаться" — почти всегда плохая идея. Heap повреждён, состояние JVM непредсказуемо. Правильное решение: дать упасть, обеспечить автоматический рестарт (Kubernetes restart policy) и собрать heap dump для анализа.

**→ Уточняющий вопрос:** Как вы будете расследовать OOM в production, если приложение уже перезапустилось? Какие данные нужно собрать заранее?

---

## 20. Какие типы OutOfMemoryError существуют?

Основные типы OOM:
- **`Java heap space`** — не хватает места в heap для объекта. Самый частый. Причины: утечка, недостаточный `-Xmx`, реальный рост нагрузки.
- **`GC overhead limit exceeded`** — JVM тратит > 98% времени на GC, освобождая < 2% heap. JVM "сдаётся" — продолжать нет смысла. Флаг `-XX:-UseGCOverheadLimit` отключает эту проверку (не рекомендуется).
- **`Metaspace`** — переполнен Metaspace (классы не выгружаются, динамическая генерация классов).
- **`Unable to create new native thread`** — ОС не может создать новый поток (исчерпаны системные ресурсы: ulimit, нативная память).
- **`Requested array size exceeds VM limit`** — попытка создать массив больше ~2 млрд элементов.
- **`Direct buffer memory`** — исчерпана память для NIO direct buffers (`ByteBuffer.allocateDirect()`), ограниченная `-XX:MaxDirectMemorySize`.
- **`Compressed class space`** — исчерпано сжатое пространство классов (при `-XX:+UseCompressedClassPointers`).

Каждый тип требует своего подхода: `Java heap space` → heap dump + MAT; `Metaspace` → анализ classloader'ов; `native thread` → проверить количество потоков (`jstack <pid> | grep "java.lang.Thread.State" | wc -l`), поднять ulimit; `Direct buffer memory` → увеличить `-XX:MaxDirectMemorySize` или найти место, где не освобождаются буферы.

❗ Ловушка: `GC overhead limit exceeded` часто путают с обычной нехваткой heap. Разница: при `Java heap space` GC не запускался много раз подряд — просто нет места. При `GC overhead` GC работал интенсивно, но толку мало — это признак более серьёзной проблемы (утечка или heap слишком мал для нагрузки).

**→ Уточняющий вопрос:** Как диагностировать `Unable to create new native thread`? Что проверить на уровне ОС и JVM?

---

## 21. Что такое memory leak и как его обнаружить?

Memory leak — постоянное накопление объектов в heap, которые недостижимы для освобождения GC (хотя технически достижимы через ссылки). Симптомы: heap монотонно растёт, после каждого Full GC baseline поднимается выше, Eventually OOM. В отличие от C/C++ утечка в Java всегда логическая: не "потеряли указатель", а "забыли удалить ссылку".

Алгоритм обнаружения: 1) **Мониторинг** — наблюдать за Old Gen через `jstat -gcutil <pid> 5s` или Prometheus+Micrometer. Если после Full GC значение не возвращается к норме — подозрение на утечку. 2) **Heap dump** — сделать два dump с интервалом (например, 30 минут под нагрузкой): `jcmd <pid> GC.heap_dump /tmp/heap1.hprof`, потом `/tmp/heap2.hprof`. 3) **Анализ в Eclipse MAT** — открыть dump, использовать "Leak Suspects Report" для автоматической диагностики. Смотреть Dominator Tree — кто удерживает больше всего памяти. Использовать "Compare Snapshots" для двух dump.

```bash
# Шаг 1: мониторинг GC
jstat -gcutil <pid> 5000
# Смотрим колонку O (Old Gen %) — должна оставаться стабильной

# Шаг 2: heap dump
jcmd <pid> GC.heap_dump /tmp/before.hprof
# ... подождать 30 мин под нагрузкой ...
jcmd <pid> GC.heap_dump /tmp/after.hprof

# Шаг 3: анализ в MAT (Eclipse Memory Analyzer)
# File → Open Heap Dump → Leak Suspects Report
# или сравнение двух dump: Compare Snapshots
```

❗ Ловушка: делать heap dump без предварительного GC (Full GC) — получите много "мусора" в dump, который затруднит анализ. Используйте `jcmd <pid> GC.run` перед dump или `jmap -dump:live,...` с ключом `live` — он делает Full GC перед сохранением.

**→ Уточняющий вопрос:** Что такое Dominator Tree в Eclipse MAT? Как он помогает найти объект, удерживающий большой граф?

---

## 22. Какие инструменты помогают анализировать память?

Инструменты делятся на несколько категорий:

**Профайлеры (живое наблюдение):** VisualVM (бесплатный, встроен в JDK до Java 9, затем отдельно) — heap monitoring, thread dump, CPU profiling. JProfiler, YourKit — коммерческие, более мощные: allocation profiling, heap walker, memory leak detection.

**Анализ heap dump:** Eclipse MAT (Memory Analyzer Tool) — стандарт де-факто для post-mortem анализа OOM. Leak Suspects, Dominator Tree, OQL запросы к объектам в dump. IntelliJ IDEA тоже умеет открывать `.hprof`.

**JVM-утилиты командной строки:** `jmap -dump:live,format=b,file=heap.hprof <pid>` — снять heap dump. `jstat -gcutil <pid> 1s` — статистика GC в реальном времени. `jcmd <pid> GC.heap_info` — быстрый обзор. `jcmd <pid> VM.native_memory` — нативная память (требует `-XX:NativeMemoryTracking=detail`).

**Java Flight Recorder + Mission Control:** JFR — built-in профайлер JVM с минимальным overhead (< 1–2%), можно включать в production. `jcmd <pid> JFR.start name=myrecording duration=60s filename=/tmp/rec.jfr`. JMC (Java Mission Control) — GUI для анализа JFR recordings: GC activity, allocation hotspots, lock contention.

```bash
# Запустить JFR профилирование в production
jcmd <pid> JFR.start name=leak_hunt duration=120s \
  settings=profile filename=/tmp/recording.jfr

# Нативная память (полная картина)
java -XX:NativeMemoryTracking=detail -jar app.jar
jcmd <pid> VM.native_memory summary

# Быстрая статистика GC
jstat -gcutil <pid> 2000 30  # каждые 2 сек, 30 итераций
```

❗ Ловушка: `jmap` во время работы приложения делает STW — в production это может быть неприемлемо. Предпочтительнее `-XX:+HeapDumpOnOutOfMemoryError` (автоматически при OOM) или JFR с `heap-statistics` event (меньший overhead).

**→ Уточняющий вопрос:** Как включить JFR в production с минимальным влиянием на производительность? Какие события отслеживать для поиска утечки памяти?

---

## 23. Что такое heap dump?

Heap dump — бинарный снимок (snapshot) всего содержимого heap в конкретный момент времени. Содержит: все живые объекты (с их типами и размерами), значения их полей, граф ссылок между объектами, информацию о classloader'ах. Формат файла — HPROF (`.hprof`), стандартный для JVM-экосистемы.

Heap dump — основной инструмент для расследования OOM и утечек памяти. Он позволяет ответить на вопросы: кто занимает больше всего памяти? кто удерживает этот объект (retention path до GC root)? сколько экземпляров определённого класса существует? В Eclipse MAT через OQL можно делать SQL-подобные запросы: `SELECT * FROM java.util.HashMap WHERE size > 10000`.

Heap dump — это слепок, а не живое наблюдение. Для полной картины утечки нужно два dump с интервалом и сравнение. Размер файла = примерно размер heap (8 ГБ heap → 8 ГБ файл). Поэтому для больших куч иногда используют `jmap -dump:live,...` (только живые объекты) или JFR heap statistics (менее детально, но без полного dump).

```java
// Программный способ (для тестов и диагностики)
import com.sun.management.HotSpotDiagnosticMXBean;
import java.lang.management.ManagementFactory;

MBeanServer server = ManagementFactory.getPlatformMBeanServer();
HotSpotDiagnosticMXBean mxBean = ManagementFactory.newPlatformMXBeanProxy(
    server, "com.sun.management:type=HotSpotDiagnostic",
    HotSpotDiagnosticMXBean.class);
mxBean.dumpHeap("/tmp/heap.hprof", true); // true = только живые объекты
```

❗ Ловушка: heap dump делает Full GC и STW при снятии. В production сервисе это может вызвать таймауты запросов на несколько секунд. Всегда предупреждайте команду перед снятием dump в prod, или используйте автоматический dump при OOM (`-XX:+HeapDumpOnOutOfMemoryError`).

**→ Уточняющий вопрос:** Как открыть heap dump от JVM с 16 ГБ heap на машине с 8 ГБ RAM? Какие опции MAT для работы с большими dump?

---

## 24. Как получить heap dump?

Способы получения heap dump:

1. **Автоматически при OOM** (рекомендуется для production):
   ```bash
   -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/crash/
   ```

2. **jcmd** (предпочтительный CLI способ, безопаснее jmap):
   ```bash
   jcmd <pid> GC.heap_dump /tmp/heap.hprof
   # Найти pid: jcmd (без аргументов) или jps -l
   ```

3. **jmap** (классический, но более рискованный):
   ```bash
   jmap -dump:live,format=b,file=/tmp/heap.hprof <pid>
   # live — только живые объекты (сначала делает Full GC)
   # без live — все объекты включая мусор
   ```

4. **VisualVM** — GUI: правой кнопкой по процессу → "Heap Dump".

5. **JMX/MBeans** — программно через `HotSpotDiagnosticMXBean.dumpHeap()` или через JMX-консоль.

6. **Kubernetes**: для контейнерного приложения нужно пробросить dump наружу:
   ```bash
   kubectl exec -it <pod> -- jcmd 1 GC.heap_dump /tmp/heap.hprof
   kubectl cp <pod>:/tmp/heap.hprof ./heap.hprof
   ```

```bash
# Полный production-ready рецепт
# Флаги при запуске
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/crash/heap-$(hostname)-$(date +%Y%m%d-%H%M%S).hprof
-XX:+ExitOnOutOfMemoryError

# Ручной dump без STW через JFR (менее детально, но безопаснее)
jcmd <pid> JFR.start name=heapstat settings=default duration=30s filename=/tmp/heap.jfr
```

❗ Ловушка: в Kubernetes pod'е без настроенного volume для dump файл создастся внутри контейнера и будет потерян при его перезапуске. Нужно либо persistent volume, либо немедленно скопировать dump до рестарта.

**→ Уточняющий вопрос:** Что делать, если OOM уже произошёл, pod перезапустился, и dump не был настроен заранее? Как попытаться воспроизвести проблему?

---

## 25. Что такое GC roots?

GC roots — стартовые точки, от которых GC начинает обход графа ссылок для маркировки живых объектов. Всё, что достижимо от roots — живо. Всё остальное — мусор. Без roots GC не знал бы, с чего начать обход.

Типы GC roots в HotSpot JVM:
- **Локальные переменные и параметры активных методов** (stack frames живых потоков)
- **Статические поля классов** (Class statics) — пока класс загружен, его static поля живы
- **JNI references** — ссылки из нативного кода (C/C++ через JNI)
- **Активные Java-потоки** (сами объекты `Thread`)
- **Мониторы синхронизации** (объекты, используемые как lock в `synchronized`)
- **Classloader-специфичные ссылки**

Практически важно: статические поля — одна из главных причин утечек. Если статический `Map` хранит объекты, они достижимы через цепочку `ClassStatic → Map → Entry → value`, пока класс загружен (то есть практически вечно в typical Spring app).

```java
public class Registry {
    // Это GC root (статическое поле)
    private static final Map<String, Handler> HANDLERS = new HashMap<>();

    // Объект добавлен в HANDLERS → достижим от GC root → никогда не будет собран
    public static void register(String name, Handler h) {
        HANDLERS.put(name, h); // если не вызвать remove — утечка
    }
}
```

❗ Ловушка: спросят "назовите GC roots" — кандидаты часто называют только стек и статику, забывая JNI references и активные потоки. Thread-объекты сами являются roots и удерживают свои ThreadLocal values.

**→ Уточняющий вопрос:** Как ThreadLocal связан с GC roots? Почему ThreadLocal без `remove()` приводит к утечке именно через GC root Thread?

---

## 26. Что такое reachability в контексте GC?

Reachability (достижимость) — наличие пути по цепочке ссылок от GC root до объекта. Java определяет несколько уровней достижимости, влияющих на то, когда GC может собрать объект.

**Strongly reachable** — объект достижим через обычные (strong) ссылки. GC не трогает. **Softly reachable** — достижим только через `SoftReference`. GC соберёт его *при нехватке памяти* (но не раньше). Используется для memory-sensitive кэшей. **Weakly reachable** — достижим только через `WeakReference`. GC соберёт при ближайшем сборе мусора. Используется в `WeakHashMap` (ключи — weak refs). **Phantom reachable** — объект уже финализирован, доступен только через `PhantomReference` + `ReferenceQueue`. Используется для пост-финализационной cleanup логики.

`WeakReference` практически применяется в кэшах с автоматическим eviction: `WeakHashMap` — ключи удаляются когда нет strong ссылок. Однако `WeakHashMap` часто неверно используют: если value удерживает key (например, value содержит поле с key-объектом) — утечка. Для production-кэшей лучше Caffeine/Guava Cache.

```java
// SoftReference — кэш, который отдаёт память при OOM
Map<String, SoftReference<byte[]>> imageCache = new HashMap<>();
imageCache.put("logo", new SoftReference<>(loadImage("logo.png")));

byte[] img = Optional.ofNullable(imageCache.get("logo"))
    .map(SoftReference::get)  // может вернуть null если GC собрал
    .orElseGet(() -> loadImage("logo.png")); // reload если evicted

// WeakReference — не препятствует GC
WeakReference<ExpensiveObject> weakRef = new WeakReference<>(new ExpensiveObject());
// ...
ExpensiveObject obj = weakRef.get(); // может вернуть null
if (obj != null) {
    obj.doWork();
} else {
    // объект уже собран GC
}
```

❗ Ловушка: `SoftReference` — не гарантия кэширования. JVM собирает soft refs по "старейший первый" принципу перед OOM. Настроить aggressiveness можно через `-XX:SoftRefLRUPolicyMSPerMB`. Если soft-кэш "промахивается" слишком часто — это не экономит, а замедляет (reload + GC overhead).

**→ Уточняющий вопрос:** Чем `WeakHashMap` отличается от `HashMap` с `WeakReference` в значениях? В каком случае `WeakHashMap` не защищает от утечки?

---

## 27. Можно ли вручную вызвать GC?

Технически — да, через `System.gc()` или `Runtime.getRuntime().gc()`. Но это лишь *подсказка* JVM, не команда. JVM может полностью проигнорировать вызов (особенно если используется `-XX:+DisableExplicitGC` — флаг, который делает System.gc() no-op). Даже если JVM выполнит GC, это произойдёт не мгновенно и не гарантированно будет полным.

В JVM вызов `System.gc()` обычно инициирует Full GC — самый дорогой вид сборки с длительной STW-паузой. В G1 это обходит всю оптимизированную логику регионального сборщика и делает однопоточный Serial-like сбор. Именно поэтому это опасно: вы можете спровоцировать 5-секундную паузу в неподходящий момент (например, под высокой нагрузкой).

`System.gc()` иногда встречается в коде для "прогрева" JVM в тестах или для финализации объектов перед проверкой. Для финализации и `PhantomReference` обработки есть `System.runFinalization()` — тоже не рекомендуется. Единственный легитимный случай — явное тестирование поведения GC в unit-тестах (но даже там нет гарантий без специальных инструментов вроде GCInspector).

```java
// Антипаттерн — вызов в production коде
public void cleanupCache() {
    cache.clear();
    System.gc(); // опасно! может вызвать Full GC с длинной паузой
}

// Правильно — доверять GC
public void cleanupCache() {
    cache.clear(); // убираем ссылки, GC сделает остальное сам
}

// Тест с NullReference (не гарантия, но часто работает)
WeakReference<Object> ref = new WeakReference<>(new Object());
System.gc(); // попытка инициировать GC в тесте
assertNull(ref.get()); // может быть flaky!
```

❗ Ловушка: в некоторых старых фреймворках и библиотеках (включая некоторые версии RMI) `System.gc()` вызывается периодически внутри. Флаг `-XX:+DisableExplicitGC` убирает это поведение. RMI по умолчанию вызывает `System.gc()` каждую минуту через `sun.rmi.dgc.client.gcInterval`.

**→ Уточняющий вопрос:** Что делает флаг `-XX:+DisableExplicitGC`? В каком сценарии его обязательно нужно использовать вместе с NIO direct buffers?

---

## 28. Почему не стоит вызывать System.gc()?

Четыре основные причины: (1) **Нет гарантий** — JVM может проигнорировать вызов полностью. (2) **Full GC с длинной паузой** — обычно инициирует самый дорогой вид сборки. (3) **Мешает оптимизациям GC** — современные GC (G1, ZGC) тщательно планируют когда и что собирать; принудительный вызов ломает эти планы. (4) **Ложная уверенность** — разработчик думает, что "очистил" память, но это иллюзия.

Особенно опасно в связке с NIO direct buffers. Direct ByteBuffers (`ByteBuffer.allocateDirect()`) освобождаются через finalizer/Cleaner — то есть не сразу при GC, а потом. Иногда разработчики вызывают `System.gc()` чтобы "подтолкнуть" освобождение direct buffers. Правильное решение — явно вызывать `((DirectBuffer) buf).cleaner().clean()` или использовать `sun.misc.Cleaner` (или Java 9+ `MemorySegment`). Флаг `-XX:+DisableExplicitGC` без `-XX:+ExplicitGCInvokesConcurrent` с NIO-heavy приложением может вызвать OOM на direct buffers.

В production коде любой `System.gc()` — code smell. На code review это должно быть поднято как проблема. Если нужно освободить память — убери ссылки, доверяй GC. Если нужно мониторить — используй JFR, Micrometer, не стимулируй GC вручную.

```java
// Антипаттерн — "форсирование" очистки
public void processLargeFile(Path file) {
    byte[] data = Files.readAllBytes(file); // 500 MB
    process(data);
    data = null;
    System.gc(); // ПЛОХО: мешает GC, может заморозить сервис
}

// Правильно — освободить ссылку, остальное GC сделает сам
public void processLargeFile(Path file) {
    // Для очень больших файлов — стриминг вместо загрузки в память
    try (InputStream is = Files.newInputStream(file)) {
        processStream(is);
    } // data никогда не жила вся в heap
}
```

❗ Ловушка: флаг `-XX:+ExplicitGCInvokesConcurrent` делает `System.gc()` конкурентным (как G1 concurrent cycle) вместо Full GC. Это снижает вред от явных вызовов, но всё равно не повод их оставлять.

**→ Уточняющий вопрос:** Как правильно освобождать NIO direct ByteBuffers в Java? Почему `System.gc()` иногда помогает в этом случае, но является плохой практикой?
