# PYTHON_АНТИПАТТЕРНЫ.md — Запрещённые конструкции и типичные ошибки ИИ
> Компенсация слабостей ИИ при генерации Python-кода.
> Сканировать код ПОСЛЕ написания. Любой ❌ = исправить до сдачи.

---

## ⚠️ КРИТИЧЕСКОЕ ПРАВИЛО

> ИИ СКАНИРУЕТ код по этому файлу ПОСЛЕ написания.
> Наличие любого паттерна из §3 (безопасность) — блокирует сдачу кода.
> Остальные паттерны — исправить перед финальным ответом.

---

## 1. АНТИПАТТЕРНЫ КОДА (типичные ошибки ИИ)

### 1.1 Несуществующие методы (галлюцинации API)

ИИ изобретает методы, которых нет в реальных библиотеках.

| ❌ Несуществует | ✅ Реальный аналог |
|----------------|-------------------|
| `df.filter_by(col > 0)` | `df[df['col'] > 0]` |
| `list.find(item)` | `next((x for x in lst if x == item), None)` |
| `dict.filter(lambda k, v: v > 0)` | `{k: v for k, v in d.items() if v > 0}` |
| `str.contains("sub")` | `"sub" in string` |
| `os.path.join_all([...])` | `os.path.join(*parts)` или `Path(*parts)` |
| `requests.get_json(url)` | `requests.get(url).json()` |
| `json.load_string(s)` | `json.loads(s)` |
| `datetime.now_utc()` | `datetime.utcnow()` или `datetime.now(UTC)` |

**Правило:** Если метод не в таблице 2.2 (зависимости) — проверить документацию перед использованием.

---

### 1.2 Устаревший синтаксис Python

| ❌ Устарело | ✅ Современно | Версия |
|------------|--------------|--------|
| `% "форматирование"` | f-строки или `.format()` | 3.6+ |
| `open(f)` без `with` | `with open(f) as fh:` | всегда |
| `Dict[str, int]` (typing) | `dict[str, int]` | 3.9+ |
| `Optional[str]` | `str \| None` | 3.10+ |
| `Union[str, int]` | `str \| int` | 3.10+ |
| `Tuple[int, str]` | `tuple[int, str]` | 3.9+ |
| `List[str]` | `list[str]` | 3.9+ |
| `.format(**locals())` | f-строки | 3.6+ |
| `raise Exception from None` без причины | Сохранять контекст или явно указывать | — |

---

### 1.3 Мутабельные аргументы по умолчанию

```python
# ❌ ЛОВУШКА — список создаётся один раз при определении функции
def append_item(item: str, container: list = []) -> list:
    container.append(item)
    return container

# Результат:
append_item("a")  # ["a"]
append_item("b")  # ["a", "b"] — неожиданно!

# ✅ ПРАВИЛЬНО
def append_item(item: str, container: list | None = None) -> list:
    if container is None:
        container = []
    container.append(item)
    return container
```

**Мутабельные типы, запрещённые как дефолт:** `list`, `dict`, `set`, `bytearray`, любые кастомные изменяемые объекты.

---

### 1.4 Проблемы с именами, перекрывающими встроенные

```python
# ❌ Перекрывает встроенные
list = [1, 2, 3]       # теперь list() не работает
id = get_user_id()     # перекрывает id()
type = "admin"         # перекрывает type()
input = "some text"    # перекрывает input()
filter = lambda x: x   # перекрывает filter()

# ✅ Добавить суффикс по смыслу
items_list = [1, 2, 3]
user_id = get_user_id()
user_type = "admin"
raw_input = "some text"
item_filter = lambda x: x
```

---

### 1.5 Неправильное сравнение

```python
# ❌ ПЛОХО
if x == None: ...
if x == True: ...
if x == False: ...
if type(x) == int: ...
if len(lst) == 0: ...

# ✅ ХОРОШО
if x is None: ...
if x is True: ...  # или просто: if x:
if x is False: ... # или: if not x:
if isinstance(x, int): ...
if not lst: ...
```

