# Многопоточность

> 📇 Справочник уровня middle. Формат: ответ по сути → механизм JVM/CPU → реальный пример → ❗ ловушка собеса → уточняющий вопрос.

**Всего вопросов: 27**

---

## 1. В чём разница между synchronized и volatile?

`volatile` гарантирует два свойства: видимость (запись в volatile-поле немедленно становится видна всем потокам через главную память) и запрет переупорядочивания инструкций вокруг этой переменной. Но `volatile` не даёт атомарности составных операций. `synchronized`, напротив, обеспечивает и видимость, и атомарность, и взаимное исключение — только один поток может исполнять синхронизированный блок в данный момент.

На уровне JMM: запись в volatile устанавливает happens-before с последующим чтением того же поля. `synchronized` устанавливает happens-before через освобождение/захват монитора, дополнительно гарантируя эксклюзивный доступ. На уровне CPU: `volatile` транслируется в инструкции с барьерами памяти (MESI-протокол сбрасывает кэш-линию), `synchronized` добавляет ещё и аппаратный lock на изменение.

Практический пример: флаг остановки сервиса — `volatile` достаточно, так как это одна запись и одно чтение. Счётчик запросов в HTTP-сервере — нужен `AtomicLong` или `synchronized`, потому что `requests++` — это read-modify-write:

```java
// НЕПРАВИЛЬНО: volatile не защищает инкремент
private volatile int requestCount = 0;
public void handleRequest() {
    requestCount++; // три операции: read, increment, write — не атомарно
}

// ПРАВИЛЬНО для счётчика:
private final AtomicInteger requestCount = new AtomicInteger(0);
public void handleRequest() {
    requestCount.incrementAndGet(); // одна CAS-инструкция
}

// ПРАВИЛЬНО для флага остановки:
private volatile boolean running = true;
public void stop() { running = false; }
public void run() { while (running) { processNext(); } }
```

❗ Ловушка собеса: кандидаты часто думают, что `volatile` делает переменную полностью потокобезопасной. Интервьюер попросит объяснить, почему `volatile int i; i++` не потокобезопасно. Ответ: `i++` компилируется в три байткод-операции, и два потока могут прочитать одно и то же значение до того, как любой из них запишет инкремент.

**→ Уточняющий вопрос:** Если я объявлю `volatile long` на 32-битной JVM, будет ли операция присвоения атомарной?

---

## 2. Что такое happens-before relationship?

Happens-before — это частичный порядок на операциях, определённый в Java Memory Model (JMM, JSR-133). Если операция A happens-before операции B, то JMM гарантирует: все эффекты памяти A (записи) будут видны при выполнении B. Это не значит, что A физически выполнится раньше B по времени; это гарантия видимости и корректного упорядочивания с точки зрения семантики программы.

Правила happens-before в JMM: разблокировка монитора happens-before захвата того же монитора; запись в `volatile` happens-before чтения того же поля; `Thread.start()` happens-before любой инструкции в запущенном потоке; всё в потоке happens-before `Thread.join()` в ожидающем потоке; транзитивность — если A hb B и B hb C, то A hb C.

На практике в сервисе обработки событий: главный поток публикует конфигурацию (`volatile config = newConfig`), рабочие потоки читают её — запись в volatile устанавливает hb, поэтому рабочие гарантированно увидят обновление. Без этой гарантии компилятор или CPU могут переупорядочить инструкции, и поток-читатель увидит частично инициализированный объект.

```java
// Публикация объекта через volatile — безопасно благодаря happens-before
class ConfigHolder {
    private volatile Config config;

    public void update(Config newConfig) {
        // Все записи внутри newConfig видны ПОСЛЕ того,
        // как другой поток прочитает this.config
        this.config = newConfig; // volatile write = happens-before барьер
    }

    public Config get() {
        return config; // volatile read — видит всё, что было до volatile write
    }
}
```

❗ Ловушка собеса: вопрос «что видит поток B, если поток A просто записал в обычное (non-volatile) поле?» — ответ: ничего не гарантируется, поток B может читать устаревшее значение из своего CPU-кэша сколь угодно долго.

**→ Уточняющий вопрос:** Устанавливает ли happens-before конструктор объекта и чтение его полей в другом потоке без дополнительной синхронизации?

---

## 3. Что такое visibility problem?

Проблема видимости возникает потому, что современные CPU имеют многоуровневые кэши (L1/L2/L3), и каждое ядро работает преимущественно со своей кэш-линией. Когда поток на ядре 1 пишет значение в переменную, оно сначала попадает в кэш этого ядра. Ядро 2, читающее ту же переменную, может получить устаревшее значение из своего кэша — до тех пор, пока не произойдёт синхронизация кэшей (cache coherency flush).

JVM дополнительно разрешает JIT-компилятору держать переменную в регистре потока и не сбрасывать её в heap вовсе, если не видит синхронизации. Это значит, что цикл `while (!flag)` может никогда не завершиться, если `flag` не объявлена `volatile` — компилятор «оптимизирует» чтение и поток будет вечно читать кэшированное `false`.

Реальный сценарий: в пуле соединений (connection pool) поток-монитор ставит флаг `drained = true` после закрытия всех соединений. Рабочие потоки проверяют этот флаг перед взятием соединения. Без `volatile` рабочие потоки могут долго не видеть `true` и продолжать брать соединения из опустошённого пула.

```java
// Демонстрация visibility problem
class VisibilityDemo {
    // Без volatile JIT может закэшировать значение в регистре
    private static boolean stopFlag = false; // ПРОБЛЕМА

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int count = 0;
            while (!stopFlag) { // может зациклиться навсегда!
                count++;
            }
            System.out.println("Stopped, count=" + count);
        });
        worker.start();
        Thread.sleep(100);
        stopFlag = true; // worker может никогда не увидеть это
    }
}

// Исправление: volatile
private static volatile boolean stopFlag = false;
```

❗ Ловушка собеса: «А `System.out.println` внутри цикла "починит" проблему?» — да, починит, потому что `println` использует `synchronized` блок, а значит устанавливает happens-before. Но это случайное совпадение, а не намеренная синхронизация.

**→ Уточняющий вопрос:** Чем visibility problem отличается от race condition — это одно и то же или разные явления?

---

## 4. Что такое монитор (monitor) в Java?

Монитор — это механизм синхронизации, встроенный в каждый объект Java. Он реализует паттерн «взаимное исключение + ожидание условия» и состоит из трёх компонентов: мьютекс (mutex) для взаимного исключения, entry set (набор потоков, ожидающих захвата монитора) и wait set (набор потоков, ожидающих условия через `wait()`).

На уровне JVM у каждого объекта есть заголовок (object header) в heap, содержащий mark word — 64/32 бита, в которых хранится состояние монитора. Пока монитор не захвачен, mark word хранит хэш-код и возраст GC. При первом захвате JVM использует biased locking (монитор «смещён» в пользу конкретного потока). При конкуренции происходит inflated locking — mark word начинает указывать на объект монитора (ObjectMonitor) в JVM heap, где хранится owner, entry queue и wait set.

Практически: каждый объект в роли lock — это монитор. В сервисе с общим кэшем `Map<String, Object> cache` можно синхронизироваться по самому `cache` или по отдельному lock-объекту, чтобы не раздражать тех, кто синхронизируется по другим причинам на том же объекте.

```java
// Явное использование монитора через wait/notify
class BoundedQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) {
            wait(); // освобождает монитор и ждёт в wait set
        }
        queue.offer(item);
        notifyAll(); // будит потоки из wait set
    }

    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();
        }
        T item = queue.poll();
        notifyAll();
        return item;
    }
}
```

❗ Ловушка собеса: `wait()` и `notify()` обязательно вызываются внутри `synchronized` блока того же объекта. Вызов вне `synchronized` бросит `IllegalMonitorStateException`. Кроме того, `wait()` всегда надо вызывать в цикле `while`, а не `if`, потому что возможны spurious wakeups.

**→ Уточняющий вопрос:** Чем отличается `notify()` от `notifyAll()` и когда опасно использовать `notify()`?

