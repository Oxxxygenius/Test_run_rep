# Поток данных в платформе

**Дата создания:** 2025-12-14
**Для кого:** Для тех, кто хочет понять, как данные трансформируются на каждом этапе

---

## 🎯 Главная идея

Данные проходят через платформу, как сырьё на фабрике:
- **Вход:** Хаотичные документы (PDF, сканы, Word)
- **Процесс:** Анализ, структурирование, генерация
- **Выход:** Готовый комплект ИД (один PDF файл со всем)

---

## 📊 Полный путь данных

### Этап 1: Загрузка документа

**Вход:**
```
Файл от пользователя:
- Название: "Раздел ОВ.pdf"
- Размер: 15 MB
- Формат: PDF
- Содержимое: Проектная документация (текст + таблицы + чертежи)
```

**Обработка:**
```javascript
// Фронтенд (React)
const file = event.target.files[0];

// Валидация
if (file.size > 50 * 1024 * 1024) {
  alert("Файл слишком большой (макс 50MB)");
  return;
}

if (!file.type.includes('pdf')) {
  alert("Только PDF файлы");
  return;
}

// Отправка на backend
const formData = new FormData();
formData.append('file', file);
formData.append('project_id', 123);
formData.append('doc_type', 'РД');

fetch('/api/v1/documents/upload', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`
  },
  body: formData
});
```

**Выход:**
```json
{
  "document_id": 456,
  "filename": "Раздел ОВ.pdf",
  "status": "uploaded",
  "task_id": "abc-123-xyz"
}
```

**Что сохраняется в БД:**
```sql
INSERT INTO documents (
  project_id,
  filename,
  file_path,
  doc_type,
  mime_type,
  file_size,
  uploaded_by
) VALUES (
  123,
  'Раздел ОВ.pdf',
  'projects/123/docs/rd_456.pdf',
  'РД',
  'application/pdf',
  15728640,
  1
);
```

---

### Этап 2: Извлечение текста (если PDF)

**Вход:**
```
Файл: projects/123/docs/rd_456.pdf
```

**Обработка (Python):**
```python
import PyMuPDF  # fitz

# Открываем PDF
doc = fitz.open("projects/123/docs/rd_456.pdf")

# Извлекаем текст со всех страниц
full_text = ""
for page_num in range(len(doc)):
    page = doc[page_num]
    full_text += page.get_text()

print(f"Извлечено {len(full_text)} символов")
```

**Выход:**
```
Текст (строка):
"ПРОЕКТНАЯ ДОКУМЕНТАЦИЯ
Раздел: Отопление и вентиляция
...
Спецификация оборудования:
1. Труба ПНД d=32мм ГОСТ 18599-2001 - 150 м
2. Фитинги соединительные - 25 шт
..."

Длина: ~50,000 символов
```

**Сохранение в БД:**
```sql
UPDATE documents
SET ocr_text = 'ПРОЕКТНАЯ ДОКУМЕНТАЦИЯ...'
WHERE id = 456;
```

---

### Этап 3: Анализ проектной документации через LLM

**Вход в GPT-4o:**
```python
prompt = f"""
Ты эксперт по строительной документации.

Задача: Проанализируй спецификацию и определи:
1. Какие виды работ нужно оформлять АОСР
2. Список материалов для каждого вида работ
3. Количество и единицы измерения

Верни результат в JSON формате:
{{
  "works": [
    {{
      "type": "Название работ",
      "materials": [
        {{"name": "...", "quantity": ..., "unit": "...", "gost": "..."}}
      ]
    }}
  ]
}}

ДОКУМЕНТ:
{full_text}
"""

response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Ты эксперт по строительной документации."},
        {"role": "user", "content": prompt}
    ],
    temperature=0.2,
    response_format={"type": "json_object"}
)

