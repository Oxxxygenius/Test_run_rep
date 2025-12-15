# REST API Reference

**Дата создания:** 2025-12-15
**Статус:** Актуально
**Версия API:** v1

---

## 📖 Оглавление

1. [Аутентификация](#аутентификация)
2. [Проекты](#проекты)
3. [Документы](#документы)
4. [Анализ РД](#анализ-рд)
5. [АОСР](#аоср)
6. [Документы качества](#документы-качества)
7. [Финальные пакеты](#финальные-пакеты)
8. [Валидация](#валидация)
9. [Пользователи](#пользователи)
10. [Коды ошибок](#коды-ошибок)

---

## Базовая информация

**Base URL:** `https://api.pto-platform.ru/api/v1`

**Аутентификация:** JWT Bearer Token

**Content-Type:** `application/json`

**Формат ответа:**
```json
{
  "success": true,
  "data": { ... },
  "message": "Успешно",
  "timestamp": "2025-12-15T10:30:00Z"
}
```

**Формат ошибки:**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Некорректные данные",
    "details": {
      "field": "email",
      "issue": "Некорректный формат email"
    }
  },
  "timestamp": "2025-12-15T10:30:00Z"
}
```

---

## Аутентификация

### POST /auth/register

Регистрация нового пользователя.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecureP@ss123",
  "full_name": "Иван Иванов",
  "company": "ООО Строй",
  "role": "engineer"
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "user_id": 123,
    "email": "user@example.com",
    "full_name": "Иван Иванов",
    "role": "engineer",
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
    "expires_in": 3600
  }
}
```

---

### POST /auth/login

Вход в систему.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecureP@ss123"
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
    "expires_in": 3600,
    "user": {
      "id": 123,
      "email": "user@example.com",
      "full_name": "Иван Иванов",
      "role": "engineer"
    }
  }
}
```

**cURL Example:**
```bash
curl -X POST https://api.pto-platform.ru/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecureP@ss123"
  }'
```

---

### POST /auth/refresh

Обновление access token.

**Request:**
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "expires_in": 3600
  }
}
```

---

### POST /auth/logout

Выход из системы (инвалидация токенов).

**Headers:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**Response: 200 OK**
```json
{
  "success": true,
  "message": "Успешно вышли из системы"
}
```

---

## Проекты

### GET /projects

Получить список всех проектов текущего пользователя.

**Headers:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**Query Parameters:**
- `page` (int, optional): Номер страницы (default: 1)
- `per_page` (int, optional): Элементов на странице (default: 20, max: 100)
- `status` (string, optional): Фильтр по статусу (`active`, `archived`, `completed`)
- `sort_by` (string, optional): Поле для сортировки (`created_at`, `updated_at`, `name`)
- `order` (string, optional): Направление сортировки (`asc`, `desc`)

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "projects": [
      {
        "id": "proj_123",
        "name": "ЖК Южный",
        "address": "г. Москва, ул. Южная, д. 10",
        "client": "ООО Застройщик",
        "contractor": "ООО Подрядчик",
        "package_format": "unified",
        "status": "active",
        "created_at": "2025-12-10T10:00:00Z",
        "updated_at": "2025-12-15T12:30:00Z",
        "stats": {
          "documents_count": 15,
          "aosr_count": 10,
          "quality_docs_count": 25
        }
      }
    ],
    "pagination": {
      "page": 1,
      "per_page": 20,
      "total": 50,
      "total_pages": 3
    }
  }
}
```

**cURL Example:**
```bash
curl -X GET "https://api.pto-platform.ru/api/v1/projects?page=1&per_page=20" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

### POST /projects

Создать новый проект.

**Request:**
```json
{
  "name": "ЖК Южный",
  "address": "г. Москва, ул. Южная, д. 10",
  "client": "ООО Застройщик",
  "contractor": "ООО Подрядчик",
  "general_contractor": "ООО Генподрядчик",
  "developer": "ООО Девелопер",
  "package_format": "unified",
  "supplier_info": {
    "name": "ООО ТоргСнаб",
    "address": "г. Москва, ул. Складская, д. 5",
    "phone": "+7 (495) 123-45-67"
  }
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "id": "proj_124",
    "name": "ЖК Южный",
    "address": "г. Москва, ул. Южная, д. 10",
    "client": "ООО Застройщик",
    "status": "active",
    "package_format": "unified",
    "created_at": "2025-12-15T12:00:00Z"
  }
}
```

---

### GET /projects/{project_id}

Получить детали проекта.

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": "proj_123",
    "name": "ЖК Южный",
    "address": "г. Москва, ул. Южная, д. 10",
    "client": "ООО Застройщик",
    "contractor": "ООО Подрядчик",
    "package_format": "unified",
    "status": "active",
    "custom_templates": {
      "general_registry": "storage/projects/123/templates/04_Шаблон общего реестра.xlsx",
      "quality_registry": "storage/projects/123/templates/04_Шаблон реестра к АОСР.xlsx",
      "aosr_form": "storage/projects/123/templates/04_Форма АОСР.xlsx"
    },
    "stats": {
      "documents_count": 15,
      "aosr_count": 10,
      "quality_docs_count": 25,
      "packages_count": 2
    },
    "created_at": "2025-12-10T10:00:00Z",
    "updated_at": "2025-12-15T12:30:00Z"
  }
}
```

---

### PATCH /projects/{project_id}

Обновить проект.

**Request:**
```json
{
  "name": "ЖК Южный (обновленное название)",
  "package_format": "repeated",
  "supplier_info": {
    "name": "ООО НовыйПоставщик"
  }
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": "proj_123",
    "name": "ЖК Южный (обновленное название)",
    "package_format": "repeated",
    "updated_at": "2025-12-15T13:00:00Z"
  }
}
```

---

### DELETE /projects/{project_id}

Удалить проект (мягкое удаление).

**Response: 200 OK**
```json
{
  "success": true,
  "message": "Проект успешно удален"
}
```

---

## Документы

### POST /projects/{project_id}/documents/upload

Загрузить документы (РД) в проект.

**Request:** multipart/form-data
```
files[]: [file1.pdf, file2.pdf, file3.pdf]
doc_type: working_documentation
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "uploaded": [
      {
        "id": "doc_456",
        "file_name": "Спецификация_1.pdf",
        "file_size": 2548000,
        "page_count": 15,
        "doc_type": "working_documentation",
        "status": "processing",
        "file_url": "storage/projects/123/documents/doc_456.pdf"
      }
    ],
    "task_id": "task_789"
  },
  "message": "Файлы загружены, началась обработка"
}
```

**cURL Example:**
```bash
curl -X POST "https://api.pto-platform.ru/api/v1/projects/proj_123/documents/upload" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -F "files[]=@Спецификация_1.pdf" \
  -F "files[]=@Спецификация_2.pdf" \
  -F "doc_type=working_documentation"
