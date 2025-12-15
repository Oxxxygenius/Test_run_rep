# Backend (Серверная часть)

**Дата создания:** 2025-12-13
**Приоритет:** ⭐⭐⭐⭐⭐ (Критический — основа платформы)

---

## Назначение

Backend отвечает за:
- API для фронтенда
- Оркестрация AI-агентов
- Обработка документов
- Бизнес-логика платформы
- Интеграции с внешними сервисами

---

## Рекомендуемая связка

### Python + FastAPI + Celery

```
┌─────────────────────────────────────────────────────┐
│                  Backend Stack                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [Web Framework]                                    │
│  └─ FastAPI (REST API + WebSockets)                 │
│                                                     │
│  [Task Queue]                                       │
│  └─ Celery + Redis (асинхронные задачи)            │
│                                                     │
│  [ORM]                                              │
│  └─ SQLAlchemy 2.0 (работа с БД)                    │
│                                                     │
│  [Validation]                                       │
│  └─ Pydantic (валидация данных)                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Конкретные инструменты

### 1. FastAPI (Web Framework)

**Что это:**
- Современный Python-фреймворк для создания API
- Автоматическая генерация документации (Swagger)
- Быстрая разработка
- Асинхронная поддержка

**Почему FastAPI:**
- ✅ Высокая производительность (как Node.js, Go)
- ✅ Автоматическая валидация через Pydantic
- ✅ Встроенная документация API
- ✅ Поддержка WebSocket для реального времени
- ✅ Легкая интеграция с async/await

**Установка:**
```bash
pip install fastapi uvicorn[standard] python-multipart
```

**Пример API:**
```python
from fastapi import FastAPI, UploadFile, File
from pydantic import BaseModel
from typing import List

app = FastAPI(title="ПТО Платформа API")

class Project(BaseModel):
    id: int
    name: str
    address: str
    status: str

class AOSRRequest(BaseModel):
    project_id: int
    work_type: str
    materials: List[dict]

@app.get("/")
def root():
    return {"message": "ПТО Platform API v1.0"}

@app.get("/projects", response_model=List[Project])
def get_projects():
    # Логика получения проектов
    return []

@app.post("/generate-aosr")
async def generate_aosr(request: AOSRRequest):
    # Запуск AI-агента для генерации АОСР
    task = generate_aosr_task.delay(request.dict())
    return {"task_id": task.id, "status": "processing"}

@app.post("/upload-document")
async def upload_document(file: UploadFile = File(...)):
    # Сохранение документа и запуск OCR
    content = await file.read()
    # Обработка...
    return {"filename": file.filename, "status": "uploaded"}

# Запуск: uvicorn main:app --reload
```

**Официальный сайт:** https://fastapi.tiangolo.com/

---

### 2. Celery (Task Queue для фоновых задач)

**Что это:**
- Распределенная очередь задач
- Для длительных операций (генерация актов, OCR, поиск документов)

**Почему Celery:**
- ✅ Асинхронное выполнение задач
- ✅ Не блокирует API
- ✅ Масштабируемость (можно добавлять workers)
- ✅ Retry механизм для надежности

**Установка:**
```bash
pip install celery redis
```

**Настройка:**
```python
# celery_app.py
from celery import Celery

app = Celery(
    'pto_platform',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/0'
)

app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='Europe/Moscow',
    enable_utc=True,
)

# Задачи
@app.task
def generate_aosr_task(data):
    """Генерация АОСР через AI-агента"""
    from agents.aosr_agent import generate_aosr
    result = generate_aosr(data)
    return result

@app.task
def process_ocr_task(file_path):
    """OCR обработка документа"""
    from services.ocr import process_document
    result = process_document(file_path)
    return result

@app.task
def search_quality_docs_task(material_name):
    """Поиск документов качества в интернете"""
    from agents.search_agent import search_documents
    result = search_documents(material_name)
    return result
```

**Запуск worker:**
```bash
celery -A celery_app worker --loglevel=info
```

**Официальный сайт:** https://docs.celeryq.dev/

---

### 3. SQLAlchemy (ORM для работы с БД)

**Что это:**
- ORM (Object-Relational Mapping) для Python
- Упрощает работу с PostgreSQL

**Установка:**
```bash
pip install sqlalchemy asyncpg psycopg2-binary
```

**Пример моделей:**
```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, JSON
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from datetime import datetime

