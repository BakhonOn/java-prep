# REST / HTTP

> 📇 Справочник уровня middle. Формат: ответ по сути → почему именно так → пример из реального API → ❗ ловушка на собесе → уточняющий вопрос.

**Всего вопросов: 17**

---

## 1. Что такое REST?

REST (Representational State Transfer) — архитектурный стиль, описанный Роем Филдингом в диссертации 2000 года, а не протокол и не стандарт. Он задаёт набор ограничений для распределённых систем поверх HTTP: ресурсы идентифицируются URI, взаимодействие с ними осуществляется стандартными HTTP-методами, состояние клиента не хранится на сервере между запросами. Ключевое слово — «стиль»: можно сделать API поверх HTTP и не быть RESTful.

Шесть ограничений REST по Филдингу: client-server, stateless, cacheable, uniform interface, layered system, code on demand (опциональное). Uniform interface — главное из них: единообразный интерфейс через URI + HTTP-методы + стандартные заголовки. Именно поэтому REST-API интуитивно понятен без документации — все действия над ресурсами предсказуемы.

В интернет-магазине REST-API выглядит так: `GET /api/v1/products` — каталог, `POST /api/v1/orders` — создать заказ, `GET /api/v1/orders/123` — детали заказа. Любой разработчик без документации угадает, что делает `DELETE /api/v1/cart/items/5`.

```http
GET /api/v1/products/42 HTTP/1.1
Host: shop.example.com
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 42,
  "name": "MacBook Pro 14",
  "price": 199999,
  "stock": 15
}
```

❗ **Ловушка собеса:** интервьюер спросит «REST — это протокол?». Чёткий ответ: нет, это архитектурный стиль. Ещё ловят на «какие ограничения REST вы знаете» — многие называют только stateless и забывают про cacheable и uniform interface.

**→ Уточняющий вопрос:** В чём разница между REST и RESTful? Все ли API поверх HTTP являются REST?

**↳ Ответ:** REST — это набор архитектурных ограничений, RESTful — прилагательное, означающее «соответствующий этим ограничениям». Не любой API поверх HTTP является REST: если нарушено хотя бы одно ограничение (например, stateful сессии на сервере, или глаголы в URL) — это не RESTful. Большинство реальных «REST API» — это HTTP API уровня 2 по Ричардсону, которые используют HTTP-методы и статус-коды, но не реализуют HATEOAS.

---

## 2. Что означает Stateless в контексте REST?

Stateless означает, что сервер не хранит никакого состояния сессии клиента между запросами. Каждый HTTP-запрос должен быть самодостаточным: он содержит всю информацию, необходимую для его обработки — идентификацию, авторизацию, контекст. Сервер обрабатывает запрос и «забывает» о клиенте.

Это ограничение даёт горизонтальное масштабирование: если состояние клиента хранится только у него (в токене, в куках), то любой из десяти инстансов сервиса может обработать любой запрос, потому что инстансу не нужна «память» о предыдущих запросах. Балансировщик нагрузки может раздавать запросы round-robin без sticky sessions.

В платёжной системе каждый запрос к `/api/v1/payments` содержит `Authorization: Bearer <jwt>`. JWT несёт в себе userId, роли, время жизни — сервер не хранит сессию, а декодирует токен на лету. При горизонтальном масштабировании на 5 pods в Kubernetes каждый pod может обработать платёж, не обращаясь к общему хранилищу сессий.

```http
POST /api/v1/payments HTTP/1.1
Host: payments.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJ1c2VySWQiOjEyMywiZW1haWwiOiJ1c2VyQGV4YW1wbGUuY29tIn0...
Content-Type: application/json

{
  "orderId": 456,
  "amount": 4999,
  "currency": "RUB"
}
```

❗ **Ловушка собеса:** «Если REST stateless, то как реализовать корзину пользователя?». Ответ: состояние хранится на клиенте (localStorage, куки) или в БД, идентифицируется через токен. Сервер не держит корзину в памяти — он читает её из БД по userId из JWT.

**→ Уточняющий вопрос:** Как stateless сочетается с JWT-токенами и как отзывать токены до истечения срока жизни, если сервер не хранит состояние?

**↳ Ответ:** Stateless не означает «сервер ничего не хранит вообще» — он не хранит сессионное состояние клиента в памяти. JWT-токены полностью совместимы: клиент несёт состояние (userId, роли) в самом токене. Отзыв до истечения срока — вынужденный компромисс: хранить blacklist отозванных токенов в Redis (с TTL = время жизни токена). Альтернатива — короткоживущие access tokens (5-15 минут) + refresh token в httpOnly куке, тогда отзыв достигается удалением refresh token.

---

## 3. Какие основные HTTP методы используются в REST?

Основные методы: GET (получить ресурс), POST (создать ресурс или выполнить действие), PUT (полная замена ресурса), PATCH (частичное обновление), DELETE (удалить ресурс). Дополнительно используются HEAD (как GET, но без тела — проверка заголовков, существования ресурса) и OPTIONS (получить список поддерживаемых методов, используется в CORS preflight).

