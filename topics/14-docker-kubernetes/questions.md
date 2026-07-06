# Docker / Kubernetes

> 📇 Справочник уровня middle. Формат: ответ по сути → как работает технически → реальный пример для Java Spring Boot → ❗ ловушка собеса → уточняющий вопрос.

**Всего вопросов: 24**

---

## 1. Что такое контейнеризация и зачем она нужна?

Контейнеризация — это упаковка приложения вместе со всеми его зависимостями (библиотеки, runtime, конфиг, переменные окружения) в единый изолированный образ, который запускается одинаково на любой машине. Проблема «у меня локально работает» исчезает, потому что образ описывает окружение полностью.

Технически контейнер использует механизмы ядра Linux: **namespaces** для изоляции (PID, network, mount, UTS, IPC) и **cgroups** для ограничения ресурсов (CPU, память, I/O). Процесс видит свою изолированную файловую систему и сетевой интерфейс, но при этом разделяет ядро хоста с другими контейнерами.

Для Java Spring Boot сервиса это означает: упаковываем JAR-файл, JRE, переменные окружения и конфиги в Docker-образ. CI/CD собирает образ один раз, прогоняет тесты и этот же образ деплоит в prod. Никакой разницы между dev/staging/prod окружениями на уровне runtime.

❗ **Ловушка собеса:** спросят «а чем отличается от VM?» — контейнер не виртуализирует железо и не содержит гостевой ОС. Это изолированный процесс на хостовом ядре. Второй момент: «если контейнеры используют одно ядро, в чём изоляция?» — ответ: namespaces изолируют видимость ресурсов, cgroups ограничивают их потребление, но уязвимость ядра потенциально затрагивает все контейнеры (в отличие от VM).

**→ Уточняющий вопрос:** Как контейнеры обеспечивают изоляцию файловой системы — что такое union filesystem и зачем нужны слои образа?

**↳ Ответ:** Docker использует OverlayFS: образ состоит из неизменяемых read-only слоёв, поверх которых при старте контейнера добавляется один write-слой (copy-on-write). Если контейнер изменяет файл — он копируется в write-слой, оригинальный слой не трогается. Слои разделяются между контейнерами — 10 контейнеров одного образа не дублируют файлы на диске, это ускоряет `docker pull` и экономит место.

---

## 2. В чём отличие контейнера от виртуальной машины?

VM запускает полный стек: гипервизор (VMware, KVM, VirtualBox) → гостевая ОС (ядро + userspace) → приложение. Каждая VM весит гигабайты и стартует минуты. Контейнер — это изолированный процесс прямо на ядре хоста, без гостевой ОС. Весит мегабайты, стартует секунды.

Технически: VM виртуализирует оборудование (CPU, память, диски через гипервизор), гостевая ОС работает как обычный пользователь этого виртуального железа. Контейнер использует только ядро хоста через namespaces/cgroups — никакой виртуализации железа нет. Образ контейнера — это набор слоёв файловой системы (overlay FS), не целый диск.

Для Java-разработчика: Spring Boot сервис в контейнере стартует за ~10 секунд, занимает ~200 МБ на диске (JRE + JAR). Та же VM с Ubuntu + JRE займёт 2–5 ГБ и стартует 1–2 минуты. На одном хосте можно запустить десятки контейнеров против единиц VM.

```
VM:         Контейнер:
+--------+  +--------+
|  App   |  |  App   |
+--------+  +--------+
| Guest  |  | Libs   |
|  OS    |  +--------+
+--------+  | Docker |
| Hyper- |  | Engine |
| visor  |  +--------+
+--------+  | Host OS Kernel |
| Hardware               |
```

❗ **Ловушка собеса:** «Контейнеры более безопасны, чем VM?» — нет, VM изолированы лучше (отдельное ядро). Контейнеры легче и быстрее, но шаренное ядро — это потенциальная поверхность атаки. В security-sensitive окружениях используют gVisor или Kata Containers (VM-like изоляция + скорость контейнеров).

**→ Уточняющий вопрос:** Что такое слои Docker-образа и как работает copy-on-write при записи в контейнере?

**↳ Ответ:** Каждая инструкция `RUN`/`COPY`/`ADD` в Dockerfile создаёт новый read-only слой — diff относительно предыдущего. При записи из контейнера OverlayFS копирует файл из нижнего слоя в верхний write-слой (copy-on-write) — образ не изменяется. Практическое следствие: если в контейнере удалить файл из нижнего слоя, он «исчезает» в write-слое, но физически занимает место в образе — вот почему надо объединять `RUN rm ...` в одну инструкцию с созданием файла.

---

## 3. Что такое Dockerfile?

Dockerfile — текстовый файл с набором инструкций, по которым Docker Engine собирает образ (image). Каждая инструкция создаёт новый слой в union filesystem. Результат сборки (`docker build`) — неизменяемый образ, который можно запустить как контейнер или опубликовать в registry.

Технически сборка работает пошагово: каждая инструкция выполняется в промежуточном контейнере, результат фиксируется как слой, промежуточный контейнер удаляется. Docker кеширует слои — если инструкция и её контекст не изменились, слой берётся из кеша. Поэтому порядок инструкций критически важен для скорости сборки.

```dockerfile
# Dockerfile для Spring Boot приложения
FROM eclipse-temurin:21-jre-alpine

# Метаданные
LABEL maintainer="team@example.com"
LABEL version="1.0"

# Рабочая директория внутри контейнера
WORKDIR /app

# Копируем JAR (имя фиксировано через Maven/Gradle)
COPY target/myapp-1.0.jar app.jar

# Порт, который слушает приложение
EXPOSE 8080

# Запуск приложения
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Собираем и запускаем:
```bash
docker build -t myapp:1.0 .
docker run -p 8080:8080 -e SPRING_PROFILES_ACTIVE=prod myapp:1.0
```

❗ **Ловушка собеса:** «Почему слои важны?» — порядок инструкций влияет на кеш. Если `COPY pom.xml` стоит перед `RUN mvn dependency:resolve`, зависимости кешируются отдельно от кода. При изменении только Java-кода зависимости не перекачиваются. Если же сразу `COPY . .` — любое изменение кода инвалидирует кеш зависимостей.

**→ Уточняющий вопрос:** Как оптимизировать кеширование слоёв Dockerfile для Maven/Gradle проекта?

**↳ Ответ:** Правило: от редко меняемого к часто меняемому. Сначала копируй `pom.xml` и скачивай зависимости (`RUN mvn dependency:go-offline`) — этот слой кешируется, пока не изменится `pom.xml`. Затем копируй `src/` и собирай. Так при изменении только кода Java зависимости не перекачиваются — сборка в CI ускоряется с минут до секунд. Для Gradle: сначала `build.gradle` + `gradlew`, затем `./gradlew dependencies`, затем `./gradlew build`.

---

## 4. Какие основные инструкции используются в Dockerfile?

Ключевые инструкции: `FROM` — базовый образ (обязателен первым), `WORKDIR` — рабочая директория (создаётся если нет), `COPY` — копирует файлы из build context в образ, `ADD` — как COPY но умеет распаковывать архивы и скачивать URL (избегать без нужды), `RUN` — выполняет команду при сборке, `ENV` — переменные окружения в образе, `EXPOSE` — документирует порт (не открывает!), `ARG` — аргумент сборки (передаётся через `--build-arg`), `VOLUME` — точка монтирования, `USER` — под каким пользователем запускать процесс, `HEALTHCHECK` — встроенная проверка здоровья, `CMD`/`ENTRYPOINT` — что запускать.

Технически `RUN` создаёт новый слой — каждая команда увеличивает размер образа. Поэтому цепочки команд объединяют через `&&` и очищают кеши в той же инструкции:

```dockerfile
FROM eclipse-temurin:21-jre-alpine