---

## 5. Как работает synchronized на уровне монитора?

При входе в `synchronized` блок JVM вставляет инструкцию `monitorenter`: JVM проверяет, не занят ли монитор объекта, и если свободен — устанавливает его владельцем текущий поток (или увеличивает счётчик входов для реентрантного захвата). При выходе из блока (нормальном или через исключение) — `monitorexit`: уменьшает счётчик входов и при нуле освобождает монитор.

На уровне байткода это видно через `javap -c`: для синхронизированного блока JVM генерирует `monitorenter`, затем `monitorexit` в конце блока и ещё один `monitorexit` в блоке обработки исключений (чтобы монитор освободился даже если вылетело исключение). Для `synchronized` метода JVM проверяет флаг `ACC_SYNCHRONIZED` в дескрипторе метода.

Оптимизации JVM: biased locking (JDK < 15) — первый захвативший поток «предвзято» владеет монитором, повторные входы не требуют CAS. Lightweight locking через CAS в стеке потока при умеренной конкуренции. Heavyweight locking (inflated monitor) через ОС-примитивы при высокой конкуренции — здесь возникает context switch и блокировка.

```java
// Что компилятор генерирует для synchronized блока
public void transfer(Account from, Account to, int amount) {
    synchronized (from) {    // monitorenter на объект from
        synchronized (to) {  // monitorenter на объект to
            from.debit(amount);
            to.credit(amount);
        }                    // monitorexit на to
    }                        // monitorexit на from
    // + ещё два monitorexit в exception handler
}
```

❗ Ловушка собеса: синхронизация на `Integer`, `String` и других interned/cached объектах может привести к неожиданному конфликту блокировок — два несвязанных компонента могут синхронизироваться на одном и том же объекте-литерале (`Integer.valueOf(1)` кэшируется в диапазоне -128..127).

**→ Уточняющий вопрос:** Начиная с какой версии JDK biased locking был отключён по умолчанию и почему?

---

## 6. В чём разница между synchronized методом и synchronized блоком?

`synchronized` метод неявно синхронизируется по `this` (для instance-методов) или по `ClassName.class` (для static-методов) и блокирует выполнение всего тела метода. `synchronized` блок требует явного указания объекта-монитора и позволяет синхронизировать только нужный участок кода, выбирая произвольный объект-локер.

Ключевые отличия: блок гибче — позволяет использовать разные локеры для разных структур данных в одном классе, тем самым уменьшая конкуренцию. Блок позволяет синхронизировать минимальный критический участок, не держа локер во время дорогих IO-операций. Синхронизированный метод синхронизируется по `this`, что означает: любой внешний код, получивший ссылку на объект, тоже может на него синхронизироваться и случайно заблокировать ваш метод.

В реальном сервисе с двумя независимыми кэшами использование двух отдельных локеров вместо `synchronized this` даёт вдвое меньше конкуренции:

```java
class UserService {
    private final Map<Long, User> userCache = new HashMap<>();
    private final Map<String, Session> sessionCache = new HashMap<>();

    // Два независимых локера — нет конкуренции между операциями с разными кэшами
    private final Object userLock = new Object();
    private final Object sessionLock = new Object();

    public User getUser(long id) {
        synchronized (userLock) {        // не блокирует sessionCache
            return userCache.get(id);
        }
    }

    public Session getSession(String token) {
        synchronized (sessionLock) {     // не блокирует userCache
            return sessionCache.get(token);
        }
    }

    // ХУЖЕ: один synchronized(this) для обоих — избыточная конкуренция
}
```

❗ Ловушка собеса: если сделать локер-поле `public` или передать ссылку на него наружу, внешний код может захватить ваш монитор и неожиданно заблокировать ваши методы. Локер должен быть `private final`.

**→ Уточняющий вопрос:** Можно ли синхронизироваться на `null`-ссылке, и что произойдёт?

---

## 7. Что такое reentrant lock?

Реентрантная (повторно входимая) блокировка — это блокировка, которую один поток может захватить несколько раз подряд, не блокируя самого себя. JVM хранит счётчик захватов и идентификатор владельца: при повторном захвате тем же потоком счётчик увеличивается, при освобождении уменьшается. Реальное освобождение происходит только когда счётчик достигает нуля.

Оба примитива — `synchronized` и `ReentrantLock` — реентрантны. Это важно при рекурсии и при вызове одного синхронизированного метода из другого в том же классе. Без реентрантности такой вызов повис бы в deadlock — поток ждал бы освобождения монитора, который сам же и держит.

`ReentrantLock` даёт дополнительные возможности: `tryLock(timeout)` для попытки захвата с таймаутом, `lockInterruptibly()` для прерываемого ожидания, `fair=true` для честной очереди (FIFO), `newCondition()` для нескольких condition variables вместо одного monitor wait set.

```java
// Реентрантность: метод A вызывает метод B, оба synchronized — не deadlock
class TreeProcessor {
    public synchronized int processNode(Node node) {
        int result = node.value;
        for (Node child : node.children) {
            result += processNode(child); // рекурсия — повторный захват монитора this
        }                                 // JVM увеличивает счётчик, не блокирует
        return result;
    }
}

// ReentrantLock с tryLock — предотвращение deadlock в переводе средств
class AccountService {
    public boolean transfer(Account from, Account to, int amount) {
        try {
            if (from.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                try {
                    if (to.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                        try {
                            from.debit(amount);
                            to.credit(amount);
                            return true;
                        } finally { to.lock.unlock(); }
                    }
                } finally { from.lock.unlock(); }
            }
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return false; // не удалось захватить оба — попробуй позже
    }
}
```

❗ Ловушка собеса: `ReentrantLock` необходимо освобождать вручную в блоке `finally`. Пропуск `unlock()` в `finally` приведёт к тому, что блокировка не будет освобождена при исключении — вечный deadlock для всех ожидающих потоков.

**→ Уточняющий вопрос:** Что такое fair и unfair режим `ReentrantLock` и как честность влияет на производительность?

---

## 8. Что такое Atomic классы?

Atomic классы из пакета `java.util.concurrent.atomic` предоставляют операции, атомарные на уровне отдельной инструкции процессора, без явных блокировок. Основные: `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference<V>`, `AtomicIntegerArray`, `AtomicStampedReference` (для борьбы с ABA). С Java 8 добавлены `LongAdder` и `LongAccumulator`, оптимизированные для высокой конкуренции.

Реализованы через `sun.misc.Unsafe` (или `VarHandle` в новых JDK), который предоставляет прямой доступ к CAS-инструкциям процессора (`CMPXCHG` на x86). Значение хранится в `volatile` поле, что обеспечивает видимость, а операции выполняются CAS-циклом (read-CAS-retry) для атомарности без блокировки.

В реальных сервисах `AtomicLong` используется как счётчик запросов в метриках, `AtomicReference` — для lock-free обновления снапшота конфигурации, `AtomicInteger` — для идентификаторов задач в очереди:

```java
// Сравнение подходов для счётчика запросов в веб-сервисе
class RequestMetrics {
    // Вариант 1: synchronized (всегда сериализует потоки)
    private int syncCounter = 0;
    public synchronized void syncIncrement() { syncCounter++; }

    // Вариант 2: AtomicLong (lock-free, хорошо при умеренной конкуренции)
    private final AtomicLong atomicCounter = new AtomicLong();
    public void atomicIncrement() { atomicCounter.incrementAndGet(); }

    // Вариант 3: LongAdder (лучше при высокой конкуренции — stripe по ядрам)
    private final LongAdder adderCounter = new LongAdder();
    public void adderIncrement() { adderCounter.increment(); }
    public long adderGet() { return adderCounter.sum(); } // менее точно в моменте
}
```

❗ Ловушка собеса: `AtomicReference.set()` и `get()` атомарны по отдельности, но последовательность `get()` + условная логика + `set()` не атомарна. Для атомарного условного обновления нужен `compareAndSet()` или `updateAndGet()`.

