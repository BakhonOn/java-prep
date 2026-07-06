# CompletableFuture и асинхронность

> 📇 Справочник уровня middle. Формат: ответ по сути → пример при необходимости → ❗ на чём ловят на собесе.

**Всего вопросов: 28**

---

## 1. Что такое CompletableFuture и чем он отличается от Future?

`CompletableFuture<T>` — класс из `java.util.concurrent`, введённый в Java 8, реализующий одновременно `Future<T>` и `CompletionStage<T>`. `Future` — простой контракт для асинхронного результата, но крайне ограниченный: единственный способ получить значение — блокирующий `get()`, нет возможности зарегистрировать колбэк, нет комбинирования нескольких future, нет обработки ошибок без try/catch вокруг `get()`.

`CompletableFuture` устраняет все эти ограничения. Во-первых, его можно завершить вручную снаружи — отсюда «Completable». Во-вторых, к нему можно прицепить цепочку колбэков (`thenApply`, `thenCompose`, `thenAccept`), которые сработают автоматически при получении результата. В-третьих, несколько CF можно объединять через `allOf`, `anyOf`, `thenCombine` без единого `get()`.

Пример: со старым `Future` вы запускаете задачу, затем блокируете поток на `get()`. С `CompletableFuture` вы описываете всю цепочку обработки декларативно — поток освобождается, и колбэки выполнятся, когда данные придут:

```java
// Future — только блокировка
Future<String> f = executor.submit(() -> callService());
String result = f.get(); // поток заблокирован

// CompletableFuture — неблокирующая цепочка
CompletableFuture.supplyAsync(() -> callService())
    .thenApply(String::toUpperCase)
    .thenAccept(System.out::println);
// поток не заблокирован, идёт дальше
```

❗ На собесе спрашивают: «А можно ли прервать выполнение задачи через `cancel()`?» — нет, `CompletableFuture.cancel()` не делает interrupt выполняющемуся потоку, он просто переводит CF в отменённое состояние. Это принципиальное отличие от `FutureTask`.

**→ Уточняющий вопрос:** Как реализован `CompletionStage` — какие гарантии он даёт относительно порядка выполнения колбэков?

**↳ Ответ:** `CompletionStage` — интерфейс, определяющий цепочечную модель: каждый этап выполняется после зависимых этапов, но не даёт гарантий порядка между независимыми ветками. Гарантирует: колбэк `thenApply` вызывается после завершения его источника (happens-before); результат источника видим внутри колбэка. Не гарантирует: порядок нескольких колбэков, прицепленных к одному CF — они могут выполниться в любом порядке параллельно.

---

## 2. Какие основные преимущества CompletableFuture перед Future?

Ключевое преимущество — возможность составлять неблокирующие конвейеры обработки данных. С `Future` код вынужден блокироваться на `get()` между каждым шагом асинхронной обработки, что фактически сводит на нет выгоду от асинхронности. `CompletableFuture` позволяет описать весь pipeline декларативно: трансформировать результат (`thenApply`), запустить следующий асинхронный шаг (`thenCompose`), объединить несколько источников (`thenCombine`, `allOf`), обработать ошибки в цепочке (`exceptionally`, `handle`).

Второе важное преимущество — ручное завершение через `complete(value)` и `completeExceptionally(ex)`. Это открывает паттерны, которые с `Future` невозможны: например, завершить CF при первом успешном ответе из нескольких источников, или вернуть fallback-значение по таймауту. Третье — богатый API для работы с несколькими CF: `allOf` ждёт завершения всех, `anyOf` — первого из набора.

```java
// Пример: параллельная загрузка пользователя и его заказов
CompletableFuture<User> userCF = supplyAsync(() -> userService.find(id), ioPool);
CompletableFuture<List<Order>> ordersCF = supplyAsync(() -> orderService.findByUser(id), ioPool);

CompletableFuture<UserDTO> result = userCF.thenCombine(ordersCF,
    (user, orders) -> new UserDTO(user, orders));
```

❗ Ловушка: «назови хотя бы три метода, которых нет в `Future`» — большинство кандидатов называют `thenApply` и останавливаются. Ожидаемый ответ: `thenCompose`, `handle`, `exceptionally`, `allOf`, `anyOf`, `complete`, `completeExceptionally`, `orTimeout`.

**→ Уточняющий вопрос:** Какой интерфейс реализует `CompletableFuture` помимо `Future`, и что он определяет?

**↳ Ответ:** `CompletionStage<T>` — интерфейс, определяющий полный API для составления асинхронных конвейеров: `thenApply`, `thenCompose`, `thenCombine`, `allOf`, `anyOf`, `exceptionally`, `handle`, `whenComplete` и их `*Async` варианты. Именно `CompletionStage` добавляет всё, чего нет в `Future`. При использовании в API методов стоит возвращать `CompletionStage<T>`, а не `CompletableFuture<T>`, чтобы скрыть методы `complete()`/`cancel()` от вызывающего.

---

## 3. Как создать CompletableFuture, который уже завершён с результатом?

`CompletableFuture.completedFuture(value)` возвращает уже выполненный CF с заданным значением. Никакой асинхронности не происходит — это синхронный объект, у которого `isDone()` сразу `true`. Любые колбэки, прицепленные к нему, выполнятся немедленно в вызывающем потоке (если не используются `*Async` варианты).

Это крайне полезно в двух случаях: при тестировании (мокировать возвращаемое значение репозитория) и в методах с early-return, когда результат уже известен без асинхронных вызовов. В Java 9 появился `CompletableFuture.failedFuture(ex)` — симметричный метод для создания CF, завершённого ошибкой.

```java
// В тесте — заглушка сервиса
when(userService.findAsync(1L))
    .thenReturn(CompletableFuture.completedFuture(new User(1L, "Alice")));

// В реальном коде — кэш-хит
public CompletableFuture<User> findUser(Long id) {
    User cached = cache.get(id);
    if (cached != null) {
        return CompletableFuture.completedFuture(cached); // сразу, без IO
    }
    return supplyAsync(() -> db.findById(id), ioPool);
}
```

❗ Ловушка: если прицепить `thenApplyAsync` к `completedFuture`, колбэк уйдёт в пул потоков — даже несмотря на то, что CF уже завершён. Разница между `thenApply` (выполнится в текущем потоке) и `thenApplyAsync` (уйдёт в пул) важна именно в этом случае.

**→ Уточняющий вопрос:** Когда именно выполнится колбэк `thenApply`, прицепленный к уже завершённому `completedFuture`?

**↳ Ответ:** Немедленно и синхронно — в том потоке, который вызывает `thenApply`. Раз CF уже завершён, нет нужды ничего ждать; вызов `thenApply` сам исполняет функцию прямо здесь. Поэтому `completedFuture(...).thenApply(fn)` — это фактически синхронный вызов `fn`. Исключение: `thenApplyAsync` всегда уходит в пул, даже если CF уже завершён.

---

## 4. В чём разница между thenApply() и thenCompose()?

`thenApply(Function<T, R>)` — синхронная трансформация результата: принимает значение T, возвращает R. Аналог `map` у Stream или Optional. Результирующий CF содержит R. `thenCompose(Function<T, CompletableFuture<R>>)` — для функций, которые сами возвращают `CompletableFuture<R>`. Он «разворачивает» вложенный CF, выдавая `CF<R>` вместо `CF<CF<R>>`. Аналог `flatMap`.

Критически важно понять: если внутри `thenApply` вернуть `CompletableFuture`, вы получите `CompletableFuture<CompletableFuture<R>>` — вложенный CF, который никогда не развернётся автоматически. Именно для таких случаев существует `thenCompose`.

```java
// thenApply — трансформация значения (синхронно)
CompletableFuture<String> nameCF = supplyAsync(() -> 42)
    .thenApply(id -> "User#" + id); // CF<String>

// НЕПРАВИЛЬНО — получим CF<CF<User>>
CompletableFuture<CompletableFuture<User>> wrong = idCF
    .thenApply(id -> userService.findAsync(id)); // findAsync возвращает CF<User>

// ПРАВИЛЬНО — thenCompose разворачивает вложенность
CompletableFuture<User> correct = idCF
    .thenCompose(id -> userService.findAsync(id)); // CF<User>

// Реальный пример: получить токен, потом загрузить данные
CompletableFuture<Data> pipeline = authService.getTokenAsync()
    .thenCompose(token -> dataService.fetchAsync(token));
```