---

### 1.6 Неэффективные паттерны

```python
# ❌ Конкатенация строк в цикле — O(n²)
result = ""
for item in items:
    result += str(item)

# ✅ join — O(n)
result = "".join(str(item) for item in items)

# ❌ Поиск в списке — O(n) при каждом вызове
for user in users:
    if user in banned_list:  # если banned_list — list, это O(n*m)
        ...

# ✅ Поиск в set — O(1)
banned_set = set(banned_list)
for user in users:
    if user in banned_set:
        ...

# ❌ Создание нового списка без нужды
existing = list(some_generator)  # если нужна только итерация

# ✅ Прямая итерация
for item in some_generator:
    ...
```

---

## 2. АНТИПАТТЕРНЫ СТРУКТУРЫ (DRY / SRP нарушения)

### 2.1 God Function (функция-бог)

```python
# ❌ Делает всё сразу — читай, валидируй, трансформируй, сохраняй
def process_user_data(file_path):
    with open(file_path) as f:
        data = json.load(f)
    # 80 строк логики...

# ✅ Декомпозиция по ответственностям
def load_user_data(file_path: Path) -> list[dict]: ...
def validate_users(users: list[dict]) -> list[dict]: ...
def transform_users(users: list[dict]) -> list[User]: ...
def save_users(users: list[User], conn: Connection) -> int: ...

def process_user_data(file_path: Path, conn: Connection) -> int:
    """Orchestrate the full user data pipeline."""
    raw = load_user_data(file_path)
    valid = validate_users(raw)
    users = transform_users(valid)
    return save_users(users, conn)
```

### 2.2 Повторяющаяся логика (нарушение DRY)

Если один и тот же блок кода встречается 2+ раза — вынести в функцию.

```python
# ❌ Дублирование
def get_admin(conn, admin_id):
    cursor = conn.execute(
        "SELECT * FROM users WHERE id = ? AND role = 'admin'", (admin_id,)
    )
    row = cursor.fetchone()
    return dict(row) if row else None

def get_moderator(conn, mod_id):
    cursor = conn.execute(
        "SELECT * FROM users WHERE id = ? AND role = 'moderator'", (mod_id,)
    )
    row = cursor.fetchone()
    return dict(row) if row else None

# ✅ DRY
def get_user_by_role(
    conn: Connection, user_id: int, role: str
) -> dict | None:
    """Fetch user by ID and role."""
    cursor = conn.execute(
        "SELECT * FROM users WHERE id = ? AND role = ?",
        (user_id, role),
    )
    row = cursor.fetchone()
    return dict(row) if row else None
```

---

## 3. АНТИПАТТЕРНЫ БЕЗОПАСНОСТИ (БЛОКИРУЮТ СДАЧУ)

### 3.1 SQL Injection

```python
# ❌ КРИТИЧНО — немедленно исправить
user = request.args.get("user")
query = f"SELECT * FROM users WHERE name = '{user}'"
cursor.execute(query)

# ❌ Тоже опасно
query = "SELECT * FROM users WHERE name = '" + user_input + "'"

# ✅ БЕЗОПАСНО — параметризованные запросы
cursor.execute(
    "SELECT * FROM users WHERE name = ?",
    (user_input,),
)
# Для PostgreSQL/psycopg2:
cursor.execute(
    "SELECT * FROM users WHERE name = %s",
    (user_input,),
)
```

### 3.2 Command Injection

```python
# ❌ КРИТИЧНО
import os, subprocess
os.system(f"ping {user_input}")
subprocess.call(f"ls {user_dir}", shell=True)

# ✅ БЕЗОПАСНО — список аргументов, no shell=True
import subprocess
subprocess.run(["ping", "-c", "1", host], check=True, timeout=5)
```

### 3.3 Небезопасная десериализация