**→ Уточняющий вопрос:** Когда `LongAdder` предпочтительнее `AtomicLong`, и какой ценой достигается его преимущество в производительности?

---

## 9. Что такое CAS (Compare-And-Swap)?

CAS — это атомарная инструкция процессора (`CMPXCHG` на x86, `LDREX/STREX` на ARM), выполняющая операцию «прочитай текущее значение по адресу памяти; если оно равно ожидаемому (expected) — запиши новое значение (update); верни, было ли обновление успешным». Вся операция выполняется как единая неделимая инструкция на уровне железа.

На уровне Java: `Unsafe.compareAndSwapInt(object, offset, expected, update)` транслируется напрямую в инструкцию `LOCK CMPXCHG`. При наличии конкуренции (другой поток успел изменить значение) CAS возвращает `false` и алгоритм выполняет spin-retry — перечитывает текущее значение и пробует снова. Это неблокирующий подход: ни один поток не засыпает, но возможен активный ожидание под нагрузкой.

Важная проблема CAS — ABA: поток A читает значение `A`, поток B меняет `A→B→A`. Когда поток A делает CAS, он видит `A` и считает, что ничего не менялось, хотя структура данных могла существенно измениться. В lock-free стеке это может привести к повреждению данных. Решение — `AtomicStampedReference`, добавляющий версионный штамп к значению.

```java
// Собственная реализация инкремента через CAS — как это работает внутри AtomicInteger
public int incrementWithCAS(AtomicInteger ref) {
    int current, next;
    do {
        current = ref.get();          // 1. читаем текущее значение
        next = current + 1;           // 2. вычисляем новое
    } while (!ref.compareAndSet(current, next)); // 3. CAS: если current не изменился — пишем next
    return next;                      // при конкуренции: повторяем с шага 1
}

// ABA-проблема в lock-free стеке
class ABADemo {
    AtomicReference<Node> head = new AtomicReference<>();

    // Поток A: читает head=NodeA, поток B: pop NodeA, push NodeC, push NodeA
    // Поток A: CAS(NodeA, NodeB) — УСПЕХ! Но next у NodeA теперь указывает на NodeC,
    // а не на то, что было раньше — стек повреждён
}

// Решение ABA: AtomicStampedReference
AtomicStampedReference<Node> stampedHead = new AtomicStampedReference<>(null, 0);
// Теперь CAS проверяет и значение, и версию
```

❗ Ловушка собеса: «CAS всегда быстрее synchronized» — неверно. При очень высокой конкуренции CAS-цикл генерирует большой трафик на шине памяти из-за постоянных cache invalidation. `synchronized` в этом случае лучше, так как проигравшие потоки засыпают, а не жгут CPU в spin-цикле.

**→ Уточняющий вопрос:** Как `AtomicStampedReference` устраняет ABA-проблему и есть ли у него недостатки?

---

## 10. Как работают AtomicInteger, AtomicLong?

`AtomicInteger` хранит значение в поле `private volatile int value`. Все операции мутации (`incrementAndGet`, `addAndGet`, `compareAndSet`, `getAndUpdate`) реализованы через `VarHandle` (Java 9+) или `Unsafe`, которые транслируются в CAS-инструкции. Операции чтения (`get`) используют `volatile` семантику для гарантии свежести значения.

Метод `updateAndGet(IntUnaryOperator op)` реализует общий паттерн CAS-цикла: читает текущее, применяет функцию, пытается CAS, при неудаче повторяет. Это позволяет атомарно применять произвольную функцию без внешней синхронизации. `getAndUpdate` аналогичен, но возвращает старое значение. `accumulateAndGet(int x, IntBinaryOperator op)` атомарно применяет бинарную операцию.

На практике `AtomicInteger` идеален для счётчиков активных соединений в пуле, идентификаторов задач, метрик запросов. `AtomicLong` — для временных меток, накопительных счётчиков байт, etc.

```java
// Connection pool с AtomicInteger счётчиком активных соединений
class ConnectionPool {
    private final int maxSize;
    private final AtomicInteger activeConnections = new AtomicInteger(0);
    private final BlockingQueue<Connection> pool;

    public Connection acquire() throws InterruptedException {
        Connection conn = pool.poll(100, TimeUnit.MILLISECONDS);
        if (conn != null) {
            activeConnections.incrementAndGet();
        }
        return conn;
    }

    public void release(Connection conn) {
        pool.offer(conn);
        activeConnections.decrementAndGet();
    }

    // Атомарная логика: взять соединение только если лимит не превышен
    public Connection acquireIfAvailable() {
        // getAndUpdate атомарно проверяет и обновляет
        int prev = activeConnections.getAndUpdate(
            current -> current < maxSize ? current + 1 : current
        );
        if (prev < maxSize) {
            return pool.poll();
        }
        return null; // лимит достигнут
    }
}
```

❗ Ловушка собеса: `AtomicInteger.intValue()` вызывает `get()` — это корректно. Но `new Integer(atomicInt.get())` или `atomicInt.intValue()` — просто чтение, без CAS. Опасность возникает когда read и последующее действие (например, условие) не атомарны — нужно использовать `compareAndSet` или `updateAndGet`.

**→ Уточняющий вопрос:** Почему `AtomicInteger` не наследует `Integer` и какое это имеет практическое значение?

---

## 11. В чём преимущество Atomic классов перед synchronized?

Главное преимущество: Atomic классы используют неблокирующие алгоритмы на основе CAS. Поток, не сумевший выполнить CAS, не засыпает (нет context switch), а немедленно повторяет попытку. Это устраняет накладные расходы ОС на переключение контекста (~1-10 мкс), пробуждение потока и системный вызов. При умеренной конкуренции (2-4 потока) Atomic заметно быстрее `synchronized`.

Дополнительные преимущества: нет риска deadlock (нет блокировок); лучшая масштабируемость при росте числа ядер при низкой-средней конкуренции; богатый API для функциональных атомарных операций (`updateAndGet`, `accumulateAndGet`). `LongAdder` идёт дальше: он разбивает счётчик на несколько ячеек по числу конкурирующих потоков (striped), что почти устраняет конкуренцию на запись.

Недостатки: при очень высокой конкуренции (десятки потоков на одну переменную) CAS-циклы создают «шторм» инвалидации кэш-линий на шине памяти (cache line bouncing). В этом случае `synchronized` с засыпанием потоков эффективнее. Кроме того, Atomic классы не решают задачу координации нескольких переменных — для этого нужна `synchronized` или `StampedLock`.

```java
// Бенчмарк-сравнение подходов при 8 потоках
// (иллюстративно — для настоящего бенчмарка использовать JMH)
class ConcurrencyComparison {
    // synchronized: потоки засыпают, меньше CPU при конкуренции
    private int syncVal = 0;
    public synchronized int syncIncrement() { return ++syncVal; }

    // AtomicInteger: хорошо при низкой/умеренной конкуренции
    private AtomicInteger atomicVal = new AtomicInteger();
    public int atomicIncrement() { return atomicVal.incrementAndGet(); }

    // LongAdder: лучший для часто инкрементируемых счётчиков
    // суммирует стрипы при чтении — итог может быть немного устаревшим
    private LongAdder adder = new LongAdder();
    public void adderIncrement() { adder.increment(); }
    public long adderGet() { return adder.sum(); }
}
```

❗ Ловушка собеса: «Atomic всегда быстрее» — распространённое заблуждение. При 32+ потоках, постоянно инкрементирующих один `AtomicLong`, производительность может быть хуже `synchronized`, потому что все ядра спорят за одну кэш-линию (64 байта). Именно для этого создан `LongAdder`.

**→ Уточняющий вопрос:** Как `LongAdder` технически обходит проблему конкуренции за кэш-линию?

---

## 12. Что такое пул потоков (Thread Pool)?

Пул потоков — это паттерн управления ресурсами: набор заранее созданных (или создаваемых по мере необходимости) потоков, переиспользуемых для выполнения задач. Вместо создания нового потока на каждую задачу (дорого: ~1-2 МБ стека ОС, системный вызов, TLB flush) задача ставится в очередь и выполняется одним из свободных потоков пула.