❗ Классическая ловушка на собесе: «что вернёт `thenApply(id -> repo.findAsync(id))`?» — ответ `CF<CF<User>>`. Интервьюер проверяет, понимает ли кандидат разницу между map и flatMap. Без этого понимания вы будете писать код, который компилируется, но не работает как ожидается.

**→ Уточняющий вопрос:** Какой метод у Stream является аналогом `thenCompose`, и почему неправильно использовать `thenApply` для асинхронных вызовов внутри цепочки?

**↳ Ответ:** Аналог — `flatMap`: оба «разворачивают» вложенность. `Stream.map(f)` → `Stream<Stream<T>>`; `Stream.flatMap(f)` → `Stream<T>`. Аналогично `thenApply(asyncFn)` → `CF<CF<T>>`; `thenCompose(asyncFn)` → `CF<T>`. Если использовать `thenApply` для асинхронного вызова, вы получите вложенный CF, который никогда не развернётся автоматически — результат внешнего CF будет не значением, а другим CF. Получить значение можно будет только через двойной `get()/join()`.

---

## 5. Что делают методы thenAccept() и thenRun()?

`thenAccept(Consumer<T>)` потребляет результат предыдущего этапа, но ничего не возвращает — возвращаемый тип `CF<Void>`. Используется, когда результат нужен для побочного эффекта (запись в базу, отправка в очередь, логирование с доступом к данным). `thenRun(Runnable)` — выполняет действие без доступа к результату вообще, просто «запустить что-то после». Тоже возвращает `CF<Void>`.

Оба метода завершают цепочку обработки значения, они не трансформируют результат — это терминальные операции. Важно не путать их с `handle` и `whenComplete`, которые тоже не меняют тип, но могут влиять на результат или ошибку.

```java
CompletableFuture.supplyAsync(() -> fetchOrder(orderId))
    .thenApply(Order::calculateTotal)          // Double — трансформация
    .thenAccept(total -> invoice.setAmount(total)) // побочный эффект с данными
    .thenRun(() -> log.info("Pipeline done"));     // действие без данных

// Разница: thenAccept видит результат, thenRun — нет
supplyAsync(() -> "hello")
    .thenAccept(s -> System.out.println("Got: " + s)) // "Got: hello"
    .thenRun(() -> System.out.println("Done"));        // "Done"
```

❗ Ловушка: оба возвращают `CF<Void>`. Попытка вызвать `.thenApply` после `thenAccept` и ожидать получить строку провалится — там `Void`. Кандидаты часто путают и пытаются «продолжить» цепочку трансформаций после `thenAccept`.

**→ Уточняющий вопрос:** Зачем использовать `thenRun` вместо просто добавления кода в конец `thenAccept`? В каком сценарии `thenRun` принципиально важен?

**↳ Ответ:** `thenRun` нужен, когда цепочки разветвляются: одна ветка обрабатывает результат через `thenAccept`, другая запускает независимое действие без доступа к результату. Добавить код в `thenAccept` невозможно, если `thenAccept` уже написан другим разработчиком или находится в другом компоненте. Другой сценарий: после `CF<Void>` (например, `thenAccept`) нужно запустить ещё одно действие — `thenRun` это позволяет, а `thenApply` нельзя (`Void` нет значения).

---

## 6. Как обрабатывать исключения в цепочке CompletableFuture?

Когда в одном из этапов цепочки возникает исключение, оно «прокидывается» дальше, минуя все обычные этапы (`thenApply`, `thenAccept` и т.д.) до первого обработчика ошибок. Это поведение аналогично тому, как исключение пропускает все `map` в стриме. Основные инструменты: `exceptionally`, `handle` и `whenComplete`.

`exceptionally(Function<Throwable, T>)` — срабатывает только при ошибке, позволяет вернуть fallback-значение того же типа. `handle(BiFunction<T, Throwable, R>)` — срабатывает всегда, получает и результат, и ошибку (одно из них null), возвращает новый результат. `whenComplete(BiConsumer<T, Throwable>)` — тоже всегда, но не меняет результат, только побочный эффект (логирование).

```java
CompletableFuture.supplyAsync(() -> externalApi.call())
    .thenApply(Response::getBody)
    .exceptionally(ex -> {
        log.error("API call failed", ex);
        return "fallback-response"; // заменяем ошибку значением
    })
    .thenAccept(body -> process(body));

// handle — полный контроль над обеими ветками
supplyAsync(() -> riskyOperation())
    .handle((result, ex) -> {
        if (ex != null) {
            metrics.increment("errors");
            return defaultValue;
        }
        return result;
    });
```

❗ Ловушка: `exceptionally` получает обёрнутое исключение типа `CompletionException`, а не исходное. Чтобы добраться до реальной причины, нужно вызвать `ex.getCause()`. Кандидаты часто пишут `instanceof IOException` и удивляются, что не срабатывает — а там `CompletionException`.

**→ Уточняющий вопрос:** Если в цепочке есть `thenApply` → `thenApply` → `exceptionally`, и первый `thenApply` бросает исключение — выполнится ли второй `thenApply`?

**↳ Ответ:** Нет. Исключение от первого `thenApply` переводит промежуточный CF в failed-состояние. Второй `thenApply` получает этот failed CF и сам переходит в failed-состояние, не выполняя свою функцию — только передаёт ошибку дальше. `exceptionally` перехватит исключение. Правило: в нормальной цепочке исключение «перешагивает» через все обычные этапы до первого обработчика.

---

## 7. В чём разница между handle(), exceptionally() и whenComplete()?

Все три позволяют реагировать на завершение CF, но с разными контрактами. `exceptionally(Function<Throwable, T>)` — только для ошибок, трансформирует исключение в значение; если ошибки нет — просто пропускает результат дальше без изменений. Идеален для задания fallback-значений.

`handle(BiFunction<T, Throwable, R>)` — срабатывает всегда (и при успехе, и при ошибке), получает оба параметра (один из них null), возвращает новый результат. Позволяет трансформировать как результат, так и ошибку. Самый гибкий из трёх. `whenComplete(BiConsumer<T, Throwable>)` — тоже срабатывает всегда, но НЕ меняет результат: если CF завершился успешно, результат пройдёт дальше без изменений; если с ошибкой — ошибка тоже пройдёт дальше. Только для побочных эффектов.

```java
// exceptionally — только замена при ошибке
cf.exceptionally(ex -> "default");

// handle — полная трансформация в любом случае
cf.handle((res, ex) -> ex == null ? res.toUpperCase() : "ERROR: " + ex.getMessage());

// whenComplete — логирование без изменения результата
cf.whenComplete((res, ex) -> {
    if (ex != null) log.error("Failed", ex);
    else metrics.record(res);
}); // результат/ошибка идут дальше как есть
```

❗ Ловушка: `whenComplete` не «поглощает» ошибку — если cf завершился ошибкой, после `whenComplete` она продолжает распространяться. Кандидаты путают его с `handle` и думают, что могут «обработать» ошибку через `whenComplete`. Обработать означает поглотить — это делает только `handle` или `exceptionally`.

**→ Уточняющий вопрос:** Может ли `handle()` сам выбросить исключение? Что произойдёт с цепочкой в этом случае?

**↳ Ответ:** Да, если функция внутри `handle()` бросит исключение, результирующий CF перейдёт в failed-состояние с этим новым исключением. Предыдущее исключение при этом теряется — заменяется новым. Это ловушка: обработчик ошибок сам стал источником ошибки, а исходная причина утеряна. Правило: внутри `handle()` оборачивать рискованный код в try/catch и логировать оба исключения.

---

## 8. Как комбинировать результаты нескольких CompletableFuture?

Для двух CF — `thenCombine(other, BiFunction)`: запускает оба независимо, когда оба завершены — вызывает функцию с обоими результатами. Для массива CF — `allOf(cf1, cf2, ...)`: завершается как `CF<Void>` когда все готовы. Для первого из набора — `anyOf(...)`: завершается с результатом первого завершившегося. Для последовательной зависимой цепочки — `thenCompose`.

