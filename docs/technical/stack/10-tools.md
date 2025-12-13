# Вспомогательные инструменты и DevOps

**Дата создания:** 2025-12-13
**Приоритет:** ⭐⭐⭐ (Средний)

---

## Назначение

Инструменты для разработки, тестирования, мониторинга и деплоя.

---

## Разработка

### 1. Git + GitHub

**Структура репозитория:**
```
pto-platform/
├── backend/
├── frontend/
├── docs/
├── .github/
│   └── workflows/
│       ├── backend-tests.yml
│       ├── frontend-tests.yml
│       └── deploy.yml
├── docker-compose.yml
└── README.md
```

---

### 2. Poetry (управление зависимостями Python)

**Установка:**
```bash
curl -sSL https://install.python-poetry.org | python3 -
```

**pyproject.toml:**
```toml
[tool.poetry]
name = "pto-platform-backend"
version = "0.1.0"

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.109.0"
uvicorn = "^0.27.0"
sqlalchemy = "^2.0.0"
langchain = "^0.1.0"
openai = "^1.10.0"

[tool.poetry.dev-dependencies]
pytest = "^7.4.0"
black = "^24.0.0"
ruff = "^0.1.0"
```

**Команды:**
```bash
poetry install
poetry add fastapi
poetry run pytest
```

---

### 3. Форматирование кода

**Black (Python):**
```bash
pip install black
black ./app
```

**Prettier (JavaScript/TypeScript):**
```bash
npm install -D prettier
npx prettier --write .
```

---

## Тестирование

### 1. pytest (Backend)

**Установка:**
```bash
pip install pytest pytest-asyncio httpx
```

**Пример теста:**
```python
# tests/test_aosr.py
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_generate_aosr():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/api/v1/aosr/generate", json={
            "project_id": 1,
            "work_type": "Монтаж труб",
            "materials": []
        })
        assert response.status_code == 200
        assert "task_id" in response.json()
```

**Запуск:**
```bash
pytest tests/ -v
```

---

### 2. Jest (Frontend)

**Установка:**
```bash
npm install -D jest @testing-library/react
```

**Пример теста:**
```typescript
// src/components/ProjectCard.test.tsx
import { render, screen } from '@testing-library/react';
import { ProjectCard } from './ProjectCard';

test('renders project name', () => {
  render(<ProjectCard project={{ id: 1, name: 'Тест' }} />);
  expect(screen.getByText('Тест')).toBeInTheDocument();
});
```

---

## Линтеры

### Python (Ruff)

```bash
pip install ruff
ruff check ./app
```

### TypeScript (ESLint)

```bash
npm install -D eslint
npx eslint src/
```

---

## Мониторинг

### 1. Sentry (мониторинг ошибок)

**Backend:**
```python
import sentry_sdk

sentry_sdk.init(
    dsn="https://...@sentry.io/...",
    traces_sample_rate=1.0,
)
```

**Frontend:**
```typescript
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: "https://...@sentry.io/...",
});
```

---

### 2. Prometheus + Grafana (метрики)

**Экспорт метрик из FastAPI:**
```bash
pip install prometheus-fastapi-instrumentator
```

```python
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()

Instrumentator().instrument(app).expose(app)
```

---

## CI/CD

### GitHub Actions

**Backend Tests:**
```yaml
# .github/workflows/backend-tests.yml
name: Backend Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest tests/
```

**Frontend Tests:**
```yaml
# .github/workflows/frontend-tests.yml
name: Frontend Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm install
      - run: npm test
```

---

## Документация API

### Swagger (встроен в FastAPI)

**Автоматически доступен:**
- http://localhost:8000/docs (Swagger UI)
- http://localhost:8000/redoc (ReDoc)

---

## Логирование

### Structlog (структурированные логи)

```bash
pip install structlog
```

```python
import structlog

logger = structlog.get_logger()

logger.info("aosr_generated", project_id=1, aosr_id=123, duration_ms=1500)
```

---

## Управление задачами

### Flower (мониторинг Celery)

```bash
pip install flower
celery -A celery_app flower
```

Доступно на http://localhost:5555

---

## Pre-commit hooks

**Установка:**
```bash
pip install pre-commit
```

**.pre-commit-config.yaml:**
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.1.1
    hooks:
      - id: black
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.14
    hooks:
      - id: ruff
```

```bash
pre-commit install
```

---

## Полезные утилиты

### 1. httpie (тестирование API)

```bash
pip install httpie
http POST localhost:8000/api/v1/projects name="Тест" address="Москва"
```

### 2. jq (парсинг JSON)

```bash
curl localhost:8000/api/v1/projects | jq '.[] | .name'
```

### 3. pgcli (интерактивный PostgreSQL CLI)

```bash
pip install pgcli
pgcli postgresql://user:pass@localhost/pto_db
```

---

## Связь с другими компонентами

- **[Backend](03-backend.md):** Тестирование, мониторинг
- **[Cloud](07-cloud.md):** CI/CD деплой
- **[Database](08-database.md):** Миграции

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-13
