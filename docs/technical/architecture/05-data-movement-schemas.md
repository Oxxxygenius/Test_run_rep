# Схемы движения данных и связей компонентов

> Текстовые схемы, показывающие как данные перемещаются между компонентами системы и как части связаны друг с другом.

---

## Содержание

1. [Общая архитектура системы](#1-общая-архитектура-системы)
2. [Схема загрузки и анализа РД](#2-схема-загрузки-и-анализа-рд)
3. [Схема генерации АОСР](#3-схема-генерации-аоср)
4. [Схема поиска документов качества](#4-схема-поиска-документов-качества)
5. [Схема валидации и создания пакета](#5-схема-валидации-и-создания-пакета)
6. [Схема взаимодействия AI-агентов](#6-схема-взаимодействия-ai-агентов)
7. [Схема работы с базой данных](#7-схема-работы-с-базой-данных)
8. [Схема аутентификации и авторизации](#8-схема-аутентификации-и-авторизации)

---

## 1. Общая архитектура системы

### 1.1 Компоненты верхнего уровня

```
┌─────────────────────────────────────────────────────────────────┐
│                        ПОЛЬЗОВАТЕЛЬ                              │
│                    (Браузер / Frontend)                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTPS (JSON/FormData)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     NGINX (Reverse Proxy)                        │
│                   - SSL Termination                              │
│                   - Static File Serving                          │
│                   - Load Balancing                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTP
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     FASTAPI APPLICATION                          │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐        │
│  │   REST API  │  │  WebSocket   │  │  Background     │        │
│  │  Endpoints  │  │   Handlers   │  │  Task Queue     │        │
│  └─────────────┘  └──────────────┘  └─────────────────┘        │
└───────┬──────────────────┬────────────────────┬─────────────────┘
        │                  │                    │
        │                  │                    │
        ▼                  ▼                    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐
│ PostgreSQL   │  │   Redis      │  │  Celery Workers          │
│              │  │              │  │  (Background Processing) │
│ - Users      │  │ - Cache      │  │                          │
│ - Projects   │  │ - Sessions   │  │  ┌─────────────────────┐ │
│ - Documents  │  │ - Pub/Sub    │  │  │  LangGraph          │ │
│ - AOSR       │  │ - Job Queue  │  │  │  AI Agents          │ │
│ - Materials  │  │              │  │  │                     │ │
└──────────────┘  └──────────────┘  │  │  1. RD Analyzer     │ │
                                     │  │  2. AOSR Generator  │ │
        ┌────────────────────────────┤  │  3. Document Search │ │
        │                            │  │  4. Validator       │ │
        ▼                            │  │  5. PDF Merger      │ │
┌──────────────────────────┐        │  │  6. OCR Processor   │ │
│  Yandex Object Storage   │        │  │  7. Quality Checker │ │
│                          │        │  └─────────────────────┘ │
│  - Project Documents     │        └──────────────────────────┘
│  - Quality Documents     │                    │
│  - Execution Schemas     │                    │
│  - Generated AOSR        │                    ▼
│  - Final Packages        │        ┌──────────────────────────┐
└──────────────────────────┘        │  External APIs           │
                                     │                          │
                                     │  - OpenAI GPT-4o         │
                                     │  - Anthropic Claude      │
                                     │  - Web Scraping Targets  │
                                     └──────────────────────────┘
```

### 1.2 Типы данных и их движение

```
INPUT DATA                  PROCESSING                    OUTPUT DATA
──────────                  ──────────                    ───────────

PDF Files                   OCR Extraction                Text Content
(15-50 MB)          ──────► (GPT-4o Vision)      ──────► (50K-200K chars)
                            [Celery Worker]
                                    │
                                    ▼
                            Semantic Analysis             Structured JSON
                            (GPT-4o + LangChain) ──────► (Work Types,
                            [AI Agent]                    Materials, etc.)
                                    │
                                    ▼
                            Data Validation               Database Records
                            (Business Logic)     ──────► (PostgreSQL)
                            [FastAPI]
                                    │
                                    ▼
                            PDF Generation                Final Documents
                            (ReportLab)          ──────► (AOSR, Packages)
                            [Celery Worker]               (5-100 MB)
                                    │
                                    ▼
                            Storage                       Object Storage
                            (S3 API)             ──────► (Yandex Cloud)
```

---

## 2. Схема загрузки и анализа РД

### 2.1 Полный цикл обработки рабочей документации

```
┌─────────────────────────────────────────────────────────────────────┐
│ ФАЗА 1: ЗАГРУЗКА ФАЙЛА                                               │
└─────────────────────────────────────────────────────────────────────┘

User Browser                Frontend                  Backend API
──────────────              ────────                  ───────────
     │                          │                          │
     │  Select File             │                          │
     │  "Раздел ОВ.pdf"         │                          │
     │  (15 MB)                 │                          │
     ├─────────────────────────►│                          │
     │                          │                          │
     │                          │  POST /api/v1/projects/  │
     │                          │  {project_id}/documents  │
     │                          │  FormData:               │
     │                          │  - file: Binary          │
     │                          │  - doc_type: "RD"        │
     │                          ├─────────────────────────►│
     │                          │                          │
     │                          │                          │ Validate File
     │                          │                          │ - Size < 100MB
     │                          │                          │ - Type: PDF
     │                          │                          │ - Check Auth
     │                          │                          │
     │                          │                          │ Save to Temp
     │                          │                          │ /tmp/uuid.pdf
     │                          │                          │
     │                          │                          │ Create DB Record
     │                          │  201 Created             │ INSERT INTO
     │                          │  {                       │ documents
     │                          │    "id": "doc_123",      │ (...values)
     │                          │    "status": "uploading" │
     │                          │  }                       │
     │                          │◄─────────────────────────┤
     │                          │                          │
     │  Upload Success          │                          │
     │◄─────────────────────────┤                          │
     │  Document ID: doc_123    │                          │
     │                          │                          │

┌─────────────────────────────────────────────────────────────────────┐
│ ФАЗА 2: ФОНОВАЯ ОБРАБОТКА (Celery Task)                             │
└─────────────────────────────────────────────────────────────────────┘

Backend API              Celery Worker           Object Storage    PostgreSQL
───────────              ─────────────           ──────────────    ──────────
     │                        │                        │               │
     │ Enqueue Task           │                        │               │
     │ process_rd_document    │                        │               │
     │ (doc_id="doc_123")     │                        │               │
     ├───────────────────────►│                        │               │
     │                        │                        │               │
     │                        │ Get Document Info      │               │
     │                        ├───────────────────────────────────────►│
     │                        │                        │               │
     │                        │ {id, path, project_id} │               │
     │                        │◄───────────────────────────────────────┤
     │                        │                        │               │
     │                        │ Upload to Storage      │               │
     │                        │ PUT /bucket/project_1/ │               │
     │                        │ documents/doc_123.pdf  │               │
     │                        ├───────────────────────►│               │
     │                        │                        │               │
     │                        │ URL: s3://...          │               │
     │                        │◄───────────────────────┤               │
     │                        │                        │               │
     │                        │ Update DB              │               │
     │                        │ status='processing'    │               │
     │                        ├───────────────────────────────────────►│
     │                        │                        │               │

┌─────────────────────────────────────────────────────────────────────┐
│ ФАЗА 3: OCR ИЗВЛЕЧЕНИЕ (GPT-4o Vision)                              │
└─────────────────────────────────────────────────────────────────────┘

Celery Worker         OCR Agent           OpenAI API        Redis Cache
─────────────         ─────────           ──────────        ───────────
     │                    │                    │                 │
     │ Start OCR          │                    │                 │
     │ ocr_agent.extract  │                    │                 │
     │ (pdf_path)         │                    │                 │
     ├───────────────────►│                    │                 │
     │                    │                    │                 │
     │                    │ Load PDF           │                 │
     │                    │ Convert pages      │                 │
     │                    │ to PNG images      │                 │
     │                    │ (PyMuPDF)          │                 │
     │                    │                    │                 │
     │                    │ For Each Page:     │                 │
     │                    │ page_1.png         │                 │
     │                    │ (Base64 encode)    │                 │
     │                    │                    │                 │
     │                    │ POST /chat/        │                 │
     │                    │ completions        │                 │
     │                    │ {                  │                 │
     │                    │   model: gpt-4o,   │                 │
     │                    │   messages: [{     │                 │
     │                    │     type: "text",  │                 │
     │                    │     content: "..." │                 │
     │                    │   }, {             │                 │
     │                    │     type: "image", │                 │
     │                    │     data: base64   │                 │
     │                    │   }]               │                 │
     │                    │ }                  │                 │
     │                    ├───────────────────►│                 │
     │                    │                    │                 │
     │                    │                    │ Process Image   │
     │                    │                    │ Extract Text    │
     │                    │                    │ (30-60 sec)     │
     │                    │                    │                 │
     │                    │ Response:          │                 │
     │                    │ {                  │                 │
     │                    │   text: "..."      │                 │
     │                    │ }                  │                 │
     │                    │◄───────────────────┤                 │
     │                    │                    │                 │
     │                    │ Cache Result       │                 │
     │                    │ SET ocr:doc_123:p1 │                 │
     │                    │ "extracted text"   │                 │
     │                    │ EX 86400           │                 │
     │                    ├────────────────────────────────────►│
     │                    │                    │                 │
     │                    │ Combine all pages  │                 │
     │                    │ full_text = ...    │                 │
     │                    │                    │                 │
     │ OCR Complete       │                    │                 │
     │ text (150K chars)  │                    │                 │
     │◄───────────────────┤                    │                 │
     │                    │                    │                 │

┌─────────────────────────────────────────────────────────────────────┐
│ ФАЗА 4: СЕМАНТИЧЕСКИЙ АНАЛИЗ (RD Analyzer Agent)                    │
└─────────────────────────────────────────────────────────────────────┘

Celery Worker      RD Analyzer Agent    LangChain         GPT-4o API
─────────────      ─────────────────    ─────────         ──────────
     │                    │                 │                  │
     │ Analyze Document   │                 │                  │
     │ rd_agent.analyze   │                 │                  │
     │ (text)             │                 │                  │
     ├───────────────────►│                 │                  │
     │                    │                 │                  │
     │                    │ Build Chain     │                  │
     │                    │ prompt | llm    │                  │
     │                    │                 │                  │
     │                    │ Invoke Chain    │                  │
     │                    ├────────────────►│                  │
     │                    │                 │                  │
     │                    │                 │ Format Prompt    │
     │                    │                 │ with Template    │
     │                    │                 │                  │
     │                    │                 │ Call GPT-4o      │
     │                    │                 │ {                │
     │                    │                 │   model: gpt-4o  │
     │                    │                 │   messages: [... │
     │                    │                 │   temperature: 0 │
     │                    │                 │   response_format│
     │                    │                 │   { type: json } │
     │                    │                 │ }                │
     │                    │                 ├─────────────────►│
     │                    │                 │                  │
     │                    │                 │                  │ Extract:
     │                    │                 │                  │ - Work types
     │                    │                 │                  │ - Materials
     │                    │                 │                  │ - Dates
     │                    │                 │                  │ - Quantities
     │                    │                 │                  │
     │                    │                 │ JSON Response    │
     │                    │                 │◄─────────────────┤
     │                    │                 │                  │
     │                    │ Structured Data │                  │
     │                    │◄────────────────┤                  │
     │                    │                 │                  │
     │ Analysis Result    │                 │                  │
     │ {                  │                 │                  │
     │   work_types: [...],                 │                  │
     │   materials: [...],                  │                  │
     │   dates: {...}                       │                  │
     │ }                  │                 │                  │
     │◄───────────────────┤                 │                  │
     │                    │                 │                  │

┌─────────────────────────────────────────────────────────────────────┐
│ ФАЗА 5: СОХРАНЕНИЕ РЕЗУЛЬТАТОВ                                       │
└─────────────────────────────────────────────────────────────────────┘

Celery Worker              PostgreSQL                  WebSocket
─────────────              ──────────                  ─────────
     │                          │                          │
     │ Save Analysis            │                          │
     │ UPDATE documents         │                          │
     │ SET                      │                          │
     │   status='analyzed',     │                          │
     │   extracted_text='...',  │                          │
     │   metadata=jsonb_data    │                          │
     │ WHERE id='doc_123'       │                          │
     ├─────────────────────────►│                          │
     │                          │                          │
     │                          │ Insert Work Types        │
     │ INSERT INTO work_types   │                          │
     │ (document_id, name, ...) │                          │
     │ VALUES (...)             │                          │
     ├─────────────────────────►│                          │
     │                          │                          │
     │                          │ Insert Materials         │
     │ INSERT INTO materials    │                          │
     │ (document_id, name, ...) │                          │
     │ VALUES (...)             │                          │
     ├─────────────────────────►│                          │
     │                          │                          │
     │ Commit Transaction       │                          │
     │◄─────────────────────────┤                          │
     │                          │                          │
     │ Notify Frontend          │                          │
     │ ws.send({                │                          │
     │   type: 'document_ready',│                          │
     │   doc_id: 'doc_123'      │                          │
     │ })                       │                          │
     ├─────────────────────────────────────────────────────►│
     │                          │                          │
     │                          │                          │ Update UI
     │                          │                          │ Show notification
     │                          │                          │ Enable AOSR
     │                          │                          │ generation
```

---

## 3. Схема генерации АОСР

### 3.1 Процесс создания АОСР из проанализированных данных

```
┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 1: ЗАПРОС НА ГЕНЕРАЦИЮ                                          │
└─────────────────────────────────────────────────────────────────────┘

Frontend              Backend API          PostgreSQL         Redis
────────              ───────────          ──────────         ─────
   │                       │                    │               │
   │ User clicks           │                    │               │
   │ "Generate AOSR"       │                    │               │
   │                       │                    │               │
   │ POST /api/v1/         │                    │               │
   │ aosr/generate         │                    │               │
   │ {                     │                    │               │
   │   project_id: "p1",   │                    │               │
   │   work_type_id: "w1"  │                    │               │
   │ }                     │                    │               │
   ├──────────────────────►│                    │               │
   │                       │                    │               │
   │                       │ Validate Request   │               │
   │                       │ Check permissions  │               │
   │                       │                    │               │
   │                       │ Check Cache        │               │
   │                       │ GET aosr:p1:w1     │               │
   │                       ├───────────────────────────────────►│
   │                       │                    │               │
   │                       │ (cache miss)       │               │
   │                       │◄───────────────────────────────────┤
   │                       │                    │               │
   │                       │ Load Data          │               │
   │                       │ SELECT * FROM      │               │
   │                       │ work_types wt      │               │
   │                       │ JOIN materials m   │               │
   │                       │ JOIN documents d   │               │
   │                       │ WHERE wt.id='w1'   │               │
   │                       ├───────────────────►│               │
   │                       │                    │               │
   │                       │ Data Bundle        │               │
   │                       │ {                  │               │
   │                       │   work_type: {...},│               │
   │                       │   materials: [...],│               │
   │                       │   project: {...}   │               │
   │                       │ }                  │               │
   │                       │◄───────────────────┤               │
   │                       │                    │               │

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 2: ФОНОВАЯ ГЕНЕРАЦИЯ (Celery)                                  │
└─────────────────────────────────────────────────────────────────────┘

Backend API         Celery Worker      AOSR Generator      GPT-4o
───────────         ─────────────      ──────────────      ──────
   │                     │                   │                │
   │ Enqueue Task        │                   │                │
   │ generate_aosr_task  │                   │                │
   │ (work_type_id)      │                   │                │
   ├────────────────────►│                   │                │
   │                     │                   │                │
   │ 202 Accepted        │                   │                │
   │ {                   │                   │                │
   │   task_id: "t_456"  │                   │                │
   │ }                   │                   │                │
   │                     │                   │                │
   │                     │ Start Generation  │                │
   │                     │ aosr_gen.generate │                │
   │                     │ (data_bundle)     │                │
   │                     ├──────────────────►│                │
   │                     │                   │                │
   │                     │                   │ Fill Template  │
   │                     │                   │ (Jinja2)       │
   │                     │                   │                │
   │                     │                   │ Enrich with AI │
   │                     │                   │ Request GPT-4o │
   │                     │                   │ for:           │
   │                     │                   │ - Descriptions │
   │                     │                   │ - Compliance   │
   │                     │                   │   checks       │
   │                     │                   │                │
   │                     │                   │ Call GPT       │
   │                     │                   ├───────────────►│
   │                     │                   │                │
   │                     │                   │                │ Generate
   │                     │                   │                │ work desc.
   │                     │                   │                │ compliance
   │                     │                   │                │ text
   │                     │                   │                │
   │                     │                   │ Text Response  │
   │                     │                   │◄───────────────┤
   │                     │                   │                │
   │                     │                   │ Merge Data     │
   │                     │                   │ with AI text   │
   │                     │                   │                │

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 3: PDF СОЗДАНИЕ (ReportLab)                                    │
└─────────────────────────────────────────────────────────────────────┘

AOSR Generator                    File System           Object Storage
──────────────                    ───────────           ──────────────
      │                                 │                     │
      │ Create PDF Canvas               │                     │
      │ c = canvas.Canvas()             │                     │
      │ pagesize=A4                     │                     │
      │                                 │                     │
      │ Draw Header                     │                     │
      │ - Title centered                │                     │
      │ - Logo (if exists)              │                     │
      │ - Document number               │                     │
      │                                 │                     │
      │ Draw Table 1: Project Info      │                     │
      │ ┌─────────────────────────┐    │                     │
      │ │ Объект: "ЖК Новый Дом"  │    │                     │
      │ │ Адрес: "..."            │    │                     │
      │ │ Застройщик: "..."       │    │                     │
      │ └─────────────────────────┘    │                     │
      │                                 │                     │
      │ Draw Table 2: Work Details      │                     │
      │ ┌─────────────────────────┐    │                     │
      │ │ Вид работ: "Монтаж ..."  │    │                     │
      │ │ Основание: СП 73.13330  │    │                     │
      │ │ Объем: 150 м²           │    │                     │
      │ └─────────────────────────┘    │                     │
      │                                 │                     │
      │ Draw Table 3: Materials         │                     │
      │ ┌───┬──────────┬────┬─────┐   │                     │
      │ │ № │ Материал │ Ед │ Кол │   │                     │
      │ ├───┼──────────┼────┼─────┤   │                     │
      │ │ 1 │ Труба... │ м  │ 500 │   │                     │
      │ │ 2 │ Фитинг...│ шт │ 120 │   │                     │
      │ └───┴──────────┴────┴─────┘   │                     │
      │                                 │                     │
      │ Draw Signatures Section         │                     │
      │ - Подрядчик: _________          │                     │
      │ - Заказчик: __________          │                     │
      │ - Стройконтроль: ______         │                     │
      │                                 │                     │
      │ Save PDF                        │                     │
      │ c.save()                        │                     │
      ├────────────────────────────────►│                     │
      │                                 │                     │
      │                                 │ File Created        │
      │                                 │ /tmp/aosr_w1.pdf    │
      │                                 │ (250 KB)            │
      │                                 │                     │
      │ Upload to Storage               │                     │
      │ PUT /bucket/project_1/          │                     │
      │ aosr/aosr_w1.pdf                │                     │
      ├─────────────────────────────────────────────────────►│
      │                                 │                     │
      │                                 │                     │ Store File
      │                                 │                     │ Return URL
      │                                 │                     │
      │ URL: s3://bucket/.../aosr_w1.pdf                      │
      │◄─────────────────────────────────────────────────────┤
      │                                 │                     │

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 4: СОХРАНЕНИЕ МЕТАДАННЫХ                                        │
└─────────────────────────────────────────────────────────────────────┘

Celery Worker          PostgreSQL            Redis            WebSocket
─────────────          ──────────            ─────            ─────────
      │                     │                  │                  │
      │ Save AOSR           │                  │                  │
      │ INSERT INTO aosr    │                  │                  │
      │ (                   │                  │                  │
      │   id,               │                  │                  │
      │   work_type_id,     │                  │                  │
      │   file_url,         │                  │                  │
      │   status,           │                  │                  │
      │   metadata          │                  │                  │
      │ ) VALUES (...)      │                  │                  │
      ├────────────────────►│                  │                  │
      │                     │                  │                  │
      │                     │ Record Saved     │                  │
      │                     │ id='aosr_123'    │                  │
      │◄────────────────────┤                  │                  │
      │                     │                  │                  │
      │ Cache Result        │                  │                  │
      │ SET aosr:p1:w1      │                  │                  │
      │ "aosr_123"          │                  │                  │
      │ EX 3600             │                  │                  │
      ├─────────────────────────────────────────►                  │
      │                     │                  │                  │
      │ Notify User         │                  │                  │
      │ ws.send({           │                  │                  │
      │   type: 'aosr_ready'│                  │                  │
      │   aosr_id: 'aosr_123'                  │                  │
      │   download_url: '...'                  │                  │
      │ })                  │                  │                  │
      ├────────────────────────────────────────────────────────────►
      │                     │                  │                  │
      │                     │                  │                  │ Show success
      │                     │                  │                  │ notification
      │                     │                  │                  │ Enable
      │                     │                  │                  │ download
```

---

## 4. Схема поиска документов качества

### 4.1 Многоуровневый поиск с веб-скрапингом

```
┌─────────────────────────────────────────────────────────────────────┐
│ УРОВЕНЬ 1: ПОИСК В ЛОКАЛЬНОЙ БАЗЕ                                    │
└─────────────────────────────────────────────────────────────────────┘

Search Agent         PostgreSQL          Redis Cache
────────────         ──────────          ───────────
     │                    │                   │
     │ Search Local DB    │                   │
     │ find_document(     │                   │
     │   material_name,   │                   │
     │   manufacturer     │                   │
     │ )                  │                   │
     │                    │                   │
     │ Check Cache        │                   │
     │ GET doc:труба:rehau│                   │
     ├───────────────────────────────────────►│
     │                    │                   │
     │ (cache miss)       │                   │
     │◄───────────────────────────────────────┤
     │                    │                   │
     │ Query Database     │                   │
     │ SELECT * FROM      │                   │
     │ quality_documents  │                   │
     │ WHERE              │                   │
     │   material_name    │                   │
     │   ILIKE '%труба%'  │                   │
     │   AND manufacturer │                   │
     │   ILIKE '%rehau%'  │                   │
     ├───────────────────►│                   │
     │                    │                   │
     │                    │ Full-text Search  │
     │                    │ using pg_trgm     │
     │                    │ similarity()      │
     │                    │                   │
     │ Results (if found) │                   │
     │◄───────────────────┤                   │
     │                    │                   │
     │ If found → DONE    │                   │
     │ If not → Level 2   │                   │
     │                    │                   │

┌─────────────────────────────────────────────────────────────────────┐
│ УРОВЕНЬ 2: СПЕЦИАЛИЗИРОВАННЫЕ САЙТЫ (Playwright)                     │
└─────────────────────────────────────────────────────────────────────┘

Search Agent       Playwright       Target Website      GPT-4o
────────────       ──────────       ──────────────      ──────
     │                  │                  │               │
     │ Search           │                  │               │
     │ Specialized      │                  │               │
     │ Sites            │                  │               │
     │                  │                  │               │
     │ For site in      │                  │               │
     │ [santech.ru,     │                  │               │
     │  petrovich.ru,   │                  │               │
     │  vodopad.ru]:    │                  │               │
     │                  │                  │               │
     │ Launch Browser   │                  │               │
     │ browser.launch   │                  │               │
     │ (headless=True)  │                  │               │
     ├─────────────────►│                  │               │
     │                  │                  │               │
     │                  │ Open Page        │               │
     │                  │ page.goto(url)   │               │
     │                  ├─────────────────►│               │
     │                  │                  │               │
     │                  │                  │ Page Loaded   │
     │                  │                  │ HTML Content  │
     │                  │◄─────────────────┤               │
     │                  │                  │               │
     │ Fill Search      │                  │               │
     │ Form             │                  │               │
     │ page.fill(       │                  │               │
     │   '#search',     │                  │               │
     │   'труба rehau'  │                  │               │
     │ )                │                  │               │
     │ page.click(      │                  │               │
     │   'button.submit'│                  │               │
     │ )                │                  │               │
     ├─────────────────►│                  │               │
     │                  │                  │               │
     │                  │ Submit Search    │               │
     │                  ├─────────────────►│               │
     │                  │                  │               │
     │                  │                  │ Search Results│
     │                  │◄─────────────────┤               │
     │                  │                  │               │
     │ Extract Links    │                  │               │
     │ pdf_links =      │                  │               │
     │ page.query_sel   │                  │               │
     │ ('a[href$=.pdf]')│                  │               │
     ├─────────────────►│                  │               │
     │                  │                  │               │
     │ Links Found      │                  │               │
     │ [url1, url2...]  │                  │               │
     │◄─────────────────┤                  │               │
     │                  │                  │               │
     │ For each link:   │                  │               │
     │                  │                  │               │
     │ Download PDF     │                  │               │
     │ page.goto(url1)  │                  │               │
     ├─────────────────►│                  │               │
     │                  │                  │               │
     │                  │ GET PDF file     │               │
     │                  ├─────────────────►│               │
     │                  │                  │               │
     │                  │ PDF Binary       │               │
     │                  │◄─────────────────┤               │
     │                  │                  │               │
     │ PDF Content      │                  │               │
     │◄─────────────────┤                  │               │
     │                  │                  │               │
     │ Check Relevance  │                  │               │
     │ with GPT-4o      │                  │               │
     │                  │                  │               │
     │ Extract text     │                  │               │
     │ from first 2     │                  │               │
     │ pages (OCR)      │                  │               │
     │                  │                  │               │
     │ Ask GPT-4o:      │                  │               │
     │ "Is this doc     │                  │               │
     │ relevant for     │                  │               │
     │ Труба REHAU?"    │                  │               │
     ├─────────────────────────────────────────────────────►
     │                  │                  │               │
     │                  │                  │               │ Analyze:
     │                  │                  │               │ - Manufacturer
     │                  │                  │               │ - Material type
     │                  │                  │               │ - Certificate
     │                  │                  │               │ - Relevance
     │                  │                  │               │
     │ Response:        │                  │               │
     │ {                │                  │               │
     │   relevant: true,│                  │               │
     │   confidence: 0.9│                  │               │
     │   reasons: [...]                    │               │
     │ }                │                  │               │
     │◄─────────────────────────────────────────────────────
     │                  │                  │               │
     │ If relevant:     │                  │               │
     │ → Save & DONE    │                  │               │
     │ If not:          │                  │               │
     │ → Try next link  │                  │               │
     │                  │                  │               │

┌─────────────────────────────────────────────────────────────────────┐
│ УРОВЕНЬ 3: ЗАПРОС РАЗРЕШЕНИЯ НА ПОИСК В GOOGLE                      │
└─────────────────────────────────────────────────────────────────────┘

Search Agent      WebSocket       Frontend         User
────────────      ─────────       ────────         ────
     │                 │              │              │
     │ Not found in    │              │              │
     │ DB or special   │              │              │
     │ sites           │              │              │
     │                 │              │              │
     │ Send notification│             │              │
     │ ws.send({       │              │              │
     │   type: "doc_   │              │              │
     │   not_found",   │              │              │
     │   material:     │              │              │
     │   "Труба REHAU",│              │              │
     │   message: "Не  │              │              │
     │   найден. Искать│              │              │
     │   в Google?"    │              │              │
     │ })              │              │              │
     ├────────────────►│              │              │
     │                 │              │              │
     │                 │ Show modal   │              │
     │                 ├─────────────►│              │
     │                 │              │              │
     │                 │              │ ┌──────────────────┐
     │                 │              │ │ ⚠ Не найдено     │
     │                 │              │ │                  │
     │                 │              │ │ Документ на      │
     │                 │              │ │ "Труба REHAU"    │
     │                 │              │ │ не найден.       │
     │                 │              │ │                  │
     │                 │              │ │ Разрешить поиск  │
     │                 │              │ │ в интернете?     │
     │                 │              │ │                  │
     │                 │              │ │ [Искать в Google]│
     │                 │              │ │ [Пропустить]     │
     │                 │              │ │ [Загружу сам]    │
     │                 │              │ └──────────────────┘
     │                 │              │              │
     │                 │              │ User clicks  │
     │                 │              │ "Искать в    │
     │                 │              │ Google"      │
     │                 │              │◄─────────────┤
     │                 │              │              │
     │                 │ Response:    │              │
     │                 │ {action:     │              │
     │                 │  "search"}   │              │
     │                 │◄─────────────┤              │
     │                 │              │              │
     │ Permission      │              │              │
     │ granted         │              │              │
     │◄────────────────┤              │              │
     │                 │              │              │
     │ Start Google    │              │              │
     │ search          │              │              │
     │                 │              │              │

┌─────────────────────────────────────────────────────────────────────┐
│ УРОВЕНЬ 3A: ПОИСК В GOOGLE (Если разрешено)                         │
└─────────────────────────────────────────────────────────────────────┘

Search Agent      Playwright      Google Search       Target Sites
────────────      ──────────      ─────────────       ────────────
     │                 │                │                   │
     │ Google Search   │                │                   │
     │ Query: "труба   │                │                   │
     │ REHAU сертификат│                │                   │
     │ качества PDF"   │                │                   │
     │                 │                │                   │
     │ Open Google     │                │                   │
     │ page.goto(      │                │                   │
     │ 'google.com/    │                │                   │
     │ search?q=...')  │                │                   │
     ├────────────────►│                │                   │
     │                 │                │                   │
     │                 │ Submit Query   │                   │
     │                 ├───────────────►│                   │
     │                 │                │                   │
     │                 │                │ Search Results    │
     │                 │◄───────────────┤                   │
     │                 │                │                   │
     │ Parse Results   │                │                   │
     │ results =       │                │                   │
     │ page.query_sel  │                │                   │
     │ ('.g a')        │                │                   │
     ├────────────────►│                │                   │
     │                 │                │                   │
     │ Top 10 URLs     │                │                   │
     │◄────────────────┤                │                   │
     │                 │                │                   │
     │ For each URL:   │                │                   │
     │                 │                │                   │
     │ Visit Page      │                │                   │
     │ page.goto(url)  │                │                   │
     ├────────────────►│                │                   │
     │                 │                │                   │
     │                 │ Navigate       │                   │
     │                 ├───────────────────────────────────►│
     │                 │                │                   │
     │                 │                │                   │ Render page
     │                 │                │                   │
     │                 │ Page Content   │                   │
     │                 │◄───────────────────────────────────┤
     │                 │                │                   │
     │ Look for PDFs   │                │                   │
     │ pdfs = page.    │                │                   │
     │ query_selector  │                │                   │
     │ ('a[href$=.pdf]'│                │                   │
     │ )               │                │                   │
     ├────────────────►│                │                   │
     │                 │                │                   │
     │ (Process PDFs   │                │                   │
     │  with GPT-4o    │                │                   │
     │  validation)    │                │                   │
     │                 │                │                   │
     │ If found:       │                │                   │
     │ source =        │                │                   │
     │ "найден в       │                │                   │
     │ интернете       │                │                   │
     │ (Google)"       │                │                   │
     │                 │                │                   │

┌─────────────────────────────────────────────────────────────────────┐
│ УРОВЕНЬ 4: ЗАПРОС РАЗРЕШЕНИЯ НА ГЕНЕРАЦИЮ AI                        │
└─────────────────────────────────────────────────────────────────────┘

Search Agent      WebSocket       Frontend         User
────────────      ─────────       ────────         ────
     │                 │              │              │
     │ Not found       │              │              │
     │ anywhere        │              │              │
     │                 │              │              │
     │ Send notification│             │              │
     │ ws.send({       │              │              │
     │   type: "doc_   │              │              │
     │   generate_     │              │              │
     │   request",     │              │              │
     │   material:     │              │              │
     │   "Труба REHAU",│              │              │
     │   message: "Не  │              │              │
     │   найден нигде. │              │              │
     │   Сгенерировать?│              │              │
     │ })              │              │              │
     ├────────────────►│              │              │
     │                 │              │              │
     │                 │ Show modal   │              │
     │                 ├─────────────►│              │
     │                 │              │              │
     │                 │              │ ┌──────────────────┐
     │                 │              │ │ ⚠ Не найдено     │
     │                 │              │ │ нигде            │
     │                 │              │ │                  │
     │                 │              │ │ Документ на      │
     │                 │              │ │ "Труба REHAU"    │
     │                 │              │ │ не найден даже   │
     │                 │              │ │ в интернете.     │
     │                 │              │ │                  │
     │                 │              │ │ Разрешить AI     │
     │                 │              │ │ сгенерировать    │
     │                 │              │ │ типовой паспорт? │
     │                 │              │ │                  │
     │                 │              │ │ [Сгенерировать]  │
     │                 │              │ │ [Загружу сам]    │
     │                 │              │ └──────────────────┘
     │                 │              │              │
     │                 │              │ User clicks  │
     │                 │              │ "Сгенерировать"    │
     │                 │              │◄─────────────┤
     │                 │              │              │
     │                 │ Response:    │              │
     │                 │ {action:     │              │
     │                 │  "generate"} │              │
     │                 │◄─────────────┤              │
     │                 │              │              │
     │ Permission      │              │              │
     │ granted         │              │              │
     │◄────────────────┤              │              │
     │                 │              │              │
     │ Start AI        │              │              │
     │ generation      │              │              │
     │                 │              │              │

┌─────────────────────────────────────────────────────────────────────┐
│ УРОВЕНЬ 4A: ГЕНЕРАЦИЯ ПАСПОРТА AI (Если разрешено)                  │
└─────────────────────────────────────────────────────────────────────┘

Search Agent        GPT-4o API       ReportLab        Object Storage
────────────        ──────────       ─────────        ──────────────
     │                   │                │                  │
     │ Generate          │                │                  │
     │ AI-based          │                │                  │
     │ Passport          │                │                  │
     │                   │                │                  │
     │ Request GPT       │                │                  │
     │ to create         │                │                  │
     │ typical passport  │                │                  │
     │ based on GOST     │                │                  │
     │                   │                │                  │
     │ POST /chat/       │                │                  │
     │ completions       │                │                  │
     │ {                 │                │                  │
     │   prompt: "Создай │                │                  │
     │   типовой паспорт │                │                  │
     │   для материала:  │                │                  │
     │   Труба REHAU...", │               │                  │
     │   material: {...} │                │                  │
     │ }                 │                │                  │
     ├──────────────────►│                │                  │
     │                   │                │                  │
     │                   │ Generate       │                  │
     │                   │ typical data:  │                  │
     │                   │ - ГОСТ refs    │                  │
     │                   │ - Typical      │                  │
     │                   │   properties   │                  │
     │                   │ - Generic      │                  │
     │                   │   specs        │                  │
     │                   │                │                  │
     │ JSON Response     │                │                  │
     │ {                 │                │                  │
     │   manufacturer:   │                │                  │
     │   "REHAU",        │                │                  │
     │   gost: "...",    │                │                  │
     │   properties: {..}│                │                  │
     │ }                 │                │                  │
     │◄──────────────────┤                │                  │
     │                   │                │                  │
     │ Generate PDF      │                │                  │
     │ create_passport   │                │                  │
     │ (data)            │                │                  │
     ├───────────────────────────────────►│                  │
     │                   │                │                  │
     │                   │                │ Create PDF       │
     │                   │                │ with LARGE       │
     │                   │                │ warning banner:  │
     │                   │                │                  │
     │                   │                │ ┌──────────────┐ │
     │                   │                │ │ ⚠️ ВНИМАНИЕ! │ │
     │                   │                │ │              │ │
     │                   │                │ │ СГЕНЕРИРОВАНО│ │
     │                   │                │ │ AI           │ │
     │                   │                │ │              │ │
     │                   │                │ │ ТРЕБУЕТСЯ    │ │
     │                   │                │ │ ПРОВЕРКА     │ │
     │                   │                │ └──────────────┘ │
     │                   │                │                  │
     │                   │                │ Red background,  │
     │                   │                │ large font       │
     │                   │                │                  │
     │ PDF Binary        │                │                  │
     │◄───────────────────────────────────┤                  │
     │                   │                │                  │
     │ Upload            │                │                  │
     │ PUT /bucket/...   │                │                  │
     ├────────────────────────────────────────────────────────►
     │                   │                │                  │
     │                   │                │                  │ Store with
     │                   │                │                  │ metadata:
     │                   │                │                  │ source=
     │                   │                │                  │ "generated_ai"
     │                   │                │                  │ needs_review=
     │                   │                │                  │ true
     │                   │                │                  │
     │ URL returned      │                │                  │
     │◄────────────────────────────────────────────────────────
     │                   │                │                  │

┌─────────────────────────────────────────────────────────────────────┐
│ РЕЗУЛЬТАТ: СОХРАНЕНИЕ НАЙДЕННОГО ДОКУМЕНТА                           │
└─────────────────────────────────────────────────────────────────────┘

Search Agent         PostgreSQL           Redis            Frontend
────────────         ──────────           ─────            ────────
     │                    │                 │                  │
     │ Save Document      │                 │                  │
     │ INSERT INTO        │                 │                  │
     │ quality_documents  │                 │                  │
     │ (                  │                 │                  │
     │   material_id,     │                 │                  │
     │   file_url,        │                 │                  │
     │   source,          │                 │                  │
     │   needs_review,    │                 │                  │
     │   confidence,      │                 │                  │
     │   metadata         │                 │                  │
     │ )                  │                 │                  │
     │ VALUES (           │                 │                  │
     │   ...,             │                 │                  │
     │   source:          │                 │                  │
     │   "найден в базе"  │                 │                  │
     │   OR "специализиро-│                 │                  │
     │   ванный сайт"     │                 │                  │
     │   OR "найден в     │                 │                  │
     │   интернете"       │                 │                  │
     │   OR "сгенерировано│                 │                  │
     │   AI",             │                 │                  │
     │   needs_review:    │                 │                  │
     │   false/true       │                 │                  │
     │ )                  │                 │                  │
     ├───────────────────►│                 │                  │
     │                    │                 │                  │
     │ Record Saved       │                 │                  │
     │◄───────────────────┤                 │                  │
     │                    │                 │                  │
     │ Cache Result       │                 │                  │
     │ SET doc:материал   │                 │                  │
     │ "doc_id"           │                 │                  │
     │ EX 86400           │                 │                  │
     ├────────────────────────────────────►│                  │
     │                    │                 │                  │
     │ Notify User        │                 │                  │
     │ ws.send({          │                 │                  │
     │   type: 'doc_found'│                 │                  │
     │   material: "..."  │                 │                  │
     │   url: "..."       │                 │                  │
     │   source: "santech"│                 │                  │
     │ })                 │                 │                  │
     ├─────────────────────────────────────────────────────────►
     │                    │                 │                  │
```

---

## 5. Схема валидации и создания пакета

### 5.1 Многоуровневая валидация

```
┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 1: ЗАПУСК ВАЛИДАЦИИ                                             │
└─────────────────────────────────────────────────────────────────────┘

Frontend            Backend API         Validator Agent    PostgreSQL
────────            ───────────         ───────────────    ──────────
   │                     │                     │                │
   │ POST /api/v1/       │                     │                │
   │ projects/{id}/      │                     │                │
   │ validate            │                     │                │
   ├────────────────────►│                     │                │
   │                     │                     │                │
   │                     │ Load Project Data   │                │
   │                     │ SELECT              │                │
   │                     │   p.*,              │                │
   │                     │   array_agg(d.*),   │                │
   │                     │   array_agg(a.*),   │                │
   │                     │   array_agg(m.*)    │                │
   │                     │ FROM projects p     │                │
   │                     │ LEFT JOIN documents d│               │
   │                     │ LEFT JOIN aosr a    │                │
   │                     │ LEFT JOIN materials m│               │
   │                     │ WHERE p.id = $1     │                │
   │                     │ GROUP BY p.id       │                │
   │                     ├────────────────────────────────────►│
   │                     │                     │                │
   │                     │ Full Project Bundle │                │
   │                     │◄────────────────────────────────────┤
   │                     │                     │                │
   │                     │ Start Validation    │                │
   │                     │ validator.validate  │                │
   │                     │ (project_bundle)    │                │
   │                     ├────────────────────►│                │
   │                     │                     │                │

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 2: ПРОВЕРКА 1 - НАЛИЧИЕ ДОКУМЕНТОВ                             │
└─────────────────────────────────────────────────────────────────────┘

Validator Agent                              Validation Results
───────────────                              ──────────────────
      │                                             │
      │ Check Document Presence                     │
      │                                             │
      │ Required Documents:                         │
      │ - РД (Рабочая документация)                 │
      │ - At least 1 АОСР                           │
      │ - Quality docs for each material            │
      │                                             │
      │ For Each Work Type:                         │
      │   check_rd = work_type.document_id != null  │
      │   if not check_rd:                          │
      │     errors.append({                         │
      │       type: "missing_rd",                   │
      │       work_type: work_type.name,            │
      │       severity: "critical"                  │
      │     })                                      │
      │                                             │
      │ For Each AOSR:                              │
      │   check_aosr = aosr.file_url != null        │
      │   if not check_aosr:                        │
      │     errors.append({                         │
      │       type: "missing_aosr",                 │
      │       aosr_id: aosr.id,                     │
      │       severity: "critical"                  │
      │     })                                      │
      │                                             │
      │ For Each Material:                          │
      │   quality_docs = material.quality_documents │
      │   if len(quality_docs) == 0:                │
      │     errors.append({                         │
      │       type: "missing_quality_doc",          │
      │       material: material.name,              │
      │       severity: "high"                      │
      │     })                                      │
      │                                             │
      │ Result Phase 1                              │
      ├────────────────────────────────────────────►│
      │                                             │
      │                                             │ {
      │                                             │   phase: "presence",
      │                                             │   status: "pass/fail",
      │                                             │   errors: [...]
      │                                             │ }

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 3: ПРОВЕРКА 2 - СОГЛАСОВАННОСТЬ ДАТ                            │
└─────────────────────────────────────────────────────────────────────┘

Validator Agent                              Validation Results
───────────────                              ──────────────────
      │                                             │
      │ Check Date Consistency                      │
      │                                             │
      │ Rule 1: Work date > Material delivery date  │
      │                                             │
      │ For Each AOSR:                              │
      │   work_date = aosr.work_completion_date     │
      │                                             │
      │   For Each Material in AOSR:                │
      │     delivery_date = material.delivery_date  │
      │                                             │
      │     if delivery_date > work_date:           │
      │       errors.append({                       │
      │         type: "date_logic_error",           │
      │         message: "Материал доставлен ПОСЛЕ  │
      │                  завершения работ",         │
      │         material: material.name,            │
      │         delivery: delivery_date,            │
      │         work: work_date,                    │
      │         severity: "critical"                │
      │       })                                    │
      │                                             │
      │ Rule 2: Certificate valid at work date      │
      │                                             │
      │   For Each Quality Document:                │
      │     cert_issue = doc.certificate_date       │
      │     cert_expiry = doc.expiry_date           │
      │                                             │
      │     if work_date < cert_issue:              │
      │       errors.append({                       │
      │         type: "cert_not_yet_valid",         │
      │         severity: "critical"                │
      │       })                                    │
      │                                             │
      │     if work_date > cert_expiry:             │
      │       errors.append({                       │
      │         type: "cert_expired",               │
      │         severity: "critical"                │
      │       })                                    │
      │                                             │
      │ Rule 3: Sequential work dates               │
      │                                             │
      │   aosr_list = sorted(aosr, key=work_date)   │
      │   for i in range(len(aosr_list) - 1):       │
      │     if aosr_list[i].date > aosr_list[i+1]:  │
      │       warnings.append({                     │
      │         type: "non_sequential_dates",       │
      │         severity: "medium"                  │
      │       })                                    │
      │                                             │
      │ Result Phase 2                              │
      ├────────────────────────────────────────────►│
      │                                             │
      │                                             │ {
      │                                             │   phase: "dates",
      │                                             │   status: "pass/fail",
      │                                             │   errors: [...],
      │                                             │   warnings: [...]
      │                                             │ }

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 4: ПРОВЕРКА 3 - СООТВЕТСТВИЕ МАТЕРИАЛОВ                        │
└─────────────────────────────────────────────────────────────────────┘

Validator Agent        GPT-4o API            Validation Results
───────────────        ──────────            ──────────────────
      │                     │                        │
      │ Check Material      │                        │
      │ Compliance          │                        │
      │                     │                        │
      │ For Each Material:  │                        │
      │                     │                        │
      │   required_props =  │                        │
      │   extract_from_rd() │                        │
      │   - ГОСТ            │                        │
      │   - Диаметр         │                        │
      │   - Давление        │                        │
      │   - Температура     │                        │
      │                     │                        │
      │   actual_props =    │                        │
      │   quality_doc.      │                        │
      │   metadata          │                        │
      │                     │                        │
      │   Ask GPT-4o to     │                        │
      │   compare specs     │                        │
      │                     │                        │
      │ POST /chat/         │                        │
      │ completions         │                        │
      │ {                   │                        │
      │   prompt: "Compare: │                        │
      │   Required: {...}   │                        │
      │   Actual: {...}     │                        │
      │   Do they match?"   │                        │
      │ }                   │                        │
      ├────────────────────►│                        │
      │                     │                        │
      │                     │ Analyze Specs          │
      │                     │ - Compare GOSTs        │
      │                     │ - Check ranges         │
      │                     │ - Verify equivalents   │
      │                     │                        │
      │ Response:           │                        │
      │ {                   │                        │
      │   compliant: false, │                        │
      │   issues: [         │                        │
      │     "Diameter       │                        │
      │      mismatch"      │                        │
      │   ]                 │                        │
      │ }                   │                        │
      │◄────────────────────┤                        │
      │                     │                        │
      │ if not compliant:   │                        │
      │   errors.append({   │                        │
      │     type:           │                        │
      │     "spec_mismatch" │                        │
      │   })                │                        │
      │                     │                        │
      │ Result Phase 3      │                        │
      ├────────────────────────────────────────────►│
      │                     │                        │
      │                     │                        │ {
      │                     │                        │   phase: "materials",
      │                     │                        │   errors: [...],
      │                     │                        │   warnings: [...]
      │                     │                        │ }

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 5: ПРОВЕРКА 4 - ПОЛНОТА ДАННЫХ                                 │
└─────────────────────────────────────────────────────────────────────┘

Validator Agent                              Validation Results
───────────────                              ──────────────────
      │                                             │
      │ Check Data Completeness                     │
      │                                             │
      │ For Each AOSR:                              │
      │                                             │
      │   Required Fields:                          │
      │   - work_type (not null)                    │
      │   - work_completion_date (not null)         │
      │   - contractor (not null)                   │
      │   - customer (not null)                     │
      │   - quantity (> 0)                          │
      │   - unit (not null)                         │
      │   - materials (length > 0)                  │
      │                                             │
      │   missing_fields = []                       │
      │                                             │
      │   if not aosr.work_type:                    │
      │     missing_fields.append("work_type")      │
      │                                             │
      │   if not aosr.work_completion_date:         │
      │     missing_fields.append("date")           │
      │                                             │
      │   if len(aosr.materials) == 0:              │
      │     missing_fields.append("materials")      │
      │                                             │
      │   if len(missing_fields) > 0:               │
      │     errors.append({                         │
      │       type: "incomplete_aosr",              │
      │       aosr_id: aosr.id,                     │
      │       missing: missing_fields,              │
      │       severity: "high"                      │
      │     })                                      │
      │                                             │
      │ For Each Quality Document:                  │
      │                                             │
      │   Required Metadata:                        │
      │   - manufacturer (not null)                 │
      │   - certificate_number (not null)           │
      │   - certificate_date (not null)             │
      │   - gost_standard (not null)                │
      │                                             │
      │   (Similar validation as above)             │
      │                                             │
      │ Result Phase 4                              │
      ├────────────────────────────────────────────►│
      │                                             │
      │                                             │ {
      │                                             │   phase: "completeness",
      │                                             │   errors: [...],
      │                                             │   warnings: [...]
      │                                             │ }

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 6: ПРОВЕРКА 5 - ЛОГИЧЕСКАЯ ЦЕЛОСТНОСТЬ                         │
└─────────────────────────────────────────────────────────────────────┘

Validator Agent        GPT-4o API            Validation Results
───────────────        ──────────            ──────────────────
      │                     │                        │
      │ Logical Consistency │                        │
      │ Check               │                        │
      │                     │                        │
      │ Build Context:      │                        │
      │ context = {         │                        │
      │   project_type: "..." │                      │
      │   work_types: [...] │                        │
      │   materials: [...]  │                        │
      │   quantities: [...] │                        │
      │ }                   │                        │
      │                     │                        │
      │ Ask GPT-4o:         │                        │
      │ "Analyze this       │                        │
      │ project for logical │                        │
      │ inconsistencies"    │                        │
      │                     │                        │
      │ POST /chat/         │                        │
      │ completions         │                        │
      │ {                   │                        │
      │   model: "gpt-4o",  │                        │
      │   messages: [{      │                        │
      │     role: "system", │                        │
      │     content: "You   │                        │
      │     are a           │                        │
      │     construction    │                        │
      │     expert..."      │                        │
      │   }, {              │                        │
      │     role: "user",   │                        │
      │     content: context│                        │
      │   }]                │                        │
      │ }                   │                        │
      ├────────────────────►│                        │
      │                     │                        │
      │                     │ Analyze:               │
      │                     │ - Material quantities  │
      │                     │   reasonable for work? │
      │                     │ - Work sequence makes  │
      │                     │   sense?               │
      │                     │ - Missing work types?  │
      │                     │ - Unusual combinations?│
      │                     │                        │
      │ Response:           │                        │
      │ {                   │                        │
      │   issues: [{        │                        │
      │     type: "warning",│                        │
      │     message: "500м  │                        │
      │     труб для 50м²   │                        │
      │     кажется много"  │                        │
      │   }]                │                        │
      │ }                   │                        │
      │◄────────────────────┤                        │
      │                     │                        │
      │ Add to warnings     │                        │
      │                     │                        │
      │ Result Phase 5      │                        │
      ├────────────────────────────────────────────►│
      │                     │                        │
      │                     │                        │ {
      │                     │                        │   phase: "logic",
      │                     │                        │   warnings: [...]
      │                     │                        │ }

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 7: СВОДНЫЙ ОТЧЕТ                                                │
└─────────────────────────────────────────────────────────────────────┘

Validator Agent        PostgreSQL          Frontend
───────────────        ──────────          ────────
      │                     │                  │
      │ Aggregate Results   │                  │
      │                     │                  │
      │ validation_report = {                  │
      │   timestamp: now(), │                  │
      │   status: "pass" |  │                  │
      │           "fail" |  │                  │
      │           "warning",│                  │
      │   phases: {         │                  │
      │     presence: {...},│                  │
      │     dates: {...},   │                  │
      │     materials: {...}│                  │
      │     completeness: {.│                  │
      │     logic: {...}    │                  │
      │   },                │                  │
      │   critical_errors: 3│                  │
      │   high_errors: 5,   │                  │
      │   warnings: 8       │                  │
      │ }                   │                  │
      │                     │                  │
      │ Save Report         │                  │
      │ INSERT INTO         │                  │
      │ validation_reports  │                  │
      │ (...)               │                  │
      ├────────────────────►│                  │
      │                     │                  │
      │ Report Saved        │                  │
      │◄────────────────────┤                  │
      │                     │                  │
      │ Return to Frontend  │                  │
      │ {                   │                  │
      │   validation_id: id,│                  │
      │   status: "...",    │                  │
      │   can_create_       │                  │
      │   package: bool,    │                  │
      │   report: {...}     │                  │
      │ }                   │                  │
      ├─────────────────────────────────────────►
      │                     │                  │
      │                     │                  │ Display:
      │                     │                  │ ✓ Checks passed
      │                     │                  │ ✗ Critical errors
      │                     │                  │ ⚠ Warnings
      │                     │                  │
      │                     │                  │ If status="pass":
      │                     │                  │   Enable "Create
      │                     │                  │   Package" button

┌─────────────────────────────────────────────────────────────────────┐
│ ЭТАП 8: СОЗДАНИЕ ФИНАЛЬНОГО ПАКЕТА (Если валидация OK)              │
└─────────────────────────────────────────────────────────────────────┘

Frontend          Backend API       Package Creator    Object Storage
────────          ───────────       ───────────────    ──────────────
   │                   │                   │                  │
   │ POST /api/v1/     │                   │                  │
   │ projects/{id}/    │                   │                  │
   │ create-package    │                   │                  │
   ├──────────────────►│                   │                  │
   │                   │                   │                  │
   │                   │ Enqueue Task      │                  │
   │                   │ create_package_   │                  │
   │                   │ task(project_id)  │                  │
   │                   ├──────────────────►│                  │
   │                   │                   │                  │
   │ 202 Accepted      │                   │                  │
   │ task_id: "..."    │                   │                  │
   │◄──────────────────┤                   │                  │
   │                   │                   │                  │
   │                   │                   │ Collect Files:   │
   │                   │                   │                  │
   │                   │                   │ Download from S3:│
   │                   │                   │ - All AOSR PDFs  │
   │                   │                   │ - Quality docs   │
   │                   │                   │ - Schemas        │
   │                   │                   │                  │
   │                   │                   │ GET /bucket/.../ │
   │                   │                   │ aosr/*.pdf       │
   │                   │                   ├─────────────────►│
   │                   │                   │                  │
   │                   │                   │ Binary files     │
   │                   │                   │◄─────────────────┤
   │                   │                   │                  │
   │                   │                   │ Generate:        │
   │                   │                   │ - Title page     │
   │                   │                   │ - Registry       │
   │                   │                   │ - Contents       │
   │                   │                   │   (ReportLab)    │
   │                   │                   │                  │
   │                   │                   │ Merge PDFs:      │
   │                   │                   │ (PyPDF2)         │
   │                   │                   │                  │
   │                   │                   │ merger =         │
   │                   │                   │ PdfMerger()      │
   │                   │                   │                  │
   │                   │                   │ merger.append(   │
   │                   │                   │   title_page.pdf)│
   │                   │                   │ merger.append(   │
   │                   │                   │   contents.pdf)  │
   │                   │                   │                  │
   │                   │                   │ for aosr in aosr_│
   │                   │                   │   merger.append( │
   │                   │                   │     aosr.pdf,    │
   │                   │                   │     bookmark=    │
   │                   │                   │     aosr.name    │
   │                   │                   │   )              │
   │                   │                   │                  │
   │                   │                   │ merger.append(   │
   │                   │                   │   registry.pdf)  │
   │                   │                   │                  │
   │                   │                   │ for doc in docs: │
   │                   │                   │   merger.append( │
   │                   │                   │     doc.pdf      │
   │                   │                   │   )              │
   │                   │                   │                  │
   │                   │                   │ merger.write(    │
   │                   │                   │   final.pdf      │
   │                   │                   │ )                │
   │                   │                   │                  │
   │                   │                   │ Upload Result    │
   │                   │                   │ PUT /bucket/.../│
   │                   │                   │ packages/        │
   │                   │                   │ project_1_       │
   │                   │                   │ final.pdf        │
   │                   │                   ├─────────────────►│
   │                   │                   │                  │
   │                   │                   │ URL returned     │
   │                   │                   │◄─────────────────┤
   │                   │                   │                  │
   │                   │ Task Complete     │                  │
   │                   │ {                 │                  │
   │                   │   package_url:    │                  │
   │                   │   "s3://..."      │                  │
   │                   │ }                 │                  │
   │                   │◄──────────────────┤                  │
   │                   │                   │                  │
   │ ws.send({         │                   │                  │
   │   type: 'package_ │                   │                  │
   │   ready',         │                   │                  │
   │   url: "..."      │                   │                  │
   │ })                │                   │                  │
   │◄──────────────────┤                   │                  │
   │                   │                   │                  │
   │ Download Package  │                   │                  │
   │ (45 MB PDF)       │                   │                  │
```

---

## 6. Схема взаимодействия AI-агентов

### 6.1 LangGraph Workflow - Последовательная обработка

```
┌─────────────────────────────────────────────────────────────────────┐
│ WORKFLOW: Обработка РД и генерация АОСР                             │
└─────────────────────────────────────────────────────────────────────┘

Trigger              StateGraph           Agents              State
───────              ──────────           ──────              ─────
   │                      │                  │                  │
   │ Start Workflow       │                  │                  │
   │ app.invoke({         │                  │                  │
   │   document_id: "..." │                  │                  │
   │ })                   │                  │                  │
   ├─────────────────────►│                  │                  │
   │                      │                  │                  │
   │                      │ Initialize State │                  │
   │                      │ state = {        │                  │
   │                      │   document_id,   │                  │
   │                      │   status: "init" │                  │
   │                      │ }                │                  │
   │                      ├─────────────────────────────────────►│
   │                      │                  │                  │
   │                      │                  │                  │ {
   │                      │                  │                  │   document_id: "d1",
   │                      │                  │                  │   status: "init",
   │                      │                  │                  │   data: null
   │                      │                  │                  │ }
   │                      │                  │                  │
   │                      │ Execute Node:    │                  │
   │                      │ "ocr_extract"    │                  │
   │                      ├─────────────────►│                  │
   │                      │                  │                  │
   │                      │                  │ OCR Agent        │
   │                      │                  │ process()        │
   │                      │                  │                  │
   │                      │                  │ - Load PDF       │
   │                      │                  │ - Extract text   │
   │                      │                  │ - Return result  │
   │                      │                  │                  │
   │                      │ Update State     │                  │
   │                      │ {                │                  │
   │                      │   extracted_text:│                  │
   │                      │   "...",         │                  │
   │                      │   status: "ocr_  │                  │
   │                      │   complete"      │                  │
   │                      │ }                │                  │
   │                      │◄─────────────────┤                  │
   │                      │                  │                  │
   │                      │ Merge State      │                  │
   │                      ├─────────────────────────────────────►│
   │                      │                  │                  │
   │                      │                  │                  │ {
   │                      │                  │                  │   document_id: "d1",
   │                      │                  │                  │   extracted_text: "...",
   │                      │                  │                  │   status: "ocr_complete"
   │                      │                  │                  │ }
   │                      │                  │                  │
   │                      │ Execute Node:    │                  │
   │                      │ "analyze_rd"     │                  │
   │                      ├─────────────────►│                  │
   │                      │                  │                  │
   │                      │                  │ RD Analyzer      │
   │                      │                  │ Agent            │
   │                      │                  │                  │
   │                      │                  │ - Parse text     │
   │                      │                  │ - Call GPT-4o    │
   │                      │                  │ - Extract data   │
   │                      │                  │                  │
   │                      │ Update State     │                  │
   │                      │ {                │                  │
   │                      │   work_types: [] │                  │
   │                      │   materials: []  │                  │
   │                      │   status:        │                  │
   │                      │   "analyzed"     │                  │
   │                      │ }                │                  │
   │                      │◄─────────────────┤                  │
   │                      │                  │                  │
   │                      │ Merge State      │                  │
   │                      ├─────────────────────────────────────►│
   │                      │                  │                  │
   │                      │                  │                  │ {
   │                      │                  │                  │   document_id: "d1",
   │                      │                  │                  │   extracted_text: "...",
   │                      │                  │                  │   work_types: [...],
   │                      │                  │                  │   materials: [...],
   │                      │                  │                  │   status: "analyzed"
   │                      │                  │                  │ }
   │                      │                  │                  │
   │                      │ Execute Node:    │                  │
   │                      │ "generate_aosr"  │                  │
   │                      ├─────────────────►│                  │
   │                      │                  │                  │
   │                      │                  │ AOSR Generator   │
   │                      │                  │                  │
   │                      │                  │ - Build template │
   │                      │                  │ - Generate PDF   │
   │                      │                  │ - Upload file    │
   │                      │                  │                  │
   │                      │ Update State     │                  │
   │                      │ {                │                  │
   │                      │   aosr_url: "..." │                 │
   │                      │   status: "done" │                  │
   │                      │ }                │                  │
   │                      │◄─────────────────┤                  │
   │                      │                  │                  │
   │                      │ Merge State      │                  │
   │                      ├─────────────────────────────────────►│
   │                      │                  │                  │
   │                      │                  │                  │ {
   │                      │                  │                  │   document_id: "d1",
   │                      │                  │                  │   extracted_text: "...",
   │                      │                  │                  │   work_types: [...],
   │                      │                  │                  │   materials: [...],
   │                      │                  │                  │   aosr_url: "s3://...",
   │                      │                  │                  │   status: "done"
   │                      │                  │                  │ }
   │                      │                  │                  │
   │ Final State          │                  │                  │
   │◄─────────────────────┤                  │                  │
   │                      │                  │                  │

Graph Definition (Python):
─────────────────────────

from langgraph.graph import StateGraph

workflow = StateGraph(State)

# Add nodes (agents)
workflow.add_node("ocr_extract", ocr_agent.process)
workflow.add_node("analyze_rd", rd_analyzer_agent.analyze)
workflow.add_node("generate_aosr", aosr_generator_agent.generate)

# Define edges (flow)
workflow.set_entry_point("ocr_extract")
workflow.add_edge("ocr_extract", "analyze_rd")
workflow.add_edge("analyze_rd", "generate_aosr")
workflow.add_edge("generate_aosr", END)

# Compile
app = workflow.compile()

# Execute
result = app.invoke({"document_id": "d1"})
```

### 6.2 Параллельное выполнение агентов

```
┌─────────────────────────────────────────────────────────────────────┐
│ WORKFLOW: Параллельный поиск документов качества                    │
└─────────────────────────────────────────────────────────────────────┘

Trigger           StateGraph         Agents (Parallel)      Results
───────           ──────────         ─────────────────      ───────
   │                   │                    │                  │
   │ Start Search      │                    │                  │
   │ for 5 materials   │                    │                  │
   ├──────────────────►│                    │                  │
   │                   │                    │                  │
   │                   │ Fork to 5 agents   │                  │
   │                   │ (parallel)         │                  │
   │                   │                    │                  │
   │                   ├───────────────────►│ Agent 1:         │
   │                   │                    │ Search "Труба"   │
   │                   │                    │                  │
   │                   ├───────────────────►│ Agent 2:         │
   │                   │                    │ Search "Фитинг"  │
   │                   │                    │                  │
   │                   ├───────────────────►│ Agent 3:         │
   │                   │                    │ Search "Утеплитель"
   │                   │                    │                  │
   │                   ├───────────────────►│ Agent 4:         │
   │                   │                    │ Search "Клапан"  │
   │                   │                    │                  │
   │                   ├───────────────────►│ Agent 5:         │
   │                   │                    │ Search "Краска"  │
   │                   │                    │                  │
   │                   │                    │ All agents work  │
   │                   │                    │ simultaneously:  │
   │                   │                    │ - DB search      │
   │                   │                    │ - Web scraping   │
   │                   │                    │ - GPT validation │
   │                   │                    │                  │
   │                   │ Collect Results    │                  │
   │                   │ (barrier - wait    │                  │
   │                   │  for all)          │                  │
   │                   │◄───────────────────┤                  │
   │                   │                    │                  │
   │                   │ Aggregate          │                  │
   │                   │ {                  │                  │
   │                   │   material_1: {    │                  │
   │                   │     doc: "url",    │                  │
   │                   │     confidence: 0.9│                  │
   │                   │   },               │                  │
   │                   │   material_2: {...}│                  │
   │                   │   ...              │                  │
   │                   │ }                  │                  │
   │                   ├───────────────────────────────────────►
   │                   │                    │                  │
   │ Results Ready     │                    │                  │
   │◄──────────────────┤                    │                  │
   │                   │                    │                  │

Time saved: 2-5 min per material × 5 = 10-25 min
With parallel: ~2-5 min total (limited by slowest search)
```

### 6.3 Условное ветвление агентов

```
┌─────────────────────────────────────────────────────────────────────┐
│ WORKFLOW: Поиск с фоллбэками                                         │
└─────────────────────────────────────────────────────────────────────┘

State                Decision Node         Agent Execution
─────                ─────────────         ───────────────
   │                      │                        │
   │ material = "Труба"   │                        │
   │ manufacturer = "..."  │                        │
   ├─────────────────────►│                        │
   │                      │                        │
   │                      │ Check Cache            │
   │                      │ if cached:             │
   │                      │   → return_cached      │
   │                      │ else:                  │
   │                      │   → search_db          │
   │                      │                        │
   │                      ├───────────────────────►│
   │                      │                        │ search_db_agent
   │                      │                        │ SELECT ...
   │                      │                        │
   │                      │ Result: null           │
   │                      │◄───────────────────────┤
   │                      │                        │
   │                      │ if found:              │
   │                      │   → END                │
   │                      │ else:                  │
   │                      │   → search_specialized │
   │                      │                        │
   │                      ├───────────────────────►│
   │                      │                        │ search_specialized
   │                      │                        │ (Playwright)
   │                      │                        │
   │                      │ Result: {url: "..."}   │
   │                      │◄───────────────────────┤
   │                      │                        │
   │                      │ if found:              │
   │                      │   → validate_with_gpt  │
   │                      │ else:                  │
   │                      │   → google_search      │
   │                      │                        │
   │                      ├───────────────────────►│
   │                      │                        │ validate_gpt_agent
   │                      │                        │ Check relevance
   │                      │                        │
   │                      │ Result: {              │
   │                      │   relevant: true,      │
   │                      │   confidence: 0.85     │
   │                      │ }                      │
   │                      │◄───────────────────────┤
   │                      │                        │
   │                      │ if confidence > 0.7:   │
   │                      │   → END (success)      │
   │                      │ else:                  │
   │                      │   → google_search      │
   │                      │                        │

Code (Conditional Edges):
────────────────────────

def route_search(state):
    if state.get('cached'):
        return "return_cached"
    if state.get('db_result'):
        return "validate"
    if state.get('specialized_result'):
        return "validate"
    if state.get('attempts') < 3:
        return "google_search"
    return "generate_placeholder"

workflow.add_conditional_edges(
    "search_db",
    route_search,
    {
        "return_cached": "end",
        "validate": "validate_gpt",
        "google_search": "google_agent",
        "generate_placeholder": "generate_agent"
    }
)
```

### 6.4 Агрегация результатов от нескольких агентов

```
┌─────────────────────────────────────────────────────────────────────┐
│ WORKFLOW: Валидация с множественными проверками                      │
└─────────────────────────────────────────────────────────────────────┘

Input Data          Parallel Validators      Aggregator         Result
──────────          ───────────────────      ──────────         ──────
     │                      │                     │                │
     │ project_data         │                     │                │
     ├─────────────────────►│                     │                │
     │                      │                     │                │
     │                      │ Fork to validators  │                │
     │                      │                     │                │
     │                      ├─────────────────────┤                │
     │                      │                     │                │
     │                      │ Validator 1:        │                │
     │                      │ check_documents()   │                │
     │                      │ → {                 │                │
     │                      │   errors: [...],    │                │
     │                      │   score: 85         │                │
     │                      │ }                   │                │
     │                      │                     │                │
     │                      │ Validator 2:        │                │
     │                      │ check_dates()       │                │
     │                      │ → {                 │                │
     │                      │   errors: [...],    │                │
     │                      │   score: 70         │                │
     │                      │ }                   │                │
     │                      │                     │                │
     │                      │ Validator 3:        │                │
     │                      │ check_materials()   │                │
     │                      │ → {                 │                │
     │                      │   errors: [...],    │                │
     │                      │   score: 90         │                │
     │                      │ }                   │                │
     │                      │                     │                │
     │                      │ Validator 4:        │                │
     │                      │ check_completeness()│                │
     │                      │ → {                 │                │
     │                      │   errors: [...],    │                │
     │                      │   score: 95         │                │
     │                      │ }                   │                │
     │                      │                     │                │
     │                      │ Validator 5 (GPT):  │                │
     │                      │ check_logic()       │                │
     │                      │ → {                 │                │
     │                      │   warnings: [...],  │                │
     │                      │   score: 80         │                │
     │                      │ }                   │                │
     │                      │                     │                │
     │                      │ All results ready   │                │
     │                      ├────────────────────►│                │
     │                      │                     │                │
     │                      │                     │ Combine:       │
     │                      │                     │                │
     │                      │                     │ all_errors = []│
     │                      │                     │ for v in       │
     │                      │                     │ validators:    │
     │                      │                     │   all_errors.  │
     │                      │                     │   extend(      │
     │                      │                     │   v.errors     │
     │                      │                     │   )            │
     │                      │                     │                │
     │                      │                     │ avg_score =    │
     │                      │                     │ mean([...])    │
     │                      │                     │ = 84           │
     │                      │                     │                │
     │                      │                     │ overall_status:│
     │                      │                     │ if critical > 0│
     │                      │                     │   → "fail"     │
     │                      │                     │ elif score<70: │
     │                      │                     │   → "warning"  │
     │                      │                     │ else:          │
     │                      │                     │   → "pass"     │
     │                      │                     │                │
     │                      │                     │ Final Report   │
     │                      │                     ├───────────────►│
     │                      │                     │                │
     │                      │                     │                │ {
     │                      │                     │                │   status: "pass",
     │                      │                     │                │   score: 84,
     │                      │                     │                │   errors: [...],
     │                      │                     │                │   warnings: [...],
     │                      │                     │                │   validators: [...]
     │                      │                     │                │ }
```

---

## 7. Схема работы с базой данных

### 7.1 Структура таблиц и связи

```
┌─────────────────────────────────────────────────────────────────────┐
│ DATABASE SCHEMA: PostgreSQL Tables & Relationships                  │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────┐
│      USERS           │
├──────────────────────┤
│ id (PK)              │──┐
│ email (UNIQUE)       │  │
│ password_hash        │  │
│ full_name            │  │
│ company              │  │
│ role                 │  │
│ created_at           │  │
│ updated_at           │  │
└──────────────────────┘  │
                          │
                          │ FK: user_id
                          │
                          ▼
┌──────────────────────────────────────────┐
│              PROJECTS                    │
├──────────────────────────────────────────┤
│ id (PK)                                  │──┐
│ user_id (FK → users.id)                  │  │
│ name                                     │  │
│ address                                  │  │
│ customer                                 │  │
│ contractor                               │  │
│ supervisor                               │  │
│ status (draft|in_progress|completed)     │  │
│ metadata (JSONB)                         │  │
│ created_at                               │  │
│ updated_at                               │  │
└──────────────────────────────────────────┘  │
                                              │
                    ┌─────────────────────────┼────────────┐
                    │                         │            │
                    │ FK: project_id          │            │
                    │                         │            │
                    ▼                         ▼            ▼
┌───────────────────────────┐  ┌──────────────────────┐  ┌─────────────────┐
│      DOCUMENTS            │  │      WORK_TYPES      │  │   EVENT_LOG     │
├───────────────────────────┤  ├──────────────────────┤  ├─────────────────┤
│ id (PK)                   │──┐ id (PK)             │──┐ id (PK)         │
│ project_id (FK)           │  │ project_id (FK)      │  │ project_id (FK) │
│ type (RD|SCHEMA|QUALITY)  │  │ document_id (FK)     │  │ user_id (FK)    │
│ file_url                  │  │ name                 │  │ event_type      │
│ file_size                 │  │ code                 │  │ entity_type     │
│ status (uploading|        │  │ quantity             │  │ entity_id       │
│   processing|analyzed|    │  │ unit                 │  │ metadata (JSONB)│
│   error)                  │  │ gost_standard        │  │ created_at      │
│ extracted_text (TEXT)     │  │ description          │  │                 │
│ metadata (JSONB)          │  │ metadata (JSONB)     │  │                 │
│ created_at                │  │ created_at           │  │                 │
│ updated_at                │  │                      │  │                 │
└───────────────────────────┘  └──────────────────────┘  └─────────────────┘
          │                              │
          │                              │ FK: work_type_id
          │                              │
          │                              ▼
          │                    ┌──────────────────────┐
          │                    │       AOSR           │
          │                    ├──────────────────────┤
          │                    │ id (PK)              │──┐
          │                    │ work_type_id (FK)    │  │
          │                    │ project_id (FK)      │  │
          │                    │ file_url             │  │
          │                    │ version              │  │
          │                    │ work_completion_date │  │
          │                    │ contractor_signature │  │
          │                    │ customer_signature   │  │
          │                    │ supervisor_signature │  │
          │                    │ status (draft|final) │  │
          │                    │ metadata (JSONB)     │  │
          │                    │ created_at           │  │
          │                    │ updated_at           │  │
          │                    └──────────────────────┘  │
          │                                              │
          │ FK: document_id                              │
          │                                              │
          ▼                                              │
┌───────────────────────────┐                           │
│      MATERIALS            │                           │
├───────────────────────────┤                           │
│ id (PK)                   │◄──────────────────────────┘
│ document_id (FK)          │     FK: aosr_id (many-to-many)
│ aosr_id (FK)              │     via AOSR_MATERIALS table
│ name                      │
│ manufacturer              │
│ quantity                  │
│ unit                      │
│ delivery_date             │
│ gost_standard             │
│ metadata (JSONB)          │
│ created_at                │
└───────────────────────────┘
          │
          │ FK: material_id
          │
          ▼
┌───────────────────────────────────┐
│     QUALITY_DOCUMENTS             │
├───────────────────────────────────┤
│ id (PK)                           │
│ material_id (FK)                  │
│ file_url                          │
│ document_type (certificate|       │
│   passport|declaration)           │
│ certificate_number                │
│ certificate_date                  │
│ expiry_date                       │
│ manufacturer                      │
│ is_generated (BOOLEAN)            │
│ source (db|web|generated)         │
│ confidence_score (0-1)            │
│ metadata (JSONB)                  │
│ created_at                        │
└───────────────────────────────────┘
```

### 7.2 Типичные запросы и оптимизация

```
┌─────────────────────────────────────────────────────────────────────┐
│ QUERY 1: Загрузка полных данных проекта                             │
└─────────────────────────────────────────────────────────────────────┘

Application          PostgreSQL                      Performance
───────────          ──────────                      ───────────
     │                    │                               │
     │ Get Project        │                               │
     │ with all related   │                               │
     │ data               │                               │
     │                    │                               │
     │ SELECT             │                               │
     │   p.*,             │                               │
     │   json_agg(DISTINCT d.*) as documents,            │
     │   json_agg(DISTINCT wt.*) as work_types,          │
     │   json_agg(DISTINCT a.*) as aosr,                 │
     │   json_agg(DISTINCT m.*) as materials,            │
     │   json_agg(DISTINCT qd.*) as quality_docs         │
     │ FROM projects p                                   │
     │ LEFT JOIN documents d ON d.project_id = p.id      │
     │ LEFT JOIN work_types wt ON wt.project_id = p.id   │
     │ LEFT JOIN aosr a ON a.project_id = p.id           │
     │ LEFT JOIN materials m ON m.document_id = d.id     │
     │ LEFT JOIN quality_documents qd                    │
     │   ON qd.material_id = m.id                        │
     │ WHERE p.id = $1                                   │
     │ GROUP BY p.id                                     │
     ├───────────────────►│                               │
     │                    │                               │
     │                    │ Execution Plan:               │
     │                    │ 1. Index Scan on projects     │
     │                    │    (Index: pk_projects)       │
     │                    │ 2. Nested Loop Left Join      │
     │                    │    with documents             │
     │                    │    (Index: idx_docs_proj)     │
     │                    │ 3. Hash Left Join with        │
     │                    │    work_types                 │
     │                    │ 4. Hash Aggregate             │
     │                    │    (json_agg)                 │
     │                    │                               │
     │                    │ Estimated cost: 45.23        │ ✓ Fast
     │                    │ Actual time: 8ms             │   (indexed)
     │                    │                               │
     │ Result:            │                               │
     │ {                  │                               │
     │   id: "p1",        │                               │
     │   name: "...",     │                               │
     │   documents: [...],│                               │
     │   work_types: [...],                              │
     │   aosr: [...],     │                               │
     │   materials: [...] │                               │
     │ }                  │                               │
     │◄───────────────────┤                               │
     │                    │                               │

Indexes Used:
─────────────
CREATE INDEX idx_docs_project ON documents(project_id);
CREATE INDEX idx_work_types_project ON work_types(project_id);
CREATE INDEX idx_aosr_project ON aosr(project_id);
CREATE INDEX idx_materials_document ON materials(document_id);
CREATE INDEX idx_quality_docs_material ON quality_documents(material_id);

┌─────────────────────────────────────────────────────────────────────┐
│ QUERY 2: Полнотекстовый поиск документов                            │
└─────────────────────────────────────────────────────────────────────┘

Application          PostgreSQL                      Performance
───────────          ──────────                      ───────────
     │                    │                               │
     │ Search for         │                               │
     │ "труба REHAU"      │                               │
     │ in quality docs    │                               │
     │                    │                               │
     │ SELECT             │                               │
     │   qd.*,            │                               │
     │   m.name as material_name,                        │
     │   similarity(      │                               │
     │     qd.metadata->>'product_name',                 │
     │     'труба REHAU'  │                               │
     │   ) as relevance   │                               │
     │ FROM quality_documents qd                         │
     │ JOIN materials m ON m.id = qd.material_id         │
     │ WHERE                                             │
     │   qd.metadata->>'product_name'                    │
     │   ILIKE '%труба%'  │                               │
     │   AND qd.manufacturer ILIKE '%REHAU%'             │
     │ ORDER BY relevance DESC                           │
     │ LIMIT 10           │                               │
     ├───────────────────►│                               │
     │                    │                               │
     │                    │ Execution Plan:               │
     │                    │ 1. Bitmap Index Scan          │
     │                    │    (GIN index on metadata)    │
     │                    │ 2. Apply pg_trgm similarity   │
     │                    │ 3. Sort by relevance          │
     │                    │                               │
     │                    │ Estimated cost: 12.45        │ ✓ Fast
     │                    │ Actual time: 3ms             │   (GIN index)
     │                    │                               │
     │ Results (10 docs)  │                               │
     │◄───────────────────┤                               │
     │                    │                               │

Extensions & Indexes:
─────────────────────
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_quality_docs_metadata
  ON quality_documents USING GIN (metadata jsonb_path_ops);
CREATE INDEX idx_quality_docs_manufacturer
  ON quality_documents USING GIN (manufacturer gin_trgm_ops);

┌─────────────────────────────────────────────────────────────────────┐
│ QUERY 3: Вставка данных с транзакцией                               │
└─────────────────────────────────────────────────────────────────────┘

Application          PostgreSQL                      ACID
───────────          ──────────                      ────
     │                    │                               │
     │ Save RD Analysis   │                               │
     │ (multiple tables)  │                               │
     │                    │                               │
     │ BEGIN              │                               │
     ├───────────────────►│                               │
     │                    │ START TRANSACTION            │ Atomicity
     │                    │                               │ ─────────
     │ UPDATE documents   │                               │ All or
     │ SET                │                               │ nothing
     │   status='analyzed'│                               │
     │   extracted_text=  │                               │
     │   'very long text' │                               │
     │   metadata=jsonb   │                               │
     │ WHERE id = $1      │                               │
     ├───────────────────►│                               │
     │                    │ Row locked                    │
     │                    │                               │
     │ INSERT INTO        │                               │
     │ work_types (...)   │                               │
     │ VALUES             │                               │
     │   ('Монтаж...'),   │                               │
     │   ('Укладка...'),  │                               │
     │   ('Изоляция...')  │                               │
     │ RETURNING id       │                               │
     ├───────────────────►│                               │
     │                    │ 3 rows inserted               │
     │ IDs: [w1,w2,w3]    │                               │
     │◄───────────────────┤                               │
     │                    │                               │
     │ INSERT INTO        │                               │
     │ materials (...)    │                               │
     │ VALUES (...)       │                               │
     ├───────────────────►│                               │
     │                    │ 15 rows inserted              │
     │                    │                               │
     │ COMMIT             │                               │
     ├───────────────────►│                               │
     │                    │ COMMIT TRANSACTION           │ Consistency
     │                    │ Write to WAL                 │ ───────────
     │                    │ Apply changes                │ Valid state
     │                    │                               │
     │ Success            │                               │ Durability
     │◄───────────────────┤                               │ ──────────
     │                    │                               │ Persisted
     │                    │                               │ to disk

If error occurs:
────────────────
     │ INSERT fails       │                               │
     │◄───────────────────┤                               │
     │                    │                               │
     │ ROLLBACK           │                               │
     ├───────────────────►│                               │
     │                    │ ROLLBACK TRANSACTION         │ Rollback
     │                    │ Undo all changes             │ ────────
     │                    │ Release locks                │ No partial
     │ Error returned     │                               │ updates
     │◄───────────────────┤                               │

┌─────────────────────────────────────────────────────────────────────┐
│ QUERY 4: JSONB операции для метаданных                              │
└─────────────────────────────────────────────────────────────────────┘

Application          PostgreSQL                      JSONB Power
───────────          ──────────                      ───────────
     │                    │                               │
     │ Find materials     │                               │
     │ with diameter > 50 │                               │
     │                    │                               │
     │ SELECT             │                               │
     │   m.*,             │                               │
     │   m.metadata->>'diameter' as diameter,            │
     │   m.metadata->>'pressure' as pressure             │
     │ FROM materials m   │                               │
     │ WHERE              │                               │
     │   (m.metadata->>'diameter')::numeric > 50         │
     │   AND m.metadata->>'material_type'                │
     │     = 'труба'      │                               │
     ├───────────────────►│                               │
     │                    │                               │
     │                    │ JSONB operators:              │
     │                    │ ->> : get as text             │
     │                    │ -> : get as jsonb             │
     │                    │ @> : contains                 │
     │                    │ ? : key exists                │
     │                    │                               │
     │                    │ GIN index scan on             │
     │                    │ metadata column               │
     │                    │                               │
     │ Results            │                               │
     │◄───────────────────┤                               │
     │                    │                               │
     │ Update nested      │                               │
     │ field in JSONB     │                               │
     │                    │                               │
     │ UPDATE materials   │                               │
     │ SET metadata =     │                               │
     │   jsonb_set(       │                               │
     │     metadata,      │                               │
     │     '{specs,diameter}',                           │
     │     '"63"',        │                               │
     │     true           │                               │
     │   )                │                               │
     │ WHERE id = $1      │                               │
     ├───────────────────►│                               │
     │                    │                               │
     │                    │ Updated:                      │
     │                    │ metadata = {                  │
     │                    │   "specs": {                  │
     │                    │     "diameter": "63",         │
     │                    │     "pressure": "10"          │
     │                    │   }                           │
     │                    │ }                             │
     │                    │                               │
     │ Success            │                               │
     │◄───────────────────┤                               │
     │                    │                               │

JSONB Best Practices:
─────────────────────
1. Use GIN indexes for fast JSONB queries
2. Store structured data in JSONB when schema varies
3. Use JSONB operators for efficient filtering
4. Regular columns for frequently queried fields
5. JSONB for flexible metadata
```

---

## 8. Схема аутентификации и авторизации

### 8.1 JWT Authentication Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ ФАЗА 1: РЕГИСТРАЦИЯ И LOGIN                                          │
└─────────────────────────────────────────────────────────────────────┘

Frontend          Backend API         PostgreSQL        Redis
────────          ───────────         ──────────        ─────
   │                   │                   │               │
   │ POST /register    │                   │               │
   │ {                 │                   │               │
   │   email: "...",   │                   │               │
   │   password: "..." │                   │               │
   │ }                 │                   │               │
   ├──────────────────►│                   │               │
   │                   │                   │               │
   │                   │ Validate Input    │               │
   │                   │ - Email format    │               │
   │                   │ - Password length │               │
   │                   │                   │               │
   │                   │ Hash Password     │               │
   │                   │ (bcrypt)          │               │
   │                   │                   │               │
   │                   │ import bcrypt     │               │
   │                   │ hash = bcrypt.    │               │
   │                   │   hashpw(         │               │
   │                   │     password,     │               │
   │                   │     bcrypt.       │               │
   │                   │     gensalt(12)   │               │
   │                   │   )               │               │
   │                   │                   │               │
   │                   │ Save User         │               │
   │                   │ INSERT INTO users │               │
   │                   │ (email, pass_hash)│               │
   │                   ├──────────────────►│               │
   │                   │                   │               │
   │                   │ User Created      │               │
   │                   │ id = "u_123"      │               │
   │                   │◄──────────────────┤               │
   │                   │                   │               │
   │ 201 Created       │                   │               │
   │ {user_id: "..."}  │                   │               │
   │◄──────────────────┤                   │               │
   │                   │                   │               │
   │ POST /login       │                   │               │
   │ {                 │                   │               │
   │   email: "...",   │                   │               │
   │   password: "..." │                   │               │
   │ }                 │                   │               │
   ├──────────────────►│                   │               │
   │                   │                   │               │
   │                   │ Get User          │               │
   │                   │ SELECT * FROM     │               │
   │                   │ users             │               │
   │                   │ WHERE email=$1    │               │
   │                   ├──────────────────►│               │
   │                   │                   │               │
   │                   │ User data         │               │
   │                   │◄──────────────────┤               │
   │                   │                   │               │
   │                   │ Verify Password   │               │
   │                   │ bcrypt.checkpw(   │               │
   │                   │   password,       │               │
   │                   │   stored_hash     │               │
   │                   │ )                 │               │
   │                   │                   │               │
   │                   │ ✓ Match           │               │
   │                   │                   │               │
   │                   │ Generate JWT      │               │
   │                   │ payload = {       │               │
   │                   │   user_id: "...", │               │
   │                   │   email: "...",   │               │
   │                   │   role: "user",   │               │
   │                   │   exp: now+1h     │               │
   │                   │ }                 │               │
   │                   │                   │               │
   │                   │ token = jwt.encode│               │
   │                   │   (payload,       │               │
   │                   │    SECRET_KEY,    │               │
   │                   │    algorithm=     │               │
   │                   │    "HS256")       │               │
   │                   │                   │               │
   │                   │ Generate Refresh  │               │
   │                   │ Token             │               │
   │                   │ refresh = jwt.    │               │
   │                   │   encode({        │               │
   │                   │     user_id,      │               │
   │                   │     exp: now+7d   │               │
   │                   │   })              │               │
   │                   │                   │               │
   │                   │ Cache Tokens      │               │
   │                   │ SET               │               │
   │                   │ session:u_123     │               │
   │                   │ {access, refresh} │               │
   │                   │ EX 604800         │               │
   │                   ├──────────────────────────────────►│
   │                   │                   │               │
   │ 200 OK            │                   │               │
   │ {                 │                   │               │
   │   access_token:   │                   │               │
   │   "eyJ...",       │                   │               │
   │   refresh_token:  │                   │               │
   │   "eyJ...",       │                   │               │
   │   expires_in: 3600│                   │               │
   │ }                 │                   │               │
   │◄──────────────────┤                   │               │
   │                   │                   │               │
   │ Store in          │                   │               │
   │ localStorage      │                   │               │
   │                   │                   │               │

┌─────────────────────────────────────────────────────────────────────┐
│ ФАЗА 2: AUTHENTICATED REQUEST                                        │
└─────────────────────────────────────────────────────────────────────┘

Frontend          Middleware          Redis           Backend
────────          ──────────          ─────           ───────
   │                   │                │                │
   │ GET /api/v1/      │                │                │
   │ projects          │                │                │
   │                   │                │                │
   │ Headers:          │                │                │
   │ Authorization:    │                │                │
   │ Bearer eyJ...     │                │                │
   ├──────────────────►│                │                │
   │                   │                │                │
   │                   │ Extract Token  │                │
   │                   │ from header    │                │
   │                   │                │                │
   │                   │ Verify JWT     │                │
   │                   │ try:           │                │
   │                   │   payload =    │                │
   │                   │   jwt.decode(  │                │
   │                   │     token,     │                │
   │                   │     SECRET_KEY,│                │
   │                   │     algorithms=│                │
   │                   │     ["HS256"]  │                │
   │                   │   )            │                │
   │                   │                │                │
   │                   │ ✓ Valid        │                │
   │                   │                │                │
   │                   │ Check Blacklist│                │
   │                   │ GET blacklist: │                │
   │                   │ {token_hash}   │                │
   │                   ├───────────────►│                │
   │                   │                │                │
   │                   │ Not blacklisted│                │
   │                   │◄───────────────┤                │
   │                   │                │                │
   │                   │ Attach User    │                │
   │                   │ to Request     │                │
   │                   │                │                │
   │                   │ request.user = │                │
   │                   │ {              │                │
   │                   │   id: "u_123", │                │
   │                   │   email: "..." │                │
   │                   │   role: "user" │                │
   │                   │ }              │                │
   │                   │                │                │
   │                   │ Forward Request│                │
   │                   ├───────────────────────────────►│
   │                   │                │                │
   │                   │                │                │ Process
   │                   │                │                │ request
   │                   │                │                │ with user
   │                   │                │                │ context
   │                   │                │                │
   │                   │ Response       │                │
   │                   │◄───────────────────────────────┤
   │                   │                │                │
   │ 200 OK            │                │                │
   │ {data: [...]}     │                │                │
   │◄──────────────────┤                │                │
   │                   │                │                │

┌─────────────────────────────────────────────────────────────────────┐
│ ФАЗА 3: TOKEN REFRESH                                                │
└─────────────────────────────────────────────────────────────────────┘

Frontend          Backend API         Redis
────────          ───────────         ─────
   │                   │                │
   │ Access token      │                │
   │ expired           │                │
   │ (401 Unauthorized)│                │
   │                   │                │
   │ POST /refresh     │                │
   │ {                 │                │
   │   refresh_token:  │                │
   │   "eyJ..."        │                │
   │ }                 │                │
   ├──────────────────►│                │
   │                   │                │
   │                   │ Verify Refresh │
   │                   │ Token          │
   │                   │ payload =      │
   │                   │ jwt.decode(    │
   │                   │   refresh_token│
   │                   │ )              │
   │                   │                │
   │                   │ ✓ Valid        │
   │                   │                │
   │                   │ Check Session  │
   │                   │ GET session:   │
   │                   │ {user_id}      │
   │                   ├───────────────►│
   │                   │                │
   │                   │ Session exists │
   │                   │◄───────────────┤
   │                   │                │
   │                   │ Generate New   │
   │                   │ Access Token   │
   │                   │                │
   │                   │ new_token =    │
   │                   │ jwt.encode({   │
   │                   │   user_id,     │
   │                   │   exp: now+1h  │
   │                   │ })             │
   │                   │                │
   │ 200 OK            │                │
   │ {                 │                │
   │   access_token:   │                │
   │   "eyJ...",       │                │
   │   expires_in: 3600│                │
   │ }                 │                │
   │◄──────────────────┤                │
   │                   │                │
   │ Update token      │                │
   │ & retry request   │                │
   │                   │                │

┌─────────────────────────────────────────────────────────────────────┐
│ ФАЗА 4: LOGOUT & TOKEN BLACKLIST                                     │
└─────────────────────────────────────────────────────────────────────┘

Frontend          Backend API         Redis
────────          ───────────         ─────
   │                   │                │
   │ POST /logout      │                │
   │ {                 │                │
   │   token: "eyJ..." │                │
   │ }                 │                │
   ├──────────────────►│                │
   │                   │                │
   │                   │ Decode Token   │
   │                   │ Get exp time   │
   │                   │                │
   │                   │ Blacklist Token│
   │                   │ SET blacklist: │
   │                   │ {token_hash}   │
   │                   │ 1              │
   │                   │ EX {ttl}       │
   │                   ├───────────────►│
   │                   │                │
   │                   │ Delete Session │
   │                   │ DEL session:   │
   │                   │ {user_id}      │
   │                   ├───────────────►│
   │                   │                │
   │ 200 OK            │                │
   │ {message: "..."}  │                │
   │◄──────────────────┤                │
   │                   │                │
   │ Clear localStorage│                │
   │                   │                │
```

### 8.2 Role-Based Access Control (RBAC)

```
┌─────────────────────────────────────────────────────────────────────┐
│ RBAC: Permission Checking                                            │
└─────────────────────────────────────────────────────────────────────┘

Request         Permission Check      User Roles       Result
───────         ────────────────      ──────────       ──────
   │                   │                   │              │
   │ DELETE /api/v1/   │                   │              │
   │ projects/{id}     │                   │              │
   ├──────────────────►│                   │              │
   │                   │                   │              │
   │                   │ Get User Role     │              │
   │                   │ from JWT          │              │
   │                   │                   │              │
   │                   │ user.role = "user"│              │
   │                   │◄──────────────────┤              │
   │                   │                   │              │
   │                   │ Check Permission  │              │
   │                   │                   │              │
   │                   │ PERMISSIONS = {   │              │
   │                   │   "user": [       │              │
   │                   │     "read:own",   │              │
   │                   │     "write:own",  │              │
   │                   │     "delete:own"  │              │
   │                   │   ],              │              │
   │                   │   "admin": [      │              │
   │                   │     "read:all",   │              │
   │                   │     "write:all",  │              │
   │                   │     "delete:all"  │              │
   │                   │   ]               │              │
   │                   │ }                 │              │
   │                   │                   │              │
   │                   │ Required:         │              │
   │                   │ "delete:projects" │              │
   │                   │                   │              │
   │                   │ Check ownership:  │              │
   │                   │ project.user_id   │              │
   │                   │ == current_user.id│              │
   │                   │                   │              │
   │                   │ ✓ Owner           │              │
   │                   │                   │              │
   │                   │ ALLOW             │              │
   │                   ├──────────────────────────────────►
   │                   │                   │              │
   │ Proceed with      │                   │              │
   │ deletion          │                   │              │
   │                   │                   │              │

Decorator Example (Python):
──────────────────────────

from functools import wraps

def require_permission(permission: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(request: Request, *args, **kwargs):
            user = request.state.user

            # Check if user has permission
            if not has_permission(user.role, permission):
                raise HTTPException(403, "Forbidden")

            # Check resource ownership for "own" permissions
            if permission.endswith(":own"):
                resource_id = kwargs.get('id')
                resource = await get_resource(resource_id)
                if resource.user_id != user.id:
                    raise HTTPException(403, "Not your resource")

            return await func(request, *args, **kwargs)
        return wrapper
    return decorator

@router.delete("/projects/{id}")
@require_permission("delete:projects")
async def delete_project(id: str, request: Request):
    # User already authorized by decorator
    await project_service.delete(id)
    return {"message": "Deleted"}
```

---

## 9. Итоговая схема: Полный цикл жизни документа

```
┌─────────────────────────────────────────────────────────────────────┐
│ COMPLETE DOCUMENT LIFECYCLE                                          │
└─────────────────────────────────────────────────────────────────────┘

User Action          System Response          Data State
───────────          ───────────────          ──────────

📤 Upload PDF        → FastAPI receives       documents:
(15 MB)                multipart/form-data     - status: 'uploading'
                                                - file_url: null
                       → Save to temp
                       → Create DB record

                     → Enqueue Celery task    documents:
                                                - status: 'processing'

                     → Upload to S3           documents:
                       (background)             - file_url: 's3://...'

🔍 OCR Extract       → GPT-4o Vision          documents:
(30-60 sec)            processes each page     - extracted_text: '...'

                     → Cache in Redis         redis:
                                                ocr:doc_id → text

🤖 AI Analysis       → RD Analyzer Agent      work_types:
(20-40 sec)            extracts structure      - 3 records inserted

                     → Save to PostgreSQL     materials:
                                                - 15 records inserted

                                              documents:
                                                - status: 'analyzed'
                                                - metadata: {...}

✅ Notify User       → WebSocket push         frontend:
                                                - Show success
                                                - Enable AOSR gen

📝 Generate AOSR     → User clicks button     aosr:
(10-20 sec)                                     - status: 'draft'
                     → AOSR Generator Agent

                     → ReportLab creates PDF  aosr:
                                                - file_url: 's3://...'
                                                - version: 1

🔎 Search Docs       → Search Agent           quality_documents:
(2-5 min/material)     4-level search          - 15 records found

                     → Web scraping           - confidence: 0.85-0.95
                     → GPT validation

✓ Validate           → Validator Agent        validation_reports:
(5-10 sec)             5-phase check           - status: 'pass'
                                                - errors: 0

📦 Create Package    → Package Creator        packages:
(30-60 sec)            merges all PDFs         - file_url: 's3://...'
                                                - size: 45 MB

⬇️ Download          → User downloads         event_log:
                       final package            - action: 'download'

═══════════════════════════════════════════════════════════════════════

TIMELINE:
─────────
0:00    Upload starts
0:10    Upload complete → Processing starts
0:40    OCR complete
1:20    Analysis complete → User notified
1:25    User generates AOSR
1:45    AOSR ready
1:50    Document search starts (parallel)
6:50    All quality docs found
6:55    Validation runs
7:05    Validation complete → Package creation enabled
7:10    User creates package
8:10    Package ready → User downloads

TOTAL TIME: ~8 minutes (vs 5 days manual)
REDUCTION: 99.9%

DATA VOLUME:
────────────
Input:  15 MB (PDF)
Stored: 45 MB (full package)
Cached: 200 KB (Redis: OCR text, analysis)
DB:     150 KB (PostgreSQL: structured data)
```

---

## Заключение

Эти схемы показывают:

1. **Общую архитектуру** - как все компоненты связаны
2. **Загрузку и анализ РД** - полный цикл с OCR и AI
3. **Генерацию АОСР** - от данных до PDF
4. **Поиск документов** - многоуровневый с web-scraping
5. **Валидацию** - 5 типов проверок с AI
6. **AI-агенты** - последовательность, параллелизм, ветвление
7. **База данных** - структура, запросы, оптимизация
8. **Аутентификация** - JWT, RBAC, безопасность
9. **Полный цикл** - от загрузки до готового пакета

Каждая схема включает:
- ASCII-диаграммы для визуализации
- Примеры кода (Python/SQL/JavaScript)
- Timing и performance метрики
- Best practices и оптимизации