Каждый метод имеет семантику, которую клиенты и промежуточные узлы (прокси, кэши) понимают и используют. GET-ответы кэшируются, POST — нет. Именно поэтому важно не использовать GET для изменения данных: поисковый бот или CDN может вызвать его произвольное число раз.

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    @GetMapping("/{id}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable Long id) {
        return ResponseEntity.ok(productService.findById(id));
    }

    @PostMapping
    public ResponseEntity<ProductDto> createProduct(@RequestBody @Valid CreateProductRequest request) {
        ProductDto created = productService.create(request);
        URI location = URI.create("/api/v1/products/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<ProductDto> replaceProduct(@PathVariable Long id,
                                                      @RequestBody @Valid ProductDto dto) {
        return ResponseEntity.ok(productService.replace(id, dto));
    }

    @PatchMapping("/{id}")
    public ResponseEntity<ProductDto> updateProduct(@PathVariable Long id,
                                                     @RequestBody Map<String, Object> updates) {
        return ResponseEntity.ok(productService.partialUpdate(id, updates));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

❗ **Ловушка собеса:** «Можно ли передавать тело в GET-запросе?». Технически HTTP не запрещает, но это нарушает семантику и большинство клиентов/серверов это игнорируют. Для сложных фильтров используйте query parameters или POST с семантикой поиска.

**→ Уточняющий вопрос:** Как бы вы реализовали сложный поиск с десятком параметров фильтрации — через GET с query params или через POST? Какие у каждого подхода плюсы и минусы?

**↳ Ответ:** GET с query params — REST-корректен, кэшируется, ссылка шарится, но URL может превысить 2000 символов и не поддерживает сложные вложенные фильтры. POST с JSON-телом — поддерживает любую сложность, нет ограничений на размер, но нарушает семантику (поиск — не создание ресурса) и не кэшируется. Компромисс: для типового поиска используйте GET; если параметры действительно сложные — `POST /products/search` с явным именованием action-эндпоинта.

---

## 4. В чём разница между PUT и PATCH?

PUT — полная замена ресурса: клиент отправляет полное представление ресурса, сервер заменяет его целиком. Если какое-то поле не указано, оно становится null или значением по умолчанию. PUT идемпотентен: два одинаковых PUT-запроса дают одинаковый результат. PATCH — частичное обновление: клиент отправляет только изменяемые поля, остальные остаются без изменений.

PUT применяется, когда клиент управляет полным состоянием ресурса и знает все поля. PATCH применяется для точечных изменений — например, обновить статус заказа или email пользователя, не трогая остальные данные. Для больших объектов PATCH экономит трафик и снижает риск случайной перезаписи полей, которых клиент не знает.

Пример: пользователь в юзер-сервисе имеет поля id, email, name, phone, address, createdAt. Нужно обновить только телефон:

```http
// PUT — нужно отправить ВЕСЬ объект
PUT /api/v1/users/123 HTTP/1.1
Content-Type: application/json

{
  "id": 123,
  "email": "user@example.com",
  "name": "Иван Иванов",
  "phone": "+79991234567",
  "address": "Москва, ул. Ленина, 1"
}

// PATCH — только изменяемое поле
PATCH /api/v1/users/123 HTTP/1.1
Content-Type: application/json

{
  "phone": "+79991234567"
}
```

```java
// Spring MVC: PATCH через Map или специальный DTO
@PatchMapping("/users/{id}")
public ResponseEntity<UserDto> patchUser(@PathVariable Long id,
                                          @RequestBody Map<String, Object> updates) {
    // Применяем только переданные поля через рефлексию или явную проверку
    UserDto updated = userService.partialUpdate(id, updates);
    return ResponseEntity.ok(updated);
}
```

❗ **Ловушка собеса:** «Является ли PATCH всегда идемпотентным?». Нет — по спецификации PATCH не обязан быть идемпотентным. Например, `PATCH /counter {"increment": 1}` при повторном вызове даст разный результат. PUT всегда идемпотентен.

**→ Уточняющий вопрос:** Как реализовать PATCH безопасно, если несколько клиентов одновременно обновляют разные поля одного ресурса? Что такое optimistic locking и как это связано с PATCH?

**↳ Ответ:** Optimistic locking — клиент получает ресурс вместе с версией (`ETag` или полем `version`), при PATCH включает её в запрос (`If-Match: "v42"`), сервер отказывает с 409 Conflict если версия устарела. Это предотвращает «потерянное обновление»: два клиента читают версию 1, один обновляет email (версия становится 2), второй пытается обновить phone с version=1 — получает 409 и должен перечитать актуальный ресурс. Без версионирования второй клиент молча перетрёт изменение первого.

---

## 5. Что такое идемпотентность (idempotency)?

Идемпотентность — свойство операции, при котором многократное выполнение с одними и теми же параметрами приводит к тому же результату, что и однократное выполнение. Ключевое уточнение: речь идёт о конечном состоянии системы, а не об ответе сервера — ответы могут отличаться (200, 404), но состояние данных остаётся одинаковым.

Идемпотентность критически важна для надёжности распределённых систем. При сетевых сбоях клиент не знает, дошёл ли запрос до сервера и был ли обработан. Если операция идемпотентна — клиент может безопасно повторить запрос без риска создать дубликаты или испортить данные. Это основа retry-логики в микросервисах и API-клиентах.

В платёжной системе это критично: если клиент отправил POST /payments и не получил ответ (таймаут), он не знает, прошёл ли платёж. Для POST (не идемпотентный) используют `Idempotency-Key` header — клиент генерирует UUID, сервер дедуплицирует по этому ключу:

```http
POST /api/v1/payments HTTP/1.1
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "orderId": 789,
  "amount": 9999,
  "currency": "RUB"
}

// При повторном запросе с тем же Idempotency-Key сервер вернёт
// тот же результат, не создавая новый платёж
HTTP/1.1 200 OK
{
  "paymentId": "pay_abc123",
  "status": "COMPLETED"
}
```

❗ **Ловушка собеса:** «Идемпотентность = безопасность (safe)?». Нет. Safe означает, что метод не изменяет состояние системы (только GET, HEAD, OPTIONS). Идемпотентный метод может изменять состояние, но одинаково при повторах. DELETE идемпотентен, но не safe — он меняет состояние при первом вызове.

**→ Уточняющий вопрос:** Как реализовать idempotency-key на сервере? Где хранить использованные ключи и как долго?

**↳ Ответ:** Хранить в Redis с TTL, равным периоду, в течение которого клиент может повторять запрос (обычно 24–48 часов). Ключ — хеш (idempotency-key + userId), значение — сериализованный ответ или статус «в обработке». Если запрос ещё не завершён — возвращать 202 Accepted или 409 Conflict, чтобы клиент подождал. БД подходит, но медленнее; Redis предпочтительней из-за встроенного TTL и скорости.

---

## 6. Какие HTTP методы идемпотентны?

Идемпотентные методы: GET, HEAD, OPTIONS, PUT, DELETE. Неидемпотентный: POST. PATCH — не обязан быть идемпотентным по спецификации, хотя на практике большинство PATCH-операций идемпотентны (если заменяем конкретное поле конкретным значением).

Это разделение закреплено в RFC 7231 и имеет практическое значение: HTTP-клиенты (браузеры, retry-логика в RestTemplate/WebClient) автоматически повторяют идемпотентные запросы при сетевых ошибках, но не повторяют POST. CDN и прокси кэшируют только безопасные методы (GET, HEAD).

```java
// Spring WebClient с retry только для идемпотентных методов
WebClient webClient = WebClient.builder()
    .baseUrl("http://product-service")
    .build();

// GET — безопасно ретраить
Mono<ProductDto> product = webClient.get()
    .uri("/api/v1/products/{id}", 42)
    .retrieve()
    .bodyToMono(ProductDto.class)
    .retryWhen(Retry.backoff(3, Duration.ofMillis(500))
        .filter(ex -> ex instanceof WebClientRequestException));

// POST — ретраить только с Idempotency-Key
Mono<OrderDto> order = webClient.post()
    .uri("/api/v1/orders")
    .header("Idempotency-Key", UUID.randomUUID().toString())
    .bodyValue(createOrderRequest)
    .retrieve()
    .bodyToMono(OrderDto.class);
```

❗ **Ловушка собеса:** «Является ли DELETE всегда идемпотентным на практике?». Технически да — по RFC. Но некоторые API возвращают 404 при повторном DELETE и считают это нарушением idempotency. Правильное понимание: повторный DELETE не меняет состояние системы (ресурс уже удалён), 404 — лишь информационный ответ, а не изменение состояния.

**→ Уточняющий вопрос:** Как настроить retry-политику в Resilience4j или Spring Retry, чтобы повторялись только идемпотентные запросы?

**↳ Ответ:** В Resilience4j через `RetryConfig` с предикатом на `HttpMethod`: retry только если метод GET, PUT, DELETE и статус ответа 5xx или исключение типа `ConnectTimeoutException`. В Spring Retry — аннотация `@Retryable` с `exclude` или кастомный `RetryPolicy` проверяющий метод. Ключевое правило: POST без idempotency-key нельзя ретраить автоматически — только вручную с явным ключом.

---

## 7. Почему GET и DELETE идемпотентны?

GET идемпотентен, потому что является read-only операцией: сколько раз ни вызови `GET /api/v1/products/42` — состояние базы данных не изменится. Это делает GET не только идемпотентным, но и «safe» (безопасным). DELETE идемпотентен по другой причине: первый вызов удаляет ресурс, второй и последующие — ничего не делают (ресурса нет), но конечное состояние одинаково — ресурс удалён.

Важно разграничить идемпотентность состояния и идемпотентность ответа. GET может вернуть разные данные при повторных вызовах (если ресурс изменился другим клиентом), и это не нарушает идемпотентность GET как метода — имеется в виду, что сам GET-запрос не меняет состояние. DELETE может вернуть 204 первый раз и 404 второй — но оба раза состояние системы одинаково: ресурс отсутствует.

```http
// Первый вызов DELETE — ресурс существует
DELETE /api/v1/orders/123 HTTP/1.1
Authorization: Bearer <token>

HTTP/1.1 204 No Content

// Второй вызов DELETE — ресурса уже нет
DELETE /api/v1/orders/123 HTTP/1.1
Authorization: Bearer <token>

HTTP/1.1 404 Not Found
{
  "error": "Order not found",
  "orderId": 123
}

// Состояние системы одинаково: заказа 123 нет.
// Ответы разные, но идемпотентность не нарушена.
```

❗ **Ловушка собеса:** «Если GET возвращает разные данные при повторных вызовах — он идемпотентен?». Да, идемпотентен — данные могут меняться другими запросами, это нормально. Идемпотентность GET означает, что сам GET не производит side effects, а не то что данные заморожены.

**→ Уточняющий вопрос:** Стоит ли возвращать 404 или 204 при повторном DELETE? Какой подход более REST-корректен и почему?

**↳ Ответ:** Оба допустимы, но 404 при повторном DELETE семантически точнее: ресурса нет — так и говорим. Возвращать 204 во второй раз (как будто только что удалил) — это «мягкая идемпотентность», удобная для клиентов, которые не хотят обрабатывать 404. Правило: если клиенты пишете вы сами — выбирайте 404 (честнее), если API публичный — 204 (дружелюбнее, меньше кода у клиентов).

---

## 8. Является ли POST идемпотентным?

Нет. POST по семантике означает «создать новый ресурс» или «выполнить действие», и каждый вызов порождает новый побочный эффект: новый заказ, новый платёж, новое письмо. Повторный POST с теми же данными создаст дубликат. Именно поэтому браузеры показывают предупреждение при обновлении страницы после отправки формы.

Чтобы сделать POST безопасным для повторных вызовов, используют паттерн Idempotency Key: клиент генерирует уникальный UUID, добавляет его в заголовок `Idempotency-Key`, сервер хранит этот ключ с результатом операции (в Redis с TTL) и при повторном запросе с тем же ключом возвращает кэшированный результат вместо повторного выполнения. Так работают Stripe, Braintree и большинство платёжных API.

```java
@PostMapping("/api/v1/orders")
public ResponseEntity<OrderDto> createOrder(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody @Valid CreateOrderRequest request,
        @AuthenticationPrincipal UserDetails user) {

    // Проверяем, не обрабатывали ли уже этот запрос
    Optional<OrderDto> cached = idempotencyService.getCachedResult(idempotencyKey);
    if (cached.isPresent()) {
        return ResponseEntity.ok(cached.get()); // возвращаем тот же результат
    }

    // Создаём заказ
    OrderDto order = orderService.create(request, user.getUsername());

    // Кэшируем результат на 24 часа
    idempotencyService.cacheResult(idempotencyKey, order, Duration.ofHours(24));

    URI location = URI.create("/api/v1/orders/" + order.getId());
    return ResponseEntity.created(location).body(order);
}
```

❗ **Ловушка собеса:** «Если я сделаю POST идемпотентным через Idempotency-Key, метод станет идемпотентным?». Нет — сам HTTP-метод POST остаётся неидемпотентным по спецификации. Idempotency-Key — это application-level механизм поверх неидемпотентного метода, не меняющий его семантику в HTTP-протоколе.

**→ Уточняющий вопрос:** Где хранить Idempotency-Key на сервере — в БД или Redis? Что делать, если первый запрос ещё обрабатывается, а второй с тем же ключом уже пришёл?

**↳ Ответ:** Redis предпочтительнее: встроенный TTL, атомарный `SET NX` (set if not exists) позволяет одной командой «застолбить» ключ и предотвратить двойную обработку. Если запрос ещё в процессе — сохраняйте статус `PROCESSING` и возвращайте 202 Accepted с `Retry-After` заголовком; когда завершится — обновить до `COMPLETED` с сериализованным ответом. В БД — вариант для систем без Redis, но нужна транзакция с `INSERT ... ON CONFLICT DO NOTHING`.

---

## 9. Какие HTTP статус коды вы знаете?

HTTP статус-коды делятся на пять классов. **2xx — успех:** 200 OK (запрос выполнен), 201 Created (ресурс создан, в Location — URI нового ресурса), 204 No Content (успех без тела ответа, используется в DELETE/PUT). **3xx — перенаправление:** 301 Moved Permanently (постоянный редирект, поисковики обновят индекс), 302 Found (временный редирект), 304 Not Modified (кэш актуален, тело не нужно).

**4xx — ошибка клиента:** 400 Bad Request (невалидный запрос — неверный JSON, отсутствующее поле), 401 Unauthorized (не аутентифицирован), 403 Forbidden (нет прав), 404 Not Found (ресурс не найден), 409 Conflict (конфликт состояния — дубликат, оптимистическая блокировка), 422 Unprocessable Entity (синтаксически верный запрос, но семантически некорректный — ошибки бизнес-валидации), 429 Too Many Requests (rate limiting). **5xx — ошибка сервера:** 500 Internal Server Error (неожиданная ошибка), 502 Bad Gateway (upstream сервис вернул ошибку), 503 Service Unavailable (сервис перегружен или на обслуживании), 504 Gateway Timeout (upstream таймаут).

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return ResponseEntity.status(422)
            .body(new ErrorResponse("VALIDATION_FAILED", errors.toString()));
    }

    @ExceptionHandler(OptimisticLockingFailureException.class)
    public ResponseEntity<ErrorResponse> handleConflict(OptimisticLockingFailureException ex) {
        return ResponseEntity.status(409)
            .body(new ErrorResponse("CONFLICT", "Resource was modified by another request"));
    }
}
```

❗ **Ловушка собеса:** Частая ошибка — использовать 400 вместо 422 для ошибок валидации, или 500 вместо 409 при конфликте. Интервьюер спросит: «Какой код вернёте, если пользователь пытается создать заказ с несуществующим товаром?». Правильный ответ зависит от контекста: если это ошибка FK-constraint — 409 или 422, а не 500.

**→ Уточняющий вопрос:** В чём разница между 400 и 422? Когда использовать каждый из них?

**↳ Ответ:** 400 — запрос невозможно распарсить (битый JSON, неверный синтаксис, отсутствующий обязательный заголовок). 422 — запрос синтаксически верен, но нарушает бизнес-правила (поле `age` отрицательное, `endDate` раньше `startDate`, email уже занят). Мнемоника: 400 — «не понял тебя», 422 — «понял, но не могу выполнить». На практике Spring Validation бросает `MethodArgumentNotValidException` — лучше возвращать 422, а не 400.

---

## 10. В чём разница между 401 и 403?

401 Unauthorized означает «вы не аутентифицированы» — сервер не знает, кто вы. Несмотря на название (Unauthorized), это об аутентификации: токена нет, он истёк или неверен. Правильная реакция клиента — перейти на страницу логина или обновить токен. 403 Forbidden означает «вы аутентифицированы, но у вас нет прав на это действие» — сервер знает, кто вы, но вы не можете выполнить эту операцию.

Практический пример из юзер-сервиса: менеджер магазина пытается обратиться к `/api/v1/admin/users`. Если его JWT истёк — 401 (нужно перелогиниться). Если JWT валиден, но роль `MANAGER` не имеет доступа к admin-эндпоинту — 403 (залогинен, но нет прав). Это разграничение важно для клиента: на 401 нужно редиректить на логин, на 403 — показывать «нет доступа» без редиректа.

```java
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException ex) throws IOException {
        // Токен отсутствует или невалиден → 401
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Authentication required");
    }
}