Компоненты `ThreadPoolExecutor`: `corePoolSize` (минимальное число потоков, живут постоянно), `maximumPoolSize` (максимум при пиковой нагрузке), `keepAliveTime` (время простоя сверх-core потоков до завершения), `workQueue` (буфер задач — `LinkedBlockingQueue`, `ArrayBlockingQueue`, `SynchronousQueue`), `RejectedExecutionHandler` (что делать, если и очередь, и пул переполнены).

В HTTP-сервере Tomcat по умолчанию использует пул `200` потоков. Каждый входящий запрос отправляется в пул. Если все 200 заняты и очередь (acceptCount) переполнена — новые соединения отклоняются. Правильный sizing пула критически важен: слишком мало — запросы ждут; слишком много — memory pressure от стеков.

```java
// Правильное создание ThreadPoolExecutor для веб-сервиса
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                          // corePoolSize: всегда 10 потоков
    50,                          // maximumPoolSize: до 50 при пиках
    60L, TimeUnit.SECONDS,       // keepAlive для сверх-core потоков
    new ArrayBlockingQueue<>(200), // ограниченная очередь (не бесконечная!)
    new ThreadFactory() {        // именованные потоки для диагностики
        private final AtomicInteger n = new AtomicInteger();
        public Thread newThread(Runnable r) {
            return new Thread(r, "request-processor-" + n.incrementAndGet());
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // при overflow — выполни в вызывающем потоке
);

// Shutdown gracefully
executor.shutdown();
executor.awaitTermination(30, TimeUnit.SECONDS);
```

❗ Ловушка собеса: `Executors.newFixedThreadPool(n)` использует `LinkedBlockingQueue` без ограничения размера. При медленном потреблении и быстром поступлении задач очередь растёт до OOM. В продакшене всегда используй `ThreadPoolExecutor` с ограниченной очередью напрямую.

**→ Уточняющий вопрос:** Как правильно подобрать `corePoolSize` для IO-bound vs CPU-bound задач?

---

## 13. Какие типы Thread Pool существуют в Java?

Java предоставляет фабричные методы `Executors` для создания типовых пулов: `newFixedThreadPool(n)` — фиксированное число потоков с неограниченной очередью; `newCachedThreadPool()` — растущий пул без лимита с `SynchronousQueue` и 60-секундным keepAlive; `newSingleThreadExecutor()` — один поток, гарантированная последовательность задач; `newScheduledThreadPool(n)` — поддержка отложенных и периодических задач; `newWorkStealingPool()` — ForkJoinPool с parallelism по числу CPU; Java 21 — `newVirtualThreadPerTaskExecutor()` — виртуальный поток на каждую задачу.

Каждый тип имеет свой профиль применения. Кроме того, напрямую `ThreadPoolExecutor` даёт полный контроль. `ForkJoinPool` используется параллельными стримами и CompletableFuture.

```java
// Сравнение типов пулов в контексте применения

// 1. Fixed: ограниченный параллелизм для CPU-bound задач
ExecutorService fixed = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);

// 2. Cached: короткие burst IO-задачи, не подходит для долгих
ExecutorService cached = Executors.newCachedThreadPool();

// 3. Scheduled: периодическая очистка кэша каждые 5 минут
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
scheduler.scheduleAtFixedRate(
    () -> cache.evictExpired(),
    5, 5, TimeUnit.MINUTES
);

// 4. Virtual threads (Java 21): IO-bound с тысячами конкурентных задач
ExecutorService virtual = Executors.newVirtualThreadPerTaskExecutor();
// Или через Thread.ofVirtual().executor()

// 5. ПРАВИЛЬНЫЙ способ для продакшена: ThreadPoolExecutor напрямую
ThreadPoolExecutor production = new ThreadPoolExecutor(
    5, 20, 30, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.AbortPolicy()
);
```

❗ Ловушка собеса: `newSingleThreadExecutor()` — это не просто `newFixedThreadPool(1)`. `SingleThreadExecutor` оборачивается в `FinalizableDelegatedExecutorService`, который не позволяет кастовать к `ThreadPoolExecutor` и изменить размер пула. Это намеренная защита инварианта "ровно один поток".

**→ Уточняющий вопрос:** Что произойдёт с задачами в очереди `newFixedThreadPool`, если вызвать `shutdownNow()`?

---

## 14. Что делает ExecutorService?

`ExecutorService` — это интерфейс, расширяющий `Executor`, который абстрагирует управление жизненным циклом пула потоков и предоставляет способы подачи задач и получения результатов. В отличие от базового `Executor.execute(Runnable)`, `ExecutorService` поддерживает `Callable` (задачи с результатом), `Future` (дескриптор асинхронного результата) и корректное завершение работы.

Ключевые методы: `submit(Callable<T>)` / `submit(Runnable)` — отправить задачу и получить `Future`; `invokeAll(Collection<Callable<T>>)` — выполнить все задачи и ждать завершения каждой; `invokeAny(Collection<Callable<T>>)` — вернуть результат первой завершившейся задачи; `shutdown()` — прекратить приём новых задач, дождаться завершения текущих; `shutdownNow()` — послать interrupt всем выполняющимся потокам и вернуть список ожидающих задач; `awaitTermination(timeout)` — ждать полного завершения.

В реальном сервисе обработки заказов: `invokeAll` используется для параллельной проверки нескольких источников (наличие товара, баланс клиента, антифрод), `invokeAny` — для геолокации по первому ответившему из нескольких провайдеров.

```java
// Паттерн корректного завершения ExecutorService
class OrderProcessor {
    private final ExecutorService executor = new ThreadPoolExecutor(
        4, 10, 60, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(500)
    );

    // invokeAll: параллельная проверка всех условий заказа
    public OrderValidation validateOrder(Order order) throws InterruptedException {
        List<Callable<ValidationResult>> checks = List.of(
            () -> inventoryService.check(order),
            () -> paymentService.checkBalance(order),
            () -> fraudService.check(order)
        );

        List<Future<ValidationResult>> futures = executor.invokeAll(
            checks, 2, TimeUnit.SECONDS // общий таймаут
        );

        return aggregateResults(futures);
    }

    // Корректный shutdown при остановке приложения (Spring @PreDestroy)
    public void shutdown() throws InterruptedException {
        executor.shutdown(); // больше не принимаем задачи
        if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
            List<Runnable> pending = executor.shutdownNow(); // interrupt текущих
            log.warn("Forced shutdown, {} tasks abandoned", pending.size());
        }
    }
}
```

❗ Ловушка собеса: после `shutdown()` пул ещё работает (завершает принятые задачи) — нельзя сразу проверять `isTerminated()`. Для ожидания полной остановки нужен `awaitTermination()`. Пропуск этого шага при деплое может привести к потере задач.

**→ Уточняющий вопрос:** Если `Future.get()` бросает `ExecutionException` — как получить исходное исключение из задачи?

---

## 15. В чём разница между Executors.newFixedThreadPool() и newCachedThreadPool()?

`newFixedThreadPool(n)` создаёт ровно `n` потоков и `LinkedBlockingQueue` без ограничения размера. Если все потоки заняты, задачи накапливаются в очереди неограниченно. Потоки живут вечно (keepAlive = 0). Подходит для ограничения параллелизма CPU-bound задач.

`newCachedThreadPool()` создаёт потоки по мере необходимости без верхнего ограничения, переиспользует свободные в течение 60 секунд. Использует `SynchronousQueue` — задача не буферизуется, а передаётся напрямую в поток (или создаётся новый). При наплыве задач создаётся неограниченное число потоков — при 10 000 одновременных запросов появится 10 000 потоков, каждый со стеком 512KB-1MB = OOM или context switch storm.

Реальная ситуация: веб-сервис получает спайк нагрузки. С `newFixedThreadPool(100)` — 100 потоков работают, остальные запросы ждут в очереди (может расти до OOM). С `newCachedThreadPool()` — 10 000 потоков создаются за секунды, память исчерпана, JVM падает. Правильное решение: `ThreadPoolExecutor` с ограниченной очередью и явным rejection policy.

