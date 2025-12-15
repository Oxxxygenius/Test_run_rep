# Работа с документами (PDF, Word, Excel)

**Дата создания:** 2025-12-13
**Приоритет:** ⭐⭐⭐⭐ (Высокий)

---

## Назначение

Обработка различных форматов документов:
- Чтение проектной документации (PDF, DOCX)
- Генерация PDF актов и реестров
- Работа с Excel спецификациями
- Объединение документов в единый PDF-комплект

---

## Рекомендуемая связка

### PyMuPDF + ReportLab + python-docx

```
┌─────────────────────────────────────────────────────┐
│            Document Processing Stack                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [PDF Reading]                                      │
│  └─ PyMuPDF (fitz)                                  │
│                                                     │
│  [PDF Generation]                                   │
│  └─ ReportLab + PyPDF2 (merge)                      │
│                                                     │
│  [Word Documents]                                   │
│  └─ python-docx                                     │
│                                                     │
│  [Excel Files]                                      │
│  └─ openpyxl + pandas                               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Конкретные инструменты

### 1. PyMuPDF (Чтение PDF)

**Установка:**
```bash
pip install pymupdf
```

**Пример чтения PDF:**
```python
import fitz  # PyMuPDF

def extract_text_from_pdf(pdf_path: str) -> str:
    """Извлечение текста из PDF"""
    doc = fitz.open(pdf_path)
    text = ""
    for page in doc:
        text += page.get_text()
    doc.close()
    return text

def extract_tables_from_pdf(pdf_path: str) -> list:
    """Извлечение таблиц из PDF"""
    doc = fitz.open(pdf_path)
    tables = []
    for page in doc:
        tabs = page.find_tables()
        for tab in tabs:
            tables.append(tab.extract())
    doc.close()
    return tables
```

**Официальный сайт:** https://pymupdf.readthedocs.io/

---

### 2. ReportLab (Генерация PDF)

**Что это:**
- Библиотека для создания PDF с нуля
- Полный контроль над макетом

**Установка:**
```bash
pip install reportlab
```

**Пример генерации АОСР:**
```python
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.lib.units import mm

# Регистрация русского шрифта
pdfmetrics.registerFont(TTFont('DejaVu', 'DejaVuSans.ttf'))

def generate_aosr_pdf(data: dict, output_path: str):
    """
    Генерация PDF акта АОСР

    ВАЖНО: Генерация выполняется согласно регламенту:
    docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку АОСР.xlsx

    Регламент определяет:
    - Обязательные разделы и поля АОСР
    - Формат таблиц материалов и оборудования
    - Требования к подписям ответственных лиц
    - Структуру документа согласно ГОСТ

    Args:
        data: Словарь с данными АОСР (номер, дата, объект, работы, материалы)
        output_path: Путь для сохранения PDF файла
    """
    c = canvas.Canvas(output_path, pagesize=A4)
    width, height = A4

    # Заголовок
    c.setFont("DejaVu", 14)
    c.drawCentredString(width/2, height - 30*mm, "АКТ ОСВИДЕТЕЛЬСТВОВАНИЯ СКРЫТЫХ РАБОТ")

    # Номер и дата
    c.setFont("DejaVu", 10)
    c.drawString(30*mm, height - 50*mm, f"№ {data['number']}")
    c.drawString(30*mm, height - 60*mm, f"Дата: {data['date']}")

    # Объект
    c.drawString(30*mm, height - 75*mm, f"Объект: {data['object_name']}")
    c.drawString(30*mm, height - 85*mm, f"Адрес: {data['address']}")

    # Таблица работ
    y = height - 110*mm
    c.drawString(30*mm, y, "Наименование работ:")
    y -= 10*mm

    for work in data['works']:
        c.drawString(35*mm, y, f"• {work['name']} - {work['volume']} {work['unit']}")
        y -= 5*mm

    # Подписи
    y = 50*mm
    c.drawString(30*mm, y, "Представитель подрядчика: _____________")
    c.drawString(30*mm, y - 10*mm, "Представитель заказчика: _____________")

    c.save()