result = json.loads(response.choices[0].message.content)
```

**Выход от GPT-4o:**
```json
{
  "works": [
    {
      "type": "Монтаж трубопроводов системы отопления",
      "description": "Монтаж трубопроводов из труб ПНД",
      "materials": [
        {
          "name": "Труба ПНД",
          "specification": "d=32мм",
          "gost": "ГОСТ 18599-2001",
          "quantity": 150,
          "unit": "м",
          "manufacturer": null
        },
        {
          "name": "Фитинги соединительные",
          "specification": "d=32мм",
          "gost": null,
          "quantity": 25,
          "unit": "шт",
          "manufacturer": null
        }
      ]
    }
  ]
}
```

**Сохранение в БД:**
```sql
INSERT INTO aosr (
  project_id,
  work_type,
  work_description,
  content,
  status
) VALUES (
  123,
  'Монтаж трубопроводов системы отопления',
  'Монтаж трубопроводов из труб ПНД',
  '{"works": [...], "materials": [...]}'::jsonb,
  'draft'
);
```

---

### Этап 4: Генерация АОСР

**Вход:**
```python
# Данные из БД
aosr_data = {
  "id": 789,
  "work_type": "Монтаж трубопроводов системы отопления",
  "materials": [
    {"name": "Труба ПНД d=32мм", "quantity": 150, "unit": "м"},
    {"name": "Фитинги соединительные", "quantity": 25, "unit": "шт"}
  ],
  "work_date": "2024-12-10",
  "project": {
    "name": "ЖК Солнечный",
    "address": "г. Москва, ул. Ленина, д. 1"
  },
  "responsible_persons": {
    "contractor": "ООО СтройПром",
    "engineer": "Иванов И.И.",
    "supervisor": "Петров П.П."
  }
}
```

**Обработка:**
```python
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas

def generate_aosr_pdf(data):
    filename = f"aosr_{data['id']}.pdf"
    c = canvas.Canvas(filename, pagesize=A4)

    # Заголовок
    c.drawString(100, 800, "АКТЫ ОСВИДЕТЕЛЬСТВОВАНИЯ СКРЫТЫХ РАБОТ")
    c.drawString(100, 780, f"Объект: {data['project']['name']}")
    c.drawString(100, 760, f"Адрес: {data['project']['address']}")

    # Таблица работ
    y = 720
    c.drawString(100, y, "Наименование работ:")
    c.drawString(100, y-20, data['work_type'])

    # Таблица материалов
    y = 660
    c.drawString(100, y, "Применённые материалы:")

    for i, material in enumerate(data['materials']):
        y -= 20
        line = f"{i+1}. {material['name']} - {material['quantity']} {material['unit']}"
        c.drawString(120, y, line)

    # Подписи
    y = 400
    c.drawString(100, y, f"Подрядчик: {data['responsible_persons']['contractor']}")
    c.drawString(100, y-20, f"Инженер ПТО: {data['responsible_persons']['engineer']}")
    c.drawString(100, y-40, f"Технический надзор: {data['responsible_persons']['supervisor']}")

    # Дата
    c.drawString(100, 100, f"Дата: {data['work_date']}")

    c.save()
    return filename
```

**Выход:**
```
Файл: aosr_789.pdf
Размер: 250 KB
Формат: PDF/A (для архивного хранения)
```

**Сохранение:**
```python
# Загружаем в Object Storage
s3_client.upload_file(
    'aosr_789.pdf',
    'pto-platform',
    'projects/123/aosr/aosr_789.pdf'
)

# Обновляем БД
UPDATE aosr
SET
  status = 'generated',
  generated_pdf_path = 'projects/123/aosr/aosr_789.pdf',
  generated_at = NOW()
WHERE id = 789;
```

---

### Этап 5: Поиск документов качества

**Вход:**
```python
material = {
  "name": "Труба ПНД",
  "gost": "ГОСТ 18599-2001",
  "quantity": 150,
  "unit": "м"
}
```

**Шаг 5.1: Поиск в локальной базе**

```sql
SELECT * FROM documents
WHERE project_id = 123
  AND doc_type IN ('сертификат', 'декларация', 'паспорт')
  AND (
    ocr_text ILIKE '%Труба ПНД%'
    OR ocr_text ILIKE '%ГОСТ 18599-2001%'
  );
```

**Если найдено:**
```json
{
  "document_id": 234,
  "filename": "Сертификат_труба_ПНД.pdf",
  "match_score": 0.85,
  "metadata": {
    "material": "Труба ПНД",
    "gost": "ГОСТ 18599-2001",
    "manufacturer": "ООО Полипластик",
    "valid_until": "2027-01-01"
  }
}
```

**Шаг 5.2: Если НЕ найдено → Поиск в интернете**

```python
from playwright.sync_api import sync_playwright

