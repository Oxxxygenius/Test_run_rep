# Схема базы данных

> Полное описание всех таблиц PostgreSQL с полями, типами данных, связями и индексами

**Дата создания:** 2025-12-15
**Приоритет:** ⭐⭐⭐⭐⭐ (Критический — основа всей системы)

---

## Содержание

1. [ERD-диаграмма](#erd-диаграмма)
2. [Таблицы пользователей и проектов](#таблицы-пользователей-и-проектов)
3. [Таблицы документов](#таблицы-документов)
4. [Таблицы работ и материалов](#таблицы-работ-и-материалов)
5. [Таблицы АОСР](#таблицы-аоср)
6. [Таблицы документов качества](#таблицы-документов-качества)
7. [Таблицы валидации и логирования](#таблицы-валидации-и-логирования)
8. [Индексы и оптимизация](#индексы-и-оптимизация)

---

## ERD-диаграмма

```
┌──────────────────────────────────────────────────────────────────────────┐
│ ОСНОВНЫЕ СВЯЗИ МЕЖДУ ТАБЛИЦАМИ                                           │
└──────────────────────────────────────────────────────────────────────────┘

users (1) ──────< (M) projects
                        │
                        ├──────< (M) documents
                        │           │
                        │           └──────< (M) document_pages
                        │
                        ├──────< (M) work_types
                        │           │
                        │           └──────< (M) work_materials (связь M-M с materials)
                        │
                        ├──────< (M) materials
                        │           │
                        │           └──────< (M) quality_documents
                        │
                        ├──────< (M) aosr
                        │           │
                        │           ├──────< (M) aosr_quality_documents
                        │           └──────< (M) aosr_versions
                        │
                        ├──────< (M) packages
                        ├──────< (M) validation_reports
                        └──────< (M) event_log

material_categories (1) ──────< (M) materials
```

---

## Таблицы пользователей и проектов

### 1. users

Хранит информацию о пользователях системы.

```sql
CREATE TABLE users (
  -- Идентификация
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL, -- bcrypt hash

  -- Профиль
  full_name VARCHAR(255) NOT NULL,
  company_name VARCHAR(255),
  phone VARCHAR(50),

  -- Роли и права
  role VARCHAR(50) NOT NULL DEFAULT 'user', -- 'user', 'admin', 'manager'
  is_active BOOLEAN DEFAULT TRUE,
  is_verified BOOLEAN DEFAULT FALSE,

  -- Метаданные
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  last_login_at TIMESTAMP,

  -- Дополнительная информация
  settings JSONB DEFAULT '{}', -- Пользовательские настройки
  metadata JSONB DEFAULT '{}'  -- Дополнительные данные
);

-- Индексы
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
```

**Примечание:** Пароли хранятся в виде bcrypt hash (никогда в открытом виде).

---

### 2. projects

Основная таблица проектов.

```sql
CREATE TABLE projects (
  -- Идентификация
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,

  -- Основная информация
  name VARCHAR(500) NOT NULL,
  address TEXT NOT NULL,

  -- Участники строительства
  developer VARCHAR(500) NOT NULL,       -- Застройщик
  general_contractor VARCHAR(500) NOT NULL, -- Генподрядчик
  contractor VARCHAR(500) NOT NULL,      -- Подрядчик
  technical_customer VARCHAR(500),       -- Технический заказчик (опционально)

  -- Период выполнения работ
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,

  -- Формат комплекта ИД
  package_format VARCHAR(20) DEFAULT 'unified', -- 'repeated' или 'unified'

  -- Данные поставщика (для отказных писем)
  supplier_info JSONB DEFAULT '{}', -- {"name": "...", "inn": "...", "address": "..."}

  -- Выбранные шаблоны (раздел 3.1 в 06-user-actions-breakdown.md)
  custom_templates JSONB DEFAULT '{}', -- {"general_registry": "path/...", ...}

  -- Статус и метаданные
  status VARCHAR(50) DEFAULT 'created', -- 'created', 'in_progress', 'completed', 'archived'
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  metadata JSONB DEFAULT '{}'
);

-- Индексы
CREATE INDEX idx_projects_user_id ON projects(user_id);
CREATE INDEX idx_projects_status ON projects(status);
CREATE INDEX idx_projects_created_at ON projects(created_at DESC);
```

**Важные поля:**
- `package_format` — определяет вариант формирования комплекта ИД (см. [10-final-package-generation.md:59-191](10-final-package-generation.md))
- `supplier_info` — используется для генерации отказных писем (см. [09-document-priorities-dependencies.md:796-837](09-document-priorities-dependencies.md))
- `custom_templates` — пути к шаблонам, выбранным пользователем (см. [06-user-actions-breakdown.md:344-351](06-user-actions-breakdown.md))

---

## Таблицы документов

### 3. documents

Хранит все загруженные документы (РД, схемы, и т.д.).

```sql
CREATE TABLE documents (
  -- Идентификация
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,

  -- Файл
  file_name VARCHAR(500) NOT NULL,
  file_url TEXT NOT NULL,         -- S3 путь: s3://bucket/project_id/documents/doc_id.pdf
  file_size BIGINT,                -- В байтах
  file_hash VARCHAR(64),           -- SHA-256 hash для дедупликации

  -- Метаданные документа
  doc_type VARCHAR(50) NOT NULL,   -- 'РД', 'схема', 'working_documentation', и т.д.
  page_count INTEGER,

  -- Извлечённый текст (OCR)
  extracted_text TEXT,             -- Результат GPT-4o OCR

  -- Статус обработки
  status VARCHAR(50) DEFAULT 'uploaded', -- 'uploaded', 'processing', 'analyzed', 'error'
  ocr_progress INTEGER DEFAULT 0,  -- Прогресс OCR (0-100%)

  -- Метаданные и ошибки
  metadata JSONB DEFAULT '{}',
  error_message TEXT,

  -- Временные метки
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  processed_at TIMESTAMP           -- Когда завершена обработка
);

-- Индексы
CREATE INDEX idx_documents_project_id ON documents(project_id);
CREATE INDEX idx_documents_doc_type ON documents(doc_type);
CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_documents_file_hash ON documents(file_hash); -- Для дедупликации
CREATE INDEX idx_documents_extracted_text_gin ON documents USING GIN(to_tsvector('russian', extracted_text)); -- Full-text search
```

**Важно:**
- `extracted_text` — полный текст из PDF, извлечённый через GPT-4o
- `file_hash` — позволяет определить дубликаты перед загрузкой

---

### 4. document_pages (опционально)

Хранит отдельные страницы документов (для детального анализа).

```sql
CREATE TABLE document_pages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID REFERENCES documents(id) ON DELETE CASCADE,

  page_number INTEGER NOT NULL,
  page_text TEXT,                  -- Текст страницы
  page_image_url TEXT,             -- Ссылка на изображение страницы (если нужно)

  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_document_pages_document_id ON document_pages(document_id);
CREATE INDEX idx_document_pages_page_number ON document_pages(page_number);
```

---

## Таблицы работ и материалов

### 5. work_types

Виды работ, извлечённые из РД.

```sql
CREATE TABLE work_types (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,

  -- Информация о работе
  name VARCHAR(500) NOT NULL,      -- Название вида работ
  description TEXT,
  volume DECIMAL(12, 2),           -- Объём работ
  unit VARCHAR(50),                -- Единица измерения: м², м³, м, шт

  -- Нормативы
  gost VARCHAR(200),               -- ГОСТ, СП
  snip VARCHAR(200),               -- СНиП

  -- Даты
  start_date DATE,
  end_date DATE,

  -- Связь с документом, откуда извлечено
  source_document_id UUID REFERENCES documents(id) ON DELETE SET NULL,

  -- Метаданные
  metadata JSONB DEFAULT '{}',     -- Дополнительные параметры
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_work_types_project_id ON work_types(project_id);
CREATE INDEX idx_work_types_source_document_id ON work_types(source_document_id);
```

---

### 6. materials

Материалы, извлечённые из РД.

```sql
CREATE TABLE materials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,

  -- Информация о материале
  name VARCHAR(500) NOT NULL,
  manufacturer VARCHAR(255),
  model VARCHAR(255),              -- Модель/артикул

  -- Количество
  quantity DECIMAL(12, 2),
  unit VARCHAR(50),                -- м, м², шт, кг, и т.д.

  -- Категория (для определения требуемых документов)
  category_code VARCHAR(50) REFERENCES material_categories(category_code),

  -- Технические характеристики
  specifications JSONB DEFAULT '{}', -- {"diameter": "20mm", "pressure": "10bar", ...}

  -- Связь с работами (многие-ко-многим через work_materials)

  -- Источник данных
  source_document_id UUID REFERENCES documents(id) ON DELETE SET NULL,

  -- Метаданные
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_materials_project_id ON materials(project_id);
CREATE INDEX idx_materials_category_code ON materials(category_code);
CREATE INDEX idx_materials_manufacturer ON materials(manufacturer);
```

---

### 7. work_materials

Связь многие-ко-многим между работами и материалами.

```sql
CREATE TABLE work_materials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  work_type_id UUID REFERENCES work_types(id) ON DELETE CASCADE,
  material_id UUID REFERENCES materials(id) ON DELETE CASCADE,

  quantity DECIMAL(12, 2),         -- Количество материала для этой работы

  created_at TIMESTAMP DEFAULT NOW(),

  UNIQUE(work_type_id, material_id) -- Уникальная связь
);

-- Индексы
CREATE INDEX idx_work_materials_work_type_id ON work_materials(work_type_id);
CREATE INDEX idx_work_materials_material_id ON work_materials(material_id);
```

---

### 8. material_categories

Категории материалов с определением требуемых документов (см. [09-document-priorities-dependencies.md:862](09-document-priorities-dependencies.md)).

```sql
CREATE TABLE material_categories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  category_name VARCHAR(100) NOT NULL,
  category_code VARCHAR(50) NOT NULL UNIQUE, -- 'gkl', 'pipes_pp', 'fasteners', и т.д.

  -- Требуемые документы (массив типов)
  required_documents JSONB NOT NULL, -- ["Паспорт качества", "Сертификат соответствия", ...]

  -- Опциональные документы
  optional_documents JSONB DEFAULT '[]',

  -- Правила генерации
  generation_rules JSONB DEFAULT '{}', -- {"passport_from_cert": true, ...}

  created_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE UNIQUE INDEX idx_material_categories_code ON material_categories(category_code);
```

**Примеры записей:**

```sql
-- ГКЛ (гипсокартон)
INSERT INTO material_categories (category_code, category_name, required_documents, generation_rules)
VALUES (
  'gkl',
  'ГКЛ (гипсокартон)',
  '["Паспорт качества", "Сертификат соответствия", "СРГ", "Пожарный сертификат"]',
  '{"passport_from_cert": true, "refusal_letter_allowed": false}'
);

-- Крепёж
INSERT INTO material_categories (category_code, category_name, required_documents, optional_documents, generation_rules)
VALUES (
  'fasteners',
  'Крепёж и метизы',
  '["Паспорт качества"]',
  '["Сертификат соответствия", "Отказное письмо"]',
  '{"refusal_letter_allowed": true, "refusal_letter_instead_of_cert": true}'
);
```

---

## Таблицы АОСР

### 9. aosr

Акты освидетельствования скрытых работ.

```sql
CREATE TABLE aosr (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  work_type_id UUID REFERENCES work_types(id) ON DELETE SET NULL,

  -- Номер и дата
  number VARCHAR(100) NOT NULL,    -- Например: "01-КЖ6-1"
  work_date DATE NOT NULL,
  work_description TEXT NOT NULL,

  -- Файлы
  generated_pdf_path TEXT,         -- S3 путь к PDF
  schema_document_id UUID REFERENCES documents(id) ON DELETE SET NULL, -- Исполнительная схема

  -- Версионирование
  version INTEGER DEFAULT 1,
  parent_version_id UUID REFERENCES aosr(id) ON DELETE SET NULL, -- Ссылка на предыдущую версию

  -- Статус
  status VARCHAR(50) DEFAULT 'draft', -- 'draft', 'approved', 'rejected'
  approved_at TIMESTAMP,
  approved_by UUID REFERENCES users(id) ON DELETE SET NULL,

  -- Подписи
  signatures JSONB DEFAULT '{}',   -- {"contractor": "Иванов И.И.", "customer": "Петров П.П."}

  -- Метаданные
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_aosr_project_id ON aosr(project_id);
CREATE INDEX idx_aosr_work_type_id ON aosr(work_type_id);
CREATE INDEX idx_aosr_status ON aosr(status);
CREATE INDEX idx_aosr_number ON aosr(number);
```

**Важно:**
- `version` — номер версии АОСР (для отслеживания изменений)
- `parent_version_id` — ссылка на предыдущую версию (цепочка версий)

---

### 10. aosr_quality_documents

Связь многие-ко-многим между АОСР и документами качества.

```sql
CREATE TABLE aosr_quality_documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aosr_id UUID REFERENCES aosr(id) ON DELETE CASCADE,
  document_id UUID REFERENCES quality_documents(id) ON DELETE CASCADE,

  created_at TIMESTAMP DEFAULT NOW(),

  UNIQUE(aosr_id, document_id)
);

-- Индексы
CREATE INDEX idx_aosr_quality_documents_aosr_id ON aosr_quality_documents(aosr_id);
CREATE INDEX idx_aosr_quality_documents_document_id ON aosr_quality_documents(document_id);
```

---

### 11. aosr_versions

История изменений АОСР (опционально, для аудита).

```sql
CREATE TABLE aosr_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aosr_id UUID REFERENCES aosr(id) ON DELETE CASCADE,

  version_number INTEGER NOT NULL,
  changes_description TEXT,
  changed_by UUID REFERENCES users(id) ON DELETE SET NULL,

  -- Снимок данных на момент версии
  snapshot JSONB NOT NULL,         -- Полная копия данных АОСР

  created_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_aosr_versions_aosr_id ON aosr_versions(aosr_id);
CREATE INDEX idx_aosr_versions_version_number ON aosr_versions(version_number);
```

---

## Таблицы документов качества

### 12. quality_documents

Документы качества (сертификаты, паспорта, и т.д.).

```sql
CREATE TABLE quality_documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  material_id UUID REFERENCES materials(id) ON DELETE SET NULL,

  -- Тип документа
  document_type VARCHAR(100) NOT NULL, -- 'Сертификат', 'СРГ', 'Паспорт качества', 'Тех.паспорт', 'Пожарный сертификат', 'Отказное письмо'

  -- Файл
  file_url TEXT NOT NULL,
  file_name VARCHAR(500),
  file_size BIGINT,

  -- Реквизиты документа
  doc_number VARCHAR(200),         -- Номер сертификата/паспорта
  issue_date DATE,                 -- Дата выдачи
  expiry_date DATE,                -- Срок действия (если применимо)
  issuer VARCHAR(500),             -- Орган выдачи

  -- Источник документа
  source VARCHAR(200) NOT NULL,    -- 'найден в базе', 'специализированный сайт (santech.ru)', 'найден в интернете (Google)', 'сгенерирован на основе Сертификата'

  -- Для сгенерированных документов
  generated_from_doc_id UUID REFERENCES quality_documents(id) ON DELETE SET NULL,
  generated_from_cert_number VARCHAR(200),

  -- Валидация
  needs_review BOOLEAN DEFAULT FALSE, -- Требуется проверка пользователем
  confidence DECIMAL(3, 2),        -- Уверенность AI (0.00 - 1.00)
  validated_by UUID REFERENCES users(id) ON DELETE SET NULL,
  validated_at TIMESTAMP,

  -- Метаданные
  metadata JSONB DEFAULT '{}',     -- Дополнительная информация (материал, производитель, и т.д.)

  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_quality_documents_project_id ON quality_documents(project_id);
CREATE INDEX idx_quality_documents_material_id ON quality_documents(material_id);
CREATE INDEX idx_quality_documents_document_type ON quality_documents(document_type);
CREATE INDEX idx_quality_documents_doc_number ON quality_documents(doc_number);
CREATE INDEX idx_quality_documents_source ON quality_documents(source);
```

**Важные поля:**
- `source` — откуда получен документ (см. [09-document-priorities-dependencies.md:360-367](09-document-priorities-dependencies.md))
- `generated_from_doc_id` — ID документа, на основе которого сгенерирован текущий (паспорт из сертификата)
- `needs_review` — флаг "требуется проверка" для сгенерированных AI документов

---

## Таблицы валидации и логирования

### 13. validation_reports

Отчёты валидации проектов.

```sql
CREATE TABLE validation_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,

  -- Статус валидации
  overall_status VARCHAR(50) NOT NULL, -- 'passed', 'passed_with_warnings', 'failed'

  -- Результаты проверок
  results JSONB NOT NULL,          -- {"document_completeness": "passed", "date_consistency": "passed", ...}

  -- Статистика
  critical_errors INTEGER DEFAULT 0,
  warnings INTEGER DEFAULT 0,
  checks_passed INTEGER DEFAULT 0,

  -- Детали
  error_details JSONB DEFAULT '[]',
  warning_details JSONB DEFAULT '[]',

  -- Рекомендации
  recommendations TEXT,

  created_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_validation_reports_project_id ON validation_reports(project_id);
CREATE INDEX idx_validation_reports_overall_status ON validation_reports(overall_status);
CREATE INDEX idx_validation_reports_created_at ON validation_reports(created_at DESC);
```

---

### 14. packages

Финальные пакеты ИД.

```sql
CREATE TABLE packages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,

  -- Файлы
  pdf_url TEXT NOT NULL,           -- Ссылка на PDF
  zip_url TEXT NOT NULL,           -- Ссылка на ZIP архив

  pdf_size BIGINT,                 -- Размер PDF в байтах
  zip_size BIGINT,                 -- Размер ZIP в байтах
  page_count INTEGER,              -- Количество страниц в PDF

  -- Статус
  status VARCHAR(50) DEFAULT 'generating', -- 'generating', 'ready', 'error'

  -- Временные ссылки
  download_links_expires_at TIMESTAMP, -- Срок действия ссылок на скачивание

  -- Метаданные
  metadata JSONB DEFAULT '{}',
  error_message TEXT,

  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_packages_project_id ON packages(project_id);
CREATE INDEX idx_packages_status ON packages(status);
CREATE INDEX idx_packages_created_at ON packages(created_at DESC);
```

---

### 15. event_log

Лог всех событий в системе (аудит).

```sql
CREATE TABLE event_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Контекст
  user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  project_id UUID REFERENCES projects(id) ON DELETE SET NULL,

  -- Событие
  event_type VARCHAR(100) NOT NULL, -- 'document_uploaded', 'aosr_generated', 'package_created', и т.д.
  event_action VARCHAR(100),       -- 'create', 'update', 'delete', 'read'

  -- Детали
  entity_type VARCHAR(100),        -- 'document', 'aosr', 'material', и т.д.
  entity_id UUID,

  -- Данные события
  event_data JSONB DEFAULT '{}',   -- Дополнительные данные события

  -- IP и метаданные
  ip_address INET,
  user_agent TEXT,

  created_at TIMESTAMP DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_event_log_user_id ON event_log(user_id);
CREATE INDEX idx_event_log_project_id ON event_log(project_id);
CREATE INDEX idx_event_log_event_type ON event_log(event_type);
CREATE INDEX idx_event_log_created_at ON event_log(created_at DESC);
```

---

## Индексы и оптимизация

### Композитные индексы

Для часто используемых запросов:

```sql
-- Поиск документов проекта по типу и статусу
CREATE INDEX idx_documents_project_type_status
ON documents(project_id, doc_type, status);

-- Поиск материалов проекта по категории
CREATE INDEX idx_materials_project_category
ON materials(project_id, category_code);

-- Поиск АОСР проекта по статусу
CREATE INDEX idx_aosr_project_status
ON aosr(project_id, status);

-- Поиск документов качества материала по типу
CREATE INDEX idx_quality_documents_material_type
ON quality_documents(material_id, document_type);
```

---

### Полнотекстовый поиск

Для поиска по извлечённому тексту документов:

```sql
-- GIN индекс для full-text search (уже создан выше)
CREATE INDEX idx_documents_extracted_text_gin
ON documents USING GIN(to_tsvector('russian', extracted_text));

-- Пример запроса
SELECT * FROM documents
WHERE to_tsvector('russian', extracted_text) @@ to_tsquery('russian', 'отопление & труба');
```

---

### Партиционирование (для больших проектов)

Для очень больших таблиц (например, `event_log`):

```sql
-- Партиционирование event_log по месяцам
CREATE TABLE event_log (
  ...
) PARTITION BY RANGE (created_at);

CREATE TABLE event_log_2024_12 PARTITION OF event_log
FOR VALUES FROM ('2024-12-01') TO ('2025-01-01');

CREATE TABLE event_log_2025_01 PARTITION OF event_log
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

---

## Триггеры для автоматического обновления

### Автоматическое обновление updated_at

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ language 'plpgsql';

-- Применяем к таблицам
CREATE TRIGGER update_projects_updated_at BEFORE UPDATE ON projects
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_documents_updated_at BEFORE UPDATE ON documents
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_materials_updated_at BEFORE UPDATE ON materials
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_work_types_updated_at BEFORE UPDATE ON work_types
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_aosr_updated_at BEFORE UPDATE ON aosr
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_quality_documents_updated_at BEFORE UPDATE ON quality_documents
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

---

## Примеры запросов

### Получить все материалы проекта с документами качества

```sql
SELECT
  m.id,
  m.name AS material_name,
  m.manufacturer,
  m.quantity,
  m.unit,
  mc.category_name,
  json_agg(
    json_build_object(
      'type', qd.document_type,
      'number', qd.doc_number,
      'source', qd.source,
      'url', qd.file_url
    )
  ) AS quality_documents
FROM materials m
LEFT JOIN material_categories mc ON m.category_code = mc.category_code
LEFT JOIN quality_documents qd ON m.id = qd.material_id
WHERE m.project_id = 'project_123'
GROUP BY m.id, m.name, m.manufacturer, m.quantity, m.unit, mc.category_name;
```

### Получить все АОСР с материалами и документами

```sql
SELECT
  a.id,
  a.number AS aosr_number,
  a.work_description,
  a.status,
  wt.name AS work_type_name,
  json_agg(DISTINCT m.name) AS materials,
  count(DISTINCT aqd.document_id) AS quality_docs_count
FROM aosr a
LEFT JOIN work_types wt ON a.work_type_id = wt.id
LEFT JOIN work_materials wm ON wt.id = wm.work_type_id
LEFT JOIN materials m ON wm.material_id = m.id
LEFT JOIN aosr_quality_documents aqd ON a.id = aqd.aosr_id
WHERE a.project_id = 'project_123'
GROUP BY a.id, a.number, a.work_description, a.status, wt.name
ORDER BY a.number;
```

---

## Миграции

Используйте Alembic для управления миграциями:

```python
# migrations/versions/001_initial_schema.py
def upgrade():
    # Создание всех таблиц
    op.create_table('users', ...)
    op.create_table('projects', ...)
    # и т.д.

def downgrade():
    op.drop_table('packages')
    op.drop_table('validation_reports')
    # и т.д. (в обратном порядке)
```

---

## Связь с другими документами

- [01-how-it-works.md](01-how-it-works.md) — как используются данные из БД
- [02-data-flow.md](02-data-flow.md) — трансформация данных между таблицами
- [06-user-actions-breakdown.md](06-user-actions-breakdown.md) — какие таблицы обновляются на каждом шаге
- [09-document-priorities-dependencies.md](09-document-priorities-dependencies.md) — таблица `material_categories`

---

**Статус:** ✅ Готово к реализации
**Последнее обновление:** 2025-12-15