# ARG доступен только во время сборки
ARG APP_VERSION=1.0.0

# ENV доступен в запущенном контейнере
ENV APP_HOME=/app \
    JAVA_OPTS="-Xms256m -Xmx512m"

WORKDIR ${APP_HOME}

# Права безопасности: не root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Кешируем зависимости отдельно от кода
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
RUN ./mvnw dependency:go-offline -B

COPY src ./src
RUN ./mvnw package -DskipTests -B

# Меняем владельца
RUN chown -R appuser:appgroup ${APP_HOME}
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -q -O- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} -jar app.jar"]
```

❗ **Ловушка собеса:** Запуск от root — грубая security-ошибка (если контейнер взломали, атакующий имеет root на файловой системе контейнера). Всегда добавляй `USER` инструкцию. Также: `EXPOSE` — только документация, реально порт открывается через `-p` при `docker run`.

**→ Уточняющий вопрос:** Почему лучше использовать `COPY` вместо `ADD` для копирования файлов проекта?

**↳ Ответ:** `ADD` делает скрытые вещи: автоматически распаковывает tar-архивы и умеет скачивать URL — это неочевидно при чтении Dockerfile и создаёт непредсказуемое поведение. Правило: `COPY` — всегда по умолчанию; `ADD` — только если явно нужна распаковка архива прямо в образе. Это делает Dockerfile читаемым и предсказуемым.

---

## 5. В чём разница между CMD и ENTRYPOINT?

`ENTRYPOINT` задаёт основной исполняемый процесс контейнера — его нельзя переопределить простой передачей аргументов в `docker run`, только через `--entrypoint`. `CMD` задаёт аргументы по умолчанию для ENTRYPOINT (или команду по умолчанию, если ENTRYPOINT не задан) — её можно полностью заменить, передав команду после имени образа в `docker run`.

Технически существуют два формата: **exec form** (`["java", "-jar", "app.jar"]`) и **shell form** (`java -jar app.jar`). Shell form запускает команду через `/bin/sh -c`, что означает: сигналы (SIGTERM при `docker stop`) получает sh, а не Java-процесс. Это критично — приложение не получит сигнал завершения и будет убито через 10 секунд по таймауту.

```dockerfile
# Правильно для Spring Boot: exec form, ENTRYPOINT фиксирует JVM
ENTRYPOINT ["java", "-jar", "app.jar"]
# CMD позволяет переопределить Spring Boot аргументы при запуске
CMD ["--spring.profiles.active=default"]

# Запуск с prod профилем (CMD переопределяется):
# docker run myapp --spring.profiles.active=prod

# Неправильно: shell form — SIGTERM не дойдёт до JVM
# ENTRYPOINT java -jar app.jar
```

Комбинация: ENTRYPOINT — программа, CMD — её параметры по умолчанию. Только CMD — полностью переопределяемая команда. Только ENTRYPOINT в exec form — фиксированная команда без параметров.

❗ **Ловушка собеса:** Shell form vs exec form — это любимый вопрос на собесах. Shell form означает PID 1 = `/bin/sh`, Java-процесс — дочерний. `docker stop` шлёт SIGTERM PID 1 (sh), sh не форвардит сигнал детям, через 10 секунд SIGKILL. Spring Boot не успевает graceful shutdown: не закрывает соединения с БД, не завершает in-flight запросы. Всегда используй exec form.

**→ Уточняющий вопрос:** Как настроить graceful shutdown в Spring Boot при работе в Docker, и какую роль играет `server.shutdown=graceful`?

**↳ Ответ:** `server.shutdown=graceful` заставляет Spring Boot при получении SIGTERM перестать принимать новые запросы и дождаться завершения текущих (таймаут — `spring.lifecycle.timeout-per-shutdown-phase`, дефолт 30s). Но это работает только если SIGTERM доходит до JVM: ENTRYPOINT должен быть в exec form `["java", "-jar", "app.jar"]`, а не shell form. В K8s добавь `lifecycle.preStop: exec: sleep 5` — это даёт время kube-proxy убрать Pod из iptables до того как придёт SIGTERM.

---

## 6. Что такое multi-stage build?

Multi-stage build — это Dockerfile с несколькими `FROM` инструкциями, где каждый стейдж — отдельная стадия сборки. Финальный образ содержит только то, что явно скопировано из предыдущих стейджей. Главная цель: разделить build-окружение (тяжёлый JDK + Maven/Gradle) от runtime-окружения (лёгкий JRE).

Технически Docker выполняет стейджи последовательно, но при сборке с `--target` можно остановиться на любом. Из предыдущих стейджей копируется только то, что нужно через `COPY --from=<стейдж>`. Build-зависимости (Maven ~/.m2, Gradle кеши, исходники) в финальный образ не попадают.

```dockerfile
# Стейдж 1: сборка (тяжёлый образ с JDK и Maven)
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build

# Кешируем зависимости
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
RUN ./mvnw dependency:go-offline -B

# Собираем JAR
COPY src ./src
RUN ./mvnw package -DskipTests -B

# Стейдж 2: распаковка для layered JAR (оптимизация)
FROM eclipse-temurin:21-jdk-alpine AS extractor
WORKDIR /extract
COPY --from=builder /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# Стейдж 3: финальный образ (лёгкий JRE)
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Создаём непривилегированного пользователя
RUN addgroup -S spring && adduser -S spring -G spring

# Копируем слои в правильном порядке (от редкоменяемых к часто)
COPY --from=extractor /extract/dependencies/ ./
COPY --from=extractor /extract/spring-boot-loader/ ./
COPY --from=extractor /extract/snapshot-dependencies/ ./
COPY --from=extractor /extract/application/ ./

USER spring
EXPOSE 8080

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Разница в размере: `eclipse-temurin:21-jdk-alpine` ~340 МБ, `eclipse-temurin:21-jre-alpine` ~90 МБ. Финальный образ Spring Boot приложения ~130 МБ вместо ~500 МБ.

❗ **Ловушка собеса:** Layered JAR (стейдж 2 в примере) — продвинутый паттерн. Spring Boot 2.3+ поддерживает `jarmode=layertools`, который разделяет JAR на слои: зависимости (меняются редко) → код приложения (меняется часто). При пересборке только изменённые слои не берутся из кеша Docker. На собесе это демонстрирует понимание оптимизации CI/CD.

**→ Уточняющий вопрос:** Какой размер у типичного production Docker образа для Spring Boot, и как его можно уменьшить?

**↳ Ответ:** Наивный Dockerfile с JDK + Maven + FAT JAR внутри даёт ~500–800 МБ. Multi-stage build с `eclipse-temurin:21-jre-alpine` снижает до ~130–200 МБ. Дальше: layered JAR разделяет зависимости и код — при пересборке только изменённый слой не берётся из кеша Docker-registry. Максимальное сокращение: GraalVM Native Image даёт ~30–80 МБ и мгновенный старт, но требует AOT-компиляции и ограничивает использование рефлексии.