Важный нюанс с `allOf`: он возвращает `CF<Void>`, результаты нужно извлекать из исходных CF после его завершения. Это частая точка ошибок.

```java
// thenCombine — два CF, один результат
CompletableFuture<String> priceStr = supplyAsync(() -> "100");
CompletableFuture<Integer> discount = supplyAsync(() -> 10);
CompletableFuture<String> summary = priceStr.thenCombine(discount,
    (price, disc) -> "Price: " + price + ", discount: " + disc + "%");

// allOf + извлечение результатов
List<CompletableFuture<String>> futures = services.stream()
    .map(s -> supplyAsync(s::call, ioPool))
    .collect(toList());

CompletableFuture<List<String>> all = CompletableFuture
    .allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream()
        .map(CompletableFuture::join) // безопасно — все уже завершены
        .collect(toList()));
```

❗ Ловушка: после `allOf(...).join()` нельзя просто вызвать `.get()` на самом `allOf` и получить результаты — там `Void`. Нужно обращаться к исходным CF. Очень частая ошибка на собесе.

**→ Уточняющий вопрос:** Что произойдёт с `allOf`, если один из переданных CF завершится с исключением?

**↳ Ответ:** `allOf` немедленно завершится с исключением — не дожидаясь остальных CF. Остальные CF при этом продолжат выполняться в фоне — они не отменяются, их ресурсы не освобождаются. Вызов `allOf.join()` бросит `CompletionException` с исходной ошибкой. Правило: если нужна частичная обработка (взять что успело) — оборачивать каждый CF в `exceptionally` с дефолтным значением до передачи в `allOf`.

---

## 9. Что делает метод allOf() и когда его использовать?

`CompletableFuture.allOf(CompletableFuture<?>... cfs)` создаёт новый CF, который завершается (без значения, `CF<Void>`) только когда завершились ВСЕ переданные CF. Если хотя бы один завершится с исключением — `allOf` тоже завершится с исключением, не дожидаясь остальных (но остальные продолжат выполняться в фоне — они не отменяются).

Типичный use case — параллельный вызов нескольких микросервисов, когда итоговый ответ требует данных от всех. Например, страница карточки товара, где нужны: данные товара, наличие, цены и отзывы — всё из разных сервисов.

```java
// Параллельный запрос к 4 микросервисам
CompletableFuture<Product> productCF = supplyAsync(() -> productService.get(id), ioPool);
CompletableFuture<Stock> stockCF = supplyAsync(() -> stockService.get(id), ioPool);
CompletableFuture<Price> priceCF = supplyAsync(() -> priceService.get(id), ioPool);
CompletableFuture<List<Review>> reviewsCF = supplyAsync(() -> reviewService.get(id), ioPool);

CompletableFuture.allOf(productCF, stockCF, priceCF, reviewsCF)
    .thenApply(v -> new ProductPage(
        productCF.join(),
        stockCF.join(),
        priceCF.join(),
        reviewsCF.join()
    ))
    .thenAccept(page -> response.render(page));
```

❗ Ловушка: `allOf` возвращает `CF<Void>` — нет метода `.getResults()`. Результаты нужно собирать вручную из исходных CF после завершения `allOf`. Кандидаты часто рисуют красивую схему с `allOf` и потом не могут объяснить, как достать результаты.

**→ Уточняющий вопрос:** Если один из 10 CF в `allOf` упадёт с исключением, продолжат ли выполняться остальные 9?

**↳ Ответ:** Да, все 9 продолжат выполняться — `allOf` не отменяет переданные CF. Сам `allOf` CF переходит в failed-состояние, но задачи в пуле потоков продолжат работу. Это означает, что ресурсы (потоки, соединения) остаются занятыми. Если нужна реальная отмена — хранить ссылки на все CF и вызывать `cancel()` на каждом при ошибке в `exceptionally` обработчике `allOf`.

---

## 10. Что делает метод anyOf() и в каких случаях он полезен?

`CompletableFuture.anyOf(CompletableFuture<?>... cfs)` возвращает `CF<Object>`, который завершается результатом ПЕРВОГО завершившегося CF из набора. Остальные CF продолжают выполняться — `anyOf` их не отменяет. Это важно понимать: «гонка» здесь означает только «кто первый ответит», а не «кто победит — остальные останавливаются».

Полезен для нескольких сценариев: запрос к нескольким репликам базы данных (берём ответ быстрейшей), запрос к нескольким CDN или зеркалам (первый ответивший), hedged requests — отправить запрос в два датацентра и взять быстрейший ответ.

```java
// Hedge-запрос: отправить в два датацентра, взять первый ответ
CompletableFuture<String> dc1 = supplyAsync(() -> callDataCenter("dc1"), ioPool);
CompletableFuture<String> dc2 = supplyAsync(() -> callDataCenter("dc2"), ioPool);

String result = (String) CompletableFuture.anyOf(dc1, dc2).join();

// Запрос к нескольким репликам с типизацией
List<CompletableFuture<Data>> replicas = replicaUrls.stream()
    .map(url -> supplyAsync(() -> fetchFrom(url), ioPool))
    .collect(toList());

CompletableFuture<Object> fastest = CompletableFuture.anyOf(
    replicas.toArray(new CompletableFuture[0]));
Data data = (Data) fastest.join(); // приходится кастовать
```

❗ Ловушка: `anyOf` возвращает `CF<Object>`, не `CF<T>` — обязателен явный каст. Если завершившийся первым CF упал с ошибкой — `anyOf` тоже завершится ошибкой, даже если остальные ещё бегут и могли бы вернуть результат. Это неочевидное поведение.

**→ Уточняющий вопрос:** Как реализовать `anyOf`, который игнорирует ошибки и ждёт первого УСПЕШНОГО результата?

**↳ Ответ:** Обернуть каждый CF так, чтобы ошибки превращались в `Optional.empty()`, успехи — в `Optional.of(value)`. Затем `anyOf` на обёрнутых CF вернёт первый завершившийся — если это `empty`, значит ошибка, смотрим дальше. Либо: создать результирующий CF вручную (`new CompletableFuture<>()`) и для каждого CF в списке добавить `whenComplete`, который при успехе вызывает `result.complete(value)`. Первый успешный CF «победит» через `complete()` (idempotent — остальные проигнорируются).

---

## 11. В чём разница между thenApply() и thenApplyAsync()?

`thenApply(fn)` выполняет функцию в потоке, который завершил предыдущий этап цепочки. Если предыдущий этап уже завершён к моменту вызова `thenApply` — функция выполнится в текущем потоке (том, который вызывает `thenApply`). Это синхронное продолжение без смены потока.

`thenApplyAsync(fn)` всегда отправляет функцию в пул потоков (по умолчанию `ForkJoinPool.commonPool()`, или в переданный `Executor`). Это даёт контроль над тем, в каком потоке выполняется конкретный шаг конвейера. Важно для: разделения IO-потоков и вычислительных потоков, предотвращения блокировки netty/reactive worker-потоков, явного управления concurrency.

```java
// thenApply — в потоке ForkJoin worker, завершившего supplyAsync
supplyAsync(() -> fetchData())          // ForkJoin thread-1
    .thenApply(d -> transform(d))       // ForkJoin thread-1 (тот же!)
    .thenApply(d -> format(d));         // может быть любой поток

// thenApplyAsync — гарантированно в пуле
supplyAsync(() -> fetchData(), ioPool)  // ioPool thread
    .thenApplyAsync(d -> transform(d))  // commonPool thread (переключение!)
    .thenApplyAsync(d -> format(d), computePool); // computePool thread
```

❗ Ловушка: поведение `thenApply` непредсказуемо — в зависимости от timing'а, функция может выполниться в вызывающем потоке ИЛИ в пуле. Если это важно (например, нельзя выполнять в Netty event loop) — всегда используйте `thenApplyAsync` с явным executor.

**→ Уточняющий вопрос:** Если предыдущий этап уже завершён до вызова `thenApply` — в каком потоке выполнится колбэк?

**↳ Ответ:** В потоке, который вызывает `thenApply` — то есть в текущем потоке вызывающего кода. Если CF ещё не завершён в момент регистрации колбэка, он выполнится в потоке, завершившем CF. Это делает поведение `thenApply` непредсказуемым в многопоточном коде: нельзя знать заранее, завершён ли CF до `thenApply` или после. Правило: если нужна предсказуемость — всегда `thenApplyAsync` с явным executor.