def search_document_online(material_name, gost):
    with sync_playwright() as p:
        browser = p.chromium.launch()
        page = browser.new_page()

        # Заходим на сайт
        page.goto("https://www.santech.ru/")

        # Ищем
        search_query = f"{material_name} {gost} сертификат"
        page.fill('input[name="q"]', search_query)
        page.click('button[type="submit"]')

        # Ждём результатов
        page.wait_for_selector('.search-results')

        # Берём первый результат
        first_result = page.query_selector('.search-results .item')
        pdf_link = first_result.query_selector('a[href$=".pdf"]')

        if pdf_link:
            pdf_url = pdf_link.get_attribute('href')

            # Скачиваем PDF
            response = page.goto(pdf_url)
            pdf_content = response.body()

            # Сохраняем
            with open('found_certificate.pdf', 'wb') as f:
                f.write(pdf_content)

            browser.close()
            return 'found_certificate.pdf'

        browser.close()
        return None
```

**Шаг 5.3: Проверка релевантности через LLM**

```python
# Извлекаем текст из найденного PDF
found_text = extract_text_from_pdf('found_certificate.pdf')

# Спрашиваем GPT-4o
prompt = f"""
Проверь, подходит ли этот сертификат для материала "{material_name} {gost}".

Сертификат:
{found_text}

Ответь в JSON:
{{
  "matches": true/false,
  "confidence": 0-1,
  "reason": "почему подходит или не подходит"
}}
"""

response = openai.chat.completions.create(...)
result = json.loads(response.choices[0].message.content)

if result['matches'] and result['confidence'] > 0.8:
    # Сохраняем документ в проект
    save_document_to_project(...)
```

**Выход:**
```json
{
  "found": true,
  "source": "internet",
  "url": "https://www.santech.ru/certificates/123.pdf",
  "confidence": 0.92,
  "saved_as": "documents/cert_456.pdf"
}
```

---

### Этап 6: Формирование финального комплекта

**Вход:**
```python
project_id = 123
```

**Сбор всех файлов:**
```python
# Получаем все АОСР проекта
aosr_list = db.query(AOSR).filter(AOSR.project_id == 123).all()

# Получаем все связанные документы
all_files = []

for aosr in aosr_list:
    # АОСР PDF
    all_files.append({
        'type': 'aosr',
        'path': aosr.generated_pdf_path,
        'title': f"АОСР №{aosr.number}"
    })

    # Исполнительная схема
    if aosr.schema_document_id:
        schema = db.query(Document).filter(Document.id == aosr.schema_document_id).first()
        all_files.append({
            'type': 'schema',
            'path': schema.file_path,
            'title': f"Исполнительная схема к АОСР №{aosr.number}"
        })

    # Документы качества
    quality_docs = db.query(AOSRQualityDocument).filter(
        AOSRQualityDocument.aosr_id == aosr.id
    ).all()

    for qd in quality_docs:
        doc = db.query(Document).filter(Document.id == qd.document_id).first()
        all_files.append({
            'type': 'quality_doc',
            'path': doc.file_path,
            'title': f"{doc.doc_type} - {doc.filename}"
        })
```

**Объединение в один PDF:**
```python
from PyPDF2 import PdfMerger

def create_final_package(files, output_path):
    merger = PdfMerger()

    # Добавляем титульный лист
    merger.append(generate_title_page())

    # Добавляем общий реестр
    merger.append(generate_general_registry(files))

    # Добавляем все файлы по порядку
    for file in files:
        # Скачиваем из Object Storage
        local_path = download_from_storage(file['path'])

        # Добавляем в итоговый PDF
        merger.append(local_path)

        # Добавляем bookmark для навигации
        merger.add_outline_item(file['title'], len(merger.pages) - 1)

    # Сохраняем
    merger.write(output_path)
    merger.close()

    return output_path
```

**Выход:**
```
Файл: ИД_ЖК_Солнечный_Полный_комплект.pdf
Размер: 45 MB
Страниц: 250
Структура:
  ├─ Титульный лист
  ├─ Общий реестр
  ├─ АОСР №1
  ├─ Исполнительная схема к АОСР №1
  ├─ Сертификат на трубу ПНД
  ├─ Паспорт качества
  ├─ АОСР №2
  └─ ...
