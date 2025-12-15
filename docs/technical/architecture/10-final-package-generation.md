# Генерация финального комплекта исполнительной документации

**Дата создания:** 2025-12-15
**Приоритет:** ⭐⭐⭐⭐⭐ (Критический — финальный результат работы платформы)

---

## 🎯 Назначение

Этот модуль отвечает за формирование итогового комплекта исполнительной документации (ИД), который включает:

1. **PDF файл** — единый документ со всеми актами, схемами и документами качества
2. **Архив ZIP** — все реестры и акты в редактируемом формате (Excel)

---

## 📚 Регламентирующие документы

При реализации платформы необходимо руководствоваться следующими регламентами:

1. **Регламент на подготовку АОСР**
   - Файл: `docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку АОСР.xlsx`
   - Содержит: Правила создания актов освидетельствования скрытых работ

2. **Регламент на проверку исполнительных схем**
   - Файл: `docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку исполнительных схем.xlsx`
   - Содержит: Правила проверки исполнительных схем

3. **Шаблон формы АОСР**
   - Файл: `docs/technical/info/04_Форма АОСР.xlsx`
   - Содержит: Стандартная форма для генерации АОСР в Excel

4. **Примеры итоговой ИД**
   - `docs/technical/info/01_ИД_Примеры/01_Пример ИД на Благоустройство.pdf`
   - `docs/technical/info/01_ИД_Примеры/01_Пример ИД на монтаж перегородок.pdf`
   - `docs/technical/info/01_ИД_Примеры/01_Пример ИД на монтаж фунтдаментной плиты.pdf`

---

## 📋 Структура финального комплекта

Платформа поддерживает **два варианта** формирования комплекта ИД в зависимости от требований проекта:

---

### Вариант 1: С повторением документов качества (Благоустройство)

**Источник:** `01_Пример ИД на Благоустройство.pdf`

**Структура:**
```
ИТОГОВЫЙ PDF
├── 1. Титульный лист
│   ├── Название проекта
│   ├── Застройщик, Подрядчик, Генподрядчик
│   └── Период выполнения работ
│
├── 2. Общий реестр исполнительной документации
│   └── Таблица со всеми документами (сквозная нумерация)
│
├── 3. АОСР №1
│   ├── Основной акт
│   ├── Исполнительная схема
│   ├── Реестр документов качества к АОСР №1 (если >3 документов)
│   └── Документы качества:
│       ├── Сертификат на материал А
│       ├── Паспорт на материал Б
│       └── Декларация на материал В
│
├── 4. АОСР №2
│   ├── Основной акт
│   ├── Исполнительная схема
│   ├── Реестр документов качества к АОСР №2
│   └── Документы качества:
│       ├── Сертификат на материал А (ПОВТОРНО!)
│       ├── Сертификат на материал Г
│       └── Паспорт на материал Д
│
└── ... (остальные АОСР)
```

**Особенности:**
- ✅ Документы качества прикладываются к **каждому АОСР** отдельно
- ✅ Если один материал используется в нескольких актах → документ дублируется
- ✅ Каждый АОСР — это самостоятельный блок с полным комплектом документов
- ✅ Удобно для сдачи отдельных этапов работ

**Когда использовать:**
- Поэтапная сдача работ
- Требование заказчика о полной комплектности каждого акта
- Разные подрядчики на разных этапах

---

### Вариант 2: Со сквозной нумерацией без повторений (Перегородки)

**Источник:** `01_Пример ИД на монтаж перегородок.pdf`

**Структура:**
```
ИТОГОВЫЙ PDF
├── 1. Титульный лист
│   ├── Название проекта
│   ├── Застройщик, Подрядчик, Генподрядчик
│   └── Период выполнения работ
│
├── 2. Общий реестр исполнительной документации
│   └── Таблица со всеми документами (сквозная нумерация)
│       Пример:
│       № 1 - АОСР №1
│       № 2 - Исполнительная схема к АОСР №1
│       № 3 - Реестр документов качества к АОСР №1
│       № 4 - АОСР №2
│       № 5 - Исполнительная схема к АОСР №2
│       № 6 - Реестр документов качества к АОСР №2
│       ...
│       № 35 - Сертификат на материал А (документ №35)
│       № 36 - Паспорт на материал Б (документ №36)
│       № 37 - Декларация на материал В (документ №37)
│
├── 3. АОСР №1
│   ├── Основной акт
│   ├── Исполнительная схема
│   └── Реестр документов качества к АОСР №1
│       └── Ссылки на документы (по номерам из общего реестра):
│           - п. 35 - Сертификат на материал А
│           - п. 36 - Паспорт на материал Б
│           - п. 37 - Декларация на материал В
│
├── 4. АОСР №2
│   ├── Основной акт
│   ├── Исполнительная схема
│   └── Реестр документов качества к АОСР №2
│       └── Ссылки на документы:
│           - п. 35 - Сертификат на материал А (ссылка на тот же документ!)
│           - п. 38 - Сертификат на материал Г
│           - п. 39 - Паспорт на материал Д
│
├── ... (остальные АОСР)
│
└── N. Папка "Документы качества" (в конце комплекта)
    ├── № 35 - Сертификат на материал А
    ├── № 36 - Паспорт на материал Б
    ├── № 37 - Декларация на материал В
    ├── № 38 - Сертификат на материал Г
    └── № 39 - Паспорт на материал Д
```

**Особенности:**
- ✅ Документы качества собраны **в конце** комплекта (один раз)
- ✅ В реестрах к АОСР указываются **ссылки** на номера документов из общего реестра
- ✅ Нет дублирования документов
- ✅ Меньший размер итогового PDF
- ✅ Сквозная нумерация всех документов

**Когда использовать:**
- Сдача всего объема работ целиком
- Экономия места (меньше дублей)
- Требование заказчика о сквозной нумерации

---

### Сравнительная таблица

| Критерий | Вариант 1 (Благоустройство) | Вариант 2 (Перегородки) |
|----------|----------------------------|-------------------------|
| **Документы качества** | К каждому АОСР | В конце комплекта |
| **Повторение документов** | Да, если материал в нескольких актах | Нет, каждый документ один раз |
| **Ссылки в реестре к АОСР** | Нет (документы сразу после акта) | Да (ссылки на № из общего реестра) |
| **Размер PDF** | Больше (из-за дублей) | Меньше |
| **Удобство поэтапной сдачи** | Высокое | Среднее |
| **Компактность** | Низкая | Высокая |

---

### Настройка варианта формирования

```python
from enum import Enum

class PackageFormat(Enum):
    """Формат формирования комплекта ИД"""
    REPEATED_DOCS = "repeated"      # Вариант 1: документы к каждому АОСР
    UNIFIED_DOCS = "unified"        # Вариант 2: документы в конце, ссылки

# В настройках проекта
project.package_format = PackageFormat.UNIFIED_DOCS  # или REPEATED_DOCS
```

---

## 🔄 Workflow генерации комплекта

### Этап 1: Сбор метаданных

