# PYTHON_СТИЛЬ.md — Стандарты написания Python-кода
> Цель — воспроизвести профессиональный, читаемый, поддерживаемый код.
> Размер файла оптимизирован (<5K токенов) — компенсация Lost in Middle.

---

## 1. ЭТАЛОННЫЕ ПАТТЕРНЫ

### Паттерн 1 — Правильная функция
```python
# ✅ ХОРОШО
def calculate_discount(
    price: float,
    discount_percent: float,
    min_price: float = 0.0,
) -> float:
    """Calculate discounted price with a minimum floor.

    Args:
        price: Original price in rubles.
        discount_percent: Discount as a percentage (0–100).
        min_price: Minimum allowed price after discount.

    Returns:
        Discounted price, never below min_price.

    Raises:
        ValueError: If discount_percent is outside 0–100 range.
    """
    if not 0 <= discount_percent <= 100:
        raise ValueError(
            f"discount_percent must be 0–100, got {discount_percent}"
        )
    discounted = price * (1 - discount_percent / 100)
    return max(discounted, min_price)
```

```python
# ❌ ПЛОХО
def calc(p, d, m=0):
    return max(p * (1 - d / 100), m)  # нет типов, нет docstring, нет валидации
```

---

### Паттерн 2 — Правильный класс
```python
# ✅ ХОРОШО
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class Order:
    """Represents a customer order.

    Attributes:
        order_id: Unique order identifier.
        items: List of (product_id, quantity) tuples.
        created_at: Order creation timestamp.
        is_paid: Payment status flag.
    """

    order_id: str
    items: list[tuple[str, int]] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.utcnow)
    is_paid: bool = False

    def total_items(self) -> int:
        """Return total number of items across all products."""
        return sum(qty for _, qty in self.items)
```

```python
# ❌ ПЛОХО
class Order:
    def __init__(self, id, items=[]):  # мутабельный дефолт!
        self.id = id
        self.items = items
```

---

### Паттерн 3 — Работа с файлами и исключениями
```python
# ✅ ХОРОШО
import logging
from pathlib import Path

logger = logging.getLogger(__name__)


def load_config(config_path: Path) -> dict:
    """Load JSON configuration from file.

    Args:
        config_path: Path to the JSON config file.

    Returns:
        Parsed configuration as dictionary.

    Raises:
        FileNotFoundError: If config file does not exist.
        json.JSONDecodeError: If file content is not valid JSON.
    """
    import json

    logger.info("Loading config from %s", config_path)
    with config_path.open(encoding="utf-8") as f:
        return json.load(f)
```

```python
# ❌ ПЛОХО
def load_config(path):
    try:
        return json.load(open(path))  # файл не закрывается!
    except:                            # голый except!
        return {}
```

---

### Паттерн 4 — Работа с БД (безопасно)
```python
# ✅ ХОРОШО
def get_user_by_email(conn: sqlite3.Connection, email: str) -> dict | None:
    """Fetch user record by email address.

    Args:
        conn: Active database connection.
        email: User email to search for.

    Returns:
        User record as dict, or None if not found.
    """
    cursor = conn.execute(
        "SELECT id, name, email FROM users WHERE email = ?",
        (email,),  # параметризованный запрос — защита от SQL injection
    )
    row = cursor.fetchone()
    if row is None:
        return None
    return {"id": row[0], "name": row[1], "email": row[2]}
```

```python
# ❌ ПЛОХО
def get_user(email):
    query = f"SELECT * FROM users WHERE email = '{email}'"  # SQL INJECTION!
    return db.execute(query).fetchone()
```

---

### Паттерн 5 — Конфигурация и секреты
```python
# ✅ ХОРОШО
import os
from dataclasses import dataclass


@dataclass(frozen=True)
class Config:
    """Application configuration loaded from environment."""

    db_url: str
    api_key: str
    debug: bool
    max_retries: int

    @classmethod
    def from_env(cls) -> "Config":
        """Create Config instance from environment variables.

        Raises:
            KeyError: If required env variable is not set.
        """
        return cls(
            db_url=os.environ["DATABASE_URL"],
            api_key=os.environ["API_KEY"],
            debug=os.environ.get("DEBUG", "false").lower() == "true",
            max_retries=int(os.environ.get("MAX_RETRIES", "3")),
        )


CONFIG = Config.from_env()
```