```

---

### GET /projects/{project_id}/documents

Получить список документов проекта.

**Query Parameters:**
- `doc_type` (string, optional): Фильтр по типу документа
- `status` (string, optional): Фильтр по статусу (`uploaded`, `processing`, `analyzed`, `error`)

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "documents": [
      {
        "id": "doc_456",
        "file_name": "Спецификация_1.pdf",
        "file_size": 2548000,
        "page_count": 15,
        "doc_type": "working_documentation",
        "status": "analyzed",
        "extracted_text_length": 45000,
        "created_at": "2025-12-15T10:00:00Z",
        "analyzed_at": "2025-12-15T10:05:00Z"
      }
    ]
  }
}
```

---

### DELETE /documents/{document_id}

Удалить документ.

**Response: 200 OK**
```json
{
  "success": true,
  "message": "Документ успешно удален"
}
```

---

## Анализ РД

### POST /projects/{project_id}/analyze

Запустить анализ загруженных РД.

**Request:**
```json
{
  "document_ids": ["doc_456", "doc_457"],
  "options": {
    "extract_work_types": true,
    "extract_materials": true,
    "auto_create_aosr_drafts": true
  }
}
```

**Response: 202 Accepted**
```json
{
  "success": true,
  "data": {
    "task_id": "task_890",
    "status": "processing",
    "estimated_time": 300
  },
  "message": "Анализ РД запущен"
}
```

---

### GET /projects/{project_id}/analysis/status

