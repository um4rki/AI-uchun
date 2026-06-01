# PYTHON_СТРУКТУРА.md — Шаблоны + Таблицы компенсации слабостей ИИ
> Типовые структуры Python-проектов + таблицы для заполнения ДО написания.
> Размер файла оптимизирован (<5K токенов) — компенсация Lost in Middle.

---

## 1. ТИПОВЫЕ ШАБЛОНЫ

### Шаблон А — «Скрипт / CLI»
**Когда:** Одноразовый скрипт, автоматизация, CLI-инструмент.

```
project/
├── main.py          # точка входа, argparse/click
├── config.py        # константы и Config.from_env()
├── utils.py         # вспомогательные функции
└── requirements.txt # зависимости
```

**Скелет main.py:**
```python
"""Script description."""
import argparse
import logging
import sys

from config import Config
from utils import process_data

logger = logging.getLogger(__name__)


def parse_args() -> argparse.Namespace:
    """Parse command-line arguments."""
    parser = argparse.ArgumentParser(description="Script description")
    parser.add_argument("input", type=str, help="Input file path")
    parser.add_argument("--verbose", action="store_true")
    return parser.parse_args()


def main() -> int:
    """Entry point. Returns exit code."""
    args = parse_args()
    logging.basicConfig(level=logging.DEBUG if args.verbose else logging.INFO)
    try:
        config = Config.from_env()
        process_data(args.input, config)
        return 0
    except Exception as e:
        logger.error("Fatal error: %s", e)
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

---

### Шаблон Б — «ООП-модуль»
**Когда:** Переиспользуемый модуль с бизнес-логикой, классы-сущности.

```
module/
├── __init__.py      # публичный API (__all__)
├── models.py        # dataclasses / Pydantic модели
├── service.py       # бизнес-логика
├── repository.py    # работа с данными
└── exceptions.py    # кастомные исключения
```

**Скелет service.py:**
```python
"""Business logic layer."""
import logging
from .exceptions import NotFoundError
from .models import Entity
from .repository import Repository

logger = logging.getLogger(__name__)


class EntityService:
    """Manages Entity lifecycle and business rules."""

    def __init__(self, repo: Repository) -> None:
        self._repo = repo

    def get(self, entity_id: int) -> Entity:
        """Retrieve entity by ID.

        Raises:
            NotFoundError: If entity does not exist.
        """
        entity = self._repo.find_by_id(entity_id)
        if entity is None:
            raise NotFoundError(f"Entity {entity_id} not found")
        return entity
```

---

### Шаблон В — «FastAPI / REST API»
**Когда:** HTTP API, микросервис, веб-сервер.

```
app/
├── main.py          # FastAPI app, lifespan, middleware
├── config.py        # Settings (pydantic-settings)
├── routers/
│   └── items.py     # APIRouter по ресурсам
├── models/
│   ├── schemas.py   # Pydantic request/response схемы
│   └── db.py        # SQLAlchemy модели
├── services/
│   └── item_service.py
└── dependencies.py  # FastAPI Depends()
```

**Скелет router:**
```python
"""Items resource router."""
from fastapi import APIRouter, Depends, HTTPException, status
from .schemas import ItemCreate, ItemResponse
from .services.item_service import ItemService
from .dependencies import get_item_service

router = APIRouter(prefix="/items", tags=["items"])