```python
# ❌ ПЛОХО
API_KEY = "sk-abc123secret"      # хардкод секрета!
DB_URL = "postgresql://admin:password@localhost/prod"  # хардкод!
```

---

## 2. ИМЕНОВАНИЕ

### Правила

| Тип | Стиль | Примеры |
|-----|-------|---------|
| Переменные, функции, методы | `snake_case` | `user_name`, `calculate_tax()` |
| Классы | `PascalCase` | `UserAccount`, `OrderProcessor` |
| Константы модульного уровня | `UPPER_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Приватные атрибуты | `_single_underscore` | `_cache`, `_validate()` |
| «Магические» методы | `__double_underscore__` | `__init__`, `__str__` |
| Дженерики / TypeVar | `PascalCase` короткие | `T`, `TKey`, `TValue` |

### Запрещённые имена

| Плохо | Почему | Хорошо |
|-------|--------|--------|
| `l`, `O`, `I` | Путаются с `1`, `0` | `line`, `output`, `index` |
| `data`, `info`, `result` в одиночку | Не несут смысла | `user_data`, `order_info` |
| `tmp`, `temp` в финальном коде | Указывает на незаконченность | Переименовать по смыслу |
| `flag`, `check`, `do_stuff` | Абстрактны | `is_valid`, `has_permission` |
| Транслит (`imya`, `schetchik`) | Нечитаемо | `name`, `counter` |

### Правило булевых переменных
```python
# ✅ ХОРОШО — начинается с is_, has_, can_, should_
is_active: bool
has_permission: bool
can_edit: bool
should_retry: bool

# ❌ ПЛОХО
active: bool
permission: bool
edit: bool
```

---

## 3. TYPE HINTS — ОБЯЗАТЕЛЬНЫЕ ПРАВИЛА

### Базовые правила

```python
# ✅ Python 3.10+ синтаксис (использовать всегда)
def process(
    items: list[str],           # не List[str]
    mapping: dict[str, int],    # не Dict[str, int]
    value: int | None = None,   # не Optional[int]
) -> tuple[bool, str]:          # не Tuple[bool, str]
    ...
```

### Обязательные аннотации

```python
# ✅ ВСЕГДА аннотировать:
# 1. Все аргументы функций и методов
# 2. Возвращаемое значение (даже None)
# 3. Атрибуты классов
# 4. Переменные, чей тип неочевиден

def send_email(
    to: str,
    subject: str,
    body: str,
    attachments: list[Path] | None = None,
    cc: list[str] | None = None,
) -> bool:  # True если отправлено успешно
    ...
```

### Кастомные типы для читаемости
```python
from typing import TypeAlias

UserId: TypeAlias = int
ProductSku: TypeAlias = str
PriceRub: TypeAlias = float

def get_price(user_id: UserId, sku: ProductSku) -> PriceRub:
    ...
```

---

## 4. DOCSTRINGS — ШАБЛОНЫ

### Формат: Google Style (обязательный)

```python
def function_name(arg1: type, arg2: type) -> return_type:
    """One-line summary (imperative mood, no period).

    Extended description if needed. Explain WHY, not WHAT.
    The code itself shows what — the docstring explains why
    and what the caller needs to know.

    Args:
        arg1: Description of arg1.
        arg2: Description of arg2. Can be multi-line
            if needed, indent continuation.

    Returns:
        Description of return value.

    Raises:
        ValueError: When arg1 is negative.
        TypeError: When arg2 is not a string.

    Example:
        >>> function_name(5, "hello")
        "hello hello hello hello hello"
    """
```

### Минимальный docstring (для простых функций)
```python
def double(n: int) -> int:
    """Return n multiplied by two."""
    return n * 2
```

### Для классов
```python
class UserRepository:
    """Manages persistence of User entities.

    Provides CRUD operations backed by a PostgreSQL database.
    All methods require an active connection passed at construction.

    Attributes:
        conn: Active database connection.
        table_name: Name of the users table.
    """