@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       AccessDeniedException ex) throws IOException {
        // Токен валиден, но прав нет → 403
        response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access denied");
    }
}

// В SecurityConfig:
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint(jwtAuthenticationEntryPoint) // → 401
    .accessDeniedHandler(customAccessDeniedHandler)        // → 403
);
```

❗ **Ловушка собеса:** Некоторые API возвращают 404 вместо 403 по соображениям безопасности — чтобы не раскрывать, существует ли ресурс. Например, GitHub возвращает 404 на приватные репозитории неавторизованному пользователю. Интервьюер может спросить: «Когда предпочесть 404 вместо 403?»

**→ Уточняющий вопрос:** Если пользователь аутентифицирован, но пытается получить чужой заказ (`GET /orders/999`), что вернёте: 403 или 404? Обоснуйте выбор.

**↳ Ответ:** 404 — безопаснее и правильнее: если ответить 403, вы раскрываете, что ресурс существует, а это утечка информации (перебором ID можно узнать чужие заказы). 404 говорит «нет такого ресурса для тебя» — и это честно с точки зрения пользователя. Правило: для персональных данных (заказы, платежи, личный профиль) возвращайте 404 при попытке доступа к чужому ресурсу; 403 оставьте для административных действий, где факт существования ресурса не секрет.

---

## 11. Что такое RESTful API дизайн?

RESTful API дизайн — это проектирование HTTP API с соблюдением принципов REST: ресурсы как существительные в URL, HTTP-методы как операции над ними, корректные статус-коды, stateless-взаимодействие, правильные заголовки. Модель зрелости Ричардсона (Richardson Maturity Model) выделяет три уровня: Level 0 — HTTP как транспорт (одна точка входа, всё через POST), Level 1 — ресурсы в URI, Level 2 — HTTP-методы и статус-коды, Level 3 — HATEOAS. Большинство «REST API» на практике находятся на Level 2.

Хороший RESTful дизайн означает предсказуемость: зная паттерн `/api/v1/{resource}/{id}`, разработчик угадает все эндпоинты. Плохой дизайн — смешивание глаголов и существительных, произвольные статус-коды, отсутствие версионирования. Это важно не только для удобства, но и для совместимости с инфраструктурой: CDN кэширует GET, retry-логика знает, что PUT можно повторить.

```
// Хороший RESTful дизайн для интернет-магазина
GET    /api/v1/products              // список товаров с пагинацией
GET    /api/v1/products?category=electronics&minPrice=1000  // фильтрация
POST   /api/v1/products              // создать товар (ADMIN)
GET    /api/v1/products/42           // товар по ID
PUT    /api/v1/products/42           // полная замена
PATCH  /api/v1/products/42           // частичное обновление
DELETE /api/v1/products/42           // удалить