---

## 7. Что такое Docker Compose?

Docker Compose — инструмент для декларативного описания и запуска мульти-контейнерных приложений через файл `docker-compose.yml`. Позволяет одной командой (`docker compose up`) поднять всю инфраструктуру: приложение + БД + Kafka + Redis, с настроенными сетями, томами и зависимостями.

Технически Compose создаёт изолированную сеть для сервисов (по умолчанию bridge network), сервисы обращаются друг к другу по имени сервиса (DNS resolution внутри сети). Порядок старта управляется через `depends_on` (но Compose ждёт только запуска контейнера, не готовности приложения внутри — для этого нужен `healthcheck` + `condition: service_healthy`).

```yaml
# docker-compose.yml для разработки Spring Boot приложения
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/mydb
      SPRING_DATASOURCE_USERNAME: myuser
      SPRING_DATASOURCE_PASSWORD: mypass
      SPRING_PROFILES_ACTIVE: dev
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - backend

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - backend

volumes:
  postgres_data:

networks:
  backend:
    driver: bridge
```

```bash
docker compose up -d          # запустить в фоне
docker compose logs -f app    # логи приложения
docker compose down -v        # остановить и удалить volumes
```

❗ **Ловушка собеса:** `depends_on` без `condition: service_healthy` не гарантирует, что БД готова принимать соединения. Spring Boot попытается подключиться к PostgreSQL, который ещё инициализируется, и упадёт. Решение: healthcheck + `condition: service_healthy`, или настроить retry в Spring через `spring.datasource.hikari.connection-timeout` + библиотеку типа Flyway с retry.

**→ Уточняющий вопрос:** Чем Docker Compose отличается от Kubernetes и когда что использовать?

**↳ Ответ:** Compose — один хост, один файл, простота: идеален для локальной разработки и небольших staging-окружений. Kubernetes — кластер машин с автоматическим масштабированием, самовосстановлением и rolling updates: нужен в production где требуется отказоустойчивость. Правило выбора: если один `docker compose up` закрывает все потребности — используй Compose; если нужно масштабировать горизонтально, деплоить без даунтайма, или распределять нагрузку по нескольким хостам — K8s.

---

## 8. Что такое Kubernetes и зачем он нужен?

Kubernetes (K8s) — система оркестрации контейнеров с открытым исходным кодом от Google. Автоматизирует деплой, масштабирование, сети, балансировку нагрузки, самовосстановление и обновления контейнеризированных приложений. Docker Compose хорош локально, но не масштабируется на prod: не умеет распределять нагрузку по машинам, перезапускать упавшие сервисы, делать rolling updates без даунтайма.

Технически K8s состоит из **control plane** (API Server, etcd, Scheduler, Controller Manager) и **worker nodes** (kubelet, kube-proxy, container runtime). Пользователь описывает желаемое состояние (Desired State) в YAML/JSON, Controller Manager непрерывно сверяет текущее состояние с желаемым и принимает меры для выравнивания (reconciliation loop).

Для Java-разработчика взаимодействие с K8s выглядит так:
```bash
# Применить конфигурацию деплоя
kubectl apply -f deployment.yaml

# Посмотреть статус
kubectl get pods -n production
kubectl describe pod myapp-7d4f9b-xk2p1 -n production

# Логи
kubectl logs -f myapp-7d4f9b-xk2p1 -n production

# Зайти в контейнер для отладки
kubectl exec -it myapp-7d4f9b-xk2p1 -n production -- /bin/sh
```

❗ **Ловушка собеса:** «K8s это замена Docker Compose?» — нет, это разные уровни. Compose — для локальной разработки (один хост, простота). K8s — для production (кластер машин, отказоустойчивость, масштабирование). Многие команды используют Compose локально и K8s в prod. Также: K8s не работает с Docker напрямую — использует Container Runtime Interface (CRI), сейчас обычно containerd.

**→ Уточняющий вопрос:** Что такое etcd и почему его потеря означает потерю кластера?

**↳ Ответ:** etcd — распределённое key-value хранилище (алгоритм Raft), единственный источник истины для всего состояния кластера: все Deployments, Pods, Services, Secrets хранятся там. При потере etcd API Server не может ни читать, ни писать состояние — кластер перестаёт функционировать. Поэтому etcd в production всегда кластер (3 или 5 нод для кворума), а бэкап делается регулярно через `etcdctl snapshot save`.

---

## 9. Что такое Pod в Kubernetes?

Pod — минимальная единица деплоя в Kubernetes, один или несколько контейнеров, которые всегда запускаются вместе на одном узле, разделяют сетевой namespace (один IP, общий localhost) и могут разделять тома. Поды эфемерны: при падении Controller создаёт новый Pod с новым IP, а не восстанавливает старый.

Технически все контейнеры внутри пода видят друг друга через `localhost`, что и является главным смыслом группировки. Каждый Pod получает уникальный IP в сети кластера (pod network overlay). Scheduler решает, на какой Node разместить Pod, исходя из ресурсов, affinity-правил, taints/tolerations.

Паттерн **sidecar**: основной контейнер (Spring Boot app) + вспомогательный (Envoy proxy, Fluentd логгер, Vault agent для секретов) в одном Pod. Sidecar прозрачно перехватывает трафик или обогащает функциональность без изменения основного приложения.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  - name: log-shipper          # sidecar
    image: fluent/fluent-bit:latest
    volumeMounts:
    - name: logs
      mountPath: /logs
  volumes:
  - name: logs
    emptyDir: {}
```

❗ **Ловушка собеса:** Pod ≠ контейнер. Разработчики часто приравнивают их. Также: `requests` и `limits` — не одно и то же. `requests` — гарантированные ресурсы для Scheduler (он учитывает при размещении). `limits` — максимум (при превышении CPU — throttling, памяти — OOMKill). Если `requests` не задать, Scheduler не знает сколько нужно и может разместить на перегруженной ноде.

**→ Уточняющий вопрос:** Почему не стоит деплоить приложения напрямую как Pod, а использовать Deployment?

**↳ Ответ:** Pod без контроллера — одиночка: упал — остался упавшим, никто не пересоздаст. Deployment управляет ReplicaSet, который отвечает за нужное число реплик и автопересоздание при сбое. Плюс Deployment даёт rolling update, rollback историю и декларативное управление версиями. Голый Pod создаётся только для одноразовых диагностических задач (`kubectl run debug --image=...`), не для production-сервисов.

---

## 10. Что такое Node в Kubernetes?

Node (узел) — физическая или виртуальная машина, входящая в кластер Kubernetes, на которой запускаются Pods. Кластер состоит из **control plane nodes** (управление кластером) и **worker nodes** (выполнение рабочих нагрузок). На каждой worker node работает: **kubelet** (агент, общается с API Server, управляет Pod lifecycle), **kube-proxy** (управляет сетевыми правилами iptables/ipvs для Service), **container runtime** (containerd или другой CRI).

Технически Scheduler при размещении пода выбирает подходящий узел по нескольким критериям: достаточно ли `requests` ресурсов, удовлетворяет ли node `nodeSelector` / `affinity` / `tolerations`. После размещения kubelet на выбранном узле получает манифест пода и запускает контейнеры через container runtime.

```bash
# Посмотреть состояние узлов
kubectl get nodes
kubectl describe node worker-1