```

---

## 5. СТРУКТУРА МОДУЛЯ (порядок элементов)

```python
"""Module docstring — one line summary.

Extended description of module purpose.
"""
# 1. Будущие импорты (если нужны)
from __future__ import annotations

# 2. Стандартная библиотека
import json
import logging
import os
from pathlib import Path

# 3. Сторонние библиотеки
import httpx
import pydantic

# 4. Локальные импорты
from .models import User
from .utils import format_date

# 5. Публичный API модуля
__all__ = ["UserService", "create_user"]

# 6. Константы
DEFAULT_TIMEOUT: int = 30
MAX_RETRIES: int = 3
logger = logging.getLogger(__name__)

# 7. Исключения (если специфичны для модуля)
class UserNotFoundError(ValueError):
    """Raised when user lookup returns no results."""

# 8. Основной код (классы, функции)
class UserService:
    ...

# 9. Точка входа (только для скриптов)
if __name__ == "__main__":
    main()
```

---

## 6. ЗАПРЕЩЁННЫЕ КОНСТРУКЦИИ

### Антипаттерны стиля

| Антипаттерн | Почему плохо | Замена |
|-------------|-------------|--------|
| `lambda x: x.name` как аргумент sorted | Нечитаемо при усложнении | `operator.attrgetter("name")` |
| `[i for i in range(n) if condition][0]` | IndexError при пустом | `next((i for i in range(n) if condition), default)` |
| `dict.get(key) or default` | Ложный default при falsy значении | `dict.get(key, default)` |
| `type(x) == int` | Не учитывает наследование | `isinstance(x, int)` |
| `== None`, `== True`, `== False` | Нарушение PEP 8 | `is None`, `is True`, `is False` |
| `print()` для логов | Нельзя отключить/уровни | `logger.info()`, `logger.error()` |

### Запрещённые паттерны безопасности (подробнее в АНТИПАТТЕРНЫ.md)
```python
# ❌ Всё это — запрещено
eval(user_input)
exec(user_code)
__import__(user_module)
pickle.loads(untrusted_data)
yaml.load(data)           # использовать yaml.safe_load()
f"SELECT ... {user_val}"  # SQL injection
os.system(user_command)   # command injection
```

---

## 7. РИТМ КОДА (аналог ритма предложений в текстах)

### Правило длины функций
- **До 10 строк** — идеально (чистая функция, одна операция)
- **10–30 строк** — норма
- **30–50 строк** — допустимо с обоснованием
- **50+ строк** → **декомпозировать обязательно**

### Правило вложенности
- **1–2 уровня** — хорошо
- **3 уровня** — подумать о декомпозиции
- **4+ уровня** → **рефакторинг обязателен** (early return, вынести в функцию)

```python
# ❌ ПЛОХО — 4 уровня вложенности
def process(data):
    if data:
        for item in data:
            if item.is_valid():
                for sub in item.children:
                    if sub.active:
                        handle(sub)

# ✅ ХОРОШО — early return + вынос логики
def process(data: list) -> None:
    """Process valid active subitems."""
    if not data:
        return
    for item in data:
        _process_item(item)

def _process_item(item: Item) -> None:
    if not item.is_valid():
        return
    active_children = [c for c in item.children if c.active]
    for child in active_children:
        handle(child)
```

---

## 8. БЫСТРАЯ ПАМЯТКА

```
✅ Type hints на каждой функции и методе
✅ Docstring на каждом классе и публичной функции
✅ Именованные константы вместо магических чисел
✅ Конкретные исключения в except (не голый except:)
✅ with для файлов и соединений
✅ Параметризованные запросы для SQL
✅ Секреты через os.environ, не в коде
✅ logging вместо print
✅ snake_case для переменных, PascalCase для классов

❌ Мутабельные аргументы по умолчанию def f(lst=[])
❌ f-строки в SQL запросах
❌ eval/exec с внешними данными
❌ Голые except: без типа
❌ Файлы без with
❌ Хардкод секретов в коде
❌ print() для отладки в финальном коде
❌ Функции длиннее 50 строк без декомпозиции
❌ Вложенность 4+ уровней без рефакторинга
```