```

**Официальный сайт:** https://www.reportlab.com/

---

### 3. python-docx (Работа с Word)

**Что это:**
- Чтение и создание файлов .docx
- Полезно для анализа проектной документации

**Установка:**
```bash
pip install python-docx
```

**Пример чтения Word:**
```python
from docx import Document

def extract_text_from_docx(docx_path: str) -> str:
    """Извлечение текста из Word документа"""
    doc = Document(docx_path)
    text = ""
    for paragraph in doc.paragraphs:
        text += paragraph.text + "\n"
    return text

def extract_tables_from_docx(docx_path: str) -> list:
    """Извлечение таблиц из Word"""
    doc = Document(docx_path)
    tables_data = []

    for table in doc.tables:
        table_data = []
        for row in table.rows:
            row_data = [cell.text for cell in row.cells]
            table_data.append(row_data)
        tables_data.append(table_data)

    return tables_data
```

---

### 4. openpyxl + pandas (Работа с Excel)

**Установка:**
```bash
pip install openpyxl pandas
```

**Пример чтения спецификации:**
```python
import pandas as pd

def read_specification_from_excel(excel_path: str) -> pd.DataFrame:
    """Чтение спецификации материалов из Excel"""
    df = pd.read_excel(excel_path, engine='openpyxl')
    return df

def parse_materials_specification(excel_path: str) -> list:
    """Парсинг спецификации материалов"""
    df = read_specification_from_excel(excel_path)

    materials = []
    for _, row in df.iterrows():
        material = {
            'name': row.get('Наименование', ''),
            'manufacturer': row.get('Производитель', ''),
            'quantity': row.get('Количество', 0),
            'unit': row.get('Единица измерения', ''),
            'gost': row.get('ГОСТ/ТУ', '')
        }
        materials.append(material)

    return materials
```

---

### 5. PyPDF2 (Объединение PDF)

**Что это:**
- Слияние нескольких PDF в один документ
- Для формирования итогового комплекта ИД

**Установка:**
```bash
pip install PyPDF2
```

**Пример объединения:**
```python
from PyPDF2 import PdfMerger

def merge_pdfs(pdf_list: list[str], output_path: str):
    """Объединение нескольких PDF в один"""
    merger = PdfMerger()

    for pdf in pdf_list:
        merger.append(pdf)

    merger.write(output_path)
    merger.close()

# Использование
merge_pdfs([
    'aosr_1.pdf',
    'aosr_2.pdf',
    'реестр.pdf',
    'документы_качества.pdf'
], 'комплект_ид.pdf')
```

---

## Генерация комплекта ИД (полный workflow)

**ВАЖНО:** Детальная архитектура генерации финального комплекта описана в [architecture/10-final-package-generation.md](../architecture/10-final-package-generation.md)

### Краткий обзор

Генерация финального комплекта включает:

1. **PDF файл** — единый документ с:
   - Титульным листом
   - Реестром исполнительной документации (с нумерацией страниц)
   - Всеми АОСР
   - Исполнительными схемами
   - Документами качества

2. **ZIP архив** со структурой:
   ```
   ├── 1. Исполнительная документация в формате PDF/
   ├── 2. Исполнительная документация в формате Excel/
   │   ├── Общий реестр.xlsx
   │   └── АОСР №1.xlsx, АОСР №2.xlsx, ...
   ├── 4. Геодезические схемы в формате DWG/
   └── 5. Паспорта, сертификаты и лабораторные заключения/
   ```

### Пример кода

```python
from typing import List
from dataclasses import dataclass

@dataclass
class AOSRData:
    number: str
    date: str
    object_name: str
    address: str
    works: List[dict]

@dataclass
class QualityDocument:
    filename: str
    doc_type: str
    material_name: str

