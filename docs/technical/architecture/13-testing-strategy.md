# Стратегия тестирования

**Дата создания:** 2025-12-15
**Для кого:** Для QA инженеров и разработчиков

---

## 🎯 Цели тестирования

1. **Надёжность** — платформа должна работать стабильно
2. **Качество AI** — агенты должны давать корректные результаты
3. **Производительность** — соответствие SLA
4. **Безопасность** — защита данных пользователей

---

## 📊 Пирамида тестирования

```
                  ┌────────────┐
                  │    E2E     │  5% (критичные пользовательские сценарии)
                  │   Tests    │
                ┌─┴────────────┴─┐
                │  Integration   │  25% (API, Celery, DB)
                │     Tests      │
              ┌─┴────────────────┴─┐
              │    Unit Tests      │  70% (функции, модули)
              └────────────────────┘
```

---

## 1️⃣ Unit-тесты (pytest)

### Целевой coverage: 80%+ общий, 95%+ критичные модули

**Что тестируем:**
- Утилиты и helper функции
- Валидация данных (Pydantic models)
- Бизнес-логика без внешних зависимостей
- Парсинг и трансформация данных

**Пример:**
```python
# tests/unit/test_utils.py
import pytest
from app.utils import physical_sheets_to_pdf_pages

def test_physical_sheets_conversion():
    assert physical_sheets_to_pdf_pages(1.0) == 2
    assert physical_sheets_to_pdf_pages(1.5) == 4  # округление вверх
    assert physical_sheets_to_pdf_pages(10.2) == 22

def test_aosr_number_generation():
    from app.utils import generate_aosr_number

    number = generate_aosr_number(project_id=123, sequence=5)
    assert number == "АОСР-123-005"
```

**Запуск:**
```bash
pytest tests/unit -v --cov=app --cov-report=html
```

---

## 2️⃣ Integration-тесты

### Целевой coverage: 90%+ для критичных интеграций

### 2.1. API Endpoints (FastAPI TestClient)

**Что тестируем:**
- Все REST API endpoints
- Аутентификация и авторизация
- Валидация входных данных
- Коды ответов и структура данных

**Пример:**
```python
# tests/integration/test_projects_api.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_project_success():
    response = client.post(
        "/api/v1/projects",
        headers={"Authorization": f"Bearer {get_test_token()}"},
        json={
            "name": "Тестовый проект",
            "address": "г. Москва",
            "package_format": "unified"
        }
    )
    assert response.status_code == 201
    data = response.json()
    assert data["success"] == True
    assert "id" in data["data"]
    assert data["data"]["name"] == "Тестовый проект"

def test_create_project_unauthorized():
    response = client.post("/api/v1/projects", json={})
    assert response.status_code == 401
```

---

### 2.2. Celery Tasks

**Что тестируем:**
- Запуск и завершение задач
- Обработка ошибок и retry
- Корректность результатов

**Пример:**
```python
# tests/integration/test_celery_tasks.py
import pytest
from app.tasks import analyze_rd_task

@pytest.mark.celery
def test_analyze_rd_task_success(test_db, sample_document):
    """Тест анализа РД"""
    result = analyze_rd_task.apply(args=[sample_document.id]).get()

    assert result["success"] == True
    assert result["work_types_count"] > 0
    assert result["materials_count"] > 0

@pytest.mark.celery
def test_analyze_rd_task_retry_on_api_error(test_db, monkeypatch):
    """Тест retry при ошибке OpenAI API"""
    # Mock OpenAI API для симуляции ошибки
    def mock_openai_error(*args, **kwargs):
        raise OpenAIError("Rate limit exceeded")

    monkeypatch.setattr("openai.ChatCompletion.create", mock_openai_error)

    with pytest.raises(Retry):
        analyze_rd_task.apply(args=[123])
```

---

### 2.3. Database Queries

**Что тестируем:**
- CRUD операции
- Транзакции
- Индексы и производительность
- Constraints и валидация на уровне БД

**Пример:**
```python
# tests/integration/test_database.py
from app.models import Project, AOSR
from sqlalchemy.orm import Session

def test_project_cascade_delete(test_db: Session):
    """Тест каскадного удаления проекта"""
    project = Project(name="Test", address="Test")
    test_db.add(project)
    test_db.commit()

    aosr = AOSR(project_id=project.id, work_type="Test")
    test_db.add(aosr)
    test_db.commit()

    test_db.delete(project)
    test_db.commit()

    # Проверяем что АОСР тоже удалился
    assert test_db.query(AOSR).filter_by(id=aosr.id).first() is None
```

---

## 3️⃣ E2E тесты (Playwright)

### Критичные пользовательские сценарии

**Что тестируем:**
- Полный workflow от создания проекта до скачивания пакета
- UI взаимодействия
- WebSocket уведомления