Проверить статус анализа.

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "task_id": "task_890",
    "status": "completed",
    "progress": 100,
    "results": {
      "work_types_found": 12,
      "materials_found": 35,
      "aosr_drafts_created": 12
    },
    "completed_at": "2025-12-15T10:10:00Z"
  }
}
```

---

### GET /projects/{project_id}/work-types

Получить виды работ из анализа РД.

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "work_types": [
      {
        "id": "wt_1",
        "name": "Монтаж трубопроводов водоснабжения",
        "volume": 500,
        "unit": "м",
        "gost": "СП 73.13330.2016",
        "extracted_from": "doc_456",
        "materials": ["mat_1", "mat_2", "mat_3"]
      }
    ]
  }
}
```

---

### GET /projects/{project_id}/materials

Получить материалы из анализа РД.

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "materials": [
      {
        "id": "mat_1",
        "name": "Труба REHAU d=16мм",
        "manufacturer": "REHAU",
        "quantity": 500,
        "unit": "м",
        "category": "Трубы и фитинги",
        "required_documents": ["Сертификат", "Паспорт качества"],
        "quality_docs_found": 2,
        "quality_docs_missing": 0
      }
    ]
  }
}
```

---

## АОСР

### POST /projects/{project_id}/aosr/generate

Сгенерировать АОСР.

**Request:**
```json
{
  "work_type_ids": ["wt_1", "wt_2"],
  "options": {
    "use_custom_template": true,
    "include_quality_docs": true,
    "format": "pdf"
  }
}
```

**Response: 202 Accepted**
```json
{
  "success": true,
  "data": {
    "task_id": "task_901",
    "status": "processing",
    "estimated_time": 120
  },
  "message": "Генерация АОСР запущена"
}
```

---

### GET /projects/{project_id}/aosr

Получить список АОСР.

**Query Parameters:**
- `status` (string, optional): Фильтр по статусу (`draft`, `generated`, `approved`)

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "aosr_list": [
      {
        "id": "aosr_1",
        "work_type": "Монтаж трубопроводов водоснабжения",
        "number": "АОСР-001",
        "status": "approved",
        "file_url": "storage/projects/123/aosr/aosr_1.pdf",
        "file_size": 1250000,
        "quality_docs_count": 3,
        "created_at": "2025-12-15T11:00:00Z",
        "approved_at": "2025-12-15T12:00:00Z"
      }
    ]
  }
}
```

---

### GET /aosr/{aosr_id}

Получить детали АОСР.

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": "aosr_1",
    "number": "АОСР-001",
    "work_type": "Монтаж трубопроводов водоснабжения",
    "status": "approved",
    "content": {
      "work_description": "Монтаж трубопроводов",
      "volume": 500,
      "unit": "м",
      "hidden_work_location": "Этаж 1, секция А"
    },
    "quality_documents": ["qd_1", "qd_2", "qd_3"],
    "file_url": "storage/projects/123/aosr/aosr_1.pdf",
    "versions": [
      {
        "version": 1,
        "created_at": "2025-12-15T11:00:00Z",
        "file_url": "storage/projects/123/aosr/aosr_1_v1.pdf"
      }
    ]
  }
}
```

---

### PATCH /aosr/{aosr_id}

Обновить АОСР.

**Request:**
```json
{
  "status": "approved",
  "content": {
    "work_description": "Монтаж трубопроводов (уточнено)"
  }
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": "aosr_1",
    "status": "approved",
    "version": 2,
    "updated_at": "2025-12-15T13:00:00Z"
  }
}
```

---

### DELETE /aosr/{aosr_id}

Удалить АОСР.

**Response: 200 OK**
```json
{
  "success": true,
  "message": "АОСР успешно удален"
}
```

---

## Документы качества

### POST /projects/{project_id}/quality-documents/search

Запустить поиск документов качества.

**Request:**
```json
{
  "material_ids": ["mat_1", "mat_2"],
  "search_strategy": {
    "check_database": true,
    "check_specialized_sites": true,
    "use_google_search": true,
    "auto_generate_if_not_found": false
  }
}
```

**Response: 202 Accepted**
```json
{
  "success": true,
  "data": {
    "task_id": "task_912",
    "status": "processing",
    "estimated_time": 180
  },
  "message": "Поиск документов качества запущен"
}
```

---

### GET /projects/{project_id}/quality-documents

Получить документы качества.

**Query Parameters:**
- `material_id` (string, optional): Фильтр по материалу
- `document_type` (string, optional): Тип документа (`certificate`, `quality_passport`, `srg`, `technical_passport`, `fire_cert`, `refusal_letter`)
- `source` (string, optional): Источник (`database`, `specialized_site`, `google`, `ai_generated`)

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "quality_documents": [
      {
        "id": "qd_1",
        "material_id": "mat_1",
        "material_name": "Труба REHAU d=16мм",
        "document_type": "certificate",
        "document_number": "С-RU.АГ76.В.03581",
        "valid_until": "2026-05-15",
        "source": "специализированный сайт (santech.ru)",
        "confidence": 0.95,
        "file_url": "storage/projects/123/quality_docs/qd_1.pdf",
        "found_at": "2025-12-15T11:30:00Z"
      }
    ]
  }
}
```