# Узнать сколько ресурсов выделено на узле
kubectl describe node worker-1 | grep -A 5 "Allocated resources"

# Пометить узел для affinity
kubectl label node worker-1 disk=ssd

# Временно вывести узел из ротации (не принимает новые поды)
kubectl cordon worker-1

# Эвакуировать поды с узла (для обслуживания)
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
```

❗ **Ловушка собеса:** «Что произойдёт если worker node упадёт?» — Controller Manager обнаружит, что kubelet перестал слать heartbeat (через ~5 минут по умолчанию, `node-monitor-grace-period`). Node переходит в статус `NotReady`, поды помечаются как Terminating и пересоздаются на других узлах. Stateless приложения переживут это прозрачно, stateful — могут потерять данные без PersistentVolume.

**→ Уточняющий вопрос:** Что такое Node Affinity и Pod Anti-Affinity, и зачем они нужны в production?

**↳ Ответ:** Node Affinity — правила размещения Pod на нодах с определёнными характеристиками (label): например, только на нодах с SSD или в нужной availability zone. Pod Anti-Affinity — запрет размещать реплики одного Deployment на одной ноде или в одной зоне. Без Anti-Affinity все 3 реплики могут оказаться на одной ноде — если нода упадёт, сервис полностью недоступен. В production Anti-Affinity по зонам (`topologyKey: topology.kubernetes.io/zone`) — стандарт HA.

---

## 11. Что такое Service в Kubernetes?

Service — абстракция, предоставляющая стабильный сетевой endpoint (IP + DNS-имя) для доступа к группе Pods. Поды эфемерны и меняют IP при пересоздании — Service решает эту проблему через **selector**: он находит все поды с нужными labels и балансирует трафик между ними. Service сохраняет свой ClusterIP и DNS-имя на всё время существования.

Технически Service не является proxy-процессом. kube-proxy на каждом узле читает конфигурацию Service и программирует правила iptables (или IPVS). Когда пакет приходит на ClusterIP:port, ядро Linux перенаправляет его на один из EndPoints (IP:port одного из Pods). DNS-имя сервиса (`myapp.default.svc.cluster.local`) резолвится в его ClusterIP через CoreDNS.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  selector:
    app: myapp          # находит поды с этим label
    version: "1.0"
  ports:
  - name: http
    protocol: TCP
    port: 80            # порт Service
    targetPort: 8080    # порт контейнера
  type: ClusterIP
```

```bash
# Из другого пода в том же namespace можно обращаться по короткому имени:
# http://myapp-service/api/users
# Из другого namespace:
# http://myapp-service.production.svc.cluster.local/api/users

kubectl get endpoints myapp-service   # посмотреть список Pod IP
kubectl get svc -n production
```

❗ **Ловушка собеса:** «Service балансирует по round-robin?» — в режиме iptables выбор endpoint случаен (stateless random), в режиме IPVS можно настроить алгоритм (round-robin, least-connections). Также: если selector не совпадает ни с одним Pod — Service существует, но EndPoints пустой, трафик уходит в никуда. Это частая причина `Connection refused` при опечатке в label.

**→ Уточняющий вопрос:** Как Service узнаёт о появлении или исчезновении подов — как работает механизм Endpoints?

**↳ Ответ:** Endpoint Controller в control plane следит за Pods через API Server watch: при появлении Pod с label, совпадающим с selector Service — добавляет его IP:port в объект EndpointSlice. При удалении Pod или провале readiness probe — убирает. kube-proxy на каждой ноде тоже подписан на EndpointSlice и обновляет правила iptables/IPVS. Всё event-driven через watch — никакого polling.

---

## 12. Какие типы Service существуют (ClusterIP, NodePort, LoadBalancer)?

**ClusterIP** (по умолчанию) — внутренний IP, доступный только внутри кластера. Используется для межсервисного взаимодействия. Spring Boot сервис обращается к PostgreSQL через ClusterIP сервис `postgres-service:5432`.

**NodePort** — открывает порт (30000–32767) на каждом узле кластера. Внешний трафик приходит на `<NodeIP>:<NodePort>` → перенаправляется в Service → Pod. Используется для разработки/тестирования, в prod неудобен (нестандартные порты, нужно знать IP узла).

**LoadBalancer** — создаёт облачный балансировщик (AWS ALB, GCP Load Balancer) с внешним IP. Включает NodePort + автоматическую настройку облачного LB. Это стандарт для production в managed Kubernetes (EKS, GKE, AKS).

**ExternalName** — DNS alias для внешнего сервиса (не входящего в кластер). Полезно для обращения к внешним БД через K8s-имя.

```yaml
# LoadBalancer для production
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
  annotations:
    # AWS-специфичные аннотации
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 443
    targetPort: 8080
---
# NodePort для staging
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080   # явно задаём порт
```

❗ **Ловушка собеса:** LoadBalancer в bare-metal кластере (не в облаке) просто зависнет в статусе `pending` — облачного провайдера нет, некому создать LB. Для bare-metal используют **MetalLB** или Ingress Controller. В production обычно Ingress + один LoadBalancer Service для Ingress Controller экономичнее, чем LoadBalancer на каждый микросервис.

**→ Уточняющий вопрос:** Чем Ingress отличается от LoadBalancer Service, и почему Ingress предпочтительнее для множества микросервисов?

**↳ Ответ:** LoadBalancer Service создаёт отдельный облачный балансировщик (и отдельный внешний IP с биллингом) для каждого Service — 10 микросервисов = 10 LB = дорого. Ingress — один LoadBalancer для одного Ingress Controller, который маршрутизирует трафик по hostname и path к десяткам ClusterIP Services. Плюс Ingress даёт TLS-терминацию, rewrite URL, rate limiting в одном месте. Практически: LoadBalancer только для самого Ingress Controller, остальные сервисы — ClusterIP.

---

## 13. Что такое ReplicaSet?

ReplicaSet — контроллер Kubernetes, обеспечивающий поддержание заданного числа одинаковых Pod-реплик в любой момент. Если Pod падает или удаляется — ReplicaSet создаёт новый. Если лишние — удаляет. Целевое число реплик описывается полем `replicas`.

Технически ReplicaSet использует `selector` для идентификации «своих» Pods и `template` как шаблон для создания новых. Controller Manager через reconciliation loop непрерывно сверяет `len(current pods matching selector)` с `spec.replicas` и действует. Сам Pod не привязан к ReplicaSet жёстко — если вручную добавить Pod с совпадающим label, ReplicaSet его «усыновит» и уменьшит создание новых.

На практике ReplicaSet напрямую почти никогда не создаётся — вместо него используется **Deployment**, который управляет ReplicaSet и добавляет rolling updates, rollback историю и другие возможности. Deployment создаёт ReplicaSet, ReplicaSet управляет Pods.

```yaml
# ReplicaSet (обычно создаётся Deployment, не вручную)
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
```

```bash
kubectl get replicasets -n production
kubectl describe rs myapp-deployment-7d4f9b8c6d
```