```python
from typing import List, Dict, Set
from dataclasses import dataclass
from datetime import date
from enum import Enum

class PackageFormat(Enum):
    """Формат формирования комплекта ИД"""
    REPEATED_DOCS = "repeated"      # Вариант 1: документы к каждому АОСР
    UNIFIED_DOCS = "unified"        # Вариант 2: документы в конце, ссылки

@dataclass
class DocumentMetadata:
    """
    Метаданные документа для реестра

    ВАЖНЫЕ ПРАВИЛА НУМЕРАЦИИ:
    1. В общем реестре нумерация начинается с ПЕРВОГО ДОКУМЕНТА (АОСР, приказ и т.д.)
    2. Количество листов САМОГО РЕЕСТРА в реестре НЕ УЧИТЫВАЕТСЯ
    3. АОСР и реестры к АОСР печатаются ДВУСТОРОННИМИ (1 лист = 2 страницы)
    4. Поэтому в поле page_count для АОСР указывается количество ЛИСТОВ, а не страниц
    """
    number: int  # № п/п в реестре (начинается с 1 для первого АОСР)
    name: str  # Наименование документа
    content: str  # Содержание документа
    doc_number: str  # № документа
    doc_date: date  # Дата документа
    page_count: int  # Количество ЛИСТОВ (для АОСР - листы, для документов - может быть страницы)
    start_page: int  # Страница начала в итоговом PDF

@dataclass
class QualityDocReference:
    """Ссылка на документ качества для Варианта 2"""
    doc_id: int  # ID документа в БД
    registry_number: int  # № п/п в общем реестре
    name: str  # Название документа
    material_name: str  # Материал, к которому относится
    file_path: str  # Путь к файлу

@dataclass
class AOSRPackage:
    """Пакет документов для одного АОСР"""
    aosr_id: int
    aosr_number: str  # Например: "01-КЖ6-1"
    aosr_date: date
    work_description: str  # Например: "Разработка грунта"

    # Файлы
    aosr_pdf_path: str  # Путь к PDF акта
    schema_pdf_path: str  # Путь к исполнительной схеме

    # Вариант 1: Пути к документам (будут вставлены после акта)
    quality_docs_paths: List[str]  # Для REPEATED_DOCS

    # Вариант 2: Ссылки на документы (документы в конце)
    quality_docs_refs: List[QualityDocReference]  # Для UNIFIED_DOCS

    # Метаданные для реестра
    metadata: List[DocumentMetadata]  # Все документы в пакете АОСР
    quality_doc_metadata: List[DocumentMetadata]  # Только документы качества (для реестра ДК)

def collect_project_documents(project_id: int, package_format: PackageFormat) -> Dict:
    """
    Собирает все документы проекта для формирования комплекта

    Args:
        project_id: ID проекта
        package_format: Формат комплекта (REPEATED_DOCS или UNIFIED_DOCS)

    Возвращает:
    {
        'project_info': {...},
        'aosr_packages': [AOSRPackage, ...],
        'unique_quality_docs': [QualityDocReference, ...],  # Для UNIFIED_DOCS
        'total_documents': int
    }
    """
    from app.models import Project, AOSR, Document, AOSRQualityDocument
    from app.database import get_db

    db = get_db()
    project = db.query(Project).filter(Project.id == project_id).first()

    # Получаем все АОСР проекта в хронологическом порядке
    aosr_list = db.query(AOSR).filter(
        AOSR.project_id == project_id,
        AOSR.status == 'approved'
    ).order_by(AOSR.work_date).all()

    packages = []
    doc_counter = 1  # Счётчик для № п/п в реестре

    # Для Варианта 2: отслеживаем уникальные документы качества
    unique_quality_docs_map = {}  # {doc_id: QualityDocReference}
    quality_docs_registry_numbers = {}  # {doc_id: registry_number}

    for aosr in aosr_list:
        metadata = []

        # 1. Сам АОСР
        # ВАЖНО: АОСР и реестры к АОСР печатаются двусторонними,
        # поэтому в общем реестре указывается количество ЛИСТОВ, а не страниц
        # 1 лист = 2 страницы при двусторонней печати
        metadata.append(DocumentMetadata(
            number=doc_counter,
            name=f"АОСР {aosr.number}",
            content=aosr.work_description,
            doc_number=aosr.number,
            doc_date=aosr.work_date,
            page_count=1,  # 1 лист (двусторонняя печать)
            start_page=0  # Будет рассчитано позже
        ))
        doc_counter += 1

        # 2. Исполнительная схема
        if aosr.schema_document_id:
            schema = db.query(Document).filter(
                Document.id == aosr.schema_document_id
            ).first()

            metadata.append(DocumentMetadata(
                number=doc_counter,
                name=f"Исполнительная схема к АОСР {aosr.number}",
                content="Приложение №1",
                doc_number=aosr.number,
                doc_date=aosr.work_date,
                page_count=1,
                start_page=0
            ))
            doc_counter += 1

        # 3. Реестр документов качества к АОСР (если документов >3)
        quality_docs = db.query(AOSRQualityDocument).filter(
            AOSRQualityDocument.aosr_id == aosr.id
        ).all()

        if len(quality_docs) > 3:
            metadata.append(DocumentMetadata(
                number=doc_counter,
                name=f"Реестр документов качества к АОСР {aosr.number}",
                content="Приложение №2",
                doc_number=aosr.number,
                doc_date=aosr.work_date,
                page_count=1,
                start_page=0
            ))
            doc_counter += 1

        # 4. Документы качества
        quality_doc_paths = []
        quality_doc_refs = []
        quality_doc_metadata = []  # Метаданные только для документов качества (для реестра ДК)

        for qd in quality_docs:
            doc = db.query(Document).filter(Document.id == qd.document_id).first()

            if package_format == PackageFormat.REPEATED_DOCS:
                # Вариант 1: Добавляем документ в список файлов для этого АОСР
                quality_doc_paths.append(doc.file_path)

                # Добавляем в общий реестр (документ будет сразу после АОСР)
                doc_metadata = DocumentMetadata(
                    number=doc_counter,
                    name=doc.doc_type,
                    content=doc.metadata.get('material_name', ''),
                    doc_number=doc.metadata.get('cert_number', ''),
                    doc_date=doc.metadata.get('issue_date', aosr.work_date),
                    page_count=doc.metadata.get('page_count', 1),
                    start_page=0
                )
                metadata.append(doc_metadata)
                quality_doc_metadata.append(doc_metadata)  # Также добавляем в список ДК
                doc_counter += 1

            else:  # UNIFIED_DOCS
                # Вариант 2: Сохраняем ссылку на документ

                # Если документ уже был добавлен в другом АОСР
                if doc.id in quality_docs_registry_numbers:
                    # Используем существующий номер
                    registry_num = quality_docs_registry_numbers[doc.id]
                else:
                    # Первое появление документа - резервируем номер
                    # (сам документ будет в конце комплекта)
                    registry_num = None  # Будет присвоен позже при добавлении уникальных документов

                    # Сохраняем уникальный документ
                    unique_quality_docs_map[doc.id] = QualityDocReference(
                        doc_id=doc.id,
                        registry_number=0,  # Будет установлен позже
                        name=doc.doc_type,
                        material_name=doc.metadata.get('material_name', ''),
                        file_path=doc.file_path
                    )

                # Сохраняем ссылку для реестра к АОСР
                quality_doc_ref = QualityDocReference(
                    doc_id=doc.id,
                    registry_number=registry_num or 0,  # Будет обновлено
                    name=doc.doc_type,
                    material_name=doc.metadata.get('material_name', ''),
                    file_path=doc.file_path
                )
                quality_doc_refs.append(quality_doc_ref)

                # Также сохраняем метаданные для реестра ДК (со ссылкой на номер)
                quality_doc_metadata.append(DocumentMetadata(
                    number=registry_num or 0,  # Номер из общего реестра
                    name=doc.doc_type,
                    content=doc.metadata.get('material_name', ''),
                    doc_number=doc.metadata.get('cert_number', ''),
                    doc_date=doc.metadata.get('issue_date', aosr.work_date),
                    page_count=doc.metadata.get('page_count', 1),
                    start_page=0
                ))

        packages.append(AOSRPackage(
            aosr_id=aosr.id,
            aosr_number=aosr.number,
            aosr_date=aosr.work_date,
            work_description=aosr.work_description,
            aosr_pdf_path=aosr.generated_pdf_path,
            schema_pdf_path=schema.file_path if aosr.schema_document_id else None,
            quality_docs_paths=quality_doc_paths,  # Для Варианта 1
            quality_docs_refs=quality_doc_refs,    # Для Варианта 2
            metadata=metadata,
            quality_doc_metadata=quality_doc_metadata  # Метаданные только для ДК
        ))

    # Для Варианта 2: Назначаем номера уникальным документам качества
    unique_quality_docs = []
    if package_format == PackageFormat.UNIFIED_DOCS:
        for doc_id, doc_ref in unique_quality_docs_map.items():
            doc_ref.registry_number = doc_counter
            quality_docs_registry_numbers[doc_id] = doc_counter
            unique_quality_docs.append(doc_ref)
            doc_counter += 1

        # Обновляем номера в ссылках
        for package in packages:
            for ref in package.quality_docs_refs:
                ref.registry_number = quality_docs_registry_numbers[ref.doc_id]

    return {
        'project_info': {
            'name': project.name,
            'address': project.address,
            'developer': project.developer,
            'contractor': project.contractor,
            'general_contractor': project.general_contractor,
            'start_date': project.start_date,
            'end_date': project.end_date,
            'package_format': package_format
        },
        'aosr_packages': packages,
        'unique_quality_docs': unique_quality_docs,  # Для Варианта 2
        'total_documents': doc_counter - 1
    }
```