GET    /api/v1/orders                // заказы текущего пользователя
POST   /api/v1/orders                // создать заказ
GET    /api/v1/orders/123            // детали заказа
POST   /api/v1/orders/123/cancel     // действие (не CRUD)

// Плохой дизайн — антипаттерны
POST   /api/v1/getProduct            // глагол в URL
GET    /api/v1/deleteOrder?id=123    // GET для изменения состояния
POST   /api/v1/order_management/doCreate  // смешение стилей
```

❗ **Ловушка собеса:** «Ваш API RESTful?» — это часто риторический вопрос. Большинство API не являются RESTful в строгом смысле Филдинга (без HATEOAS). Правильный ответ: «Мы следуем Level 2 по модели Ричардсона — HTTP-методы и статус-коды используем корректно, HATEOAS не реализуем, так как это усложнило бы клиенты».

**→ Уточняющий вопрос:** Как вы обрабатываете действия, которые не укладываются в CRUD — например, отмена заказа, публикация товара, блокировка пользователя?

**↳ Ответ:** Три паттерна: 1) `POST /{resource}/{id}/{action}` — наиболее читаемо (`POST /orders/123/cancel`), рекомендован Google API Design Guide; 2) PATCH с изменением поля-статуса (`PATCH /orders/123` с `{"status": "CANCELLED"}`) — проще, но теряет бизнес-семантику и не позволяет добавить специфичную валидацию; 3) отдельный ресурс-событие (`POST /order-cancellations`). Предпочитайте вариант 1 для явных бизнес-действий — код становится самодокументируемым.

---

## 12. Как правильно именовать REST endpoints?

Правила именования REST endpoints: использовать существительные (не глаголы), множественное число для коллекций (`/products`, `/orders`, `/users`), id для конкретного ресурса (`/products/42`), вложенность для иерархических отношений (`/orders/123/items`), строчные буквы, дефисы вместо underscores (`/order-items`, не `/order_items`), без расширений файлов (`.json` не нужен — для этого есть `Accept` header).

Глубина вложенности не должна превышать 2-3 уровней — глубже становится неудобно. Если ресурс может существовать независимо, лучше дать ему отдельный эндпоинт. Параметры фильтрации, сортировки и пагинации — в query string. Версия API — в пути или в заголовке.

```
// Коллекции — множественное число
GET /api/v1/users                    // список пользователей
GET /api/v1/users/123                // конкретный пользователь

