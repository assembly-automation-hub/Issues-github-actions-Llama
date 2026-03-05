# Labels Documentation

Полное руководство по использованию лейблов в AI Auto-Issue Generator.

---

## Как это работает

Когда ты делаешь **push** или открываешь/обновляешь **Pull Request**, GitHub Actions запускает `process_event.py`. Скрипт читает лейблы и решает, в какой роли выступить AI-анализатор — аудитор безопасности, архитектор, QA-инженер и т.д.

```
Твоё действие (push / PR)
        ↓
GitHub Actions запускает process_event.py
        ↓
Скрипт читает лейблы
        ↓
Выбирает роль и промпт для модели
        ↓
Создаёт Issue с анализом + комментарий в PR
```

Каждый Issue содержит:
- **Severity** в заголовке (`[CRITICAL]`, `[HIGH]`, `[MEDIUM]`, `[LOW]`)
- **Секцию Problem** — что именно не так и где
- **Code Reference** — конкретный проблемный код
- **Suggested Fix** — как исправить
- **Permalink** — прямая ссылка на файл и строку в GitHub

---

## Как активировать лейблы

### В Pull Request
Добавь лейбл через GitHub UI — правая панель PR → **Labels** → выбери нужный.
Workflow сработает на событие `labeled`.

### В commit message (push)
Вставь лейбл в квадратных скобках в текст коммита:

```
git commit -m "refactor auth module [security]"
git commit -m "add stripe integration [deps][review]"
git commit -m "update service layer [arch]"
```

Можно указывать **несколько лейблов** — сработает первый по приоритету (сверху вниз по списку ниже).

---

## Все лейблы

### 🔒 `[security]` / `[sec]` / `[audit]`

**Роль модели:** Strict Security Auditor

**Что анализирует:**
- OWASP Top 10 уязвимости
- SQL/NoSQL инъекции
- Небезопасная работа с токенами, паролями, секретами
- Открытые endpoints без авторизации
- Небезопасная десериализация
- Hardcoded credentials в коде

**Пример Issue:**
```
[HIGH] SQL Injection vulnerability in user search endpoint

## Problem
В файле `api/users.py`, строка 47, входные данные передаются
напрямую в SQL-запрос без параметризации.

## Code Reference
query = f"SELECT * FROM users WHERE name = '{user_input}'"

## Suggested Fix
Использовать параметризованные запросы:
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))

## Permalink
https://github.com/org/repo/blob/abc1234/api/users.py#L47
```

---

### 👁 `[review]` / `[refactor]` / `[code-review]`

**Роль модели:** Strict Code Reviewer

**Что анализирует:**
- Нарушения SOLID-принципов
- Нарушения DRY (дублирование кода)
- Слишком длинные функции / God-классы
- Плохое именование переменных
- Отсутствие обработки ошибок
- Магические числа и строки

**Пример Issue:**
```
[MEDIUM] Single Responsibility Principle violation in OrderService

## Problem
Класс `OrderService` в `services/order.py` (строки 12–180)
одновременно обрабатывает бизнес-логику, отправляет email
и пишет в БД — три разные ответственности.

## Suggested Fix
Разделить на OrderService, OrderNotifier, OrderRepository.
```

---

### 🧪 `[qa]` / `[test]` / `[testing]`

**Роль модели:** QA Engineer

**Что анализирует:**
- Непокрытые edge cases
- Отсутствующие unit/integration тесты
- Функции без тестов
- Некорректная обработка null/None/пустых значений
- Граничные условия (пустой массив, отрицательные числа, очень длинные строки)

**Пример Issue:**
```
[MEDIUM] Missing edge case tests for payment processing

## Problem
Функция `calculate_total()` в `utils/cart.py` не покрыта тестами
для случаев: пустая корзина, количество товара = 0, отрицательная цена.

## Suggested Fix
def test_calculate_total_empty_cart():
    assert calculate_total([]) == 0

def test_calculate_total_zero_quantity():
    ...
```

---

### ⚡ `[perf]` / `[performance]` / `[optimize]`

**Роль модели:** Performance Expert

**Что анализирует:**
- O(n²) и выше алгоритмическая сложность
- N+1 запросы к БД
- Отсутствие кэширования там, где оно нужно
- Загрузка лишних данных (SELECT * вместо конкретных полей)
- Блокирующие операции в async коде
- Утечки памяти

**Пример Issue:**
```
[HIGH] N+1 database query problem in product listing

## Problem
В `views/products.py`, строка 34, для каждого продукта
в цикле выполняется отдельный запрос к таблице categories.
При 1000 продуктах = 1001 запрос к БД.

## Suggested Fix
Использовать select_related() или prefetch_related():
Product.objects.prefetch_related('category').all()
```

---