---

## 12. Какой пул потоков используется по умолчанию для async методов?

По умолчанию все `*Async` методы без явного `Executor` используют `ForkJoinPool.commonPool()`. Это единый пул на всё JVM-приложение, его размер по умолчанию равен `Runtime.getRuntime().availableProcessors() - 1` (минус 1, чтобы оставить главный поток). На машине с 8 ядрами — 7 потоков.

`commonPool` — пул для CPU-интенсивных, неблокирующих задач. Он использует work-stealing: свободные потоки «воруют» задачи из очередей занятых потоков, что эффективно для коротких вычислительных задач. Проблема возникает, когда в `commonPool` попадают блокирующие IO-операции — потоки зависают в ожидании, пул деградирует, и страдают все другие задачи в приложении, которые тоже используют `commonPool`.

```java
// Плохо: HTTP-запросы в commonPool — блокируют всем
CompletableFuture.supplyAsync(() -> httpClient.get(url)); // commonPool!

// Хорошо: свой пул для IO
ExecutorService ioPool = Executors.newFixedThreadPool(
    100, new ThreadFactoryBuilder().setNameFormat("io-%d").build());

CompletableFuture.supplyAsync(() -> httpClient.get(url), ioPool);

// Проверить размер commonPool
System.out.println(ForkJoinPool.commonPool().getParallelism()); // 7 на 8 ядрах
```

❗ Ловушка: можно увеличить `commonPool` через системное свойство `-Djava.util.concurrent.ForkJoinPool.common.parallelism=16`, но это не решает проблему блокирующих задач — лишь маскирует. Правильное решение — отдельный пул для IO.

**→ Уточняющий вопрос:** Что произойдёт с приложением, если все потоки `commonPool` заблокируются на IO-операциях?

**↳ Ответ:** Новые задачи, которые должны выполняться в `commonPool`, встанут в очередь и будут ждать освобождения потоков. На сервере это выглядит как резкий рост latency или полная остановка: входящие HTTP-запросы обрабатываются, но параллельные CF-задачи зависают. ForkJoinPool имеет механизм ManagedBlocker для регистрации блокировок и компенсации потоков, но `CompletableFuture` им не пользуется. Итог: starvation других задач приложения.

---

## 13. Как указать свой Executor для CompletableFuture?

Любой `*Async` метод имеет перегрузку с `Executor` в качестве последнего аргумента: `supplyAsync(supplier, executor)`, `thenApplyAsync(fn, executor)`, `thenComposeAsync(fn, executor)` и т.д. Передавая свой executor, вы полностью контролируете, какой пул потоков выполняет данный этап.

Для IO-операций (HTTP-запросы, запросы к БД, чтение файлов) — использовать cached или fixed пул с большим количеством потоков (`Executors.newFixedThreadPool(N)` или виртуальные потоки в Java 21). Для CPU-задач — размер пула равен числу ядер. Разные части одной цепочки могут использовать разные пулы.

```java
// Разные пулы для разных типов задач
ExecutorService ioPool = Executors.newFixedThreadPool(50);
ExecutorService cpuPool = ForkJoinPool.commonPool(); // для CPU-работы OK

CompletableFuture.supplyAsync(() -> db.query(sql), ioPool)        // IO в ioPool
    .thenApplyAsync(rows -> expensiveTransform(rows), cpuPool)    // CPU в cpuPool
    .thenAcceptAsync(result -> kafka.send(result), ioPool);       // IO снова в ioPool

// Java 21: виртуальные потоки для IO
ExecutorService virtualPool = Executors.newVirtualThreadPerTaskExecutor();
CompletableFuture.supplyAsync(() -> blockingHttpCall(), virtualPool);
```

❗ Ловушка: если в середине цепочки передать executor только в один `thenApplyAsync`, следующие `thenApply` (без Async) выполнятся в этом же пуле. Нужно явно контролировать каждый шаг, требующий конкретного пула. Кандидаты часто думают, что один `supplyAsync(task, myPool)` «прикрепляет» весь pipeline к этому пулу — это не так.

**→ Уточняющий вопрос:** Зачем в Java 21 для IO в `CompletableFuture` рекомендуют `newVirtualThreadPerTaskExecutor()`?

**↳ Ответ:** Виртуальные потоки монтируются на platform-потоки и автоматически «размонтируются» при блокировке — platform-поток освобождается для других виртуальных потоков. Это значит: блокирующий IO в виртуальном потоке не занимает реальный OS-поток. Можно создать тысячи виртуальных потоков без накладных расходов. `newVirtualThreadPerTaskExecutor()` создаёт по одному виртуальному потоку на задачу — идеально для IO-bound `CompletableFuture`, больше не нужно управлять размером пула вручную.

---

## 14. Что такое блокирующий код и как его отличить от неблокирующего?

Блокирующий код — это любой код, который заставляет поток ждать в режиме простоя: поток занят, но не делает полезной работы. Главные маркеры: `Thread.sleep()`, `Object.wait()`, `Future.get()`, `CompletableFuture.join()` в середине pipeline, синхронные JDBC-запросы, синхронные HTTP-клиенты (`RestTemplate`, `HttpURLConnection`), блокирующее чтение файлов (`FileInputStream`).

Неблокирующий код не занимает поток во время ожидания. Поток либо не ждёт вообще (передаёт задачу и уходит), либо использует async/reactive механизмы. Примеры: колбэки `CompletableFuture`, реактивные клиенты (WebClient), NIO, async JDBC (R2DBC).

```java
// БЛОКИРУЮЩИЙ — поток занят на время запроса
String result = restTemplate.getForObject(url, String.class); // поток ждёт

// НЕБЛОКИРУЮЩИЙ — поток свободен, колбэк вызовется потом
webClient.get().uri(url)
    .retrieve()
    .bodyToMono(String.class)
    .subscribe(result -> process(result)); // поток сразу свободен

// Как отличить: поток застревает "внутри" вызова или сразу возвращается?
// Если метод возвращает Future/CF/Mono/Flux — скорее всего неблокирующий
// Если возвращает конкретный результат — блокирующий
```

❗ Ловушка: `CompletableFuture.join()` — блокирующий вызов, несмотря на то, что CF "асинхронный". Вызов `join()` в Netty event loop или в reactive pipeline приведёт к дедлоку. Кандидаты часто считают, что раз используют CF — значит код неблокирующий.

**→ Уточняющий вопрос:** Почему вызов `join()` внутри колбэка `thenApply` в Reactor pipeline может привести к дедлоку?

**↳ Ответ:** Reactor использует небольшой пул event loop потоков (например, Netty). Если внутри `flatMap` вызвать `cf.join()` — event loop поток заблокируется, ожидая завершения CF. Но сам CF может тоже ждать event loop потока для выполнения своих колбэков — круговое ожидание, дедлок. Правило: в Reactor никогда не блокировать event loop поток; использовать `Mono.fromCompletionStage(cf)` для интеграции CF в реактивный pipeline.

---

## 15. Почему важно избегать блокирующих операций в CompletableFuture?

`CompletableFuture` по умолчанию использует `ForkJoinPool.commonPool()` с ограниченным числом потоков (N-1 по числу ядер). Если блокирующая операция займёт поток на время IO (даже на 50мс), то при нескольких параллельных запросах пул может полностью исчерпаться. Новые задачи встанут в очередь и будут ждать — что при HTTP-сервере с нагрузкой выглядит как деградация latency или полная остановка обработки запросов.

ForkJoinPool использует work-stealing и оптимизирован для коротких, CPU-bound задач. Блокирующие операции в нём — антипаттерн: поток висит, stealing не помогает, пул деградирует. Представьте кухню с 7 поварами, каждый из которых ушёл ждать доставки продуктов — готовить некому.