```java
// Сравнение поведения при спайке нагрузки
class ThreadPoolComparison {

    // РИСК OOM: бесконечная очередь при fixed
    ExecutorService fixed = Executors.newFixedThreadPool(10);
    // При 100k задач: 10 потоков, 99990 задач в LinkedBlockingQueue -> OOM

    // РИСК OOM: бесконечные потоки при cached
    ExecutorService cached = Executors.newCachedThreadPool();
    // При 100k задач: 100k потоков -> OOM + context switch hell

    // ПРАВИЛЬНО: явные ограничения + rejection policy
    ThreadPoolExecutor safe = new ThreadPoolExecutor(
        10,    // core: всегда живые потоки
        50,    // max: при пике создаём до 50
        30, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(200), // буфер 200 задач
        new ThreadPoolExecutor.CallerRunsPolicy()
        // overflow -> задача выполнится в вызывающем потоке
        // это создаёт backpressure на источник задач
    );
}
```

❗ Ловушка собеса: интервьюер часто спрашивает «что произойдёт, если в `newCachedThreadPool` отправить 100 000 задач». Правильный ответ: JVM попытается создать 100 000 потоков (примерно 100 ГБ стека) и упадёт с `OutOfMemoryError: unable to create native thread`.

**→ Уточняющий вопрос:** Что такое `CallerRunsPolicy` и как оно помогает реализовать backpressure?

---

## 16. Что такое ForkJoinPool?

`ForkJoinPool` — специализированный пул потоков, оптимизированный для рекурсивных divide-and-conquer задач. Идея: большая задача разбивается (`fork`) на подзадачи, каждая подзадача может разбиваться дальше, результаты объединяются (`join`). Центральный элемент — work stealing: у каждого потока своя двусторонняя очередь (`deque`), свободный поток крадёт задачи с конца очереди занятого потока, минимизируя простой.

`ForkJoinPool` — общий пул по умолчанию для parallel streams (`IntStream.range(...).parallel()`) и `CompletableFuture.supplyAsync()` (без явного executor). Параллелизм common pool = `Runtime.getRuntime().availableProcessors() - 1`. Это означает, что долгая блокирующая задача в `CompletableFuture.supplyAsync()` без своего executor заблокирует общий пул и повлияет на parallel streams во всём приложении.

Задачи реализуются через `RecursiveTask<V>` (возвращает результат) или `RecursiveAction` (без результата):

```java
// ForkJoinPool для параллельного вычисления суммы массива
class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000;
    private final long[] array;
    private final int from, to;

    @Override
    protected Long compute() {
        if (to - from <= THRESHOLD) {
            // Базовый случай: считаем напрямую
            long sum = 0;
            for (int i = from; i < to; i++) sum += array[i];
            return sum;
        }
        // Divide: разбиваем пополам
        int mid = (from + to) / 2;
        SumTask left = new SumTask(array, from, mid);
        SumTask right = new SumTask(array, mid, to);
        left.fork();            // async запуск левой подзадачи
        long rightResult = right.compute(); // текущий поток выполняет правую
        long leftResult = left.join();      // ждём левую
        return leftResult + rightResult;
    }
}

// ВАЖНО: для IO-bound задач в CompletableFuture — НЕ используй common pool!
// Создай свой executor
ExecutorService ioExecutor = Executors.newFixedThreadPool(50);
CompletableFuture.supplyAsync(() -> callExternalApi(), ioExecutor);
```

❗ Ловушка собеса: использование блокирующих операций (JDBC, HTTP) внутри `parallel()` stream или `CompletableFuture.supplyAsync()` без явного executor блокирует потоки common ForkJoinPool. При 4 ядрах common pool имеет 3 потока — три медленных DB-запроса парализуют все параллельные вычисления в приложении.

**→ Уточняющий вопрос:** Как `ManagedBlocker` в ForkJoinPool помогает при необходимости использовать блокирующие операции?

---

## 17. Что такое deadlock (взаимная блокировка)?

Deadlock — состояние системы, когда два или более потоков навечно заблокированы, ожидая ресурсы, удерживаемые друг другом. Граф ожидания образует цикл: поток A ждёт ресурс у потока B, поток B ждёт ресурс у потока A — ни один не может продвинуться. Это не исключение; ни один поток не завершается, JVM не обнаруживает deadlock автоматически и не восстанавливается.

На практике классический сценарий — банковский перевод: нужно заблокировать оба счёта для атомарности. Если один поток блокирует счёт `A` затем `B`, а другой — `B` затем `A`, и они делают это одновременно, оба окажутся в вечном ожидании. Deadlock в продакшене проявляется как зависший сервис с 0% CPU и 100% потоков в состоянии `BLOCKED` (видно в thread dump через `kill -3` или `jstack`).

```java
// Классический deadlock на примере банковского перевода
class DeadlockDemo {
    static final Object lockA = new Object(); // счёт A
    static final Object lockB = new Object(); // счёт B

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {              // поток 1 захватывает A
                sleepMs(100);                   // "работает" с A
                synchronized (lockB) {          // ждёт B... вечно
                    System.out.println("T1 done");
                }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (lockB) {              // поток 2 захватывает B
                sleepMs(100);                   // "работает" с B
                synchronized (lockA) {          // ждёт A... вечно
                    System.out.println("T2 done");
                }
            }
        });

        t1.start(); t2.start();
        // Оба потока зависнут. JVM не упадёт.
        // jstack покажет: "Found one Java-level deadlock"
    }
}
```

❗ Ловушка собеса: «JVM бросит исключение при deadlock» — нет. JVM только детектирует deadlock через ` ThreadMXBean.findDeadlockedThreads()` при запросе, но не прерывает потоки. Инструмент обнаружения: `jstack <pid>` покажет «Found one Java-level deadlock» с трассировкой.

**→ Уточняющий вопрос:** Как с помощью `jstack` диагностировать deadlock в работающем Java-процессе?

---

## 18. Какие условия необходимы для возникновения deadlock?

Четыре условия Коффмана (1971) — необходимые и достаточные для deadlock. Все четыре должны выполняться одновременно:

1. **Взаимное исключение** (Mutual Exclusion): ресурс может использоваться только одним потоком одновременно. Если ресурс доступен нескольким потокам сразу — deadlock невозможен.

2. **Удержание с ожиданием** (Hold and Wait): поток удерживает минимум один ресурс и ожидает ещё один. Если запрещено удерживать ресурсы при ожидании (взять все сразу или не брать) — deadlock невозможен.

3. **Отсутствие принудительного освобождения** (No Preemption): ресурс не может быть принудительно отнят у потока. Если ОС/JVM могут отнять ресурс — deadlock невозможен.

4. **Циклическое ожидание** (Circular Wait): существует цикл A ждёт B, B ждёт C, ..., N ждёт A. Нарушение порядка захвата ресурсов устраняет это условие.

```java
// Демонстрация всех четырёх условий в контексте очереди задач
// 1. Mutual exclusion: synchronized блоки — только один поток входит
// 2. Hold & wait: поток держит queue.lock и ждёт cache.lock
// 3. No preemption: synchronized нельзя отнять принудительно
// 4. Circular wait: TaskProducer: queue -> cache; CacheReloader: cache -> queue

class TaskProducer {
    void produce() {
        synchronized (queue) {          // держит queue
            synchronized (cache) {      // ждёт cache -> deadlock если CacheReloader активен
                processAndCache();
            }
        }
    }
}

class CacheReloader {
    void reload() {
        synchronized (cache) {          // держит cache
            synchronized (queue) {      // ждёт queue -> deadlock если TaskProducer активен
                reloadFromQueue();
            }
        }
    }
}
```

❗ Ловушка собеса: интервьюер может спросить «достаточно ли трёх условий Коффмана?» — нет, нужны все четыре. Устранение любого одного условия предотвращает deadlock. На практике самое доступное для устранения — циклическое ожидание (установить глобальный порядок захвата ресурсов).