def generate_full_id_package(project_id: int, output_pdf_path: str, output_zip_path: str):
    """
    Генерация полного комплекта исполнительной документации

    Возвращает:
        dict: {
            'pdf_path': str,  # Путь к финальному PDF
            'zip_path': str   # Путь к архиву с редактируемыми файлами
        }
    """
    from PyPDF2 import PdfMerger
    import zipfile

    merger = PdfMerger()
    temp_files = []

    # 1. Титульный лист
    title_pdf = f"temp_title_{project_id}.pdf"
    generate_title_page(project_id, title_pdf)
    merger.append(title_pdf)
    temp_files.append(title_pdf)

    # 2. Общий реестр (с правильной нумерацией страниц)
    registry_pdf = f"temp_registry_{project_id}.pdf"
    generate_main_registry(project_id, registry_pdf)
    merger.append(registry_pdf)
    temp_files.append(registry_pdf)

    # 3. АОСР с приложениями
    aosr_list = get_aosr_list(project_id)
    for aosr in aosr_list:
        # АОСР
        aosr_pdf = f"temp_aosr_{aosr.id}.pdf"
        generate_aosr_pdf(aosr, aosr_pdf)
        merger.append(aosr_pdf)
        temp_files.append(aosr_pdf)

        # Исполнительная схема
        if aosr.schema_path:
            merger.append(aosr.schema_path)

        # Реестр документов качества для этого АОСР
        docs_registry_pdf = f"temp_docs_reg_{aosr.id}.pdf"
        generate_quality_docs_registry(aosr.id, docs_registry_pdf)
        merger.append(docs_registry_pdf)
        temp_files.append(docs_registry_pdf)

        # Документы качества
        quality_docs = get_quality_docs_for_aosr(aosr.id)
        for doc in quality_docs:
            merger.append(doc.path)

    # 4. Сохранение итогового PDF
    merger.write(output_pdf_path)
    merger.close()

    # 5. Создание ZIP архива с редактируемыми файлами
    create_editable_archive(project_id, aosr_list, output_zip_path)

    # 6. Удаление временных файлов
    import os
    for temp_file in temp_files:
        os.remove(temp_file)

    return {
        'pdf_path': output_pdf_path,
        'zip_path': output_zip_path
    }