Base = declarative_base()

class Project(Base):
    __tablename__ = "projects"

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    address = Column(String, nullable=False)
    client = Column(String)  # Заказчик
    contractor = Column(String)  # Подрядчик
    general_contractor = Column(String)  # Генподрядчик
    developer = Column(String)  # Застройщик
    package_format = Column(String, default="unified")  # Формат комплекта ИД: "repeated" или "unified"
    created_at = Column(DateTime, default=datetime.utcnow)
    created_by = Column(Integer, ForeignKey("users.id"))

    documents = relationship("Document", back_populates="project")
    aosr_list = relationship("AOSR", back_populates="project")

class Document(Base):
    __tablename__ = "documents"

    id = Column(Integer, primary_key=True)
    project_id = Column(Integer, ForeignKey("projects.id"))
    filename = Column(String)
    doc_type = Column(String)  # сертификат, паспорт, декларация
    ocr_text = Column(String)
    metadata = Column(JSON)  # реквизиты документа
    created_at = Column(DateTime, default=datetime.utcnow)

    project = relationship("Project", back_populates="documents")

class AOSR(Base):
    __tablename__ = "aosr"

    id = Column(Integer, primary_key=True)
    project_id = Column(Integer, ForeignKey("projects.id"))
    work_type = Column(String)
    content = Column(JSON)  # структурированное содержимое акта
    status = Column(String)  # draft, generated, approved
    created_at = Column(DateTime, default=datetime.utcnow)

    project = relationship("Project", back_populates="aosr_list")
```

**Официальный сайт:** https://www.sqlalchemy.org/

---

### 4. Pydantic (Валидация данных)

**Что это:**
- Библиотека для валидации данных
- Встроена в FastAPI

**Пример схем:**
```python
from pydantic import BaseModel, Field, validator
from typing import List, Optional
from datetime import date

class MaterialSchema(BaseModel):
    name: str = Field(..., min_length=1)
    manufacturer: str
    gost: Optional[str] = None
    quantity: float = Field(..., gt=0)
    unit: str

class AOSRCreateSchema(BaseModel):
    project_id: int
    work_type: str
    work_description: str
    materials: List[MaterialSchema]
    work_date: date

    @validator('work_date')
    def validate_work_date(cls, v):
        if v > date.today():
            raise ValueError('Дата работ не может быть в будущем')
        return v

class DocumentUploadSchema(BaseModel):
    doc_type: str
    project_id: int

    @validator('doc_type')
    def validate_doc_type(cls, v):
        allowed = ['сертификат', 'паспорт', 'декларация', 'отказное письмо']
        if v not in allowed:
            raise ValueError(f'Неизвестный тип документа: {v}')
        return v
```

**Официальный сайт:** https://docs.pydantic.dev/

---

## Структура проекта

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI приложение
│   ├── config.py               # Конфигурация (переменные окружения)
│   ├── database.py             # Подключение к БД
│   │
│   ├── models/                 # SQLAlchemy модели
│   │   ├── __init__.py
│   │   ├── project.py
│   │   ├── document.py
│   │   └── aosr.py
│   │
│   ├── schemas/                # Pydantic схемы
│   │   ├── __init__.py
│   │   ├── project.py
│   │   ├── document.py
│   │   └── aosr.py
│   │
│   ├── api/                    # API endpoints
│   │   ├── __init__.py
│   │   ├── projects.py
│   │   ├── documents.py
│   │   ├── aosr.py
│   │   └── agents.py
│   │
│   ├── agents/                 # AI-агенты
│   │   ├── __init__.py
│   │   ├── rd_analyzer.py      # Анализ РД
│   │   ├── aosr_generator.py   # Генерация АОСР
│   │   ├── doc_searcher.py     # Поиск документов
│   │   ├── doc_generator.py    # Генерация документов
│   │   └── validator.py        # Проверка комплектности
│   │
│   ├── services/               # Бизнес-логика
│   │   ├── __init__.py
│   │   ├── ocr.py
│   │   ├── pdf_generator.py
│   │   └── file_storage.py
│   │
│   └── tasks/                  # Celery задачи
│       ├── __init__.py
│       ├── ocr_tasks.py
│       ├── generation_tasks.py
│       └── search_tasks.py
│
├── tests/                      # Тесты
│   ├── test_api.py
│   ├── test_agents.py
│   └── test_services.py
│
├── alembic/                    # Миграции БД
│   ├── versions/
│   └── env.py
│
├── requirements.txt
├── .env                        # Переменные окружения
└── README.md
```