---

### Этап 2: Генерация реестра исполнительной документации

```python
from reportlab.lib.pagesizes import A4
from reportlab.lib import colors
from reportlab.lib.units import mm
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont

# Регистрация русского шрифта
pdfmetrics.registerFont(TTFont('DejaVu', 'fonts/DejaVuSans.ttf'))
pdfmetrics.registerFont(TTFont('DejaVuBold', 'fonts/DejaVuSans-Bold.ttf'))

def generate_registry_pdf(
    project_info: Dict,
    all_metadata: List[DocumentMetadata],
    output_path: str
) -> str:
    """
    Генерирует реестр исполнительной документации в формате PDF

    Формат соответствует примеру из 01_Пример ИД на монтаж фунтдаментной плиты.pdf
    """
    from reportlab.lib.pagesizes import A4
    from reportlab.pdfgen import canvas

    c = canvas.Canvas(output_path, pagesize=A4)
    width, height = A4

    # Заголовок реестра
    c.setFont("DejaVuBold", 14)
    c.drawCentredString(width/2, height - 30*mm, "РЕЕСТР ИСПОЛНИТЕЛЬНОЙ ДОКУМЕНТАЦИИ")

    c.setFont("DejaVu", 10)
    c.drawCentredString(width/2, height - 40*mm, f"№ {project_info.get('registry_number', 'КЖ-6-БКРТПБ')}")

    # Информация о проекте
    y = height - 55*mm
    c.setFont("DejaVu", 9)
    c.drawString(20*mm, y, f"Застройщик: {project_info['developer']}")
    y -= 5*mm
    c.drawString(20*mm, y, f"Подрядчик: {project_info['contractor']}")
    y -= 5*mm
    c.drawString(20*mm, y, f"Генподрядчик: {project_info['general_contractor']}")
    y -= 5*mm
    c.drawString(20*mm, y, f"Объект: {project_info['name']}")
    y -= 5*mm
    c.drawString(20*mm, y, f"Адрес: {project_info['address']}")

    # Таблица реестра
    y -= 10*mm

    # Заголовки столбцов
    table_headers = [
        "№ п/п",
        "Наименование документа",
        "Содержание документа",
        "№ документа",
        "Дата документа",
        "Кол-во листов",
        "Страница по списку"
    ]

    # Ширины столбцов (в мм)
    col_widths = [10, 45, 40, 25, 20, 15, 15]

    # Рисуем заголовки таблицы
    c.setFont("DejaVuBold", 8)
    x_start = 10*mm
    x = x_start

    for i, header in enumerate(table_headers):
        # Рисуем границы ячейки
        c.rect(x, y - 10*mm, col_widths[i]*mm, 10*mm)
        # Текст заголовка
        c.drawString(x + 1*mm, y - 7*mm, header)
        x += col_widths[i]*mm

    y -= 10*mm

    # Данные таблицы
    c.setFont("DejaVu", 7)

    for doc in all_metadata:
        # Проверка, нужна ли новая страница
        if y < 30*mm:
            c.showPage()
            y = height - 20*mm
            c.setFont("DejaVu", 7)

        x = x_start
        row_data = [
            str(doc.number),
            doc.name,
            doc.content,
            doc.doc_number,
            doc.doc_date.strftime("%d.%m.%Y"),
            str(doc.page_count),
            str(doc.start_page)
        ]

        # Рисуем строку таблицы
        for i, value in enumerate(row_data):
            c.rect(x, y - 8*mm, col_widths[i]*mm, 8*mm)
            c.drawString(x + 1*mm, y - 5*mm, value[:20])  # Обрезаем длинный текст
            x += col_widths[i]*mm

        y -= 8*mm

    c.save()
    return output_path


def generate_registry_excel(
    project_info: Dict,
    all_metadata: List[DocumentMetadata],
    output_path: str
) -> str:
    """
    Генерирует реестр в формате Excel на основе шаблона

    Использует шаблон: docs/technical/info/04_Шаблоны_Формы/04_Шаблон общего реестра.xlsx

    Args:
        project_info: Информация о проекте (название, адрес, участники)
        all_metadata: Список метаданных всех документов
        output_path: Путь для сохранения Excel файла

    Returns:
        str: Путь к сгенерированному файлу
    """
    import openpyxl
    from openpyxl.styles import Font, Alignment, Border, Side

    # Загружаем шаблон
    template_path = "docs/technical/info/04_Шаблоны_Формы/04_Шаблон общего реестра.xlsx"

    if os.path.exists(template_path):
        # Используем шаблон
        wb = openpyxl.load_workbook(template_path)
        ws = wb.active

        # Заполняем информацию о проекте (предполагаем, что в шаблоне есть именованные ячейки)
        # Если в шаблоне нет именованных ячеек - используем координаты
        try:
            # Пытаемся использовать именованные ячейки
            ws['developer_cell'] = project_info.get('developer', '')
            ws['contractor_cell'] = project_info.get('contractor', '')
            ws['object_name_cell'] = project_info.get('name', '')
            ws['address_cell'] = project_info.get('address', '')
        except:
            # Fallback на фиксированные координаты (по аналогии со старым кодом)
            ws['A3'] = f"Застройщик: {project_info.get('developer', '')}"
            ws['A4'] = f"Подрядчик: {project_info.get('contractor', '')}"
            ws['A5'] = f"Объект: {project_info.get('name', '')}"
            ws['A6'] = f"Адрес: {project_info.get('address', '')}"

        # Находим первую строку для данных (после заголовков)
        # Предполагаем, что данные начинаются со строки 9
        start_row = 9

        # Заполняем данные документов
        for idx, doc in enumerate(all_metadata):
            row = start_row + idx
            ws.cell(row, 1).value = doc.number
            ws.cell(row, 2).value = doc.name
            ws.cell(row, 3).value = doc.content
            ws.cell(row, 4).value = doc.doc_number
            ws.cell(row, 5).value = doc.doc_date.strftime("%d.%m.%Y") if doc.doc_date else ""
            ws.cell(row, 6).value = doc.page_count
            ws.cell(row, 7).value = doc.start_page

            # Копируем стиль из шаблона (если есть)
            for col in range(1, 8):
                cell = ws.cell(row, col)
                if row == start_row and idx == 0:
                    # Копируем границы из шаблона
                    template_cell = ws.cell(start_row - 1, col)
                    if template_cell.border:
                        cell.border = template_cell.border
                else:
                    # Применяем стандартные границы
                    cell.border = Border(
                        left=Side(style='thin'),
                        right=Side(style='thin'),
                        top=Side(style='thin'),
                        bottom=Side(style='thin')
                    )

    else:
        # Если шаблона нет - создаём таблицу программно
        wb = openpyxl.Workbook()
        ws = wb.active
        ws.title = "Реестр ИД"

        # Заголовок
        ws.merge_cells('A1:G1')
        ws['A1'] = "РЕЕСТР ИСПОЛНИТЕЛЬНОЙ ДОКУМЕНТАЦИИ"
        ws['A1'].font = Font(bold=True, size=14)
        ws['A1'].alignment = Alignment(horizontal='center')

        # Информация о проекте
        ws['A3'] = f"Застройщик: {project_info.get('developer', '')}"
        ws['A4'] = f"Подрядчик: {project_info.get('contractor', '')}"
        ws['A5'] = f"Объект: {project_info.get('name', '')}"
        ws['A6'] = f"Адрес: {project_info.get('address', '')}"

        # Заголовки таблицы
        headers = [
            "№ п/п",
            "Наименование документа",
            "Содержание документа",
            "№ документа",
            "Дата документа",
            "Кол-во листов",
            "Страница по списку"
        ]

        for col, header in enumerate(headers, start=1):
            cell = ws.cell(row=8, column=col)
            cell.value = header
            cell.font = Font(bold=True)
            cell.alignment = Alignment(horizontal='center', vertical='center')
            cell.border = Border(
                left=Side(style='thin'),
                right=Side(style='thin'),
                top=Side(style='thin'),
                bottom=Side(style='thin')
            )

        # Данные
        for row, doc in enumerate(all_metadata, start=9):
            ws.cell(row, 1).value = doc.number
            ws.cell(row, 2).value = doc.name
            ws.cell(row, 3).value = doc.content
            ws.cell(row, 4).value = doc.doc_number
            ws.cell(row, 5).value = doc.doc_date.strftime("%d.%m.%Y") if doc.doc_date else ""
            ws.cell(row, 6).value = doc.page_count
            ws.cell(row, 7).value = doc.start_page

            # Границы для всех ячеек
            for col in range(1, 8):
                ws.cell(row, col).border = Border(
                    left=Side(style='thin'),
                    right=Side(style='thin'),
                    top=Side(style='thin'),
                    bottom=Side(style='thin')
                )

        # Ширины столбцов
        ws.column_dimensions['A'].width = 8
        ws.column_dimensions['B'].width = 40
        ws.column_dimensions['C'].width = 35
        ws.column_dimensions['D'].width = 20
        ws.column_dimensions['E'].width = 15
        ws.column_dimensions['F'].width = 12
        ws.column_dimensions['G'].width = 12

    wb.save(output_path)
    return output_path
```