def create_editable_archive(project_id: int, aosr_list: List, output_zip_path: str):
    """
    Создаёт ZIP архив с редактируемыми версиями документов
    """
    import zipfile
    import openpyxl

    with zipfile.ZipFile(output_zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:

        # Папка 1: PDF файлы
        for aosr in aosr_list:
            zipf.write(aosr.pdf_path, f"1. Исполнительная документация в формате PDF/АОСР {aosr.number}.pdf")

        # Папка 2: Excel файлы
        # Общий реестр
        registry_excel = f"temp_registry_{project_id}.xlsx"
        generate_registry_excel(project_id, registry_excel)
        zipf.write(registry_excel, "2. Исполнительная документация в формате Excel/Общий реестр.xlsx")
        os.remove(registry_excel)

        # АОСР в Excel
        for aosr in aosr_list:
            aosr_excel = f"temp_aosr_{aosr.id}.xlsx"
            generate_aosr_excel(aosr, aosr_excel)
            zipf.write(aosr_excel, f"2. Исполнительная документация в формате Excel/АОСР {aosr.number}.xlsx")
            os.remove(aosr_excel)

        # Папка 5: Документы качества
        for aosr in aosr_list:
            quality_docs = get_quality_docs_for_aosr(aosr.id)
            for i, doc in enumerate(quality_docs, 1):
                zipf.write(doc.path, f"5. Паспорта, сертификаты и лабораторные заключения/{i:03d}_{doc.filename}")


def generate_registry_excel(project_id: int, output_path: str):
    """
    Генерирует реестр исполнительной документации в Excel
    """
    import openpyxl
    from openpyxl.styles import Font, Alignment, Border, Side

    wb = openpyxl.Workbook()
    ws = wb.active
    ws.title = "Реестр ИД"

    # Заголовок
    ws.merge_cells('A1:G1')
    ws['A1'] = "РЕЕСТР ИСПОЛНИТЕЛЬНОЙ ДОКУМЕНТАЦИИ"
    ws['A1'].font = Font(bold=True, size=14)
    ws['A1'].alignment = Alignment(horizontal='center')

    # Заголовки таблицы
    headers = ["№ п/п", "Наименование документа", "Содержание документа",
               "№ документа", "Дата документа", "Кол-во листов", "Страница по списку"]

    for col, header in enumerate(headers, start=1):
        cell = ws.cell(row=3, column=col)
        cell.value = header
        cell.font = Font(bold=True)
        cell.alignment = Alignment(horizontal='center')

    # Заполнение данных
    # ... (код заполнения таблицы)

    wb.save(output_path)


def generate_aosr_excel(aosr: AOSRData, output_path: str):
    """
    Генерирует АОСР в формате Excel на основе шаблона

    ВАЖНО: Генерация выполняется согласно регламенту:
    docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку АОСР.xlsx

    Используется шаблон:
    docs/technical/info/04_Форма АОСР.xlsx

    Args:
        aosr: Данные АОСР (номер, дата, объект, работы)
        output_path: Путь для сохранения Excel файла
    """
    import openpyxl

    # Загружаем шаблон из регламентирующих документов
    template_path = "docs/technical/info/04_Форма АОСР.xlsx"
    wb = openpyxl.load_workbook(template_path)
    ws = wb.active

    # Заполняем поля согласно регламенту
    ws['B5'] = aosr.number
    ws['E5'] = aosr.date.strftime("%d.%m.%Y")
    ws['B10'] = aosr.work_description

    wb.save(output_path)
```

**Смотрите полную реализацию:** [architecture/10-final-package-generation.md](../architecture/10-final-package-generation.md)

---

## Шаблоны документов

### Использование Jinja2 для шаблонов

**Установка:**
```bash
pip install jinja2
```

**Пример шаблона АОСР (HTML → PDF):**

```python
from jinja2 import Template
from weasyprint import HTML

# Шаблон АОСР
aosr_template = """
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <style>
        body { font-family: 'DejaVu Sans', sans-serif; }
        .header { text-align: center; font-weight: bold; }
        table { width: 100%; border-collapse: collapse; }
        td, th { border: 1px solid black; padding: 5px; }
    </style>
</head>
<body>
    <div class="header">
        <h2>АКТ ОСВИДЕТЕЛЬСТВОВАНИЯ СКРЫТЫХ РАБОТ</h2>
        <p>№ {{ number }} от {{ date }}</p>
    </div>

    <p><strong>Объект:</strong> {{ object_name }}</p>
    <p><strong>Адрес:</strong> {{ address }}</p>

    <h3>Наименование работ:</h3>
    <table>
        <tr>
            <th>№</th>
            <th>Наименование</th>
            <th>Объем</th>
            <th>Единица измерения</th>
        </tr>
        {% for work in works %}
        <tr>
            <td>{{ loop.index }}</td>
            <td>{{ work.name }}</td>
            <td>{{ work.volume }}</td>
            <td>{{ work.unit }}</td>
        </tr>
        {% endfor %}
    </table>
</body>
</html>
"""

def generate_aosr_from_template(data: dict, output_path: str):
    """
    Генерация АОСР из HTML шаблона

    ВАЖНО: Генерация выполняется согласно регламенту:
    docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку АОСР.xlsx

    Args:
        data: Данные для заполнения шаблона (номер, дата, объект, работы)
        output_path: Путь для сохранения PDF файла
    """
    template = Template(aosr_template)
    html_content = template.render(**data)

    # Конвертация HTML в PDF
    HTML(string=html_content).write_pdf(output_path)
```

**Установка WeasyPrint:**
```bash
pip install weasyprint
```

---

## Альтернативные инструменты

### 1. pdfkit (HTML → PDF)

**Что это:**
- Обертка над wkhtmltopdf
- Проще, чем WeasyPrint

**Установка:**
```bash
pip install pdfkit
# Также нужен wkhtmltopdf: https://wkhtmltopdf.org/downloads.html
```

---

### 2. fpdf2 (альтернатива ReportLab)

**Что это:**
- Более простой API, чем ReportLab
- Хорошая поддержка UTF-8

---

## Рекомендации

### ✅ Лучшие практики

1. **Используйте шаблоны (Jinja2) для генерации**
2. **Кэшируйте сгенерированные PDF**
3. **Асинхронная генерация через Celery**
4. **Валидация данных перед генерацией**
5. **Логирование всех генераций**

---

## Связь с другими компонентами

- **[AI/LLM](01-ai-llm.md):** Анализ содержимого документов
- **[OCR](02-ocr.md):** Распознавание сканов
- **[Backend](03-backend.md):** API для генерации и скачивания
- **[Cloud](07-cloud.md):** Хранение PDF файлов

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-13