```python
# ❌ КРИТИЧНО — pickle выполняет произвольный код
import pickle
data = pickle.loads(user_provided_bytes)

# ❌ yaml.load выполняет Python-код
import yaml
config = yaml.load(file_content)

# ✅ БЕЗОПАСНО
import json
data = json.loads(user_provided_json)  # json — безопасен

import yaml
config = yaml.safe_load(file_content)  # safe_load — без выполнения кода
```

### 3.4 Хардкод секретов

```python
# ❌ КРИТИЧНО — никогда не коммитить
API_KEY = "sk-proj-abc123"
DB_PASSWORD = "admin123"
SECRET_TOKEN = "eyJhbGciOi..."
JWT_SECRET = "my_super_secret"

# ✅ БЕЗОПАСНО — из окружения
import os
API_KEY = os.environ["API_KEY"]             # обязательная переменная
DB_PASSWORD = os.environ.get("DB_PASSWORD") # опциональная

# Ещё лучше — через pydantic-settings или python-dotenv
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    api_key: str
    db_password: str

    class Config:
        env_file = ".env"
```

### 3.5 Небезопасный Path Traversal

```python
# ❌ Пользователь может передать ../../etc/passwd
filename = request.args.get("file")
with open(f"/uploads/{filename}") as f:
    return f.read()

# ✅ БЕЗОПАСНО — resolve() + проверка prefix
from pathlib import Path

UPLOAD_DIR = Path("/uploads").resolve()

def safe_open(filename: str) -> str:
    """Open file only within UPLOAD_DIR."""
    requested = (UPLOAD_DIR / filename).resolve()
    if not str(requested).startswith(str(UPLOAD_DIR)):
        raise PermissionError(f"Access denied: {filename}")
    return requested.read_text(encoding="utf-8")
```

### 3.6 Выполнение пользовательского кода

```python
# ❌ КРИТИЧНО — выполняет произвольный Python
result = eval(user_expression)
exec(user_code)

# ❌ Опасно
module = __import__(user_module_name)

# ✅ Если нужна «формула» — парсить вручную или использовать ast.literal_eval
import ast
# literal_eval безопасен — только литералы Python (числа, строки, списки, dict)
value = ast.literal_eval(user_input)  # "{'a': 1}" → {'a': 1}
```

---

## 4. СВОДНАЯ ТАБЛИЦА ПРОВЕРКИ

> Сканировать код после написания. Все ❌ — исправить.

### Блокирующие (безопасность)

| Проверка | Команда grep | Статус |
|----------|-------------|--------|
| f-строки в SQL | `f".*SELECT\|INSERT\|UPDATE\|DELETE` | ✅ / ❌ |
| Хардкод секретов | `password\s*=\s*["']` | ✅ / ❌ |
| eval/exec с вводом | `eval(\|exec(` | ✅ / ❌ |
| pickle.loads | `pickle\.loads` | ✅ / ❌ |
| yaml.load (не safe) | `yaml\.load[^s]` | ✅ / ❌ |
| shell=True | `shell=True` | ✅ / ❌ |

### Некритичные (качество)

| Проверка | Статус |
|----------|--------|
| Голые except: без типа | ✅ / ❌ |
| Мутабельные дефолты def f(x=[]) | ✅ / ❌ |
| print() в финальном коде | ✅ / ❌ |
| Несуществующие методы библиотек | ✅ / ❌ |
| Устаревший синтаксис typing (List, Dict, Optional) | ✅ / ❌ |
| Функции > 50 строк | ✅ / ❌ |
| Вложенность > 3 уровней | ✅ / ❌ |
| Дублирование кода (DRY) | ✅ / ❌ |
| Перекрытие встроенных имён | ✅ / ❌ |

---

*Файл: PYTHON_АНТИПАТТЕРНЫ.md · Версия 1.0*
*Использовать вместе с PYTHON_ИНСТРУКЦИЯ.md · PYTHON_СТИЛЬ.md · PYTHON_СТРУКТУРА.md*