// Вложенные ресурсы
GET /api/v1/users/123/orders         // заказы пользователя
GET /api/v1/orders/456/items         // позиции заказа
GET /api/v1/orders/456/items/2       // конкретная позиция заказа

// Фильтрация, сортировка, пагинация в query params
GET /api/v1/products?category=phones&minPrice=5000&sort=price&order=asc&page=1&size=20

// Действия, не вписывающиеся в CRUD — через субресурс-глагол
POST /api/v1/orders/456/cancel
POST /api/v1/orders/456/confirm
POST /api/v1/users/123/block

// Антипаттерны
GET  /api/v1/getUsers                // ❌ глагол
POST /api/v1/user/Create             // ❌ глагол + единственное число
GET  /api/v1/Users/GetById/123       // ❌ camelCase + глагол
```

❗ **Ловушка собеса:** «Как именовать эндпоинт для поиска?». Варианты: `GET /products?query=iphone` (рекомендуется), `GET /products/search?q=iphone` (допустимо, хотя `search` — глагол), `POST /products/search` (если параметры сложные). Важно выбрать один подход и придерживаться его во всём API.

**→ Уточняющий вопрос:** Как именовать эндпоинт для получения текущего пользователя — `/users/me` или `/users/{id}`? В чём разница и когда использовать `/me`?

**↳ Ответ:** `/users/me` — устоявшийся паттерн (GitHub, Spotify, Google) для «текущего аутентифицированного пользователя»; клиент не знает свой ID заранее, не нужен лишний запрос. `/users/{id}` требует, чтобы клиент уже знал userId (из JWT). Используйте `/me` если userId хранится только в токене на сервере, или для UX — проще. Проблема `/me`: технически `me` — не числовой ID, нарушает однородность API, но это широко принятое исключение.

---

## 13. Стоит ли использовать глаголы в URL?

Нет, глаголы в URL — антипаттерн в REST. Действие над ресурсом должно выражаться HTTP-методом, а не URL. `POST /users` вместо `GET /createUser`, `DELETE /users/42` вместо `POST /deleteUser?id=42`. Это принцип Uniform Interface: HTTP уже имеет словарь глаголов (GET, POST, PUT, PATCH, DELETE), дублировать их в URL — избыточность и нарушение семантики.

Есть обоснованное исключение: действия, которые не ложатся на стандартные CRUD-операции. Отмена заказа, подтверждение платежа, блокировка пользователя — это не «обновление поля status», а бизнес-транзакции с собственной логикой и валидацией. Для таких случаев допустим паттерн `POST /{resource}/{id}/{action}`: `POST /orders/123/cancel`, `POST /payments/456/refund`. Это Google API Design Guide-совместимый подход («custom methods»).

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    // Хорошо: HTTP-метод выражает действие
    @PostMapping                          // создать заказ
    @GetMapping("/{id}")                  // получить заказ
    @PatchMapping("/{id}")                // обновить поля заказа
    @DeleteMapping("/{id}")               // удалить черновик

    // Хорошо: действие не в CRUD — субресурс с POST
    @PostMapping("/{id}/cancel")
    public ResponseEntity<OrderDto> cancelOrder(@PathVariable Long id,
                                                 @RequestBody CancelOrderRequest request) {
        // Бизнес-логика отмены, не просто смена статуса
        OrderDto cancelled = orderService.cancel(id, request.getReason());
        return ResponseEntity.ok(cancelled);
    }

    @PostMapping("/{id}/confirm")
    public ResponseEntity<OrderDto> confirmOrder(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.confirm(id));
    }

    // Плохо: глаголы в URL
    // @PostMapping("/createOrder")
    // @GetMapping("/getOrderById")
    // @PostMapping("/cancelOrder/{id}")
}
```