**Пример:**
```python
# tests/e2e/test_full_workflow.py
from playwright.sync_api import Page, expect

def test_full_project_workflow(page: Page, logged_in_user):
    """Тест полного цикла работы с проектом"""

    # 1. Создание проекта
    page.goto("/projects/new")
    page.fill('input[name="name"]', "Тестовый ЖК")
    page.fill('input[name="address"]', "г. Москва")
    page.click('button:text("Создать")')
    expect(page).to_have_url("/projects/proj_*")

    # 2. Загрузка РД
    page.set_input_files('input[type="file"]', "test_data/rd_sample.pdf")
    expect(page.locator('text="Документ загружен"')).to_be_visible()

    # 3. Анализ РД
    page.click('button:text("Анализировать РД")')
    expect(page.locator('text="Анализ завершен"')).to_be_visible(timeout=120000)

    # 4. Генерация АОСР
    page.click('button:text("Сгенерировать АОСР")')
    expect(page.locator('.aosr-item')).to_have_count(10)

    # 5. Формирование пакета
    page.click('button:text("Создать пакет ИД")')
    expect(page.locator('text="Пакет готов"')).to_be_visible(timeout=60000)

    # 6. Скачивание
    with page.expect_download() as download_info:
        page.click('button:text("Скачать PDF")')
    download = download_info.value
    assert download.suggested_filename.endswith(".pdf")
```

---

## 4️⃣ Тестирование AI-агентов

### Mock vs Real API

**Development:** Mock OpenAI API
```python
# conftest.py
@pytest.fixture
def mock_openai():
    with patch("openai.ChatCompletion.create") as mock:
        mock.return_value = {
            "choices": [{
                "message": {
                    "content": json.dumps({
                        "work_types": [...],
                        "materials": [...]
                    })
                }
            }]
        }
        yield mock
```

**Staging/Production:** Real API с тестовыми данными
```python
@pytest.mark.slow
@pytest.mark.requires_openai
def test_real_openai_analysis():
    """Тест с реальным OpenAI API"""
    result = analyze_rd_with_gpt(test_document)
    assert result["work_types"] is not None
```

---

## 5️⃣ Performance тесты (Locust)

**Что тестируем:**
- Производительность под нагрузкой
- Время отклика API
- Пропускная способность

**Пример:**
```python
# tests/performance/locustfile.py
from locust import HttpUser, task, between

class PlatformUser(HttpUser):
    wait_time = between(1, 3)

    def on_start(self):
        """Логин перед тестами"""
        response = self.client.post("/api/v1/auth/login", json={
            "email": "test@example.com",
            "password": "password"
        })
        self.token = response.json()["data"]["access_token"]

    @task(3)
    def get_projects(self):
        """Получить список проектов"""
        self.client.get(
            "/api/v1/projects",
            headers={"Authorization": f"Bearer {self.token}"}
        )

    @task(1)
    def create_project(self):
        """Создать проект"""
        self.client.post(
            "/api/v1/projects",
            headers={"Authorization": f"Bearer {self.token}"},
            json={"name": "Load Test", "address": "Test"}
        )
```

**Запуск:**
```bash
locust -f tests/performance/locustfile.py --host=https://api.pto-platform.ru
```

---

## 6️⃣ Security тесты

**Что тестируем:**
- SQL Injection
- XSS атаки
- CSRF protection
- Rate limiting
- JWT security

**Инструменты:**
- OWASP ZAP
- Bandit (статический анализ)
- Safety (проверка зависимостей)

**Пример:**
```bash
# Проверка уязвимостей в зависимостях
safety check

# Статический анализ безопасности
bandit -r app/ -ll

# OWASP ZAP scan
zap-cli quick-scan https://api.pto-platform.ru
```

---

## 📋 Coverage targets

| Модуль | Target Coverage |
|--------|----------------|
| **API routes** | 95%+ |
| **AI agents** | 90%+ |
| **Database models** | 95%+ |
| **Celery tasks** | 90%+ |
| **Utils/helpers** | 85%+ |
| **Frontend components** | 70%+ |

---

## 🚀 CI/CD Pipeline

```yaml
# .github/workflows/tests.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Setup Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Run unit tests
        run: pytest tests/unit -v --cov=app --cov-report=xml

      - name: Run integration tests
        run: pytest tests/integration -v
        env:
          DATABASE_URL: postgresql://test:test@localhost/test_db
          REDIS_URL: redis://localhost:6379/0

      - name: Upload coverage
        uses: codecov/codecov-action@v2
        with:
          files: ./coverage.xml

      - name: Security scan
        run: |
          bandit -r app/ -ll
          safety check
```

---

## 📊 Метрики качества

**Минимальные требования для merge в main:**
- ✅ Все тесты проходят
- ✅ Coverage ≥ 80%
- ✅ Нет critical security issues
- ✅ Код прошел линтер (black, flake8)

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-15