---

### Этап 3: Объединение всех документов в единый PDF

```python
from PyPDF2 import PdfMerger, PdfReader
import os

def merge_all_documents_to_pdf(
    project_id: int,
    package_format: PackageFormat,
    output_pdf_path: str
) -> str:
    """
    Объединяет все документы проекта в единый PDF файл
    с правильной нумерацией страниц в реестре

    Поддерживает два варианта формирования:
    - REPEATED_DOCS: документы качества после каждого АОСР
    - UNIFIED_DOCS: документы качества в конце комплекта
    """

    # 1. Собираем все документы
    data = collect_project_documents(project_id, package_format)
    project_info = data['project_info']
    aosr_packages = data['aosr_packages']
    unique_quality_docs = data.get('unique_quality_docs', [])

    # 2. Создаём временные файлы
    temp_dir = f"temp_{project_id}"
    os.makedirs(temp_dir, exist_ok=True)

    # 3. Генерируем титульный лист
    title_page_path = os.path.join(temp_dir, "00_title.pdf")
    generate_title_page(project_info, title_page_path)

    # 4. Подсчёт страниц для реестра
    all_metadata = []
    current_page = 1  # Начинаем с 1 (титульный лист)

    # Титульный лист
    current_page += 1  # +1 страница для титульника

    # Реестр займёт ~2 страницы (рассчитаем позже точно)
    registry_start_page = current_page
    registry_page_count = calculate_registry_pages(aosr_packages)
    current_page += registry_page_count

    # Подсчитываем страницы для каждого документа
    if package_format == PackageFormat.REPEATED_DOCS:
        # Вариант 1: Документы после каждого АОСР
        for package in aosr_packages:
            for doc_meta in package.metadata:
                # Скачиваем PDF из storage чтобы узнать реальное кол-во страниц
                if doc_meta.name.startswith("АОСР"):
                    pdf_path = download_from_storage(package.aosr_pdf_path)
                elif "схема" in doc_meta.name.lower():
                    pdf_path = download_from_storage(package.schema_pdf_path)
                elif "Реестр документов качества" in doc_meta.name:
                    # Генерируем реестр документов качества к АОСР
                    pdf_path = generate_quality_docs_registry_for_aosr(
                        package,
                        os.path.join(temp_dir, f"reg_{package.aosr_id}.pdf")
                    )
                else:
                    # Документ качества - находим его индекс
                    quality_docs_start = 2  # АОСР + схема
                    if len(package.quality_docs_paths) > 3:
                        quality_docs_start = 3  # АОСР + схема + реестр

                    doc_index = doc_meta.number - package.metadata[0].number - quality_docs_start
                    pdf_path = download_from_storage(package.quality_docs_paths[doc_index])

                # Читаем реальное количество страниц
                reader = PdfReader(pdf_path)
                actual_page_count = len(reader.pages)

                # Обновляем метаданные
                doc_meta.start_page = current_page
                doc_meta.page_count = actual_page_count
                current_page += actual_page_count

                all_metadata.append(doc_meta)

    else:  # UNIFIED_DOCS
        # Вариант 2: Документы в конце комплекта
        for package in aosr_packages:
            for doc_meta in package.metadata:
                # Скачиваем PDF
                if doc_meta.name.startswith("АОСР"):
                    pdf_path = download_from_storage(package.aosr_pdf_path)
                elif "схема" in doc_meta.name.lower():
                    pdf_path = download_from_storage(package.schema_pdf_path)
                elif "Реестр документов качества" in doc_meta.name:
                    # Генерируем реестр с ссылками на документы
                    pdf_path = generate_quality_docs_registry_with_refs(
                        package,
                        os.path.join(temp_dir, f"reg_{package.aosr_id}.pdf")
                    )
                else:
                    continue  # Документы качества будут в конце

                # Читаем реальное количество страниц
                reader = PdfReader(pdf_path)
                actual_page_count = len(reader.pages)

                # Обновляем метаданные
                doc_meta.start_page = current_page
                doc_meta.page_count = actual_page_count
                current_page += actual_page_count

                all_metadata.append(doc_meta)

        # Добавляем уникальные документы качества в конец
        for doc_ref in unique_quality_docs:
            pdf_path = download_from_storage(doc_ref.file_path)
            reader = PdfReader(pdf_path)
            actual_page_count = len(reader.pages)

            all_metadata.append(DocumentMetadata(
                number=doc_ref.registry_number,
                name=doc_ref.name,
                content=doc_ref.material_name,
                doc_number="",
                doc_date=None,
                page_count=actual_page_count,
                start_page=current_page
            ))
            current_page += actual_page_count

    # 5. Генерируем реестр с правильными номерами страниц
    registry_pdf_path = os.path.join(temp_dir, "01_registry.pdf")
    generate_registry_pdf(project_info, all_metadata, registry_pdf_path)

    # 6. Объединяем все PDF в один
    merger = PdfMerger()

    # Титульный лист
    merger.append(title_page_path)

    # Реестр
    merger.append(registry_pdf_path)

    if package_format == PackageFormat.REPEATED_DOCS:
        # Вариант 1: АОСР + схема + реестр ДК + документы качества
        for package in aosr_packages:
            # АОСР
            aosr_pdf = download_from_storage(package.aosr_pdf_path)
            merger.append(aosr_pdf)

            # Исполнительная схема
            if package.schema_pdf_path:
                schema_pdf = download_from_storage(package.schema_pdf_path)
                merger.append(schema_pdf)

            # Реестр документов качества (если >3 документов)
            if len(package.quality_docs_paths) > 3:
                reg_path = os.path.join(temp_dir, f"reg_{package.aosr_id}.pdf")
                merger.append(reg_path)

            # Документы качества
            for quality_doc_path in package.quality_docs_paths:
                quality_pdf = download_from_storage(quality_doc_path)
                merger.append(quality_pdf)

    else:  # UNIFIED_DOCS
        # Вариант 2: АОСР + схема + реестр ДК (с ссылками)
        for package in aosr_packages:
            # АОСР
            aosr_pdf = download_from_storage(package.aosr_pdf_path)
            merger.append(aosr_pdf)

            # Исполнительная схема
            if package.schema_pdf_path:
                schema_pdf = download_from_storage(package.schema_pdf_path)
                merger.append(schema_pdf)

            # Реестр документов качества с ссылками (если >3 документов)
            if len(package.quality_docs_refs) > 3:
                reg_path = os.path.join(temp_dir, f"reg_{package.aosr_id}.pdf")
                merger.append(reg_path)

        # Документы качества в конце (только уникальные)
        for doc_ref in unique_quality_docs:
            quality_pdf = download_from_storage(doc_ref.file_path)
            merger.append(quality_pdf)

    # 7. Сохраняем итоговый PDF
    merger.write(output_pdf_path)
    merger.close()

    # 8. Очистка временных файлов
    import shutil
    shutil.rmtree(temp_dir)

    return output_pdf_path


def generate_quality_docs_registry_with_refs(
    package: AOSRPackage,
    output_path: str
) -> str:
    """
    Генерирует реестр документов качества к АОСР
    с ссылками на документы из общего реестра (Вариант 2)
    """
    from reportlab.pdfgen import canvas
    from reportlab.lib.pagesizes import A4

    c = canvas.Canvas(output_path, pagesize=A4)
    width, height = A4

    # Заголовок
    c.setFont("DejaVuBold", 12)
    c.drawCentredString(width/2, height - 30*mm, f"РЕЕСТР ДОКУМЕНТОВ КАЧЕСТВА")
    c.drawCentredString(width/2, height - 40*mm, f"К АОСР {package.aosr_number}")

    # Таблица
    y = height - 60*mm
    c.setFont("DejaVuBold", 9)

    # Заголовки
    c.drawString(20*mm, y, "№ п/п в")
    c.drawString(20*mm, y - 5*mm, "общем реестре")
    c.drawString(50*mm, y, "Наименование документа")
    c.drawString(120*mm, y, "Материал")

    y -= 15*mm

    # Данные
    c.setFont("DejaVu", 8)
    for ref in package.quality_docs_refs:
        c.drawString(25*mm, y, str(ref.registry_number))
        c.drawString(50*mm, y, ref.name[:30])
        c.drawString(120*mm, y, ref.material_name[:25])
        y -= 7*mm

        if y < 40*mm:
            c.showPage()
            y = height - 40*mm
            c.setFont("DejaVu", 8)

    c.save()
    return output_path


def generate_quality_docs_registry_for_aosr(
    package: AOSRPackage,
    output_path: str
) -> str:
    """
    Генерирует реестр документов качества к АОСР
    для Варианта 1 (документы идут сразу после реестра)
    """
    from reportlab.pdfgen import canvas
    from reportlab.lib.pagesizes import A4

    c = canvas.Canvas(output_path, pagesize=A4)
    width, height = A4

    # Заголовок
    c.setFont("DejaVuBold", 12)
    c.drawCentredString(width/2, height - 30*mm, f"РЕЕСТР ДОКУМЕНТОВ КАЧЕСТВА")
    c.drawCentredString(width/2, height - 40*mm, f"К АОСР {package.aosr_number}")

    # Таблица
    y = height - 60*mm
    c.setFont("DejaVuBold", 9)

    # Заголовки
    c.drawString(20*mm, y, "№ п/п")
    c.drawString(40*mm, y, "Наименование документа")
    c.drawString(110*mm, y, "Материал")
    c.drawString(150*mm, y, "Кол-во листов")

    y -= 15*mm

    # Данные
    c.setFont("DejaVu", 8)
    doc_num = 1
    # Находим документы качества в метаданных
    for meta in package.metadata:
        if meta.name not in [f"АОСР {package.aosr_number}",
                            f"Исполнительная схема к АОСР {package.aosr_number}",
                            f"Реестр документов качества к АОСР {package.aosr_number}"]:
            c.drawString(20*mm, y, str(doc_num))
            c.drawString(40*mm, y, meta.name[:35])
            c.drawString(110*mm, y, meta.content[:20])
            c.drawString(155*mm, y, str(meta.page_count))
            y -= 7*mm
            doc_num += 1

            if y < 40*mm:
                c.showPage()
                y = height - 40*mm
                c.setFont("DejaVu", 8)

    c.save()
    return output_path


def generate_title_page(project_info: Dict, output_path: str) -> str:
    """Генерирует титульный лист комплекта ИД"""
    from reportlab.pdfgen import canvas
    from reportlab.lib.pagesizes import A4

    c = canvas.Canvas(output_path, pagesize=A4)
    width, height = A4

    # Заголовок
    c.setFont("DejaVuBold", 18)
    c.drawCentredString(width/2, height - 80*mm, "ИСПОЛНИТЕЛЬНАЯ ДОКУМЕНТАЦИЯ")

    c.setFont("DejaVu", 12)
    y = height - 100*mm

    c.drawCentredString(width/2, y, f"Объект: {project_info['name']}")
    y -= 10*mm
    c.drawCentredString(width/2, y, f"Адрес: {project_info['address']}")

    # Участники стройки
    y -= 30*mm
    c.setFont("DejaVu", 10)
    c.drawString(40*mm, y, f"Застройщик: {project_info['developer']}")
    y -= 7*mm
    c.drawString(40*mm, y, f"Подрядчик: {project_info['contractor']}")
    y -= 7*mm
    c.drawString(40*mm, y, f"Генподрядчик: {project_info['general_contractor']}")

    # Период работ
    y -= 20*mm
    c.drawString(40*mm, y, f"Период выполнения работ:")
    y -= 7*mm
    c.drawString(40*mm, y,
                 f"с {project_info['start_date'].strftime('%d.%m.%Y')} "
                 f"по {project_info['end_date'].strftime('%d.%m.%Y')}")

    # Год
    c.setFont("DejaVuBold", 14)
    c.drawCentredString(width/2, 40*mm, f"{project_info['end_date'].year} г.")

    c.save()
    return output_path
```

