# База данных

**Дата создания:** 2025-12-13
**Приоритет:** ⭐⭐⭐⭐⭐ (Критический)

---

## Назначение

База данных для хранения:
- Проектов
- Документов и их метаданных
- АОСР и других актов
- Пользователей
- История генераций

---

## Рекомендуемое решение

### PostgreSQL 15

**Почему PostgreSQL:**
- ✅ Поддержка JSON (для гибких структур)
- ✅ Надежность и ACID
- ✅ Полнотекстовый поиск
- ✅ Отличная поддержка в Python (SQLAlchemy, psycopg2)

---

## Схема базы данных

### Основные таблицы

```sql
-- Пользователи
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    role VARCHAR(50) DEFAULT 'user',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Проекты
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address TEXT NOT NULL,
    client VARCHAR(255),  -- Заказчик
    contractor VARCHAR(255),  -- Подрядчик
    general_contractor VARCHAR(255),  -- Генподрядчик
    developer VARCHAR(255),  -- Застройщик
    package_format VARCHAR(20) DEFAULT 'unified',  -- Формат комплекта ИД: 'repeated' или 'unified'
    custom_templates JSONB,  -- Пользовательские шаблоны для АОСР и реестров
    status VARCHAR(50) DEFAULT 'active', -- active, completed, archived
    created_by INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Документы
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    filename VARCHAR(255) NOT NULL,
    file_path VARCHAR(500) NOT NULL, -- Путь в Object Storage
    doc_type VARCHAR(100), -- сертификат, паспорт, декларация, РД, схема
    mime_type VARCHAR(100),
    file_size BIGINT,

    -- OCR данные
    ocr_text TEXT,
    ocr_confidence FLOAT,

    -- Извлеченные метаданные (JSON)
    metadata JSONB,
    -- Пример: {
    --   "document_number": "...",
    --   "issue_date": "2024-01-15",
    --   "expiry_date": "2027-01-15",
    --   "manufacturer": "...",
    --   "material_name": "...",
    --   "gost": "..."
    -- }

    uploaded_by INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- АОСР
CREATE TABLE aosr (
    id SERIAL PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    number VARCHAR(100) NOT NULL,
    work_type VARCHAR(255) NOT NULL,
    work_description TEXT,
    work_date DATE,

    -- Структурированное содержимое (JSON)
    content JSONB NOT NULL,
    -- Пример: {
    --   "works": [
    --     {"name": "Монтаж труб", "volume": 50, "unit": "м"}
    --   ],
    --   "materials": [
    --     {"name": "Труба ПНД", "quantity": 50, "unit": "м"}
    --   ],
    --   "responsible_persons": {...}
    -- }

    -- Связанные документы
    schema_document_id INTEGER REFERENCES documents(id),

    status VARCHAR(50) DEFAULT 'draft', -- draft, generated, approved
    generated_pdf_path VARCHAR(500),

    created_by INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Связь АОСР с документами качества
CREATE TABLE aosr_quality_documents (
    id SERIAL PRIMARY KEY,
    aosr_id INTEGER REFERENCES aosr(id) ON DELETE CASCADE,
    document_id INTEGER REFERENCES documents(id) ON DELETE CASCADE,
    material_name VARCHAR(255),
    relevance_score FLOAT, -- 0-1, насколько документ подходит
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(aosr_id, document_id)
);

-- Задачи генерации (для мониторинга Celery)
CREATE TABLE generation_tasks (
    id SERIAL PRIMARY KEY,
    task_id VARCHAR(255) UNIQUE NOT NULL, -- Celery task ID
    task_type VARCHAR(100) NOT NULL, -- generate_aosr, search_docs, etc
    project_id INTEGER REFERENCES projects(id),
    aosr_id INTEGER REFERENCES aosr(id),

    status VARCHAR(50) DEFAULT 'pending', -- pending, running, completed, failed
    progress INTEGER DEFAULT 0, -- 0-100
    result JSONB,
    error_message TEXT,

    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- История поиска документов
CREATE TABLE document_search_history (
    id SERIAL PRIMARY KEY,
    material_name VARCHAR(255) NOT NULL,
    manufacturer VARCHAR(255),
    gost VARCHAR(100),

    found_documents JSONB, -- Массив найденных документов
    search_duration INTEGER, -- миллисекунды

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Индексы для производительности
CREATE INDEX idx_documents_project_id ON documents(project_id);
CREATE INDEX idx_documents_doc_type ON documents(doc_type);
CREATE INDEX idx_aosr_project_id ON aosr(project_id);
CREATE INDEX idx_aosr_status ON aosr(status);
CREATE INDEX idx_generation_tasks_status ON generation_tasks(status);

-- Полнотекстовый поиск
CREATE INDEX idx_documents_ocr_text ON documents USING GIN(to_tsvector('russian', ocr_text));
```

---

## SQLAlchemy модели