❗ **Ловушка собеса:** «Зачем нужен Deployment если есть ReplicaSet?» — ReplicaSet не умеет rolling update. При изменении `image` он удалит все старые поды и создаст новые — это даунтайм. Deployment создаёт новый ReplicaSet с новой версией и постепенно переводит трафик, сохраняя историю для rollback. В 99% случаев используйте Deployment.

**→ Уточняющий вопрос:** Как Deployment управляет несколькими ReplicaSet во время rolling update?

**↳ Ответ:** При обновлении Deployment создаёт новый ReplicaSet с новым pod template (новым образом). Затем постепенно увеличивает `replicas` нового RS и уменьшает `replicas` старого, соблюдая `maxSurge` и `maxUnavailable`. Старые RS не удаляются — остаются с `replicas: 0` для возможного rollback (количество хранимых ревизий задаётся `revisionHistoryLimit`, дефолт 10). `kubectl rollout undo` просто возвращает реплики в предыдущий RS.

---

## 14. Как происходит масштабирование в Kubernetes?

**Горизонтальное масштабирование (scale out)** — изменение числа реплик Pod. Вручную: `kubectl scale deployment myapp --replicas=5` или изменить `spec.replicas` в YAML. Автоматически — через HorizontalPodAutoscaler (HPA). **Вертикальное масштабирование** — изменение ресурсов (CPU/memory) для Pod через VerticalPodAutoscaler (VPA) или ручное редактирование `resources`.

Технически при горизонтальном масштабировании Scheduler размещает новые Pods на существующих узлах, если хватает ресурсов. Если нет — Cluster Autoscaler (в облаке) добавляет новые узлы. Уменьшение реплик: Kubernetes выбирает «лишние» поды (по приоритету: поды без readiness, поды на перегруженных узлах) и gracefully завершает их (SIGTERM + `terminationGracePeriodSeconds`).

```bash
# Ручное масштабирование
kubectl scale deployment myapp --replicas=5 -n production

# Просмотр текущего числа реплик
kubectl get deployment myapp -n production

# HPA — автоматическое масштабирование
kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=10

# Статус HPA
kubectl get hpa -n production
kubectl describe hpa myapp -n production
```

Для Spring Boot важно убедиться, что приложение stateless — состояние сессий не хранится in-memory (используй Redis), нет локальных файлов. Только тогда масштабирование работает корректно.

❗ **Ловушка собеса:** «Масштабировать Spring Boot приложение легко, просто добавь реплики» — это правда только если приложение stateless. Если Spring Security хранит сессии в памяти (in-memory session), при переключении на другой Pod пользователь будет разлогинен. Решение: `spring-session` с Redis backend. Аналогично — кэш Caffeine/Ehcache не шарится между репликами, нужен Redis/Hazelcast.

**→ Уточняющий вопрос:** Как Spring Boot приложение должно быть спроектировано, чтобы корректно масштабироваться в Kubernetes?

**↳ Ответ:** Три правила stateless: не хранить сессии в памяти (используй Spring Session + Redis), не хранить файлы локально (используй S3/объектное хранилище), не держать локальный кэш Caffeine/Ehcache как источник истины (используй Redis/Hazelcast). Плюс: настрой `server.shutdown=graceful` и readiness probe — K8s не пустит трафик на не готовый Pod и даст время завершить текущие запросы при shutdown.

---

## 15. Что такое HorizontalPodAutoscaler?

HorizontalPodAutoscaler (HPA) — контроллер Kubernetes, автоматически изменяющий число реплик Deployment/ReplicaSet на основе метрик. По умолчанию работает с CPU/memory (через Metrics Server), расширяется на кастомные метрики через Custom Metrics API (Prometheus Adapter, KEDA).

Технически HPA раз в 15 секунд (настраивается) опрашивает Metrics Server, вычисляет желаемое число реплик по формуле: `desiredReplicas = ceil(currentReplicas * (currentMetric / desiredMetric))`. Есть защита от flapping: scale-down происходит медленнее (по умолчанию 5 минут охлаждения), scale-up быстрее. HPA уважает `minReplicas` и `maxReplicas`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # целевое использование CPU 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 5 мин охлаждения перед scale-down
    scaleUp:
      stabilizationWindowSeconds: 30
```

Для Spring Boot критично правильно задать `resources.requests.cpu` — HPA считает `currentMetric` как процент от `requests`. Если `requests: 250m` и приложение использует 200m CPU → 80% → HPA будет масштабировать.

❗ **Ловушка собеса:** HPA не работает без Metrics Server (не входит в базовую установку K8s). Без него HPA будет в статусе `unknown`. Также: если `resources.requests` не задан, HPA не может вычислить процент использования. Масштабирование по памяти ненадёжно для Java — JVM резервирует память заранее (heap + metaspace + off-heap), потребление не снижается при низкой нагрузке. Лучше масштабировать по CPU или кастомным метрикам (RPS, queue length через KEDA).

**→ Уточняющий вопрос:** Что такое KEDA и чем оно лучше стандартного HPA для event-driven приложений?

**↳ Ответ:** KEDA (Kubernetes Event-Driven Autoscaling) масштабирует Deployment на основе длины очереди сообщений (Kafka consumer lag, RabbitMQ queue depth) — стандартный HPA умеет только CPU/memory. Для Kafka Consumer: KEDA смотрит lag — 10 000 непрочитанных сообщений → поднимает реплики; lag = 0 → может масштабировать до 0 реплик (HPA не умеет). Это идеально для batch-обработчиков и event-driven микросервисов, где нагрузка нерегулярная.

---

## 16. В чём разница между ConfigMap и Secret?

**ConfigMap** хранит нечувствительную конфигурацию: URL баз данных, feature flags, конфиги приложения, свойства Spring Boot. Хранится в etcd открытым текстом. **Secret** хранит чувствительные данные: пароли, API-ключи, TLS-сертификаты, JWT-секреты. Хранится в etcd в base64-encoded виде.

Технически оба монтируются в Pod одинаково: как переменные окружения (`env`/`envFrom`) или как файлы (volume mount). Разница в доступе: к Secret применяются RBAC-правила, можно ограничить `get`/`list` на уровне namespace. В etcd при включённом **Encryption at Rest** Secrets шифруются AES-CBC/AES-GCM, ConfigMap — нет.

```yaml
# ConfigMap для Spring Boot
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  application.properties: |
    server.port=8080
    spring.datasource.url=jdbc:postgresql://postgres-service:5432/mydb
    logging.level.com.example=INFO
    cache.ttl=3600

---
# Secret для чувствительных данных
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
stringData:           # stringData автоматически base64-кодирует
  db-password: "supersecretpassword"
  jwt-secret: "my-jwt-signing-key"

---
# Deployment: монтируем оба
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        # ConfigMap как файл (Spring Boot подхватит application.properties)
        volumeMounts:
        - name: config
          mountPath: /app/config
        # Секреты как переменные окружения
        env:
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: jwt-secret
      volumes:
      - name: config
        configMap:
          name: myapp-config