---

### Этап 4: Создание архива с редактируемыми файлами

```python
import zipfile
import os

def create_editable_archive(
    project_id: int,
    final_pdf_path: str,
    output_zip_path: str,
    package_format: PackageFormat
) -> str:
    """
    Создаёт ZIP архив со всеми актами и реестрами в редактируемом формате

    Args:
        project_id: ID проекта
        final_pdf_path: Путь к финальному объединённому PDF файлу
        output_zip_path: Путь для сохранения ZIP архива
        package_format: Формат комплекта (REPEATED_DOCS или UNIFIED_DOCS)

    Структура архива (по примеру из папки "Итоговый архив"):
    ├── 1. Исполнительная документация в формате PDF/
    │   └── ИД_Полный_комплект.pdf (итоговый PDF файл)
    │
    ├── 2. Исполнительная документация в формате Excel/
    │   ├── Общий реестр.xlsx
    │   ├── АОСР №1.xlsx
    │   ├── АОСР №2.xlsx
    │   ├── Акты испытаний.xlsx
    │   └── ...
    │
    ├── 3. Рабочая документация в формате PDF/
    │   └── [РД которую загрузил пользователь]
    │   └── (если не загружал - папка остаётся пустой)
    │
    ├── 4. Геодезические схемы в формате DWG/
    │   └── (пустая папка, заполняется пользователем вручную)
    │
    └── 5. Паспорта, сертификаты и лабораторные заключения/
        ├── АОСР №1/
        │   ├── Сертификат на материал А.pdf
        │   └── Паспорт на материал Б.pdf
        ├── АОСР №2/
        │   └── Декларация на материал В.pdf
        └── ...
        (документы качества сортируются по видам работ для каждого АОСР)
    """

    data = collect_project_documents(project_id, package_format)
    project_info = data['project_info']
    aosr_packages = data['aosr_packages']

    with zipfile.ZipFile(output_zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:

        # 1. Исполнительная документация в формате PDF (итоговый PDF файл)
        pdf_folder = "1. Исполнительная документация в формате PDF/"

        # Добавляем ТОЛЬКО финальный объединённый PDF файл
        final_pdf_filename = f"ИД_{project_info['object_name']}_полный_комплект.pdf"
        zipf.write(final_pdf_path, f"{pdf_folder}{final_pdf_filename}")

        # 2. Исполнительная документация в формате Excel
        excel_folder = "2. Исполнительная документация в формате Excel/"

        # Общий реестр
        all_metadata = []
        for package in aosr_packages:
            all_metadata.extend(package.metadata)

        registry_excel_path = f"temp_registry_{project_id}.xlsx"
        generate_registry_excel(project_info, all_metadata, registry_excel_path)
        zipf.write(registry_excel_path, f"{excel_folder}Общий реестр.xlsx")
        os.remove(registry_excel_path)

        # АОСР в Excel (на основе шаблона 04_Форма АОСР.xlsx)
        for package in aosr_packages:
            aosr_excel_path = f"temp_aosr_{package.aosr_id}.xlsx"
            generate_aosr_excel(package, aosr_excel_path)
            zipf.write(aosr_excel_path, f"{excel_folder}АОСР {package.aosr_number}.xlsx")
            os.remove(aosr_excel_path)

        # Реестры документов качества к АОСР (если >3 документов)
        for package in aosr_packages:
            if len(package.quality_docs) > 3:
                quality_reg_path = f"temp_quality_reg_{package.aosr_id}.xlsx"
                generate_quality_docs_registry_excel(package, quality_reg_path)
                zipf.write(quality_reg_path, f"{excel_folder}Реестр_документов_качества_АОСР_{package.aosr_number}.xlsx")
                os.remove(quality_reg_path)

        # 3. Рабочая документация в формате PDF (загруженная пользователем)
        rd_folder = "3. Рабочая документация в формате PDF/"

        # Получаем рабочую документацию из БД
        working_docs = db.query(Document).filter(
            Document.project_id == project_id,
            Document.doc_type == "working_documentation"
        ).all()

        if working_docs:
            # Если пользователь загружал РД - добавляем файлы
            for rd_doc in working_docs:
                rd_pdf = download_from_storage(rd_doc.file_path)
                filename = rd_doc.metadata.get('original_filename', os.path.basename(rd_pdf))
                zipf.write(rd_pdf, f"{rd_folder}{filename}")
        else:
            # Если РД не загружалась - создаём пустую папку с .gitkeep
            zipf.writestr(f"{rd_folder}.gitkeep", "")

        # 4. Геодезические схемы в формате DWG (пустая папка, заполняется пользователем)
        dwg_folder = "4. Геодезические схемы в формате DWG/"
        zipf.writestr(f"{dwg_folder}.gitkeep", "")

        # 5. Паспорта, сертификаты и лабораторные заключения
        # Документы качества сортируются по видам работ для каждого АОСР
        quality_folder = "5. Паспорта, сертификаты и лабораторные заключения/"

        for package in aosr_packages:
            # Создаём подпапку для каждого АОСР
            aosr_subfolder = f"{quality_folder}АОСР {package.aosr_number}/"

            # Используем quality_docs_paths (актуально для обоих вариантов)
            # В варианте REPEATED_DOCS - это документы для каждого АОСР
            # В варианте UNIFIED_DOCS - это те же документы, но они будут размещены по папкам
            if package.quality_docs_paths:
                for quality_doc_path in package.quality_docs_paths:
                    quality_pdf = download_from_storage(quality_doc_path)
                    filename = os.path.basename(quality_pdf)
                    zipf.write(quality_pdf, f"{aosr_subfolder}{filename}")
            else:
                # Если нет документов качества к этому АОСР - создаём пустую подпапку
                zipf.writestr(f"{aosr_subfolder}.gitkeep", "")

    return output_zip_path


def generate_quality_docs_registry_excel(package: AOSRPackage, output_path: str) -> str:
    """
    Генерирует реестр документов качества к АОСР в формате Excel на основе шаблона

    Использует шаблон: docs/technical/info/04_Шаблоны_Формы/04_Шаблон реестра к АОСР.xlsx

    Используется когда к АОСР прикладывается более 3-х документов

    Args:
        package: Пакет документов АОСР
        output_path: Путь для сохранения Excel файла

    Returns:
        str: Путь к сгенерированному файлу
    """
    import openpyxl
    from openpyxl.styles import Font, Alignment, Border, Side

    # Загружаем шаблон
    template_path = "docs/technical/info/04_Шаблоны_Формы/04_Шаблон реестра к АОСР.xlsx"

    if os.path.exists(template_path):
        # Используем шаблон
        wb = openpyxl.load_workbook(template_path)
        ws = wb.active

        # Заполняем заголовок (если в шаблоне есть именованная ячейка)
        try:
            ws['aosr_number_cell'] = package.aosr_number
        except:
            # Fallback: обновляем заголовок в ячейке A1
            ws['A1'] = f"РЕЕСТР ДОКУМЕНТОВ КАЧЕСТВА К АОСР №{package.aosr_number}"

        # Находим первую строку для данных (предполагаем строка 3)
        start_row = 3

    else:
        # Если шаблона нет - создаём таблицу программно
        wb = openpyxl.Workbook()
        ws = wb.active
        ws.title = f"Реестр ДК АОСР {package.aosr_number}"

        # Заголовок
        ws['A1'] = f"РЕЕСТР ДОКУМЕНТОВ КАЧЕСТВА К АОСР №{package.aosr_number}"
        ws['A1'].font = Font(bold=True, size=12)
        ws.merge_cells('A1:G1')
        ws['A1'].alignment = Alignment(horizontal='center')

        start_row = 3

    # Шапка таблицы
    headers = ['№ п/п', 'Наименование', 'Содержание', '№ документа', 'Дата', 'Кол-во листов', 'Страница']
    ws.append(headers)

    for cell in ws[2]:
        cell.font = Font(bold=True)
        cell.alignment = Alignment(horizontal='center', vertical='center')
        cell.border = Border(
            left=Side(style='thin'),
            right=Side(style='thin'),
            top=Side(style='thin'),
            bottom=Side(style='thin')
        )

    # Заполняем данные из метаданных документов качества
    row_num = 3
    for idx, doc_metadata in enumerate(package.quality_doc_metadata, 1):
        ws.cell(row=row_num, column=1, value=idx)
        ws.cell(row=row_num, column=2, value=doc_metadata.name)
        ws.cell(row=row_num, column=3, value=doc_metadata.content)
        ws.cell(row=row_num, column=4, value=doc_metadata.doc_number)
        ws.cell(row=row_num, column=5, value=doc_metadata.doc_date.strftime('%d.%m.%Y') if doc_metadata.doc_date else '')
        ws.cell(row=row_num, column=6, value=doc_metadata.page_count)
        ws.cell(row=row_num, column=7, value=doc_metadata.start_page)

        # Форматирование ячеек
        for col in range(1, 8):
            cell = ws.cell(row=row_num, column=col)
            cell.border = Border(
                left=Side(style='thin'),
                right=Side(style='thin'),
                top=Side(style='thin'),
                bottom=Side(style='thin')
            )
            if col in [1, 6, 7]:  # Числовые колонки - по центру
                cell.alignment = Alignment(horizontal='center')

        row_num += 1

    # Автоширина столбцов
    for column in ws.columns:
        max_length = 0
        column = list(column)
        for cell in column:
            try:
                if len(str(cell.value)) > max_length:
                    max_length = len(cell.value)
            except:
                pass
        adjusted_width = (max_length + 2)
        ws.column_dimensions[column[0].column_letter].width = adjusted_width

    wb.save(output_path)
    return output_path


def generate_aosr_excel(package: AOSRPackage, output_path: str) -> str:
    """
    Генерирует АОСР в формате Excel (редактируемый)
    На основе шаблона из info/04_Форма АОСР.xlsx
    """
    import openpyxl
    from openpyxl.styles import Font, Alignment, Border, Side

    # Загружаем шаблон АОСР
    template_path = "docs/technical/info/04_Форма АОСР.xlsx"

    if os.path.exists(template_path):
        wb = openpyxl.load_workbook(template_path)
        ws = wb.active

        # Заполняем поля шаблона
        # (это зависит от структуры шаблона, нужно изучить файл)
        # Пример заполнения:
        ws['B5'] = package.aosr_number  # Номер акта
        ws['E5'] = package.aosr_date.strftime("%d.%m.%Y")  # Дата
        ws['B10'] = package.work_description  # Описание работ

        # ... заполнение других полей

    else:
        # Если шаблона нет, создаём простую таблицу
        wb = openpyxl.Workbook()
        ws = wb.active
        ws.title = f"АОСР {package.aosr_number}"

        ws['A1'] = "АКТ ОСВИДЕТЕЛЬСТВОВАНИЯ СКРЫТЫХ РАБОТ"
        ws['A1'].font = Font(bold=True, size=14)

        ws['A3'] = f"№ {package.aosr_number}"
        ws['A4'] = f"Дата: {package.aosr_date.strftime('%d.%m.%Y')}"
        ws['A6'] = f"Наименование работ: {package.work_description}"

    wb.save(output_path)
    return output_path
```