### 📦 `[deps]` / `[dependencies]`

**Роль модели:** Security & Dependency Auditor

**Что анализирует:**
- Известные CVE-уязвимости в новых зависимостях
- Совместимость лицензий (MIT / Apache / GPL — GPL может быть проблемой)
- Активность поддержки пакета (последний коммит, количество issues)
- Размер добавляемого пакета и его влияние на бандл
- Транзитивные зависимости с рисками
- Дублирующие зависимости (уже есть аналог в проекте)

**Пример Issue:**
```
[HIGH] Dependency risk: lodash@3.10.1 has known prototype pollution CVE

## Problem
В `requirements.txt` добавлен `some-lib==1.2.0`, который тянет
lodash@3.10.1 — уязвимость CVE-2019-10744 (prototype pollution).

## Suggested Fix
Обновить до lodash@4.17.21 или заменить на нативные методы JS.
```

---

### 🏛 `[arch]` / `[architecture]`

**Роль модели:** Software Architect

**Что анализирует:**
- Нарушение разделения слоёв (например, бизнес-логика в контроллере)
- Tight coupling между модулями
- Нарушение Dependency Inversion (зависимость от конкретики, не абстракции)
- Anti-patterns: God Object, Spaghetti Logic, Shotgun Surgery
- Магические числа и глобальные состояния
- Неправильное место для кода (утилита в сервисе, модель в контроллере)

**Пример Issue:**
```
[MEDIUM] Business logic leaking into controller layer

## Problem
В `controllers/checkout.py`, строки 67–102, содержится расчёт
скидок и налогов — это бизнес-логика, которая должна быть
в `services/pricing.py`.

## Suggested Fix
Вынести в PricingService.calculate_final_price(cart, user)
и вызывать из контроллера одной строкой.
```

---

### 📋 `[pm]` / `[release]` / `[product]`

**Роль модели:** Product Manager

**Что анализирует:**
- Описывает изменения с точки зрения пользователя, не кода
- Генерирует Release Notes
- Указывает что изменилось, что улучшилось, что сломалось (breaking changes)
- Пишет понятно для нетехнической аудитории

**Пример Issue:**
```
[LOW] Release Notes: User authentication improvements

## What's New
Пользователи теперь могут входить через Google OAuth.
Время сессии увеличено с 1 часа до 24 часов.

## Improvements
Страница логина загружается на 40% быстрее.

## Breaking Changes
Старые API-токены формата v1 больше не поддерживаются.
Необходимо перевыпустить токены через /api/v2/auth.
```

---

### 📝 Без лейбла (default)

**Роль модели:** General Analyst

**Что анализирует:**
- Стандартная документация изменений
- Общее описание что было изменено и зачем
- Используется когда ни один лейбл не совпал

---

## Severity уровни

Каждый Issue получает автоматический severity на основе анализа:

| Уровень | Значение | Примеры |
|---------|----------|---------|
| `[CRITICAL]` | Требует немедленного исправления | RCE, утечка данных, падение прода |
| `[HIGH]` | Исправить в текущем спринте | SQL-инъекция, N+1 на главной странице |
| `[MEDIUM]` | Запланировать исправление | Нарушение SOLID, отсутствие тестов |
| `[LOW]` | Технический долг | Именование, стиль кода, минорный рефакторинг |

Severity автоматически добавляется как GitHub лейбл: `severity: critical`, `severity: high` и т.д.

---

## Защита от дублей

Скрипт автоматически:

1. **Не создаёт** новый Issue если открытый Issue для этого же коммита/PR уже существует
2. **Не переоткрывает** Issue если похожий Issue уже был найден и закрыт
3. **Пропускает merge-коммиты** — коммит с 2+ родителями (автоматический при merge PR) игнорируется

---

## Комментарий в PR

При анализе Pull Request, помимо Issue, скрипт автоматически оставляет **краткий комментарий прямо в PR**:

```
🤖 AI Analysis Summary

Обнаружена потенциальная SQL-инъекция в модуле аутентификации.
Входные данные не санируются перед передачей в запрос.
Рекомендуется использовать параметризованные запросы.

Severity: HIGH

📋 Full details: #42
```

---

## Быстрый справочник

| Лейбл | Псевдонимы | Роль |
|-------|-----------|------|
| `security` | `sec`, `audit` | Security Auditor (OWASP) |
| `review` | `refactor`, `code-review` | Code Reviewer (SOLID/DRY) |
| `qa` | `test`, `testing` | QA Engineer |
| `perf` | `performance`, `optimize` | Performance Expert |
| `deps` | `dependencies` | Dependency Auditor |
| `arch` | `architecture` | Software Architect |
| `pm` | `release`, `product` | Product Manager |
| _(none)_ | — | General Documentation |
