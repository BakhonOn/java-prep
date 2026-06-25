# 🎯 Подготовка к собеседованию Middle Java Developer

516 вопросов · 20 тем · готовая система обучения через Claude Code.

## Что внутри

```
java-prep/
├── CLAUDE.md              ← инструкции для Claude Code (режимы: справочник/интервьюер/код/инфра)
├── docker-compose.yml     ← Postgres + Kafka для практических тем
├── topics/                ← 20 папок по темам
│   └── XX-тема/
│       ├── questions.md   ← вопросы + место для ответов
│       └── NOTES.md       ← твои заметки
├── anki/
│   └── java_interview_cards.tsv  ← импорт в Anki (516 карточек)
└── playground/            ← сюда Claude Code будет писать запускаемые примеры
```

## Шаг 1. Установка Claude Code

> Нужна подписка Claude **Pro / Max / Team / Enterprise** или оплата через **Console (API)**.
> Бесплатный план Claude.ai доступа к Claude Code НЕ даёт.

**macOS / Linux:**
```bash
curl -fsSL https://claude.com/install.sh | bash    # либо: npm install -g @anthropic-ai/claude-code
```

**Windows:** нужен WSL2 (Ubuntu), внутри него — команда выше.

Требования: macOS / Linux / Windows 10+ (через WSL2), 4 ГБ RAM (рекомендуется 8).

После установки:
```bash
cd java-prep
claude          # запустится, попросит авторизоваться в браузере
```

## Шаг 2. Как учиться

Claude Code автоматически читает `CLAUDE.md`, поэтому достаточно команд на русском:

| Что сказать | Что произойдёт |
|---|---|
| `объясни тему 09 (многопоточность)` | развёрнутый ответ уровня middle + подводные камни |
| `проведи собес по Stream API` | режим интервьюера: вопрос → твой ответ → разбор |
| `покажи на коде разницу thenApply и thenCompose` | напишет, запустит .java, покажет вывод |
| `подними docker и покажи грязное чтение` | поднимет Postgres, продемонстрирует вживую |
| `заполни ответы в topics/01-databases-sql` | сгенерирует ответы прямо в questions.md |

## Шаг 3. Флешкарты (Anki)

1. Установи [Anki](https://apps.ankiweb.net/).
2. Сначала сгенерируй ответы (попроси Claude Code заполнить колонку ответов в TSV, либо заполни сам).
3. File → Import → выбери `anki/java_interview_cards.tsv`, разделитель Tab, поля: Front / Back / Tags.

## Карта тем по способу изучения

🧪 **Практика на коде:** Records/Generics, CompletableFuture, Иммутабельность, String, HashMap, Многопоточность, Stream API, Исключения, Коллекции, Паттерны
🐳 **Инфраструктура (Docker):** Hibernate/JPA, Транзакции, Kafka, БД/SQL
📇 **Зубрёжка:** SOLID, Микросервисы, Docker/K8s, REST/HTTP, Spring, GC

## ⚠️ Безопасность
Claude Code имеет прямой доступ к файловой системе. Этот учебный проект — изолированная папка, рисков нет. Но не запускай агента в автономном режиме там, где лежит что-то ценное без бэкапа.