❗ **Ловушка собеса:** «Как реализовать логин через REST?». Многие отвечают `POST /login` — это глагол! Правильнее: `POST /auth/sessions` (создание сессии) или `POST /auth/tokens` (создание токена). Хотя `/login` и `/logout` настолько устоялись, что их часто принимают как допустимое исключение.

**→ Уточняющий вопрос:** Как бы вы реализовали эндпоинт для массовой операции — например, массовое обновление статуса 50 заказов одним запросом? Как именовать и какой метод использовать?

**↳ Ответ:** `POST /orders/bulk-cancel` или `PATCH /orders` с массивом в теле — оба приемлемы. POST предпочтительнее для bulk-actions, потому что это операция с побочными эффектами и неопределённой идемпотентностью. Тело: `{"ids": [1,2,3], "status": "CANCELLED"}`. Ответ: 207 Multi-Status с индивидуальным статусом каждой операции (часть может упасть). Ключевое решение — транзакционность: делать всё-или-ничего или best-effort с частичным успехом.

---

## 14. Что такое HATEOAS?

HATEOAS (Hypermedia as the Engine of Application State) — принцип REST уровня 3 по модели Ричардсона, при котором ответ сервера содержит не только данные, но и ссылки на доступные следующие действия. Клиент не знает заранее URL эндпоинтов — он «обнаруживает» их из ответов, как пользователь, переходящий по гиперссылкам на сайте. Сервер через links управляет тем, какие операции доступны в текущем состоянии ресурса.

Практическая ценность: если заказ в статусе PENDING, ответ содержит ссылки `cancel` и `confirm`. Если заказ уже CANCELLED — только ссылку `reorder`. Клиент не нужно жёстко кодировать бизнес-логику состояний — он просто проверяет наличие нужной ссылки. Минус: усложняет клиент и сервер, не совместим с большинством фреймворков из коробки. Поэтому в реальных проектах HATEOAS реализуют редко — обычно ограничиваются Level 2.