**→ Уточняющий вопрос:** Какое из четырёх условий Коффмана проще всего устранить программно и как именно?

---

## 19. Как предотвратить deadlock?

Главная практическая стратегия: установить глобальный порядок захвата блокировок и всегда следовать ему. Если все потоки захватывают ресурсы в одном порядке (например, по системному ID объекта), циклическое ожидание становится невозможным. В банковском примере: всегда блокировать счёт с меньшим ID первым.

Вторая стратегия: `tryLock(timeout)` из `ReentrantLock`. Поток пытается захватить блокировку за ограниченное время; при неудаче освобождает уже захваченные и повторяет. Это нарушает условие "no preemption" с точки зрения самого потока. Третья стратегия: избегать вложенных блокировок — взять lock, сделать работу, отпустить, взять следующий. Четвёртая: использовать неблокирующие структуры данных (`ConcurrentHashMap`, `CopyOnWriteArrayList`) где возможно.

```java
// Предотвращение deadlock в банковском переводе — три подхода

// Подход 1: глобальный порядок по ID (устраняет circular wait)
void transfer1(Account from, Account to, int amount) {
    Account first = from.id < to.id ? from : to;
    Account second = from.id < to.id ? to : from;
    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}

// Подход 2: tryLock с таймаутом и retry
void transfer2(Account from, Account to, int amount) throws InterruptedException {
    while (true) {
        if (from.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
            try {
                if (to.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                    try {
                        from.debit(amount); to.credit(amount); return;
                    } finally { to.lock.unlock(); }
                }
            } finally { from.lock.unlock(); }
        }
        // Не удалось — ждём случайное время (exponential backoff)
        Thread.sleep(ThreadLocalRandom.current().nextLong(10, 100));
    }
}

// Подход 3: один глобальный lock (проще, но хуже масштабируется)
private static final ReentrantLock globalLock = new ReentrantLock();
void transfer3(Account from, Account to, int amount) {
    globalLock.lock();
    try { from.debit(amount); to.credit(amount); }
    finally { globalLock.unlock(); }
}
```

❗ Ловушка собеса: `tryLock` с retry без random jitter может привести к livelock — оба потока одновременно освобождают блокировки, одновременно пробуют снова и снова мешают друг другу. Добавление случайного backoff решает livelock.

**→ Уточняющий вопрос:** Чем livelock отличается от deadlock, и как его распознать в thread dump?

---

## 20. Что такое race condition?

Race condition (состояние гонки) — ошибка, при которой корректность результата зависит от порядка и тайминга выполнения операций в нескольких потоках. В отличие от visibility problem (поток не видит актуальное значение), race condition возникает даже если видимость обеспечена: два потока видят актуальное значение, но конкурируют за его изменение.

Классический пример: `check-then-act`. Поток A проверяет `if (balance >= amount)`, поток B делает то же самое, оба видят достаточный баланс, оба выполняют списание. В результате баланс уходит в минус. Промежуток между проверкой и действием не атомарен — это и есть race condition. На уровне байткода: `getfield`, `if_icmplt`, `putfield` — три отдельные инструкции, между которыми может вклиниться другой поток.

Реальный сценарий: кэш с lazy initialization. Два потока одновременно обращаются за значением, оба видят `null`, оба создают объект — один из них будет потерян или будут два разных экземпляра (проблема для singleton-ресурсов вроде DB-соединения).

```java
// Race condition в наивной реализации кэша
class UnsafeCache<K, V> {
    private final Map<K, V> map = new HashMap<>();

    // race condition: два потока могут одновременно пройти if (map.get == null)
    public V get(K key, Supplier<V> loader) {
        V value = map.get(key);      // read
        if (value == null) {         // check
            value = loader.get();    // act (вычисление)
            map.put(key, value);     // write — может затереть другой поток
        }
        return value;
    }
}

// Исправление 1: computeIfAbsent — атомарная check-and-put операция
class SafeCache<K, V> {
    private final ConcurrentHashMap<K, V> map = new ConcurrentHashMap<>();

    public V get(K key, Supplier<V> loader) {
        return map.computeIfAbsent(key, k -> loader.get()); // атомарно
    }
}

// Исправление 2: double-checked locking для дорогой инициализации
class ExpensiveResource {
    private volatile Resource instance; // volatile обязателен!

    public Resource get() {
        if (instance == null) {                   // первая проверка (без lock)
            synchronized (this) {
                if (instance == null) {            // вторая проверка (с lock)
                    instance = new Resource();     // volatile запись
                }
            }
        }
        return instance; // volatile чтение
    }
}
```

❗ Ловушка собеса: double-checked locking без `volatile` — классическая ловушка. Без `volatile` JIT может переупорядочить инструкции в конструкторе: другой поток увидит ненулевую ссылку на частично инициализированный объект. `volatile` запрещает такое переупорядочивание.

**→ Уточняющий вопрос:** Почему `ConcurrentHashMap.computeIfAbsent` не полностью заменяет `synchronized` блок при дорогостоящем вычислении значения?

---

## 21. Как избежать race condition?

Базовая стратегия: сделать операцию check-then-act атомарной. Три подхода в порядке предпочтительности: 1) иммутабельность — если объект не изменяется, race condition невозможен; 2) ограничение потока (thread confinement) — изменяемое состояние доступно только одному потоку (Netty EventLoop, LMAX Disruptor); 3) синхронизация — `synchronized`, `Lock`, атомарные классы, `ConcurrentHashMap`.

Выбор механизма синхронизации зависит от задачи: одна переменная с простыми операциями — `Atomic*`; несколько переменных в одной транзакции — `synchronized` блок; long-lived операции с таймаутом и прерыванием — `ReentrantLock`; часто-читаемые редко-пишемые данные — `ReadWriteLock` или `StampedLock`.

В практике: счётчик активных пользователей — `AtomicInteger`; обновление профиля пользователя (несколько полей) — `synchronized` на объекте пользователя; маршрутизация запросов по конфигурации — `volatile` + иммутабельная конфигурация; кэш сессий — `ConcurrentHashMap`.

```java
// Различные стратегии для реального сервиса
class SessionService {
    // Иммутабельность: сессия не меняется после создания
    record Session(String token, long userId, Instant expiresAt) {}

    // Thread confinement: каждый пользователь обрабатывается в своём потоке
    private final ConcurrentHashMap<String, Session> sessions = new ConcurrentHashMap<>();

    // ReadWriteLock: много чтений, редкие обновления конфигурации
    private final ReadWriteLock configLock = new ReentrantReadWriteLock();
    private RateLimitConfig config;

    public boolean isRateLimited(String userId) {
        configLock.readLock().lock(); // много потоков читают одновременно
        try { return config.check(userId); }
        finally { configLock.readLock().unlock(); }
    }

    public void updateConfig(RateLimitConfig newConfig) {
        configLock.writeLock().lock(); // эксклюзивный доступ при обновлении
        try { this.config = newConfig; }
        finally { configLock.writeLock().unlock(); }
    }

    // StampedLock (Java 8+): оптимистичное чтение без блокировки
    private final StampedLock sl = new StampedLock();
    private double rate;

    public double getRate() {
        long stamp = sl.tryOptimisticRead(); // не блокирует!
        double r = rate;
        if (!sl.validate(stamp)) {           // проверяем, не было ли записи
            stamp = sl.readLock();           // деградируем до обычного readlock
            try { r = rate; }
            finally { sl.unlockRead(stamp); }
        }
        return r;
    }
}
```

❗ Ловушка собеса: `Collections.synchronizedMap()` делает каждый отдельный метод атомарным, но не их последовательность. `if (!map.containsKey(k)) map.put(k, v)` всё равно race condition — два потока пройдут `containsKey(false)` до первого `put`. Нужен `ConcurrentHashMap.putIfAbsent()`.

**→ Уточняющий вопрос:** Чем `StampedLock` отличается от `ReentrantReadWriteLock` и когда оптимистичное чтение оправдано?

---

## 22. Что такое Virtual Threads в Java 21?