```java
// АНТИПАТТЕРН: блокировка commonPool
CompletableFuture.supplyAsync(() -> {
    Thread.sleep(200); // блокирует поток commonPool на 200мс!
    return callExternalApi(); // ещё блокировка!
});

// ПРАВИЛЬНО: свой пул для IO-операций
ExecutorService ioPool = Executors.newFixedThreadPool(100);
CompletableFuture.supplyAsync(() -> {
    return callExternalApi(); // блокирует поток ioPool — нормально
}, ioPool);

// Или виртуальные потоки (Java 21) — блокировка не проблема
ExecutorService virtual = Executors.newVirtualThreadPerTaskExecutor();
CompletableFuture.supplyAsync(() -> callExternalApi(), virtual);
```

❗ Ловушка: многие кандидаты говорят «нужно избегать блокировок» — но не могут объяснить, ПОЧЕМУ именно для commonPool это критично. Правильный ответ: ограниченный размер пула + блокировка = голодание (starvation) других задач.

**→ Уточняющий вопрос:** Как виртуальные потоки Java 21 меняют правила работы с блокирующим IO в CompletableFuture?

**↳ Ответ:** С виртуальными потоками правило «не блокируй commonPool» становится менее критичным: блокирующий IO в виртуальном потоке освобождает platform-поток для других задач. Можно писать синхронный блокирующий код и передавать его в `supplyAsync(task, virtualExecutor)` без страха голодания пула. Главное ограничение остаётся: не использовать виртуальные потоки для CPU-intensive задач — там лучше ForkJoinPool с platform-потоками.

---

## 16. Как правильно выполнить несколько параллельных запросов к микросервисам?

Правильный паттерн: запустить каждый запрос через `supplyAsync` с явным IO-пулом (не commonPool), собрать все CF в `allOf`, дождаться завершения, затем синхронно извлечь результаты через `join()` (что безопасно — все CF уже завершены). Каждый сервис работает параллельно, суммарное время ≈ времени самого медленного запроса, а не сумме всех.

Важно также обрабатывать ошибки: если один микросервис недоступен, `allOf` упадёт. Нужно решить: либо обернуть каждый CF в `exceptionally` с дефолтным значением, либо отловить ошибку на уровне `allOf` и решать, что делать с частичными данными.

```java
ExecutorService ioPool = Executors.newFixedThreadPool(50);

// Запускаем параллельно
CompletableFuture<UserDto> userCF =
    supplyAsync(() -> userService.getUser(userId), ioPool);
CompletableFuture<List<Order>> ordersCF =
    supplyAsync(() -> orderService.getOrders(userId), ioPool);
CompletableFuture<Loyalty> loyaltyCF =
    supplyAsync(() -> loyaltyService.getPoints(userId), ioPool)
        .exceptionally(ex -> Loyalty.empty()); // дефолт при недоступности

// Ждём всех
CompletableFuture<DashboardDto> dashboard = CompletableFuture
    .allOf(userCF, ordersCF, loyaltyCF)
    .thenApply(v -> new DashboardDto(
        userCF.join(),     // безопасно — CF уже завершён
        ordersCF.join(),
        loyaltyCF.join()
    ));

// С таймаутом на всю операцию (Java 9+)
DashboardDto result = dashboard
    .orTimeout(3, TimeUnit.SECONDS)
    .join();
```

❗ Ловушка: использовать `commonPool` для HTTP-запросов. Под нагрузкой несколько десятков параллельных запросов заблокируют весь commonPool, и приложение перестанет обрабатывать другие задачи. Всегда отдельный пул для IO.

**→ Уточняющий вопрос:** Как изменится поведение, если один из трёх сервисов отвечает за 5 секунд, а таймаут выставлен в 3 секунды — что произойдёт с остальными двумя запросами?

**↳ Ответ:** По истечении 3 секунд `orTimeout` переводит итоговый CF в failed-состояние с `TimeoutException`. Два быстрых запроса при этом могут уже завершиться — их CF содержат результаты, но они не используются. Медленный третий запрос продолжит выполняться в IO-пуле и завершится через 5 секунд, но результат некуда деть. Ресурсы (соединение, поток) третьего сервиса остаются занятыми ещё 2 секунды — это плата за то, что `orTimeout` не отменяет задачи.

---

## 17. Что делает метод supplyAsync() и когда его использовать?

`CompletableFuture.supplyAsync(Supplier<T>)` запускает переданный `Supplier` асинхронно в пуле потоков и возвращает `CompletableFuture<T>`, который завершится результатом этого `Supplier`. Это стандартный способ начать асинхронное вычисление, дающее значение. Если нужно выполнить задачу без возвращаемого значения — используют `runAsync(Runnable)`.

`supplyAsync` — это точка входа в асинхронный pipeline. Хорошая практика: всегда передавать явный `Executor` вторым аргументом, особенно если задача выполняет IO. Метод оборачивает исключения Supplier в `CompletionException` и завершает CF с ошибкой — они не теряются.

```java
// Базовое использование
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    return externalService.fetchData(); // запустится в commonPool
});

// Правильно — явный IO-пул
CompletableFuture<User> userCF = CompletableFuture.supplyAsync(
    () -> userRepository.findById(id),
    ioExecutor
);

// Цепочка начинается с supplyAsync
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> fetchRaw(), ioPool)
    .thenApply(raw -> parse(raw))
    .thenApply(parsed -> format(parsed));

// Отличие от runAsync — нет возвращаемого значения
CompletableFuture<Void> fireAndForget = CompletableFuture.runAsync(
    () -> auditLog.record(event), auditPool
);
```

❗ Ловушка: если `Supplier` бросит исключение — CF завершится с ошибкой, но без явной обработки через `exceptionally`/`handle` оно «потеряется» — не упадёт на вызывающего, пока не вызовут `get()`/`join()`. Fire-and-forget паттерн без обработки ошибок — распространённая утечка ошибок.

**→ Уточняющий вопрос:** Чем `supplyAsync` отличается от `runAsync`, и когда использовать второй?

**↳ Ответ:** `supplyAsync(Supplier<T>)` возвращает `CF<T>` с результатом; `runAsync(Runnable)` возвращает `CF<Void>` без результата. Используйте `runAsync` для fire-and-forget задач: запись в audit log, отправка события в Kafka, обновление кэша — когда не нужен ответ и не нужно дожидаться завершения (или достаточно `CF<Void>` для синхронизации через `allOf`). Правило: если задача возвращает значение — `supplyAsync`; если побочный эффект без результата — `runAsync`.

---

## 18. Как отменить выполнение CompletableFuture?

`cf.cancel(mayInterruptIfRunning)` переводит CF в состояние «отменён» (`isCancelled()` возвращает `true`, `get()` и `join()` бросят `CancellationException`). Однако в отличие от `FutureTask`, параметр `mayInterruptIfRunning` у `CompletableFuture` игнорируется — фактически поток, выполняющий задачу, interrupt НЕ получает. Задача продолжает выполняться в пуле, просто её результат будет проигнорирован.

Это фундаментальное ограничение `CompletableFuture`. Настоящая отмена с прерыванием требует отдельного механизма: хранить ссылку на выполняющий поток или `Future` от `ExecutorService.submit()` и вызывать `future.cancel(true)` напрямую.

```java
// Отмена только меняет состояние CF — задача продолжает бежать в пуле
CompletableFuture<String> cf = supplyAsync(() -> {
    Thread.sleep(10_000); // этот sleep НЕ прервётся
    return "result";
}, executor);

cf.cancel(true); // mayInterruptIfRunning игнорируется!
System.out.println(cf.isCancelled()); // true
cf.join(); // бросит CancellationException

// Реальная отмена с прерыванием потока
ExecutorService pool = Executors.newSingleThreadExecutor();
Future<?> rawFuture = pool.submit(() -> longRunningTask()); // обычный Future
CompletableFuture<Void> cf2 = CompletableFuture.runAsync(() -> longRunningTask(), pool);

rawFuture.cancel(true); // ВОТ это прервёт поток
```

❗ Ловушка: 90% кандидатов не знают, что `cancel(true)` не прерывает поток в `CompletableFuture`. Это одно из самых частых «сюрпризов» API. Задача продолжает потреблять ресурсы, просто результат некуда деть.

**→ Уточняющий вопрос:** Как реализовать реальную отмену с прерыванием потока, используя `CompletableFuture`?