---

### Этап 5: API endpoint для генерации комплекта

```python
from fastapi import APIRouter, BackgroundTasks
from app.schemas import FinalPackageResponse
from app.tasks import generate_final_package_task

router = APIRouter()

@router.post("/projects/{project_id}/generate-final-package")
async def generate_final_package(
    project_id: int,
    background_tasks: BackgroundTasks
) -> FinalPackageResponse:
    """
    Запускает генерацию финального комплекта ИД

    Возвращает:
    - task_id для отслеживания прогресса
    - estimated_time для информирования пользователя
    """

    # Запускаем задачу в фоне через Celery
    task = generate_final_package_task.delay(project_id)

    return FinalPackageResponse(
        task_id=task.id,
        status="processing",
        message="Генерация комплекта ИД запущена",
        estimated_time_seconds=120  # ~2 минуты
    )


@router.get("/projects/{project_id}/final-package/status/{task_id}")
async def get_package_generation_status(
    project_id: int,
    task_id: str
):
    """Проверка статуса генерации комплекта"""
    from celery.result import AsyncResult

    task = AsyncResult(task_id)

    if task.state == 'PENDING':
        return {
            "status": "pending",
            "progress": 0,
            "message": "Задача в очереди"
        }
    elif task.state == 'PROGRESS':
        return {
            "status": "processing",
            "progress": task.info.get('current', 0),
            "total": task.info.get('total', 100),
            "message": task.info.get('message', 'Обработка...')
        }
    elif task.state == 'SUCCESS':
        result = task.result
        return {
            "status": "completed",
            "progress": 100,
            "pdf_url": result['pdf_url'],
            "zip_url": result['zip_url'],
            "message": "Комплект ИД готов к скачиванию"
        }
    else:
        return {
            "status": "failed",
            "error": str(task.info)
        }


@router.get("/projects/{project_id}/download/pdf")
async def download_final_pdf(project_id: int):
    """Скачивание финального PDF"""
    from fastapi.responses import FileResponse

    # Путь к готовому PDF
    pdf_path = f"storage/projects/{project_id}/final_package.pdf"

    return FileResponse(
        pdf_path,
        media_type="application/pdf",
        filename=f"ИД_Проект_{project_id}.pdf"
    )


@router.get("/projects/{project_id}/download/archive")
async def download_editable_archive(project_id: int):
    """Скачивание архива с редактируемыми файлами"""
    from fastapi.responses import FileResponse

    zip_path = f"storage/projects/{project_id}/final_archive.zip"

    return FileResponse(
        zip_path,
        media_type="application/zip",
        filename=f"ИД_Архив_Проект_{project_id}.zip"
    )
```