Virtual Threads (Project Loom) — лёгкие потоки, управляемые JVM, а не операционной системой. Обычный платформенный поток (OS thread) — это обёртка над потоком ОС со стеком ~512KB-2MB. Virtual thread — это Java-объект в heap с начальным стеком ~килобайты, растущим по необходимости. JVM маппит виртуальные потоки на пул несущих (carrier) потоков — обычных ОС-потоков, число которых равно числу CPU.

Когда виртуальный поток выполняет блокирующую операцию (I/O, sleep, lock), JVM выполняет «unmount» — сохраняет стек виртуального потока в heap, освобождает несущий поток для другого виртуального, а когда операция завершается — «mount» обратно. С точки зрения кода: ты пишешь синхронный блокирующий код, JVM делает его неблокирующим за тебя.

Это кардинальное изменение: вместо реактивного программирования (CompletableFuture, WebFlux) с callback hell можно писать простой последовательный код и получать такую же масштабируемость. Tomcat на virtual threads обрабатывает миллионы одновременных соединений с линейным кодом.

```java
// Сравнение: реактивный vs Virtual Threads подход

// БЫЛО: реактивный подход (сложно читать, дебажить)
CompletableFuture<Response> handleRequest(Request req) {
    return userService.findUser(req.userId())        // async
        .thenCompose(user -> orderService.getOrders(user.id())) // async
        .thenCompose(orders -> paymentService.getBalance(req.userId())) // async
        .thenApply(balance -> new Response(/*...*/));
}

// СТАЛО: Virtual Threads (Java 21) — линейный код, та же масштабируемость
Response handleRequest(Request req) {
    // Все вызовы блокирующие, но JVM unmount'ит поток во время ожидания
    User user = userService.findUser(req.userId());    // блокирует VT, не OS thread
    List<Order> orders = orderService.getOrders(user.id());
    BigDecimal balance = paymentService.getBalance(req.userId());
    return new Response(user, orders, balance);
}

// Создание virtual threads
Thread vt = Thread.ofVirtual().name("request-handler").start(() -> handleRequest(req));

// ExecutorService для virtual threads
try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Request req : requests) {
        exec.submit(() -> handleRequest(req)); // каждый запрос — свой virtual thread
    }
} // try-with-resources вызывает shutdown + awaitTermination
```

❗ Ловушка собеса: `synchronized` блок «приковывает» (pin) виртуальный поток к несущему — поток ОС не освобождается при блокировке, что уничтожает масштабируемость. В Java 21 нужно заменять `synchronized` на `ReentrantLock` в критических путях. В Java 24 это ограничение снято.

**→ Уточняющий вопрос:** Что такое «pinning» виртуального потока и как его обнаружить с помощью JDK Flight Recorder?

---

## 23. В чём преимущества Virtual Threads перед обычными потоками?

Главное преимущество — цена создания и переключения. Платформенный поток: ~1-2 МБ стека ОС, системный вызов для создания (~100 мкс), context switch через ядро ОС. Virtual thread: несколько сотен байт в heap, создаётся в пространстве JVM (~1 мкс), переключение (mount/unmount) — JVM операция без syscall (~микросекунды против ~микросекунд context switch).

Масштабируемость: 10 000 одновременных виртуальных потоков потребляют меньше памяти, чем 1 000 платформенных. Это позволяет иметь поток на каждое соединение/запрос — модель "thread per request" возвращается, но без ограничений. При 100k одновременных HTTP-соединений: платформенные потоки — OOM; virtual threads — работает нормально.

Второе преимущество — простота кода. Реактивное программирование (WebFlux, CompletableFuture chains) требует специального мышления: нельзя использовать ThreadLocal, сложно дебажить stack trace, нельзя блокировать поток. Virtual threads снимают эти ограничения.

```java
// Демонстрация масштабируемости: 100k concurrent "requests"
class VirtualThreadScaleDemo {
    public static void main(String[] args) throws InterruptedException {
        int taskCount = 100_000;
        CountDownLatch latch = new CountDownLatch(taskCount);

        // Virtual threads: займёт ~несколько секунд, ~десятки МБ
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < taskCount; i++) {
                executor.submit(() -> {
                    Thread.sleep(Duration.ofSeconds(1)); // симуляция IO
                    latch.countDown();
                    return null;
                });
            }
        }

        // Платформенные потоки: OutOfMemoryError при попытке создать 100k потоков
        // ExecutorService platformExec = Executors.newCachedThreadPool();
        // -> java.lang.OutOfMemoryError: unable to create native thread
    }
}

// ThreadLocal работает с virtual threads, но осторожно:
// virtual thread-per-task создаётся и уничтожается часто,
// поэтому InheritableThreadLocal может вести себя неожиданно
```

❗ Ловушка собеса: Virtual Threads не улучшают производительность CPU-bound задач. Если задача 100% времени вычисляет (нет IO, нет блокировок), виртуальный поток никогда не unmount'ится и не даёт преимущества. Для CPU-bound нужен `ForkJoinPool` с параллелизмом = числу CPU.

**→ Уточняющий вопрос:** Как ThreadLocal взаимодействует с Virtual Threads и почему рекомендуется использовать ScopedValue вместо ThreadLocal в Loom-приложениях?

---

## 24. Когда стоит использовать Virtual Threads?

Virtual Threads идеальны для IO-bound задач с высоким concurrency: HTTP-сервера, микросервисы с вызовами внешних API, сервисы с частыми запросами к БД. Если типичный запрос проводит 80%+ времени в ожидании IO (сеть, диск, JDBC) — Virtual Threads позволяют масштабироваться до числа одновременных соединений, близкого к бесконечному, с простым блокирующим кодом.

Не стоит использовать Virtual Threads для: CPU-intensive вычислений (нет benefit, лучше ForkJoinPool); задач с `synchronized` на горячих путях (pinning в Java 21); задач, активно использующих ThreadLocal как кэш (частое создание/уничтожение VT делает ThreadLocal-кэши неэффективными); кода, управляющего пулами соединений — пул соединений с 10 соединениями и 1 000 000 VT будет работать, но большинство VT будут ждать соединение.

Практическое руководство: Spring Boot 3.2+ поддерживает virtual threads для Tomcat. Включить одной строкой:

```java
// Spring Boot 3.2+: включить virtual threads для всего
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
    return protocolHandler ->
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}

// Или через properties:
// spring.threads.virtual.enabled=true

// Хороший пример использования: параллельные внешние API вызовы
class ProductAggregator {
    public ProductDetails getDetails(String productId) throws Exception {
        // Три IO-bound вызова параллельно через virtual threads
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            var price = scope.fork(() -> priceService.getPrice(productId));
            var stock = scope.fork(() -> inventoryService.getStock(productId));
            var reviews = scope.fork(() -> reviewService.getReviews(productId));

            scope.join().throwIfFailed();
            return new ProductDetails(price.get(), stock.get(), reviews.get());
        }
    }
}
```

❗ Ловушка собеса: «Virtual Threads решают все проблемы многопоточности» — нет. Race condition, visibility problem и deadlock никуда не исчезают. Virtual threads решают только проблему масштабируемости IO-bound кода. Синхронизация данных по-прежнему необходима.

**→ Уточняющий вопрос:** Как ведёт себя `ThreadLocal` в virtual threads и почему Java 21 вводит `ScopedValue` как альтернативу?

---

## 25. Что такое structured concurrency?

Structured Concurrency (JEP 428/437/453, preview в Java 21-22, финализируется) — это подход к параллельному программированию, основанный на принципе: параллельные задачи, порождённые в блоке кода, должны завершиться до выхода из этого блока. Это аналог структурного программирования (блоки `{}`), но для потоков. `StructuredTaskScope` — основная реализация.

Ключевая проблема, которую решает: при разбивке задачи на параллельные подзадачи через `ExecutorService` ошибки и отмены сложно распространять. Если одна подзадача упала, остальные продолжают работать. Если вызывающий код прерван, подзадачи не получают сигнал. `StructuredTaskScope` решает это: при выходе из блока (даже с исключением) все незавершённые подзадачи отменяются.