```java
// Spring HATEOAS
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @GetMapping("/{id}")
    public EntityModel<OrderDto> getOrder(@PathVariable Long id) {
        OrderDto order = orderService.findById(id);
        EntityModel<OrderDto> model = EntityModel.of(order);

        // Всегда добавляем self-ссылку
        model.add(linkTo(methodOn(OrderController.class).getOrder(id)).withSelfRel());

        // Динамически добавляем действия в зависимости от состояния
        if (order.getStatus() == OrderStatus.PENDING) {
            model.add(linkTo(methodOn(OrderController.class).cancelOrder(id, null))
                .withRel("cancel"));
            model.add(linkTo(methodOn(OrderController.class).confirmOrder(id))
                .withRel("confirm"));
        }
        if (order.getStatus() == OrderStatus.DELIVERED) {
            model.add(linkTo(methodOn(OrderController.class).createReturn(id, null))
                .withRel("return"));
        }
        return model;
    }
}
```

```json
{
  "id": 123,
  "status": "PENDING",
  "total": 9999,
  "_links": {
    "self":    { "href": "/api/v1/orders/123" },
    "cancel":  { "href": "/api/v1/orders/123/cancel" },
    "confirm": { "href": "/api/v1/orders/123/confirm" }
  }
}
```

❗ **Ловушка собеса:** «Используете ли HATEOAS в своих API?». Честный ответ ценится больше, чем «да, конечно». Правильно: «HATEOAS — это Level 3 зрелости REST. В большинстве проектов мы останавливаемся на Level 2, потому что HATEOAS значительно усложняет клиент и сервер, а клиенты всё равно привязываются к конкретной схеме данных».

**→ Уточняющий вопрос:** Какие альтернативы HATEOAS существуют для управления переходами состояний ресурса — например, как GraphQL или gRPC решают эту проблему?

**↳ Ответ:** GraphQL решает через схему типов и явные mutations — клиент запрашивает только нужные поля, доступные действия описаны в схеме. gRPC — через явное определение методов в `.proto` файле, никакого «discovery» нет. Альтернативы HATEOAS в REST: OpenAPI/Swagger-спека как «внешний HATEOAS» (клиент читает документацию, а не ссылки в ответе), или поле `allowedActions: ["cancel", "confirm"]` в ответе — компромисс между HATEOAS и простотой.

---

## 15. Как организовать версионирование REST API?

Существует три основных подхода. **URL-версионирование** (`/api/v1/products`) — самый распространённый: версия видна явно, легко тестировать в браузере, поддерживается CDN и кэшами. Минус: URL должен идентифицировать ресурс, а не его версию. **Header-версионирование** (`Accept: application/vnd.shop.v2+json` или кастомный `API-Version: 2`) — семантически чище (URL не меняется), но сложнее в отладке и тестировании. **Query parameter** (`/products?version=2`) — не рекомендуется: загрязняет query string, плохо кэшируется.

На практике большинство компаний (GitHub, Stripe, Twitter) используют URL-версионирование из-за простоты. Стратегия эволюции: v1 должна жить достаточно долго для миграции клиентов (обычно 6-12 месяцев после выхода v2), для устаревших эндпоинтов добавляют заголовок `Deprecation: true` и `Sunset: <date>`. Семантическое версионирование (semver) в REST API обычно не применяют — мажорная версия в URL достаточна.

```java
// Вариант 1: URL-версионирование через разные контроллеры
@RestController
@RequestMapping("/api/v1/products")
public class ProductV1Controller {
    @GetMapping("/{id}")
    public ProductDtoV1 getProduct(@PathVariable Long id) { ... }
}

@RestController
@RequestMapping("/api/v2/products")
public class ProductV2Controller {
    @GetMapping("/{id}")
    public ProductDtoV2 getProduct(@PathVariable Long id) { ... }
}

// Вариант 2: Header-версионирование
@GetMapping(value = "/products/{id}", headers = "API-Version=1")
public ProductDtoV1 getProductV1(@PathVariable Long id) { ... }

@GetMapping(value = "/products/{id}", headers = "API-Version=2")
public ProductDtoV2 getProductV2(@PathVariable Long id) { ... }

// Вариант 3: Content Negotiation (Accept header)
@GetMapping(value = "/products/{id}",
            produces = "application/vnd.shop.v1+json")
public ProductDtoV1 getProductV1(@PathVariable Long id) { ... }

@GetMapping(value = "/products/{id}",
            produces = "application/vnd.shop.v2+json")
public ProductDtoV2 getProductV2(@PathVariable Long id) { ... }
```

❗ **Ловушка собеса:** «Нужно ли версионировать API с первой версии?». Да — добавить `/v1` сразу дешевле, чем потом мигрировать клиентов. Ещё ловушка: «Как версионировать только часть API?». Ответ: обычно версионируют весь API единообразно, а не отдельные эндпоинты — иначе матрица версий становится неуправляемой.

**→ Уточняющий вопрос:** Как организовать backward compatibility — какие изменения API считаются breaking changes, а какие нет?

**↳ Ответ:** Non-breaking: добавить новое поле в ответ, добавить новый опциональный параметр, добавить новый эндпоинт, расширить enum новым значением. Breaking: удалить поле из ответа, переименовать поле, изменить тип поля, удалить эндпоинт, изменить семантику статус-кода. Правило Postel: «будь либерален в том, что принимаешь, строг в том, что отдаёшь» — клиенты должны игнорировать неизвестные поля в ответе (что делают Jackson и большинство JSON-клиентов по умолчанию).

---

## 16. Что такое Content-Type header?

`Content-Type` — HTTP-заголовок, который указывает MIME-тип тела запроса или ответа. Он сообщает получателю (сервер получает его от клиента в запросе, клиент — от сервера в ответе), в каком формате передаётся тело и как его парсить. Без этого заголовка получатель вынужден угадывать формат или отказывать в обработке.