---

### Этап 6: Celery задача с отслеживанием прогресса

```python
from celery import Task
from app.celery_app import app

class ProgressTask(Task):
    """Базовый класс для задач с отслеживанием прогресса"""

    def update_progress(self, current, total, message):
        self.update_state(
            state='PROGRESS',
            meta={
                'current': current,
                'total': total,
                'message': message
            }
        )

@app.task(bind=True, base=ProgressTask)
def generate_final_package_task(self, project_id: int):
    """
    Celery задача для генерации финального комплекта ИД
    с отслеживанием прогресса
    """

    try:
        # Этап 1: Сбор метаданных (10%)
        self.update_progress(10, 100, "Сбор метаданных проекта...")
        data = collect_project_documents(project_id)

        # Этап 2: Генерация реестра (20%)
        self.update_progress(20, 100, "Генерация реестра документов...")
        # ... код генерации реестра

        # Этап 3: Генерация титульного листа (30%)
        self.update_progress(30, 100, "Генерация титульного листа...")
        # ... код генерации титульника

        # Этап 4: Объединение PDF (50%)
        self.update_progress(50, 100, "Объединение документов в PDF...")
        pdf_path = f"storage/projects/{project_id}/final_package.pdf"
        merge_all_documents_to_pdf(project_id, pdf_path)

        # Этап 5: Создание архива (70%)
        self.update_progress(70, 100, "Создание архива с редактируемыми файлами...")
        zip_path = f"storage/projects/{project_id}/final_archive.zip"
        create_editable_archive(project_id, zip_path)

        # Этап 6: Загрузка в облако (90%)
        self.update_progress(90, 100, "Загрузка файлов в облачное хранилище...")
        pdf_url = upload_to_storage(pdf_path)
        zip_url = upload_to_storage(zip_path)

        # Этап 7: Завершение (100%)
        self.update_progress(100, 100, "Готово!")

        return {
            'pdf_url': pdf_url,
            'zip_url': zip_url,
            'status': 'completed'
        }

    except Exception as e:
        self.update_state(
            state='FAILURE',
            meta={'error': str(e)}
        )
        raise
```