Два встроенных шаблона: `ShutdownOnFailure` — отменяет всё при первой ошибке (all-or-nothing для параллельных вызовов); `ShutdownOnSuccess` — отменяет всё при первом успехе (race to first result, например, геолокация по нескольким провайдерам).

```java
// Structured concurrency: параллельный запрос к нескольким сервисам
class OrderService {

    // ShutdownOnFailure: все подзадачи должны успешно завершиться
    public OrderSummary createOrder(OrderRequest request) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            // Запускаем три параллельных вызова
            Subtask<User> userTask = scope.fork(() ->
                userService.getUser(request.userId()));
            Subtask<Inventory> inventoryTask = scope.fork(() ->
                inventoryService.check(request.items()));
            Subtask<PaymentResult> paymentTask = scope.fork(() ->
                paymentService.charge(request.payment()));

            // Ждём завершения ВСЕХ; если одна упала — отменяем остальные
            scope.join().throwIfFailed(e -> new OrderException("Order failed", e));

            // Сюда попадаем только если все три успешны
            return new OrderSummary(userTask.get(), inventoryTask.get(), paymentTask.get());
        }
        // При выходе из try: все незавершённые задачи отменены автоматически
    }

    // ShutdownOnSuccess: возвращаем результат первого успешного
    public GeoLocation locateDevice(String deviceId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<GeoLocation>()) {
            scope.fork(() -> gpsProvider.locate(deviceId));
            scope.fork(() -> cellTowerProvider.locate(deviceId));
            scope.fork(() -> wifiProvider.locate(deviceId));

            scope.join(); // ждём первого успеха — остальные отменятся
            return scope.result(); // результат победителя
        }
    }
}
```

❗ Ловушка собеса: Structured Concurrency — это preview API, менявшийся между версиями JDK (JEP 428 в Java 19, JEP 453 в Java 21). На собесе важно сказать, что API стабилизирован в Java 25. Также: `StructuredTaskScope` требует использования с virtual threads (не работает с platform threads в некоторых JDK-сборках).

**→ Уточняющий вопрос:** Как `StructuredTaskScope` обрабатывает ситуацию, когда родительский поток прерван во время `scope.join()`?

---

## 26. В чём разница между Thread и Runnable?

`Thread` — это класс, представляющий сам поток выполнения. Он хранит стек, состояние, приоритет, группу потоков и содержит логику планирования через JVM/ОС. `Runnable` — функциональный интерфейс с единственным методом `run()`, представляющий задачу (что выполнять), но не поток (кто выполняет). Это разделение задачи и механизма исполнения.

Наследование от `Thread` — антипаттерн по нескольким причинам: Java не поддерживает множественное наследование (нельзя унаследовать ещё один класс), класс-наследник намертво привязан к конкретному механизму выполнения (нельзя запустить в `ExecutorService` без обёртки), нарушается принцип единственной ответственности — класс одновременно содержит бизнес-логику и управляет потоком.

Современный подход: реализовывать `Runnable` (или `Callable`) и передавать в `ExecutorService`. Это даёт гибкость: та же задача может выполняться в пуле, в виртуальном потоке, или напрямую в `new Thread(task).start()` — без изменения кода задачи.

```java
// АНТИПАТТЕРН: наследование от Thread
class RequestHandlerThread extends Thread {
    // Нельзя использовать в ExecutorService без переписывания
    // Нельзя наследовать другой класс
    @Override
    public void run() {
        handleRequest();
    }
}

// ПРАВИЛЬНО: реализация Runnable
class RequestHandler implements Runnable {
    private final Request request;

    @Override
    public void run() {
        handleRequest(request);
    }
}

// Та же задача — разные механизмы выполнения
RequestHandler handler = new RequestHandler(request);

// Вариант 1: прямой поток (редко нужно)
new Thread(handler).start();

// Вариант 2: пул потоков (типично для сервисов)
executorService.submit(handler);

// Вариант 3: virtual thread (Java 21)
Thread.ofVirtual().start(handler);

// Вариант 4: лямбда (если логика простая)
executorService.submit(() -> handleRequest(request));
```

❗ Ловушка собеса: при наследовании от `Thread` и переопределении `run()` — если вызвать `thread.start()`, выполнится переопределённый `run()`. Если ошибочно вызвать `thread.run()` напрямую — выполнение произойдёт в вызывающем потоке, а не в новом. Не будет ни нового потока, ни исключения, только скрытый баг.

**→ Уточняющий вопрос:** Можно ли вызвать `start()` на одном и том же объекте `Thread` дважды, и что произойдёт?

---

## 27. Что такое Callable и Future?

`Callable<V>` — функциональный интерфейс с методом `V call() throws Exception`. В отличие от `Runnable.run()`, метод `call()` может возвращать результат типа `V` и выбрасывать checked-исключения. `Callable` — предпочтительный способ описания задач, возвращающих результат: парсинг файла, HTTP-запрос, запрос к БД.

`Future<V>` — дескриптор результата асинхронной задачи. Методы: `get()` — блокирует вызывающий поток до завершения задачи и возвращает результат (или выбрасывает `ExecutionException`); `get(timeout, unit)` — блокирует с таймаутом; `cancel(mayInterruptIfRunning)` — попытка отмены; `isDone()` — проверка без блокировки; `isCancelled()`.

Важное ограничение `Future`: нельзя зарегистрировать callback (реакцию на завершение), нельзя цеплять задачи (`thenApply`, `thenCompose`), нет удобного способа комбинировать несколько Future. Для этого Java 8+ предоставляет `CompletableFuture`, который реализует `Future` и добавляет реактивный API.

```java
// Callable + Future: параллельная загрузка данных из нескольких источников
class DashboardService {
    private final ExecutorService executor;

    public Dashboard getDashboard(long userId) throws Exception {
        // Запускаем три задачи параллельно
        Future<UserStats> statsFuture = executor.submit(
            () -> analyticsService.getUserStats(userId) // Callable<UserStats>
        );
        Future<List<Order>> ordersFuture = executor.submit(
            () -> orderService.getRecentOrders(userId)  // Callable<List<Order>>
        );
        Future<Notifications> notifFuture = executor.submit(
            () -> notificationService.getUnread(userId)  // Callable<Notifications>
        );

        // Собираем результаты (блокируем по очереди)
        try {
            UserStats stats = statsFuture.get(2, TimeUnit.SECONDS);
            List<Order> orders = ordersFuture.get(2, TimeUnit.SECONDS);
            Notifications notifications = notifFuture.get(2, TimeUnit.SECONDS);
            return new Dashboard(stats, orders, notifications);
        } catch (TimeoutException e) {
            // Отменяем незавершённые задачи
            statsFuture.cancel(true);
            ordersFuture.cancel(true);
            notifFuture.cancel(true);
            throw new ServiceUnavailableException("Dashboard timeout");
        } catch (ExecutionException e) {
            throw new RuntimeException("Task failed", e.getCause()); // разворачиваем
        }
    }
}

// CompletableFuture: более мощная альтернатива
CompletableFuture<Dashboard> getDashboardAsync(long userId) {
    return CompletableFuture.allOf(
        CompletableFuture.supplyAsync(() -> analyticsService.getUserStats(userId), executor),
        CompletableFuture.supplyAsync(() -> orderService.getRecentOrders(userId), executor),
        CompletableFuture.supplyAsync(() -> notificationService.getUnread(userId), executor)
    ).thenApply(v -> buildDashboard(userId));
}
```

❗ Ловушка собеса: `Future.get()` выбрасывает `ExecutionException`, оборачивающий исходное исключение из задачи. Кандидаты часто забывают вызвать `e.getCause()` для получения реального исключения. Кроме того, `cancel(false)` только помечает задачу как отменённую, но не прерывает уже выполняющийся код — нужен `cancel(true)` и обработка `InterruptedException` внутри задачи.

**→ Уточняющий вопрос:** Чем `CompletableFuture` отличается от `Future` и когда предпочтительнее использовать `CompletableFuture.get()` vs `join()`?