```

---

## 🔄 Диаграмма трансформации данных

```
RAW DATA (Вход)
├─ PDF файл (15 MB, неструктурированный)
│
│ ↓ [OCR / Text Extraction]
│
├─ Текст (50,000 символов, plain text)
│
│ ↓ [LLM Analysis]
│
├─ JSON структура (работы + материалы)
│  {
│    "works": [...],
│    "materials": [...]
│  }
│
│ ↓ [Template Filling]
│
├─ Formatted AOSR (PDF, 250 KB)
│
│ ↓ [Document Search]
│
├─ Quality Documents (сертификаты, паспорта)
│
│ ↓ [PDF Merge]
│
└─ FINAL PACKAGE (PDF, 45 MB, структурированный)
```

---

## 📦 Формат хранения на каждом этапе

| Этап | Формат | Где хранится | Размер (примерно) |
|------|--------|--------------|-------------------|
| Загрузка | PDF | Object Storage | 15 MB |
| Извлечение текста | TEXT | PostgreSQL (поле `ocr_text`) | 50 KB |
| Анализ LLM | JSON | PostgreSQL (поле `content` JSONB) | 5 KB |
| Генерация АОСР | PDF | Object Storage | 250 KB |
| Найденные документы | PDF | Object Storage | 1-5 MB каждый |
| Финальный комплект | PDF | Object Storage | 45 MB |

---

## 🔐 Метаданные документов

**Каждый документ имеет метаданные:**

```json
{
  "document_id": 456,
  "project_id": 123,
  "filename": "Раздел ОВ.pdf",
  "doc_type": "РД",
  "uploaded_at": "2024-12-14T10:00:00Z",
  "uploaded_by": "user_1",
  "file_size": 15728640,
  "mime_type": "application/pdf",
  "ocr_status": "completed",
  "ocr_confidence": 0.95,
  "metadata": {
    "extracted_from_ocr": {
      "document_number": "123-ОВ-ПД",
      "issue_date": "2024-01-15",
      "project_name": "ЖК Солнечный",
      "section": "Отопление и вентиляция"
    }
  }
}
```

---

## ⚡ Оптимизации потока данных

### 1. Кэширование результатов LLM

```python
import hashlib
import redis

redis_client = redis.Redis()

def get_cached_llm_result(prompt):
    # Создаём hash промпта
    prompt_hash = hashlib.sha256(prompt.encode()).hexdigest()

    # Проверяем кэш
    cached = redis_client.get(f"llm:{prompt_hash}")
    if cached:
        return json.loads(cached)

    # Если нет в кэше → запрос к LLM
    result = call_openai(prompt)

    # Сохраняем в кэш (на 24 часа)
    redis_client.setex(f"llm:{prompt_hash}", 86400, json.dumps(result))

    return result
```

### 2. Потоковая обработка больших PDF

```python
def process_large_pdf_in_chunks(pdf_path, chunk_size=10):
    """Обрабатывает PDF по частям (по 10 страниц)"""
    doc = fitz.open(pdf_path)
    total_pages = len(doc)

    for start in range(0, total_pages, chunk_size):
        end = min(start + chunk_size, total_pages)

        # Извлекаем текст из чанка
        chunk_text = ""
        for page_num in range(start, end):
            chunk_text += doc[page_num].get_text()

        # Обрабатываем чанк
        yield process_chunk(chunk_text)
```

### 3. Параллельная обработка материалов

```python
from concurrent.futures import ThreadPoolExecutor

def search_all_materials_parallel(materials):
    """Ищет документы для всех материалов параллельно"""
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = [
            executor.submit(search_document, material)
            for material in materials
        ]

        results = [future.result() for future in futures]
        return results
```

---

## 🎯 Следующие шаги

Прочитайте:
- [03-agents-interaction.md](03-agents-interaction.md) — Как ИИ-агенты координируются
- [04-scaling-strategy.md](04-scaling-strategy.md) — Как масштабировать поток данных

---

**Статус:** ✅ Актуально
**Последнее обновление:** 2025-12-14