**↳ Ответ:** Хранить обычный `Future<?>` от `ExecutorService.submit(task)` рядом с CF. При необходимости отменить — вызвать `rawFuture.cancel(true)`, что пошлёт interrupt в выполняющий поток. Сам CF дополнительно завершить через `cf.cancel(false)`. Либо в самой задаче периодически проверять `Thread.currentThread().isInterrupted()` и выходить при прерывании. Правило: CF API не предоставляет прерывания — используйте `ExecutorService.submit()` для получения `Future` с реальным управлением.

---

## 19. Что произойдёт, если в цепочке CompletableFuture возникнет исключение?

Исключение переводит CF в «failed» состояние и распространяется вниз по цепочке, пропуская все обычные этапы (`thenApply`, `thenAccept`, `thenRun`, `thenCombine`). Пропущенные этапы «передают» ошибку дальше без выполнения. Первый встреченный обработчик ошибок (`exceptionally`, `handle`) перехватывает её.

Если цепочка завершается без обработчика — исключение «поглощается» внутри CF. Оно всплывёт при вызове `get()` как `ExecutionException` (checked), при `join()` — как `CompletionException` (unchecked). Если никто не вызовет `get()`/`join()` — исключение пропадёт тихо (silent exception — серьёзный антипаттерн).

```java
CompletableFuture.supplyAsync(() -> {
        throw new RuntimeException("Сервис недоступен");
    })
    .thenApply(s -> s.toUpperCase())  // ПРОПУСКАЕТСЯ
    .thenApply(s -> s + "!")           // ПРОПУСКАЕТСЯ
    .exceptionally(ex -> {
        // ex — CompletionException, причина в ex.getCause()
        log.error("Ошибка: {}", ex.getCause().getMessage());
        return "FALLBACK";
    })
    .thenAccept(System.out::println); // "FALLBACK"

// Тихое поглощение ошибки — антипаттерн!
CompletableFuture.supplyAsync(() -> riskyOperation()); // ошибка потеряна
// Всегда добавляйте:
    .exceptionally(ex -> { log.error("...", ex); return null; });
```

❗ Ловушка: «silent exceptions» — исключение возникает, задача не выполняется, но никто об этом не знает. Очень сложно дебажить в production. Правило: каждая «финальная» CF-цепочка должна иметь обработчик ошибок.

**→ Уточняющий вопрос:** Как исключение в одном из CF передаётся через `allOf`? Если один CF упал — что происходит с остальными и с самим `allOf`?

**↳ Ответ:** `allOf` CF немедленно переходит в failed-состояние с исключением упавшего CF. Остальные CF не затрагиваются — продолжают выполняться в фоне (нет отмены). Результирующий `allOf.join()` бросит `CompletionException`. Следующие этапы после `allOf` (типа `thenApply`) не выполнятся — перейдут в failed. Шаблон устойчивой агрегации: оборачивать каждый CF в `.exceptionally(e -> defaultValue)` до `allOf`, тогда `allOf` всегда успешен.

---

## 20. Можно ли повторно использовать один CompletableFuture в нескольких цепочках?

Да, от одного CF можно строить несколько независимых цепочек — это называется «fan-out» паттерн. Каждый вызов `thenApply`, `thenCompose` и т.д. создаёт новый `CompletableFuture`, не изменяя исходный. Когда исходный CF завершится, все прикреплённые к нему цепочки получат результат и начнут выполняться независимо друг от друга.

Это мощный паттерн: один источник данных — несколько потребителей. Например, ответ от API нужно и записать в кэш, и обработать для ответа клиенту, и отправить в аналитику.

```java
CompletableFuture<User> userCF = supplyAsync(() -> userService.fetch(id), ioPool);

// Три независимые цепочки от одного CF
CompletableFuture<Void> cacheChain = userCF
    .thenAcceptAsync(user -> cache.put(id, user), cachePool);

CompletableFuture<UserDto> responseChain = userCF
    .thenApply(User::toDto);

CompletableFuture<Void> analyticsChain = userCF
    .thenAcceptAsync(user -> analytics.record(user), analyticsPool);

// userCF не изменён — он общий корень для всех трёх цепочек
// Все три запустятся когда userCF завершится

CompletableFuture.allOf(cacheChain, analyticsChain)
    .thenCompose(v -> responseChain)
    .thenAccept(dto -> response.send(dto));
```

❗ Ловушка: если к одному CF одновременно прикреплено много цепочек, порядок их запуска при завершении не гарантирован. Если между ними есть зависимости — их нужно явно выражать через `thenCompose`/`allOf`, а не полагаться на порядок.

**→ Уточняющий вопрос:** Что произойдёт, если один из fan-out колбэков бросит исключение — повлияет ли это на другие цепочки?

**↳ Ответ:** Нет — каждая цепочка независима. Исключение в `cacheChain` (например, Redis недоступен) переводит в failed-состояние только `cacheChain`. `responseChain` и `analyticsChain` получат результат из исходного `userCF` и выполнятся нормально. Исходный `userCF` не меняется при ошибках в его fan-out ветках. Правило: fan-out изолирует ошибки между ветками — это главное преимущество паттерна перед последовательными вызовами.

---

## 21. Как реализовать timeout для CompletableFuture?

В Java 9+ есть два встроенных метода: `orTimeout(long, TimeUnit)` завершает CF исключением `TimeoutException` по истечении времени; `completeOnTimeout(T value, long, TimeUnit)` завершает CF с указанным дефолтным значением по истечении времени. Оба неблокирующие и используют `ScheduledExecutorService` внутри JDK.

До Java 9 таймаут реализовывался вручную: через `ScheduledExecutorService.schedule()` с отдельным CF, который завершает основной через `complete(value)` или `completeExceptionally(ex)` по истечении времени.

```java
// Java 9+ — элегантно
CompletableFuture<String> result = supplyAsync(() -> slowApi(), ioPool)
    .orTimeout(2, TimeUnit.SECONDS); // TimeoutException при превышении

// С дефолтным значением вместо ошибки
CompletableFuture<String> withFallback = supplyAsync(() -> slowApi(), ioPool)
    .completeOnTimeout("default-response", 2, TimeUnit.SECONDS);

// Java 8 — ручная реализация
ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
CompletableFuture<String> cf = supplyAsync(() -> slowApi(), ioPool);

scheduler.schedule(
    () -> cf.completeExceptionally(new TimeoutException("Timeout after 2s")),
    2, TimeUnit.SECONDS
);

// Полный пример: параллельные вызовы с общим таймаутом
CompletableFuture<List<String>> allData = CompletableFuture
    .allOf(cf1, cf2, cf3)
    .orTimeout(5, TimeUnit.SECONDS)
    .thenApply(v -> List.of(cf1.join(), cf2.join(), cf3.join()));
```

❗ Ловушка: `orTimeout` не отменяет выполняющуюся задачу — та продолжает бежать в фоне. Только CF переходит в состояние ошибки, ресурсы не освобождаются. Для реальной отмены нужен дополнительный механизм.

**→ Уточняющий вопрос:** Чем `orTimeout` отличается от `completeOnTimeout`, и в каком бизнес-сценарии лучше использовать второй?

**↳ Ответ:** `orTimeout` завершает CF с `TimeoutException` (ошибка), `completeOnTimeout` завершает с дефолтным значением (успех). Используйте `completeOnTimeout` там, где пустой/дефолтный ответ приемлем для пользователя: рекомендации (показать популярное), персонализация (показать обычную версию), нотификации (отложить). Используйте `orTimeout` там, где операция без ответа — ошибка: авторизация, расчёт итоговой цены. Правило: если есть разумный fallback — `completeOnTimeout`; если нет — `orTimeout`.

---

## 22. Что делает метод orTimeout() в Java 9+?

`cf.orTimeout(long timeout, TimeUnit unit)` возвращает тот же CF (это не создаёт новый!) и регистрирует отложенное завершение с `TimeoutException`. Если CF не завершится за указанное время — он принудительно завершается исключением `TimeoutException`, что является подклассом `CancellationException`. Если CF завершится раньше — таймер отменяется автоматически.

Внутри JDK метод использует `Delayer` — внутренний `ScheduledThreadPoolExecutor` с одним daemon-потоком, общий на все `orTimeout` вызовы. Это важно: `orTimeout` не создаёт новый `ScheduledExecutorService` на каждый вызов — используется общий, лёгкий механизм.