```python
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, Float, Date, BigInteger
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True, nullable=False)
    password_hash = Column(String(255), nullable=False)
    full_name = Column(String(255))
    role = Column(String(50), default='user')
    created_at = Column(DateTime, default=datetime.utcnow)

    projects = relationship("Project", back_populates="creator")

class Project(Base):
    __tablename__ = "projects"

    id = Column(Integer, primary_key=True)
    name = Column(String(255), nullable=False)
    address = Column(Text, nullable=False)
    client = Column(String(255))  # Заказчик
    contractor = Column(String(255))  # Подрядчик
    general_contractor = Column(String(255))  # Генподрядчик
    developer = Column(String(255))  # Застройщик
    package_format = Column(String(20), default='unified')  # Формат комплекта ИД: 'repeated' или 'unified'
    custom_templates = Column(JSONB)  # Пользовательские шаблоны для АОСР и реестров (JSONB для PostgreSQL)
    status = Column(String(50), default='active')
    created_by = Column(Integer, ForeignKey('users.id'))
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    creator = relationship("User", back_populates="projects")
    documents = relationship("Document", back_populates="project", cascade="all, delete-orphan")
    aosr_list = relationship("AOSR", back_populates="project", cascade="all, delete-orphan")

class Document(Base):
    __tablename__ = "documents"

    id = Column(Integer, primary_key=True)
    project_id = Column(Integer, ForeignKey('projects.id'))
    filename = Column(String(255), nullable=False)
    file_path = Column(String(500), nullable=False)
    doc_type = Column(String(100))
    mime_type = Column(String(100))
    file_size = Column(BigInteger)
    ocr_text = Column(Text)
    ocr_confidence = Column(Float)
    metadata = Column(JSONB)
    uploaded_by = Column(Integer, ForeignKey('users.id'))
    created_at = Column(DateTime, default=datetime.utcnow)

    project = relationship("Project", back_populates="documents")

class AOSR(Base):
    __tablename__ = "aosr"

    id = Column(Integer, primary_key=True)
    project_id = Column(Integer, ForeignKey('projects.id'))
    number = Column(String(100), nullable=False)
    work_type = Column(String(255), nullable=False)
    work_description = Column(Text)
    work_date = Column(Date)
    content = Column(JSONB, nullable=False)
    schema_document_id = Column(Integer, ForeignKey('documents.id'))
    status = Column(String(50), default='draft')
    generated_pdf_path = Column(String(500))
    created_by = Column(Integer, ForeignKey('users.id'))
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    project = relationship("Project", back_populates="aosr_list")
    quality_documents = relationship("AOSRQualityDocument", back_populates="aosr")
```

---

## Миграции (Alembic)

**Установка:**
```bash
pip install alembic
alembic init alembic
```

**Создание миграции:**
```bash
alembic revision --autogenerate -m "Initial tables"
alembic upgrade head
```

---

## Примеры запросов

### Получение проекта со всеми АОСР

```python
from sqlalchemy.orm import Session, joinedload

def get_project_with_aosr(session: Session, project_id: int):
    project = session.query(Project)\
        .options(joinedload(Project.aosr_list))\
        .filter(Project.id == project_id)\
        .first()
    return project
```

### Поиск документов по тексту (полнотекстовый поиск)

```python
from sqlalchemy import func

def search_documents_by_text(session: Session, query: str):
    results = session.query(Document)\
        .filter(func.to_tsvector('russian', Document.ocr_text)\
        .match(query))\
        .all()
    return results
```

### Статистика по проекту

```python
def get_project_stats(session: Session, project_id: int):
    from sqlalchemy import func

    stats = {
        'total_documents': session.query(func.count(Document.id))\
            .filter(Document.project_id == project_id).scalar(),
        'total_aosr': session.query(func.count(AOSR.id))\
            .filter(AOSR.project_id == project_id).scalar(),
        'aosr_by_status': session.query(AOSR.status, func.count(AOSR.id))\
            .filter(AOSR.project_id == project_id)\
            .group_by(AOSR.status).all()
    }
    return stats
```

---

## Бэкапы

### Автоматический бэкап PostgreSQL

```bash
#!/bin/bash
# backup.sh

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_DIR="/backups/postgres"
DB_NAME="pto_db"

pg_dump $DB_NAME | gzip > $BACKUP_DIR/pto_db_$TIMESTAMP.sql.gz

# Удалить бэкапы старше 30 дней
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete
```

**Cron job:**
```bash
# Ежедневно в 2:00 ночи
0 2 * * * /opt/scripts/backup.sh
```

---

## Оптимизация производительности

### 1. Индексы

```sql
-- JSONB индекс для быстрого поиска по метаданным
CREATE INDEX idx_documents_metadata_gin ON documents USING GIN(metadata);

-- Пример поиска
SELECT * FROM documents
WHERE metadata @> '{"manufacturer": "Полипластик"}';
```

### 2. Партиционирование (для больших таблиц)

```sql
-- Партиционирование documents по project_id (если много проектов)
CREATE TABLE documents (
    ...
) PARTITION BY HASH (project_id);

CREATE TABLE documents_p0 PARTITION OF documents FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE documents_p1 PARTITION OF documents FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE documents_p2 PARTITION OF documents FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE documents_p3 PARTITION OF documents FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

## Связь с другими компонентами

- **[Backend](03-backend.md):** SQLAlchemy ORM
- **[Cloud](07-cloud.md):** Managed PostgreSQL
- **[OCR](02-ocr.md):** Хранение OCR результатов

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-13