---

## 🔧 Вспомогательные функции

```python
def download_from_storage(file_path: str) -> str:
    """Скачивает файл из Object Storage во временную папку"""
    import boto3
    from app.config import settings

    s3_client = boto3.client(
        's3',
        endpoint_url=settings.S3_ENDPOINT,
        aws_access_key_id=settings.S3_ACCESS_KEY,
        aws_secret_access_key=settings.S3_SECRET_KEY
    )

    temp_path = f"temp/{os.path.basename(file_path)}"
    s3_client.download_file(
        settings.S3_BUCKET,
        file_path,
        temp_path
    )

    return temp_path


def upload_to_storage(local_path: str) -> str:
    """Загружает файл в Object Storage и возвращает URL"""
    import boto3
    from app.config import settings

    s3_client = boto3.client(
        's3',
        endpoint_url=settings.S3_ENDPOINT,
        aws_access_key_id=settings.S3_ACCESS_KEY,
        aws_secret_access_key=settings.S3_SECRET_KEY
    )

    key = f"final_packages/{os.path.basename(local_path)}"

    s3_client.upload_file(
        local_path,
        settings.S3_BUCKET,
        key
    )

    # Генерируем подписанный URL (действителен 7 дней)
    url = s3_client.generate_presigned_url(
        'get_object',
        Params={
            'Bucket': settings.S3_BUCKET,
            'Key': key
        },
        ExpiresIn=7 * 24 * 60 * 60  # 7 дней
    )

    return url


def calculate_registry_pages(aosr_packages: List[AOSRPackage]) -> int:
    """Рассчитывает количество страниц, которое займёт реестр"""

    total_rows = sum(len(p.metadata) for p in aosr_packages)

    # Примерно 25-30 строк на страницу A4
    rows_per_page = 25

    pages = (total_rows // rows_per_page) + 1

    return pages
```

---

## 📊 Диаграмма workflow

```
Пользователь нажимает "Скачать комплект ИД"
          │
          ▼
┌──────────────────────────┐
│  POST /generate-final-   │
│       package            │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│  Celery задача запущена  │
│  (в фоне)                │
└───────────┬──────────────┘
            │
            ├─► [10%] Сбор метаданных (collect_project_documents)
            │
            ├─► [20%] Генерация реестра PDF (generate_registry_pdf)
            │
            ├─► [30%] Генерация титульного листа (generate_title_page)
            │
            ├─► [50%] Объединение PDF (merge_all_documents_to_pdf)
            │         ├─ Титульный лист
            │         ├─ Реестр
            │         ├─ АОСР №1 + схема + документы
            │         ├─ АОСР №2 + схема + документы
            │         └─ ...
            │
            ├─► [70%] Создание архива ZIP (create_editable_archive)
            │         ├─ PDF папка
            │         ├─ Excel папка (реестры + АОСР)
            │         ├─ DWG папка (схемы)
            │         └─ Документы качества
            │
            ├─► [90%] Загрузка в облако (upload_to_storage)
            │
            └─► [100%] Готово!
                      │
                      ▼
                Возврат URL для скачивания
```

---

## ⚡ Оптимизации

### 1. Кэширование готовых комплектов

```python
from functools import lru_cache
import hashlib

def get_package_cache_key(project_id: int) -> str:
    """
    Генерирует ключ кэша на основе хеша всех документов проекта

    Если хотя бы один документ изменился — кэш инвалидируется
    """
    from app.database import get_db
    from app.models import AOSR, Document

    db = get_db()

    # Получаем даты последнего обновления всех документов
    aosr_updates = db.query(AOSR.updated_at).filter(
        AOSR.project_id == project_id
    ).all()

    docs_updates = db.query(Document.updated_at).filter(
        Document.project_id == project_id
    ).all()

    # Создаём хеш
    hash_input = f"{project_id}_{'_'.join(str(d) for d in aosr_updates + docs_updates)}"
    cache_key = hashlib.md5(hash_input.encode()).hexdigest()

    return cache_key


@app.task(bind=True, base=ProgressTask)
def generate_final_package_task_cached(self, project_id: int):
    """Версия с кэшированием"""

    cache_key = get_package_cache_key(project_id)
    cached_path = f"cache/packages/{cache_key}.pdf"

    # Проверяем кэш
    if os.path.exists(cached_path):
        return {
            'pdf_url': upload_to_storage(cached_path),
            'from_cache': True
        }

    # Если не в кэше — генерируем
    result = generate_final_package_task(self, project_id)

    # Сохраняем в кэш
    import shutil
    shutil.copy(result['pdf_path'], cached_path)

    return result
```

---

### 2. Параллельная обработка АОСР

```python
from concurrent.futures import ThreadPoolExecutor

def merge_all_documents_to_pdf_parallel(project_id: int, output_pdf_path: str):
    """Параллельное скачивание PDF из storage"""

    data = collect_project_documents(project_id)

    # Скачиваем все PDF параллельно
    all_pdf_paths = []

    for package in data['aosr_packages']:
        all_pdf_paths.append(package.aosr_pdf_path)
        if package.schema_pdf_path:
            all_pdf_paths.append(package.schema_pdf_path)
        all_pdf_paths.extend(package.quality_docs)

    # Параллельное скачивание
    with ThreadPoolExecutor(max_workers=10) as executor:
        local_paths = list(executor.map(download_from_storage, all_pdf_paths))

    # Объединяем
    merger = PdfMerger()
    for path in local_paths:
        merger.append(path)

    merger.write(output_pdf_path)
    merger.close()
```

---

## 📝 Обновление архитектуры

Этот модуль дополняет существующие файлы архитектуры:

1. **[01-how-it-works.md](01-how-it-works.md)** — добавлен новый этап "Формирование финального комплекта"
2. **[02-data-flow.md](02-data-flow.md)** — добавлена схема трансформации данных для финального PDF
3. **[06-user-actions-breakdown.md](06-user-actions-breakdown.md)** — добавлено действие "Скачать комплект ИД"

---

## 🔗 Связь с другими компонентами

- **[05-documents.md](../stack/05-documents.md):** PyPDF2, ReportLab, openpyxl
- **[03-backend.md](../stack/03-backend.md):** FastAPI endpoints, Celery задачи
- **[07-cloud.md](../stack/07-cloud.md):** Object Storage для хранения итоговых файлов
- **[08-database.md](../stack/08-database.md):** Метаданные документов для реестра

---

**Статус:** ✅ Готово к реализации
**Последнее обновление:** 2025-12-15