Основные значения в REST API: `application/json` (JSON-тело — самый распространённый), `application/xml` (XML), `multipart/form-data` (загрузка файлов), `application/x-www-form-urlencoded` (данные формы), `application/octet-stream` (бинарные данные). Spring MVC автоматически выбирает `Content-Type` ответа на основе `produces` в аннотации и зарегистрированных `HttpMessageConverter`. Если клиент отправил `Content-Type: application/xml`, а сервер не поддерживает XML — вернётся 415 Unsupported Media Type.

```java
// Spring автоматически выставляет Content-Type: application/json
// для методов, возвращающих объекты (через MappingJackson2HttpMessageConverter)
@PostMapping(value = "/api/v1/orders",
             consumes = "application/json",   // принимаем только JSON
             produces = "application/json")   // отвечаем только JSON
public ResponseEntity<OrderDto> createOrder(@RequestBody CreateOrderRequest request) {
    return ResponseEntity.status(201).body(orderService.create(request));
}

// Эндпоинт для загрузки файлов
@PostMapping(value = "/api/v1/products/{id}/images",
             consumes = "multipart/form-data")
public ResponseEntity<ImageDto> uploadImage(@PathVariable Long id,
                                             @RequestParam("file") MultipartFile file) {
    ImageDto image = productService.uploadImage(id, file);
    return ResponseEntity.status(201).body(image);
}
```

```http
// Запрос с JSON-телом
POST /api/v1/orders HTTP/1.1
Content-Type: application/json
Content-Length: 85

{"productId": 42, "quantity": 2, "deliveryAddress": "Москва, Арбат 1"}

// Если Content-Type не соответствует телу — 400 или 415
POST /api/v1/orders HTTP/1.1
Content-Type: application/xml   ← сервер ожидает JSON

HTTP/1.1 415 Unsupported Media Type
```

❗ **Ловушка собеса:** «Что произойдёт, если не указать Content-Type в POST-запросе с JSON-телом?». Spring по умолчанию предполагает `application/octet-stream` или отказывает с 415. Ещё частая ошибка: `Content-Type: application/json` при отправке `multipart/form-data` — Jackson не сможет парсить multipart и выбросит исключение.

**→ Уточняющий вопрос:** Как настроить Spring MVC для поддержки как JSON, так и XML в зависимости от Content-Type запроса? Что такое HttpMessageConverter и как добавить свой?

**↳ Ответ:** `HttpMessageConverter` — стратегия сериализации/десериализации тела запроса/ответа. Spring регистрирует `MappingJackson2HttpMessageConverter` для JSON и `Jaxb2RootElementHttpMessageConverter` для XML автоматически при наличии зависимостей. Кастомный конвертер: реализовать `HttpMessageConverter<T>`, переопределить `supports()`, `read()`, `write()` и зарегистрировать через `WebMvcConfigurer.extendMessageConverters()`. Для одновременной поддержки JSON и XML достаточно добавить зависимость `jackson-dataformat-xml`.

---

## 17. Что такое Accept header?

`Accept` — HTTP-заголовок запроса, которым клиент сообщает серверу, какие форматы ответа он понимает и предпочитает. Сервер выбирает подходящий формат из доступных и выставляет соответствующий `Content-Type` в ответе. Этот механизм называется **content negotiation** (согласование содержимого). Если сервер не может вернуть ни один из запрошенных форматов — возвращает 406 Not Acceptable.

Значение `Accept` может содержать несколько форматов с весами (`q` — quality factor): `Accept: application/json;q=0.9, application/xml;q=0.8, */*;q=0.1`. Чем выше q (от 0 до 1), тем предпочтительнее формат. `*/*` означает «любой формат». В REST API `Accept: application/json` почти всегда используется по умолчанию в современных клиентах. Для версионирования через content negotiation используют vendor MIME-types: `Accept: application/vnd.shop.v2+json`.

```java
// Spring MVC автоматически делает content negotiation
// через ContentNegotiationStrategy и HttpMessageConverter
@GetMapping(value = "/api/v1/products/{id}",
            produces = {"application/json", "application/xml"})
public ResponseEntity<ProductDto> getProduct(@PathVariable Long id) {
    // Spring сам выберет нужный конвертер на основе Accept header
    return ResponseEntity.ok(productService.findById(id));
}

// Кастомный content negotiation для версионирования
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(false)
            .ignoreAcceptHeader(false)
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

```http
// Клиент просит JSON
GET /api/v1/products/42 HTTP/1.1
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json
{"id": 42, "name": "iPhone 15", "price": 89999}

// Клиент просит XML
GET /api/v1/products/42 HTTP/1.1
Accept: application/xml

HTTP/1.1 200 OK
Content-Type: application/xml
<product><id>42</id><name>iPhone 15</name><price>89999</price></product>

// Сервер не поддерживает формат
GET /api/v1/products/42 HTTP/1.1
Accept: text/csv

HTTP/1.1 406 Not Acceptable
```

❗ **Ловушка собеса:** «В чём разница между `Content-Type` и `Accept`?». Content-Type описывает, что отправляю я (тело моего запроса). Accept описывает, что хочу получить в ответе. Частая ошибка Junior — путать эти два заголовка или использовать Content-Type там, где нужен Accept.

**→ Уточняющий вопрос:** Если клиент шлёт `Accept: */*`, какой формат вернёт Spring по умолчанию? Как изменить этот default и почему это важно для API-first дизайна?

**↳ Ответ:** Spring вернёт первый зарегистрированный `HttpMessageConverter`, поддерживающий возвращаемый тип — обычно JSON (Jackson регистрируется первым). Изменить через `ContentNegotiationConfigurer.defaultContentType(MediaType.APPLICATION_JSON)`. Это важно для API-first: если вдруг XML-конвертер окажется первым (например, после добавления зависимости), клиенты без явного `Accept` неожиданно получат XML. Явный default защищает от таких сюрпризов.