@router.post("/", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
async def create_item(
    payload: ItemCreate,
    service: ItemService = Depends(get_item_service),
) -> ItemResponse:
    """Create a new item."""
    return await service.create(payload)
```

---

### Шаблон Г — «Data Pipeline / ETL»
**Когда:** Обработка данных, трансформации, аналитика.

```
pipeline/
├── extract.py       # получение данных (API, файл, БД)
├── transform.py     # преобразования
├── load.py          # сохранение результата
├── validate.py      # схема + проверки данных
├── config.py
└── main.py          # оркестрация шагов
```

**Скелет pipeline:**
```python
"""ETL pipeline orchestration."""
import logging
from pathlib import Path
from .extract import extract_from_csv
from .transform import clean_and_enrich
from .load import save_to_db
from .validate import validate_schema

logger = logging.getLogger(__name__)


def run_pipeline(source: Path, conn_str: str) -> dict:
    """Run full ETL pipeline.

    Returns:
        Summary dict with counts: extracted, transformed, loaded, failed.
    """
    logger.info("Starting pipeline for %s", source)
    raw = extract_from_csv(source)
    validated = validate_schema(raw)
    transformed = clean_and_enrich(validated)
    loaded = save_to_db(transformed, conn_str)
    return {"extracted": len(raw), "loaded": loaded}
```

---

### Шаблон Д — «Automation / Scheduler»
**Когда:** Периодические задачи, боты, мониторинг.

```
bot/
├── main.py          # точка входа, scheduler
├── tasks/
│   ├── __init__.py
│   └── report.py    # конкретные задачи
├── clients/
│   └── api_client.py  # внешние API
├── config.py
└── storage.py       # состояние между запусками
```

---

## 2. ТАБЛИЦЫ ДЛЯ ЗАПОЛНЕНИЯ ДО НАПИСАНИЯ

### 2.1 Таблица компонентов
> Заполнить ПЕРЕД написанием. Фиксирует всё, что будет в коде.

| Имя | Тип | Файл | Ответственность | Зависит от |
|-----|-----|------|-----------------|-----------|
| `Config` | dataclass | config.py | Хранит настройки из env | os |
| `UserService` | class | service.py | Бизнес-логика пользователей | UserRepo |
| `get_user` | function | service.py | Получить юзера по ID | UserRepo.find |
| *(добавить)* | | | | |

---

### 2.2 Таблица зависимостей
> Все импорты ПЕРЕД написанием. При написании — только из этой таблицы.

| Пакет | Тип | Версия | Что используем | Импорт |
|-------|-----|--------|----------------|--------|
| `pathlib` | stdlib | — | Работа с путями | `from pathlib import Path` |
| `logging` | stdlib | — | Логирование | `import logging` |
| `dataclasses` | stdlib | — | Модели | `from dataclasses import dataclass, field` |
| `httpx` | third-party | `>=0.27` | HTTP-запросы | `import httpx` |
| `pydantic` | third-party | `>=2.0` | Валидация | `from pydantic import BaseModel` |
| *(добавить)* | | | | |

---

### 2.3 Таблица сигнатур
> Полные сигнатуры функций ПЕРЕД написанием. Копировать при написании.

| Функция/метод | Аргументы | Возвращает | Исключения |
|---------------|-----------|------------|-----------|
| `Config.from_env()` | `cls` | `Config` | `KeyError` |
| `get_user(conn, user_id)` | `conn: Connection, user_id: int` | `User \| None` | — |
| `process_order(order)` | `order: Order` | `Receipt` | `ValueError`, `PaymentError` |
| *(добавить)* | | | |

---

## 3. ТАБЛИЦЫ ВЕРИФИКАЦИИ (заполнять ПОСЛЕ написания)

### 3.1 Таблица безопасности
> Каждая точка ввода данных — отдельная строка. ❌ = нельзя сдавать.

| Место в коде | Тип ввода | Защита | Статус |
|--------------|-----------|--------|--------|
| `get_user(email)` | Пользовательский | Параметризованный SQL | ✅ |
| `Config.from_env()` | Env переменные | Валидация типов | ✅ |
| `load_file(path)` | Путь к файлу | Path.resolve() + белый список | ✅ / ❌ |
| *(добавить все точки ввода)* | | | |

**Дополнительная проверка:**

| Проверка | Статус |
|----------|--------|
| Нет хардкод-секретов (`grep -r "password\|secret\|api_key" --include="*.py"`) | ✅ / ❌ |
| Нет f-строк в SQL | ✅ / ❌ |
| Нет `eval()`/`exec()` с внешними данными | ✅ / ❌ |
| Нет `pickle.loads()` с ненадёжными данными | ✅ / ❌ |

---

### 3.2 Таблица исключений
> Каждый try/except — отдельная строка. Голый `except:` = ❌.

| Место в коде | Тип исключения | Обработка | Правильно? |
|--------------|----------------|-----------|-----------|
| `int(user_input)` | `ValueError` | log + raise | ✅ |
| `open(path)` | `FileNotFoundError` | return None | ✅ |
| `except Exception as e` | Слишком широко | — | ⚠️ уточнить |
| `except:` | Голый | — | ❌ |
| *(добавить все try/except)* | | | |

---

### 3.3 Таблица сложности
> Проверить вложенные циклы и алгоритмы.

| Функция | Сложность | Обоснование | Можно улучшить? |
|---------|-----------|-------------|----------------|
| `find_duplicates(lst)` | O(n) | Использует set | — |
| `match_users(a, b)` | O(n²) | Вложенный цикл | ⚠️ → dict lookup O(n) |
| *(добавить нетривиальные функции)* | | | |

---

### 3.4 Чеклист качества кода

| Критерий | Статус |
|----------|--------|
| Type hints на всех публичных функциях | ✅ / ❌ |
| Docstrings на всех классах и публичных функциях | ✅ / ❌ |
| Нет магических чисел (только именованные константы) | ✅ / ❌ |
| Нет дублирования логики (DRY) | ✅ / ❌ |
| Каждая функция ≤ 50 строк | ✅ / ❌ |
| Вложенность не превышает 3 уровня | ✅ / ❌ |
| Импорты в правильном порядке (stdlib → third-party → local) | ✅ / ❌ |
| `__all__` определён в публичных модулях | ✅ / ❌ |
| Нет `print()` — только `logging` | ✅ / ❌ |
| Файлы и соединения закрываются через `with` | ✅ / ❌ |
| Нет мутабельных аргументов по умолчанию | ✅ / ❌ |
| Антипаттерны из PYTHON_АНТИПАТТЕРНЫ.md отсутствуют | ✅ / ❌ |

---

*Файл: PYTHON_СТРУКТУРА.md · Версия 1.0*
*Использовать вместе с PYTHON_СТИЛЬ.md и PYTHON_АНТИПАТТЕРНЫ.md*