```

❗ **Ловушка собеса:** **Secret НЕ шифруется по умолчанию** — только base64. `echo "supersecretpassword" | base64` — это кодирование, не шифрование. Любой, кто может читать etcd или делать `kubectl get secret -o yaml`, увидит данные. Шифрование at-rest надо явно настраивать через EncryptionConfiguration API Server. В production используют внешние vault: HashiCorp Vault + Vault Agent Injector, AWS Secrets Manager + External Secrets Operator, или Sealed Secrets.

**→ Уточняющий вопрос:** Как безопасно передать секреты в Kubernetes в production — какие инструменты используются вместо нативных Secrets?

**↳ Ответ:** Три production-подхода: 1) **External Secrets Operator** — синхронизирует секреты из AWS Secrets Manager/HashiCorp Vault в K8s Secrets автоматически, реальный секрет живёт во внешней системе; 2) **Sealed Secrets** — шифрует Secret публичным ключом кластера, зашифрованный файл хранится в Git (GitOps-friendly, никто не видит значение); 3) **Vault Agent Injector** — sidecar инжектирует секреты прямо в Pod как файлы, минуя etcd полностью.

---

## 17. Что такое liveness probe?

Liveness probe — механизм Kubernetes для проверки «жив ли контейнер» в смысле «корректно ли работает процесс». Если liveness probe падает заданное число раз подряд (`failureThreshold`), Kubernetes перезапускает контейнер. Предназначена для ситуаций: deadlock в потоках, бесконечный цикл, неотвечающий процесс — когда перезапуск реально поможет.

Технически kubelet выполняет probe по расписанию (каждые `periodSeconds`). Три типа: **HTTP GET** (ожидает 2xx/3xx ответ), **TCP Socket** (проверяет открытость порта), **Exec** (выполняет команду в контейнере, успех = exit code 0). Для Spring Boot стандарт — HTTP GET на `/actuator/health/liveness`.

```yaml
# Spring Boot Actuator автоматически предоставляет /actuator/health/liveness
# При добавлении зависимости spring-boot-starter-actuator

containers:
- name: myapp
  image: myapp:1.0
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    initialDelaySeconds: 60    # ждём 60с перед первой проверкой (время старта JVM)
    periodSeconds: 10           # проверять каждые 10с
    timeoutSeconds: 5           # таймаут ответа
    failureThreshold: 3         # 3 неудачи → перезапуск
    successThreshold: 1         # достаточно 1 успеха после восстановления
```

```yaml
# application.properties
management.health.livenessstate.enabled=true
management.health.readinessstate.enabled=true
management.endpoint.health.probes.enabled=true
management.endpoint.health.show-details=always
```

Spring Boot 2.3+ предоставляет отдельные endpoints `/actuator/health/liveness` и `/actuator/health/readiness` из коробки через `ApplicationAvailability`.

❗ **Ловушка собеса:** Неправильный `initialDelaySeconds` — классическая ошибка. Spring Boot с Hibernate, Flyway, большим контекстом может стартовать 30–60 секунд. Если `initialDelaySeconds: 15`, liveness сработает раньше старта, Kubernetes начнёт бесконечно перезапускать поды (CrashLoopBackOff). Используй `startupProbe` вместо `initialDelaySeconds` — он даёт время на старт, потом передаёт контроль liveness.

**→ Уточняющий вопрос:** Что такое startupProbe и чем он лучше `initialDelaySeconds` для медленно стартующих Spring Boot приложений?

**↳ Ответ:** `startupProbe` — отдельная probe только для фазы старта: пока она не прошла успешно, liveness и readiness не запускаются. Безопаснее чем `initialDelaySeconds` потому что адаптивна: `failureThreshold × periodSeconds` задаёт максимальное время ожидания, но если Spring Boot стартанул быстро — probe сразу передаёт управление liveness. `initialDelaySeconds` — жёсткая задержка: слишком маленькая убьёт медленный старт, слишком большая замедляет обнаружение краша.

---

## 18. Что такое readiness probe?

Readiness probe — механизм проверки «готов ли Pod принимать трафик». Если readiness probe падает, Pod исключается из endpoints Service (трафик на него не направляется), но контейнер НЕ перезапускается. Когда probe снова проходит — Pod возвращается в ротацию. Используется для: прогрева кэша при старте, временной недоступности зависимостей (БД перегружена), graceful shutdown.

Технически kube-proxy обновляет правила iptables на основе EndpointSlices. Когда Pod не проходит readiness, Endpoint Controller удаляет его из EndpointSlice, kube-proxy перестаёт направлять туда трафик. Это происходит быстрее, чем удаление Pod — тонкая, но важная деталь для zero-downtime деплоя.

```yaml
containers:
- name: myapp
  image: myapp:1.0
  readinessProbe:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    initialDelaySeconds: 20    # меньше чем у liveness — нужно попасть в Service быстрее
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 3
    successThreshold: 2        # 2 успеха нужно чтобы снова попасть в ротацию
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    initialDelaySeconds: 60
    periodSeconds: 10
    failureThreshold: 3
```

В Spring Boot `readiness` endpoint отражает `ReadinessState`. Можно программно менять состояние:
```java
@Autowired
private ApplicationEventPublisher eventPublisher;

// Когда приложение прогрелось (кэш загружен)
eventPublisher.publishEvent(
    new AvailabilityChangeEvent<>(this, ReadinessState.ACCEPTING_TRAFFIC)
);

// При проблемах с зависимостями
eventPublisher.publishEvent(
    new AvailabilityChangeEvent<>(this, ReadinessState.REFUSING_TRAFFIC)
);
```

❗ **Ловушка собеса:** Liveness ≠ Readiness — это разные смыслы, частая путаница на собесах. Простое правило: liveness = «нужно ли перезапустить?», readiness = «можно ли слать трафик?». Ошибка — использовать только liveness или вешать на liveness проверку БД. Если БД недоступна, liveness зафейлит → Pod перезапустится → снова не может достучаться до БД → бесконечный рестарт. Доступность внешних зависимостей — только в readiness.

**→ Уточняющий вопрос:** Как настроить readiness probe для graceful shutdown Spring Boot в Kubernetes при rolling update?

**↳ Ответ:** При `server.shutdown=graceful` Spring Boot автоматически переводит `ReadinessState` в `REFUSING_TRAFFIC` при получении SIGTERM — readiness probe начнёт падать и Pod исключится из Service endpoints. Добавь `lifecycle.preStop: exec: sleep 5` — kube-proxy обновляет iptables с задержкой, sleep даёт ему время убрать Pod из ротации до того, как придёт SIGTERM. Порядок: `preStop sleep` → SIGTERM → Spring graceful shutdown → Pod terminated.

---

## 19. Зачем нужны health checks?

Health checks — это механизм самодиагностики приложения, без которого Kubernetes «слеп»: он видит что процесс запущен, но не знает работает ли он корректно. Без health checks K8s отправляет трафик на Pod, который стартует (ещё не готов), завис в deadlock, или потерял соединение с БД — результат: ошибки для пользователей.

Технически K8s использует три типа probes: **startup** (даёт время на старт, блокирует liveness/readiness), **liveness** (триггерит перезапуск при заморозке), **readiness** (управляет включением в Service). Spring Boot Actuator предоставляет `/actuator/health` с детальными проверками подсистем — DB, Redis, Kafka, Disk space — каждую можно включить/выключить.

```yaml
# application.properties: подключаем health indicators
management.endpoint.health.enabled=true
management.endpoint.health.probes.enabled=true
management.health.db.enabled=true
management.health.redis.enabled=true
management.health.kafka.enabled=true
management.endpoints.web.exposure.include=health,info,metrics,prometheus