```java
// orTimeout — стандартный таймаут с исключением
CompletableFuture<Response> apicall = supplyAsync(() -> api.call(req), ioPool)
    .orTimeout(3, TimeUnit.SECONDS)
    .exceptionally(ex -> {
        if (ex.getCause() instanceof TimeoutException) {
            return Response.timeout(); // обработка таймаута
        }
        return Response.error(ex);
    });

// Цепочка: таймаут + retry
supplyAsync(() -> api.call(req), ioPool)
    .orTimeout(1, TimeUnit.SECONDS)
    .exceptionally(ex -> retryWithNewInstance(req)) // при любой ошибке
    .orTimeout(3, TimeUnit.SECONDS); // финальный таймаут на retry
```

❗ Ловушка: `orTimeout` возвращает `this` — тот же CF, а не новый. Кандидаты иногда думают, что это «обёртка». Кроме того, `TimeoutException` приходит как `CompletionException` в `exceptionally`, нужно проверять `ex.getCause() instanceof TimeoutException`.

**→ Уточняющий вопрос:** Если CF завершится успешно за 1 секунду, а `orTimeout` был выставлен на 5 секунд — что произойдёт с запланированным отложенным завершением?

**↳ Ответ:** Запланированная задача в `Delayer` будет автоматически отменена — JDK отслеживает завершение CF и снимает отложенное завершение. Ресурсы освобождаются корректно, нет утечки задач в `ScheduledExecutorService`. Это безопасное поведение: `orTimeout` — это «на всякий случай», не дополнительная нагрузка для быстрых CF.

---

## 23. В чём разница между thenCombine() и thenCompose()?

`thenCompose(Function<T, CF<R>>)` — последовательная цепочка с зависимостью: результат первого CF нужен для запуска второго. Это «и тогда» (and then). Два CF выполняются последовательно. Аналог `flatMap`.

`thenCombine(CF<U>, BiFunction<T, U, R>)` — параллельная комбинация: два CF выполняются независимо, и когда оба завершились — вызывается функция с обоими результатами. Это «оба одновременно, потом объединить». Принципиально разные паттерны использования.

```java
// thenCompose — последовательно, второй зависит от первого
// Получить userId, затем запросить профиль по userId
CompletableFuture<Profile> profile = supplyAsync(() -> authService.getUserId(token))
    .thenCompose(userId -> supplyAsync(() -> profileService.get(userId), ioPool));
// Общее время: время(auth) + время(profile) — последовательно

// thenCombine — параллельно, независимые источники
// Получить профиль и настройки независимо, объединить
CompletableFuture<Profile> profileCF = supplyAsync(() -> profileService.get(id), ioPool);
CompletableFuture<Settings> settingsCF = supplyAsync(() -> settingsService.get(id), ioPool);

CompletableFuture<UserView> view = profileCF.thenCombine(settingsCF,
    (profile, settings) -> new UserView(profile, settings));
// Общее время: max(время(profile), время(settings)) — параллельно
```

❗ Ловушка: использовать `thenCompose` там, где CF независимы (теряем параллелизм), или `thenCombine` там, где второй CF зависит от результата первого (логическая ошибка). Вопрос «когда что использовать» — обязательный на собесе.

**→ Уточняющий вопрос:** Как с помощью `thenCombine` объединить не два, а три независимых CF?

**↳ Ответ:** Цепочкой двух `thenCombine`: сначала объединить первые два CF в промежуточный объект, затем объединить результат с третьим CF. Либо использовать `allOf(cf1, cf2, cf3).thenApply(v -> ...)` — более читаемо при трёх и более CF. `thenCombine` хорошо работает для двух, но становится громоздким для трёх и более. Правило: для 2 CF — `thenCombine`; для 3+ — `allOf` с последующим извлечением через `join()`.

---

## 24. Как тестировать код с CompletableFuture?

Основная стратегия: для юнит-тестов заменять реальные асинхронные вызовы на `CompletableFuture.completedFuture(value)` (успех) и `CompletableFuture.failedFuture(ex)` (ошибка). Это делает тесты детерминированными и не требует ждать реальной асинхронности. В тесте для получения результата вызывать `.join()` или `.get()` — это блокирующие вызовы, которые в тестах приемлемы.

Для интеграционных тестов с реальной асинхронностью — использовать `Awaitility` или просто `join()` с таймаутом. Важно тестировать не только «happy path», но и ветку ошибок: проверять, что `exceptionally`/`handle` отрабатывают корректно.

```java
// Мок возвращает уже завершённый CF
@Mock
UserService userService;

@Test
void testFetchUser_success() {
    User expected = new User(1L, "Alice");
    when(userService.fetchAsync(1L))
        .thenReturn(CompletableFuture.completedFuture(expected));

    UserDto dto = userController.getUser(1L).join(); // блокируем в тесте
    assertEquals("Alice", dto.getName());
}

@Test
void testFetchUser_serviceDown() {
    when(userService.fetchAsync(1L))
        .thenReturn(CompletableFuture.failedFuture(new ServiceException("down")));

    UserDto dto = userController.getUser(1L).join();
    assertEquals("FALLBACK", dto.getName()); // проверяем fallback
}

// Awaitility для реальной асинхронности
Awaitility.await()
    .atMost(5, SECONDS)
    .until(() -> resultHolder.get() != null);
```

❗ Ловушка: `CompletableFuture.failedFuture(ex)` появился только в Java 9. В Java 8 нужно создавать вручную: `cf = new CompletableFuture<>(); cf.completeExceptionally(ex); return cf;`. Кандидаты на Java 8 проектах могут не знать этот нюанс.

**→ Уточняющий вопрос:** Как в JUnit 5 правильно проверить, что метод возвращает CF, завершённый с конкретным исключением?

**↳ Ответ:** Использовать `assertThrows` с `cf.join()`: `CompletionException ex = assertThrows(CompletionException.class, () -> cf.join())`. Затем проверить `ex.getCause()` на нужный тип. Либо: `ExecutionException ex = assertThrows(ExecutionException.class, () -> cf.get())` и проверить `ex.getCause()`. Правило: реальное исключение всегда в `getCause()`, а не само по себе — не забывать об этом при проверке `instanceof`.

---

## 25. В каких случаях лучше использовать CompletableFuture, а в каких реактивное программирование?

`CompletableFuture` подходит для сценариев с одним асинхронным результатом: вызов одного микросервиса, параллельный запрос к нескольким сервисам с объединением результатов, асинхронное выполнение IO-задачи с продолжением. CF хорошо интегрируется с императивным кодом, прост для понимания команды, не требует reactive-стека. Стандартная библиотека Java, без дополнительных зависимостей.

Реактивное программирование (Project Reactor / RxJava) необходимо, когда: источник данных — поток (stream) событий, а не один результат (WebSocket, Kafka, SSE); нужен backpressure (потребитель не успевает за производителем); требуется богатый набор операторов (window, buffer, debounce, merge); строится полностью неблокирующий стек (WebFlux + R2DBC + реактивный Kafka).

```java
// CompletableFuture — один результат, простая композиция
CompletableFuture<DashboardDto> dashboard = CompletableFuture
    .allOf(userCF, ordersCF)
    .thenApply(v -> new DashboardDto(userCF.join(), ordersCF.join()));

// Reactor — поток событий с backpressure
Flux<Order> orderStream = kafkaReceiver.receive()
    .map(record -> deserialize(record.value()))
    .filter(order -> order.getAmount() > 1000)
    .flatMap(order -> enrichWithUserData(order)) // async enrichment
    .buffer(Duration.ofSeconds(1))               // батчинг
    .flatMap(batch -> batchSave(batch));          // сохранение батчами
```

❗ Ловушка: смешивать оба подхода неосторожно. Блокирующий `cf.join()` внутри Reactor pipeline убьёт event loop. Если использовать CF внутри Flux — нужно оборачивать через `Mono.fromCompletionStage(cf)`, а не блокировать. Это частая ошибка при миграции кода.

**→ Уточняющий вопрос:** Как правильно интегрировать `CompletableFuture` внутрь Reactor цепочки (Mono/Flux)?