---

### POST /projects/{project_id}/quality-documents/upload

Загрузить документ качества вручную.

**Request:** multipart/form-data
```
file: certificate.pdf
material_id: mat_1
document_type: certificate
document_number: С-RU.АГ76.В.03581
valid_until: 2026-05-15
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "id": "qd_10",
    "material_id": "mat_1",
    "document_type": "certificate",
    "file_url": "storage/projects/123/quality_docs/qd_10.pdf",
    "source": "uploaded_manually"
  }
}
```

---

### DELETE /quality-documents/{qd_id}

Удалить документ качества.

**Response: 200 OK**
```json
{
  "success": true,
  "message": "Документ качества успешно удален"
}
```

---

## Финальные пакеты

### POST /projects/{project_id}/packages/generate

Сгенерировать финальный пакет ИД.

**Request:**
```json
{
  "package_format": "unified",
  "options": {
    "include_editable_files": true,
    "include_registries": true,
    "include_title_page": true,
    "generate_zip_archive": true
  }
}
```

**Response: 202 Accepted**
```json
{
  "success": true,
  "data": {
    "task_id": "task_920",
    "status": "processing",
    "estimated_time": 60
  },
  "message": "Генерация финального пакета запущена"
}
```

---

### GET /projects/{project_id}/packages

Получить список пакетов.

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "packages": [
      {
        "id": "pkg_1",
        "package_format": "unified",
        "status": "completed",
        "pdf_file_url": "storage/projects/123/packages/pkg_1/final.pdf",
        "zip_file_url": "storage/projects/123/packages/pkg_1/archive.zip",
        "pdf_size": 15840000,
        "zip_size": 25600000,
        "total_pages": 250,
        "structure": {
          "title_page": 1,
          "general_registry": 3,
          "aosr_count": 10,
          "quality_docs_count": 25,
          "schemas_count": 5
        },
        "created_at": "2025-12-15T14:00:00Z"
      }
    ]
  }
}
```

---

### GET /packages/{package_id}/download

Скачать финальный пакет.

**Query Parameters:**
- `type` (string): Тип файла (`pdf`, `zip`)

**Response: 302 Redirect**
Перенаправление на signed URL для скачивания.

**cURL Example:**
```bash
curl -X GET "https://api.pto-platform.ru/api/v1/packages/pkg_1/download?type=pdf" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -L -O
```

---

## Валидация

### POST /projects/{project_id}/validate

Запустить валидацию комплектности.

**Request:**
```json
{
  "check_completeness": true,
  "check_dates": true,
  "check_logic": true
}
```

**Response: 202 Accepted**
```json
{
  "success": true,
  "data": {
    "task_id": "task_930",
    "status": "processing",
    "estimated_time": 30
  },
  "message": "Валидация запущена"
}
```

---

### GET /projects/{project_id}/validation-reports

Получить отчеты валидации.

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "reports": [
      {
        "id": "vr_1",
        "is_valid": false,
        "issues": [
          {
            "severity": "error",
            "category": "missing_document",
            "message": "Отсутствует сертификат для материала 'Труба REHAU d=20мм'",
            "material_id": "mat_5"
          },
          {
            "severity": "warning",
            "category": "date_inconsistency",
            "message": "Дата документа качества (2023-05-10) раньше даты работ (2025-12-10)",
            "aosr_id": "aosr_3",
            "quality_doc_id": "qd_7"
          }
        ],
        "created_at": "2025-12-15T15:00:00Z"
      }
    ]
  }
}
```