---

## Конфигурация (settings)

```python
# app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Database
    DATABASE_URL: str = "postgresql://user:password@localhost/pto_db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # OpenAI
    OPENAI_API_KEY: str

    # File Storage
    UPLOAD_DIR: str = "./uploads"
    MAX_UPLOAD_SIZE: int = 50 * 1024 * 1024  # 50MB

    # Security
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60 * 24 * 7  # 7 дней

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## Альтернативные варианты

### Django + Django REST Framework

**Когда использовать:**
- Если нужна админ-панель из коробки
- Если команда знакома с Django
- Для более традиционного подхода

**Недостатки:**
- Медленнее FastAPI
- Больше boilerplate кода

---

### Node.js + Express + TypeScript

**Когда использовать:**
- Если команда знает JavaScript/TypeScript
- Если нужна максимальная производительность на I/O операциях

**Недостатки:**
- Меньше библиотек для ML/AI
- Сложнее интеграция с Python AI-стеком

---

## API Endpoints (примерная структура)

```
# Проекты
GET    /api/v1/projects              # Список проектов
POST   /api/v1/projects              # Создание проекта
GET    /api/v1/projects/{id}         # Получение проекта
PUT    /api/v1/projects/{id}         # Обновление проекта
DELETE /api/v1/projects/{id}         # Удаление проекта

# Документы
POST   /api/v1/documents/upload      # Загрузка документа
GET    /api/v1/documents             # Список документов
GET    /api/v1/documents/{id}        # Получение документа
DELETE /api/v1/documents/{id}        # Удаление документа

# АОСР
POST   /api/v1/aosr/generate         # Генерация АОСР
GET    /api/v1/aosr                  # Список актов
GET    /api/v1/aosr/{id}             # Получение акта
PUT    /api/v1/aosr/{id}             # Редактирование акта

# AI-агенты
POST   /api/v1/agents/analyze-rd     # Анализ РД
POST   /api/v1/agents/search-docs    # Поиск документов качества
POST   /api/v1/agents/validate       # Проверка комплектности

# Задачи (мониторинг Celery)
GET    /api/v1/tasks/{task_id}       # Статус задачи

# Экспорт
GET    /api/v1/export/pdf/{project_id}  # Скачать комплект ИД
```

---

## Аутентификация и авторизация

### JWT (JSON Web Tokens)

**Установка:**
```bash
pip install python-jose[cryptography] passlib[bcrypt]
```

**Реализация:**
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from datetime import datetime, timedelta

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)
    return encoded_jwt

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return user_id
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

---

## Логирование

```python
import logging
from logging.handlers import RotatingFileHandler

# Настройка логирования
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        RotatingFileHandler('app.log', maxBytes=10485760, backupCount=5),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

# Использование
logger.info("Запущена генерация АОСР для проекта ID: 123")
logger.error("Ошибка при обработке документа", exc_info=True)
```

---

## Рекомендации

### ✅ Лучшие практики

1. **Используйте async/await для I/O операций**
2. **Валидируйте все входные данные через Pydantic**
3. **Длительные задачи выносите в Celery**
4. **Используйте environment variables для secrets**
5. **Пишите unit-тесты (pytest)**
6. **Добавьте health check endpoint**
7. **Используйте миграции БД (Alembic)**

---

## Связь с другими компонентами

- **[Frontend](04-frontend.md):** REST API + WebSocket
- **[Database](08-database.md):** SQLAlchemy ORM
- **[AI/LLM](01-ai-llm.md):** Вызов агентов из API
- **[Cloud](07-cloud.md):** Деплой backend на сервер

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-13