**↳ Ответ:** Использовать `Mono.fromCompletionStage(cf)` или `Mono.fromFuture(cf)` — оба оборачивают CF в `Mono` без блокировки. Внутри `flatMap` возвращать `Mono.fromCompletionStage(() -> serviceCall())` — обратите внимание на `Supplier`, чтобы CF создавался при подписке, а не заранее. Никогда не вызывать `cf.join()` внутри реактивного pipeline — это блокировка event loop. Правило: CF и Reactor совместимы через `Mono.from*`, но смешивать блокирующий и реактивный код нельзя.

---

## 26. Что делает метод join() и чем он отличается от get()?

Оба метода блокируют текущий поток до завершения CF и возвращают результат. Ключевое различие — в обработке исключений. `get()` унаследован от `Future` и бросает checked-исключения: `ExecutionException` (если задача завершилась с ошибкой) и `InterruptedException` (если поток был прерван). Это требует try/catch или объявления throws.

`join()` бросает только unchecked `CompletionException` (обёртка вокруг реального исключения), которая в свою очередь является `RuntimeException`. Это делает `join()` предпочтительным в лямбдах, Stream API и местах, где checked exceptions неудобны. Поведение при прерывании потока: `get()` сразу бросает `InterruptedException`, `join()` не реагирует на interrupt — продолжает ждать.

```java
// get() — checked exceptions, нужен try/catch
try {
    String result = cf.get(5, TimeUnit.SECONDS);
} catch (ExecutionException e) {
    Throwable cause = e.getCause(); // реальное исключение здесь
} catch (InterruptedException | TimeoutException e) {
    Thread.currentThread().interrupt();
}

// join() — unchecked, удобен в стримах и лямбдах
List<String> results = futures.stream()
    .map(CompletableFuture::join) // не нужен try/catch
    .collect(toList());

// join() НЕ реагирует на interrupt — это может быть проблемой
// Если поток прерван, join() продолжит ждать!
// get() в этом случае бросит InterruptedException
```

❗ Ловушка: `join()` игнорирует прерывание потока — это важно знать для корректного shutdown приложения. Если нужна реакция на interrupt (например, в `ExecutorService.shutdown()`), используйте `get()`. Другая ловушка: `join()` в нескольких потоках на одном CF — дедлок, если CF никогда не завершится.

**→ Уточняющий вопрос:** Почему `join()` без таймаута опасен в production-коде, и как правильно ограничить время ожидания?

**↳ Ответ:** `join()` без таймаута блокирует поток навсегда, если CF никогда не завершится (зависший сетевой вызов без таймаута, дедлок). В production это приведёт к исчерпанию потоков сервера. Правильно: использовать `cf.orTimeout(5, SECONDS).join()` (Java 9+) или `cf.get(5, SECONDS)` (Java 8). Альтернатива: настраивать таймауты на уровне HTTP-клиентов/соединений, чтобы CF завершались сами по себе, а `join()` всегда находил завершённый CF.

---

## 27. Можно ли вручную завершить CompletableFuture результатом?

Да, именно это делает «Completable» в названии — CF можно завершить снаружи, из любого потока. Методы: `complete(T value)` устанавливает результат и возвращает `true` (если CF ещё не был завершён); `completeExceptionally(Throwable ex)` завершает с ошибкой; `cancel(boolean)` переводит в состояние отменён. Все методы idempotent: повторный вызов на уже завершённом CF возвращает `false` и не меняет результат.

Паттерны применения: реализация таймаутов (завершить с ошибкой по истечении времени), Promise-паттерн (создать CF, передать его слушателям, завершить когда придёт событие), адаптация callback-based API к CF.

```java
// Promise-паттерн: адаптация callback API
public CompletableFuture<Response> makeRequest(Request req) {
    CompletableFuture<Response> promise = new CompletableFuture<>();

    asyncHttpClient.execute(req, new Callback() {
        @Override
        public void onSuccess(Response resp) {
            promise.complete(resp); // завершаем CF из колбэка
        }
        @Override
        public void onError(Exception ex) {
            promise.completeExceptionally(ex);
        }
    });

    return promise; // возвращаем незавершённый CF
}

// complete() возвращает false если CF уже завершён
CompletableFuture<String> cf = new CompletableFuture<>();
cf.complete("first");
boolean secondResult = cf.complete("second"); // false — проигнорировано
cf.join(); // "first"
```

❗ Ловушка: `complete()` возвращает `boolean` — его стоит проверять в гонках. Если два потока одновременно вызовут `complete()`, победит первый, второй получит `false`. В race condition без проверки возвращаемого значения можно потерять информацию о том, «победил» ли ваш `complete()`.

**→ Уточняющий вопрос:** Как реализовать «первый успешный из N» — несколько параллельных запросов, где CF завершается результатом первого успешного, а остальные игнорируются?

**↳ Ответ:** Создать результирующий `CompletableFuture<T> result = new CompletableFuture<>()`. Для каждого из N запросов добавить `cf.whenComplete((v, ex) -> { if (ex == null) result.complete(v); })`. Первый успешный CF вызовет `result.complete()`, остальные вызовы `complete()` будут проигнорированы (idempotent). Если все упали — нужен счётчик ошибок и `result.completeExceptionally(lastEx)` когда счётчик достиг N. Это надёжнее `anyOf`, который завершается ошибкой при первом failed CF.

---

## 28. Как реализовать retry логику с помощью CompletableFuture?

Базовый паттерн — рекурсивная функция, которая при ошибке запускает следующую попытку через `thenCompose` или `exceptionally`. Ключевые аспекты: ограничить число попыток, добавить задержку между попытками (exponential backoff), не блокировать поток между попытками.

Для неблокирующей задержки используют `CompletableFuture.delayedExecutor(delay, unit)` (Java 9+) или `ScheduledExecutorService`. В production окружениях правильнее использовать готовые библиотеки: Resilience4j (retry, circuit breaker, rate limiter), Spring Retry — они обрабатывают edge cases, которые легко пропустить при самостоятельной реализации.

```java
// Рекурсивный retry с экспоненциальным backoff (Java 9+)
public static <T> CompletableFuture<T> retry(
        Supplier<CompletableFuture<T>> taskSupplier,
        int maxAttempts,
        long delayMs) {

    return taskSupplier.get().exceptionallyCompose(ex -> {
        if (maxAttempts <= 1) {
            return CompletableFuture.failedFuture(ex); // попытки кончились
        }
        long nextDelay = delayMs * 2; // exponential backoff
        // неблокирующая задержка через delayedExecutor
        Executor delayed = CompletableFuture.delayedExecutor(
            delayMs, TimeUnit.MILLISECONDS);
        return CompletableFuture
            .supplyAsync(() -> null, delayed) // "подождать"
            .thenCompose(v -> retry(taskSupplier, maxAttempts - 1, nextDelay));
    });
}

// Использование
CompletableFuture<Response> result = retry(
    () -> supplyAsync(() -> externalApi.call(), ioPool),
    3,     // максимум 3 попытки
    100    // первая задержка 100мс, потом 200мс, потом 400мс
);

// Resilience4j в Spring Boot — production вариант
@Retry(name = "externalApi", fallbackMethod = "fallback")
public CompletableFuture<Response> callApi(Request req) {
    return supplyAsync(() -> externalApi.call(req), ioPool);
}
```

❗ Ловушка: наивный retry с `Thread.sleep()` внутри `exceptionally` блокирует поток пула между попытками. При большом числе параллельных операций это быстро исчерпает пул. `delayedExecutor` (Java 9+) решает это без блокировки. Также: retry без jitter (случайного разброса задержки) при большом числе клиентов создаёт «thundering herd» — все ретраят синхронно.

**→ Уточняющий вопрос:** Что такое «thundering herd» в контексте retry, и как jitter помогает его избежать?

**↳ Ответ:** Thundering herd — когда N клиентов получают ошибку одновременно (например, сервис упал), затем все N синхронно ретраят через одинаковый интервал, создавая мощный spike нагрузки на восстанавливающийся сервис — он снова падает. Jitter (случайный разброс задержки) распределяет retry во времени: вместо пика через 1 сек — равномерная нагрузка в течение 0–2 сек. Правило: всегда добавлять jitter к backoff при retry, особенно в высоконагруженных системах с большим числом клиентов.