---

## Пользователи

### GET /users/me

Получить профиль текущего пользователя.

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": 123,
    "email": "user@example.com",
    "full_name": "Иван Иванов",
    "company": "ООО Строй",
    "role": "engineer",
    "created_at": "2025-11-01T10:00:00Z"
  }
}
```

---

### PATCH /users/me

Обновить профиль.

**Request:**
```json
{
  "full_name": "Иван Петрович Иванов",
  "company": "ООО Новая Строй"
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": 123,
    "full_name": "Иван Петрович Иванов",
    "company": "ООО Новая Строй",
    "updated_at": "2025-12-15T16:00:00Z"
  }
}
```

---

## Коды ошибок

| Код HTTP | Код ошибки | Описание |
|----------|------------|----------|
| 400 | `VALIDATION_ERROR` | Некорректные данные в запросе |
| 400 | `INVALID_FILE_TYPE` | Неподдерживаемый тип файла |
| 400 | `FILE_TOO_LARGE` | Файл превышает максимальный размер (50 MB) |
| 401 | `UNAUTHORIZED` | Отсутствует или невалидный токен |
| 401 | `TOKEN_EXPIRED` | Токен истек |
| 401 | `INVALID_CREDENTIALS` | Неверный email или пароль |
| 403 | `FORBIDDEN` | Недостаточно прав доступа |
| 404 | `NOT_FOUND` | Ресурс не найден |
| 409 | `ALREADY_EXISTS` | Ресурс уже существует (например, email) |
| 422 | `TASK_FAILED` | Фоновая задача завершилась с ошибкой |
| 429 | `RATE_LIMIT_EXCEEDED` | Превышен лимит запросов |
| 500 | `INTERNAL_SERVER_ERROR` | Внутренняя ошибка сервера |
| 503 | `SERVICE_UNAVAILABLE` | Сервис временно недоступен |

---

## Rate Limiting

**Лимиты:**
- Анонимные запросы: 100 запросов/час
- Аутентифицированные: 1000 запросов/час
- File uploads: 50 файлов/час

**Headers:**
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1734268800
```

---

## WebSocket Events

**Connection URL:** `wss://api.pto-platform.ru/ws`

**Authentication:**
```json
{
  "type": "auth",
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Event Types:**
```json
// Прогресс задачи
{
  "type": "task_progress",
  "task_id": "task_890",
  "progress": 45,
  "message": "Анализ документа 3 из 5"
}

// Задача завершена
{
  "type": "task_completed",
  "task_id": "task_890",
  "result": { ... }
}

// Задача ошибка
{
  "type": "task_error",
  "task_id": "task_890",
  "error": "Ошибка анализа документа"
}
```

---

## Пример полного workflow

```bash
# 1. Регистрация
curl -X POST https://api.pto-platform.ru/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "SecureP@ss123"}'

# 2. Логин
TOKEN=$(curl -X POST https://api.pto-platform.ru/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "SecureP@ss123"}' \
  | jq -r '.data.access_token')

# 3. Создать проект
PROJECT_ID=$(curl -X POST https://api.pto-platform.ru/api/v1/projects \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "ЖК Южный", "address": "г. Москва"}' \
  | jq -r '.data.id')

# 4. Загрузить документы
curl -X POST https://api.pto-platform.ru/api/v1/projects/$PROJECT_ID/documents/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "files[]=@document1.pdf" \
  -F "doc_type=working_documentation"

# 5. Запустить анализ РД
curl -X POST https://api.pto-platform.ru/api/v1/projects/$PROJECT_ID/analyze \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"options": {"extract_work_types": true}}'

# 6. Сгенерировать АОСР
curl -X POST https://api.pto-platform.ru/api/v1/projects/$PROJECT_ID/aosr/generate \
  -H "Authorization: Bearer $TOKEN"

# 7. Сгенерировать финальный пакет
curl -X POST https://api.pto-platform.ru/api/v1/projects/$PROJECT_ID/packages/generate \
  -H "Authorization: Bearer $TOKEN"

# 8. Скачать пакет
curl -X GET "https://api.pto-platform.ru/api/v1/packages/pkg_1/download?type=pdf" \
  -H "Authorization: Bearer $TOKEN" \
  -L -O
```

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-15
**Версия:** 1.0