# Для production — скрываем детали от анонимов
management.endpoint.health.show-details=when_authorized
management.endpoint.health.show-components=always
```

```java
// Кастомный HealthIndicator для критичной зависимости
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    
    @Autowired
    private ExternalApiClient client;
    
    @Override
    public Health health() {
        try {
            boolean available = client.ping();
            return available
                ? Health.up().withDetail("api", "reachable").build()
                : Health.down().withDetail("api", "unreachable").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

❗ **Ловушка собеса:** «Подключим проверку БД к liveness» — смертельная ошибка при массовых сбоях. Представь: PostgreSQL перегружен, 10 реплик Spring Boot одновременно зафейлили liveness → K8s перезапускает все 10 → они снова все стучатся в БД при старте → ещё большая нагрузка. Cascade failure. Правило: liveness проверяет только внутреннее состояние JVM/процесса, readiness — внешние зависимости.

**→ Уточняющий вопрос:** Как Spring Boot Actuator разделяет health indicators между liveness и readiness группами?

**↳ Ответ:** Spring Boot 2.3+ создаёт две группы автоматически: `liveness` содержит только `LivenessStateHealthIndicator`, `readiness` — `ReadinessStateHealthIndicator` + все health indicators зависимостей (db, redis, kafka). Кастомный HealthIndicator попадает в обе группы по умолчанию; явно задаётся через `management.endpoint.health.group.readiness.include=db,redis,myCheck`. Правило: в liveness — ничего про внешние зависимости, иначе получишь cascade restart при сбое БД.

---

## 20. Что такое Ingress в Kubernetes?

Ingress — ресурс Kubernetes, описывающий правила маршрутизации внешнего HTTP(S)-трафика внутрь кластера: по hostname (`api.example.com`), по path (`/api/v1/*`), с TLS-терминацией. Ingress сам по себе — только декларативный манифест, реальную маршрутизацию выполняет **Ingress Controller** (nginx, Traefik, AWS ALB Ingress Controller, Istio) — отдельный Pod, читающий Ingress-ресурсы через Kubernetes API.

Технически схема: внешний трафик → LoadBalancer Service → Ingress Controller Pod → ClusterIP Service → application Pod. Ingress Controller следит за изменениями Ingress-ресурсов (через API watch) и динамически обновляет конфигурацию прокси (nginx.conf или аналог). TLS-сертификаты хранятся в Secret и автоматически монтируются в Ingress Controller.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"   # автовыпуск сертификата
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-secret    # cert-manager автоматически заполнит
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

Преимущество перед LoadBalancer Service: один LoadBalancer (дорогой в облаке) → один Ingress Controller → маршрутизирует между десятками микросервисов по hostname/path.

❗ **Ловушка собеса:** Ingress Controller ≠ Ingress. Без установленного Ingress Controller манифест Ingress просто лежит в etcd и ничего не делает. Также: `path: /api/v1` с `pathType: Prefix` и `pathType: Exact` ведут себя по-разному. `Exact` требует точного совпадения, `Prefix` — совпадение префикса пути. Частая ошибка: `pathType: Prefix` с `/api/v1` перехватит `/api/v1`, `/api/v1/users`, `/api/v10` (!)  — нужно учитывать.

**→ Уточняющий вопрос:** Что такое cert-manager и как он автоматизирует выпуск Let's Encrypt сертификатов для Ingress?

**↳ Ответ:** cert-manager — K8s-оператор, следящий за Ingress с аннотацией `cert-manager.io/cluster-issuer`. При обнаружении автоматически запрашивает сертификат через ACME (Let's Encrypt), проходит HTTP-01 или DNS-01 challenge, сохраняет в Secret указанный в `spec.tls.secretName` и отслеживает renewal (обновляет за 30 дней до истечения). Без cert-manager всё это делается вручную каждые 90 дней — с ним это полностью автоматизировано.

---

## 21. Что такое namespace?

Namespace — механизм виртуального разделения ресурсов (Pods, Services, Deployments, ConfigMaps) внутри одного физического кластера Kubernetes. Имена ресурсов уникальны в пределах namespace, но могут совпадать в разных. Используется для: разделения окружений (dev/staging/prod), разграничения команд, применения RBAC-политик и Resource Quotas.

Технически namespace — это просто метаданные (метка) на ресурсах, хранящихся в etcd. Сетевой изоляции между namespace по умолчанию нет — Pod из `namespace-a` может обращаться к Service в `namespace-b` по полному DNS-имени: `service-name.namespace-b.svc.cluster.local`. Изоляцию сети обеспечивают **NetworkPolicy**.

```bash
# Создание namespace
kubectl create namespace production
kubectl create namespace staging

# Применение ресурса в конкретный namespace
kubectl apply -f deployment.yaml -n production

# По умолчанию kubectl работает с namespace 'default'
# Сменить дефолтный namespace:
kubectl config set-context --current --namespace=production

# Просмотр ресурсов
kubectl get all -n production
kubectl get pods --all-namespaces
```

```yaml
# ResourceQuota: ограничиваем потребление ресурсов в namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    count/pods: "50"

---
# LimitRange: дефолтные лимиты для новых контейнеров
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 250m
    type: Container
```

❗ **Ловушка собеса:** Namespace ≠ сетевая изоляция. По умолчанию Pod в `dev` может достучаться до `prod` базы данных по DNS. Для настоящей изоляции нужны NetworkPolicy. Также: системные namespace (`kube-system`, `kube-public`, `kube-node-lease`) удалять нельзя — это убьёт кластер. При удалении пользовательского namespace все ресурсы внутри удаляются каскадно — будь осторожен.

**→ Уточняющий вопрос:** Как настроить NetworkPolicy для запрета межnamespace трафика при мультитенантном кластере?

**↳ Ответ:** По умолчанию K8s разрешает весь трафик между namespace. Создай `default-deny` NetworkPolicy в каждом namespace с пустым `ingress: []` — запрет всего входящего. Затем явно разреши нужное через `namespaceSelector` с label нужного namespace. Важно: NetworkPolicy работает только если CNI-плагин её поддерживает — Calico и Cilium да, flannel нет. Для строгой изоляции tenant'ов Cilium с `CiliumNetworkPolicy` даёт более гибкий контроль.

---

## 22. Как организовать rolling update в Kubernetes?

Rolling update — стратегия обновления Deployment, при которой новые поды создаются постепенно, а старые гасятся только после того, как новые прошли readiness probe. В результате сервис доступен на протяжении всего обновления без даунтайма.

Технически Deployment создаёт новый ReplicaSet с новым образом и начинает увеличивать его реплики, одновременно уменьшая старый. Скорость управляется параметрами: `maxSurge` — сколько новых Pod сверх желаемых можно создать (`25%` или абсолютное число), `maxUnavailable` — сколько Pod из желаемых могут быть недоступны. При `maxUnavailable: 0` — zero-downtime гарантирован (всегда N доступных Pod).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # макс. 5 подов в процессе обновления
      maxUnavailable: 0     # всегда минимум 4 пода доступны
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:2.0    # меняем версию → начнётся rolling update
        # Критично для rolling update: readiness probe!
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 5
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 5"]  # даём время kube-proxy обновить правила
      terminationGracePeriodSeconds: 60
```

```bash
# Обновить образ (триггер rolling update)
kubectl set image deployment/myapp myapp=myapp:2.0 -n production

# Следить за прогрессом
kubectl rollout status deployment/myapp -n production

# История обновлений
kubectl rollout history deployment/myapp -n production

# Откатиться на предыдущую версию
kubectl rollout undo deployment/myapp -n production

# Откатиться на конкретную ревизию
kubectl rollout undo deployment/myapp --to-revision=2 -n production
```

❗ **Ловушка собеса:** Rolling update без readiness probe — это не zero-downtime. Kubernetes не знает что новый Pod готов, он помечает его Ready сразу как контейнер стартовал. Трафик пойдёт на Spring Boot который ещё инициализирует Hibernate/Flyway → 500 ошибки. Также: `preStop: sleep 5` — важный паттерн. kube-proxy обновляет iptables с небольшой задержкой, без sleep в preStop трафик продолжает идти на Pod ещё несколько секунд после начала завершения → connection reset.

**→ Уточняющий вопрос:** Чем blue-green деплой отличается от rolling update и когда он предпочтительнее?

**↳ Ответ:** Rolling update постепенно заменяет поды — в какой-то момент в production одновременно работают и старая, и новая версия (проблема при несовместимых изменениях API или схемы БД). Blue-green поднимает полноценный второй stack (green) рядом со старым (blue) и переключает трафик мгновенно через Ingress/Service selector — при проблеме откат за секунды. Предпочтительнее при: несовместимых изменениях схемы, критичных деплоях, когда нужен мгновенный rollback без состояния смешанных версий.

---

## 23. Что такое StatefulSet и когда его использовать?

StatefulSet — контроллер Kubernetes для приложений с постоянным состоянием (stateful). В отличие от Deployment, он гарантирует: **постоянную идентичность Pod** (имена pod-0, pod-1, pod-2 сохраняются при пересоздании), **стабильные сетевые имена** (через Headless Service), **упорядоченный запуск/остановку** (pod-0 → pod-1 → pod-2), **индивидуальные PersistentVolumeClaim** для каждого Pod.

Технически StatefulSet использует Headless Service (`clusterIP: None`): каждый Pod получает DNS-запись вида `pod-0.service-name.namespace.svc.cluster.local`. Поды StatefulSet запускаются последовательно — следующий стартует только после того, как предыдущий прошёл readiness. Остановка — в обратном порядке. PVC не удаляются при удалении StatefulSet — данные сохраняются.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless   # Headless Service для DNS
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:           # отдельный PVC для каждого Pod
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None           # Headless: нет виртуального IP, прямой DNS к подам
  selector:
    app: postgres
  ports:
  - port: 5432
```

Когда использовать: PostgreSQL, MySQL, MongoDB кластеры, Apache Kafka, Apache Zookeeper, Elasticsearch — всё где нужна персистентность данных или упорядоченная топология кластера (leader-follower).

❗ **Ловушка собеса:** «StatefulSet для всего stateful» — нет. Если управляемый облачный сервис (RDS, Cloud SQL) доступен — используй его, не разворачивай БД в K8s без острой необходимости. Операционная сложность резко возрастает: бэкапы, репликация, failover — всё вручную. Для Java-разработчика важно: Spring Boot сервис сам должен быть stateless (Deployment), а StatefulSet — для инфраструктуры (БД, брокеры).

**→ Уточняющий вопрос:** Как организовать бэкап PersistentVolume в Kubernetes и что такое VolumeSnapshot?

**↳ Ответ:** VolumeSnapshot — K8s API для снапшотов PV через CSI-драйверы (AWS EBS, GCP PD, Azure Disk): создаёшь объект `VolumeSnapshot`, CSI-драйвер делает снапшот на уровне облака, восстанавливаешь через `dataSource: volumeSnapshot` в новом PVC. Для комплексных бэкапов — Velero: бэкапит и данные PV (через restic/Kopia), и K8s объекты вместе, что позволяет восстановить весь namespace. Без бэкапов PV — StatefulSet в prod это риск необратимой потери данных.

---

## 24. Как мониторить приложения в Kubernetes?

Мониторинг в K8s строится из трёх компонентов: **метрики** (числовые временные ряды — CPU, RPS, latency), **логи** (текстовые события), **трейсы** (распределённые цепочки запросов). Стандартный стек: **Prometheus** (сбор метрик) + **Grafana** (визуализация) + **Alertmanager** (алерты); **Loki** или **Elasticsearch** (логи); **Jaeger** или **Tempo** (трейсы). Для Java/Spring Boot добавляется **Micrometer** как instrumentation библиотека.

Технически Prometheus работает по pull-модели: сам ходит на `/actuator/prometheus` endpoint каждого Pod и собирает метрики. ServiceMonitor (Prometheus Operator CRD) описывает какие сервисы мониторить. kube-state-metrics экспортирует метрики о состоянии K8s-объектов (pod restarts, deployment replicas, PVC status).

```yaml
# application.properties: включаем Prometheus endpoint
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.prometheus.enabled=true
management.metrics.export.prometheus.enabled=true

# Кастомные метрики через Micrometer
```

```java
@RestController
public class OrderController {
    
    private final Counter orderCounter;
    private final Timer orderTimer;
    
    public OrderController(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created")
            .tag("status", "success")
            .description("Total orders created")
            .register(registry);
        this.orderTimer = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .register(registry);
    }
    
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest request) {
        return orderTimer.record(() -> {
            Order order = orderService.create(request);
            orderCounter.increment();
            return order;
        });
    }
}
```

```yaml
# ServiceMonitor для Prometheus Operator
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: http
    path: /actuator/prometheus
    interval: 15s
---
# PrometheusRule: алерт на высокий error rate
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
spec:
  groups:
  - name: myapp
    rules:
    - alert: HighErrorRate
      expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.1
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on {{ $labels.instance }}"
```

```bash
# Базовый мониторинг через kubectl
kubectl top pods -n production              # CPU/memory потребление
kubectl top nodes                           # ресурсы узлов
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl logs -f deployment/myapp -n production --all-containers
```

❗ **Ловушка собеса:** «Просто смотрим логи через kubectl logs» — это не мониторинг. При рестарте Pod логи теряются (если нет log aggregation). Правильный ответ: централизованный сбор логов (Fluentd/Fluent Bit → Elasticsearch/Loki), структурированные логи в JSON (`logstash-logback-encoder` для Spring Boot), корреляция логов с трейсами через `traceId` (Spring Cloud Sleuth / Micrometer Tracing + Zipkin/Jaeger). На собесе ценится понимание полного observability стека, а не только «смотрю в Grafana».

**→ Уточняющий вопрос:** Как настроить распределённую трассировку (distributed tracing) между микросервисами в Spring Boot с Kubernetes?

**↳ Ответ:** Подключи Micrometer Tracing + Brave/OpenTelemetry: автоматически создаётся `traceId`, который пробрасывается через HTTP-заголовки (`traceparent`) между сервисами. Трейсы экспортируются в Jaeger/Zipkin/Tempo. В K8s: деплой Jaeger через Helm, в `application.properties`: `management.tracing.sampling.probability=1.0` и endpoint коллектора. Logback автоматически включает `traceId` в каждое лог-сообщение при наличии Micrometer Tracing в classpath — это связывает логи и трейсы.
