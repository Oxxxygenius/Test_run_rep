# OCR и распознавание документов

**Дата создания:** 2025-12-13
**Приоритет:** ⭐⭐⭐⭐⭐ (Критический — для анализа сканов документов качества)

---

## Назначение

OCR (Optical Character Recognition) нужен для:
- Распознавания текста со сканов документов качества (сертификаты, паспорта, декларации)
- Извлечения структурированных данных из PDF
- Классификации типов документов

---

## Рекомендуемая связка

### Основной стек: GPT-4o Vision + Tesseract OCR

```
┌─────────────────────────────────────────────────────┐
│              OCR Pipeline                           │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. Предобработка изображения                       │
│     └─ OpenCV / Pillow                              │
│                                                     │
│  2. OCR распознавание                               │
│     ├─ Tesseract OCR (базовое распознавание)        │
│     └─ GPT-4o Vision (анализ + извлечение данных)   │
│                                                     │
│  3. Постобработка + классификация                   │
│     └─ LangChain + GPT-4o                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Конкретные инструменты

### 1. GPT-4o Vision (Основной инструмент)

**Что это:**
- Мультимодальная модель OpenAI, которая "видит" изображения
- Может анализировать сканы документов напрямую
- Извлекает структурированные данные из изображений

**Преимущества:**
- ✅ Не нужна предобработка изображений
- ✅ Понимает контекст и структуру документа
- ✅ Высокая точность распознавания
- ✅ Работает даже с плохим качеством сканов

**Стоимость:**
- $10 за 1M input токенов (текст)
- $0.00765 за изображение (detail: high)
- $0.00255 за изображение (detail: low)

**Установка:**
```bash
pip install openai pillow
```

**Пример использования:**
```python
from openai import OpenAI
import base64

client = OpenAI(api_key="sk-...")

# Загрузка изображения
def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode('utf-8')

base64_image = encode_image("сертификат.jpg")

# Анализ документа
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": """Это скан документа качества.
                    Извлеки следующие данные в JSON формате:
                    - тип документа (сертификат/паспорт/декларация)
                    - наименование материала
                    - производитель
                    - номер документа
                    - дата выдачи
                    - срок действия
                    - ГОСТ/ТУ
                    """
                },
                {
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:image/jpeg;base64,{base64_image}",
                        "detail": "high"
                    }
                }
            ]
        }
    ],
    max_tokens=1000
)

import json
result = json.loads(response.choices[0].message.content)
print(result)
```

**Официальный сайт:** https://platform.openai.com/docs/guides/vision

---

### 2. Tesseract OCR (Резервный/вспомогательный)

**Что это:**
- Open-source OCR движок от Google
- Бесплатный
- Хорошая поддержка русского языка

**Когда использовать:**
- Для быстрого извлечения текста перед отправкой в GPT-4
- Для уменьшения стоимости (извлекли текст → отправили текст вместо изображения)
- Для работы offline

**Установка:**

**Windows:**
```bash
# Скачать установщик с GitHub
# https://github.com/UB-Mannheim/tesseract/wiki

# Установить tesseract-ocr
# Добавить в PATH: C:\Program Files\Tesseract-OCR

# Python библиотека
pip install pytesseract pillow
```

**Linux:**
```bash
sudo apt-get install tesseract-ocr tesseract-ocr-rus
pip install pytesseract pillow
```

**Пример использования:**
```python
import pytesseract
from PIL import Image

# Указать путь к tesseract (Windows)
pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'

# Загрузка изображения
image = Image.open('сертификат.jpg')

# Распознавание текста (русский + английский)
text = pytesseract.image_to_string(image, lang='rus+eng')

print(text)
```

**Официальный сайт:** https://github.com/tesseract-ocr/tesseract

---

### 3. PyMuPDF (fitz) — для работы с PDF

**Что это:**
- Библиотека для извлечения текста и изображений из PDF
- Очень быстрая
- Поддерживает как текстовые PDF, так и сканы

**Установка:**
```bash
pip install pymupdf
```

**Пример использования:**
```python
import fitz  # PyMuPDF

# Открытие PDF
pdf_document = fitz.open("документ_качества.pdf")

# Извлечение текста со всех страниц
full_text = ""
for page_num in range(len(pdf_document)):
    page = pdf_document[page_num]
    full_text += page.get_text()

print(full_text)

# Извлечение изображений (если PDF это скан)
for page_num in range(len(pdf_document)):
    page = pdf_document[page_num]
    image_list = page.get_images(full=True)

    for img_index, img in enumerate(image_list):
        xref = img[0]
        base_image = pdf_document.extract_image(xref)
        image_bytes = base_image["image"]

        # Сохранение изображения
        with open(f"page_{page_num}_img_{img_index}.png", "wb") as f:
            f.write(image_bytes)

pdf_document.close()
```

**Официальный сайт:** https://pymupdf.readthedocs.io/

---

### 4. OpenCV + Pillow (Предобработка изображений)

**Что это:**
- Библиотеки для улучшения качества сканов перед OCR
- Увеличивают точность распознавания

**Типичные операции:**
- Удаление шума
- Бинаризация (черно-белое)
- Выравнивание перекошенных сканов
- Увеличение контраста

**Установка:**
```bash
pip install opencv-python pillow numpy
```

**Пример предобработки:**
```python
import cv2
import numpy as np

# Загрузка изображения
image = cv2.imread('плохой_скан.jpg')

# Преобразование в grayscale
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

# Удаление шума
denoised = cv2.fastNlMeansDenoising(gray, None, 10, 7, 21)

# Бинаризация (Otsu's method)
_, binary = cv2.threshold(denoised, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)

# Увеличение резкости
kernel = np.array([[-1,-1,-1],
                   [-1, 9,-1],
                   [-1,-1,-1]])
sharpened = cv2.filter2D(binary, -1, kernel)

# Сохранение
cv2.imwrite('улучшенный_скан.jpg', sharpened)

# Теперь можно отправить в Tesseract или GPT-4o Vision
```

**Официальный сайт:** https://opencv.org/

---

## Альтернативные варианты

### Российские OCR-сервисы

#### 1. Smart Engines (Лидер в России)

**Что это:**
- Российская компания, специализирующаяся на OCR
- Высокая точность распознавания (98-99%)
- Специализированные модели для документов РФ

**Преимущества:**
- ✅ Российская инфраструктура (импортозамещение)
- ✅ Предобученные модели для российских документов
- ✅ On-premise развертывание доступно
- ✅ Высокая скорость обработки

**Стоимость:**
- Индивидуальная (зависит от объемов)
- Есть cloud API и on-premise лицензии

**API пример:**
```python
import requests

url = "https://smartengines.ru/api/v1/recognize"
headers = {"Authorization": "Bearer YOUR_API_KEY"}

with open("документ.jpg", "rb") as f:
    files = {"image": f}
    response = requests.post(url, headers=headers, files=files)

result = response.json()
print(result['text'])
```

**Официальный сайт:** https://smartengines.com/

---

#### 2. ABBYY FineReader Engine

**Что это:**
- Мировой лидер в OCR
- Российская компания ABBYY
- Отличная поддержка русского языка

**Преимущества:**
- ✅ Самая высокая точность на рынке
- ✅ Поддержка 200+ языков
- ✅ SDK для интеграции

**Недостатки:**
- ❌ Дорого (enterprise решение)
- ❌ Сложная интеграция

**Стоимость:**
- От $3000 за лицензию SDK
- Индивидуальные тарифы для Cloud API

**Официальный сайт:** https://www.abbyy.com/finereader-engine/

---

### Cloud OCR API

#### 1. Google Cloud Vision API

**Что это:**
- OCR от Google
- Часть Google Cloud Platform
- Поддержка 50+ языков

**Преимущества:**
- ✅ Высокая точность
- ✅ Простая интеграция
- ✅ Масштабируемость

**Стоимость:**
- Первые 1000 запросов/месяц — бесплатно
- Далее: $1.50 за 1000 изображений

**Установка:**
```bash
pip install google-cloud-vision
```

**Пример использования:**
```python
from google.cloud import vision

client = vision.ImageAnnotatorClient()

with open('документ.jpg', 'rb') as image_file:
    content = image_file.read()

image = vision.Image(content=content)
response = client.text_detection(image=image)
texts = response.text_annotations

if texts:
    print(texts[0].description)
```

**Официальный сайт:** https://cloud.google.com/vision/docs/ocr

---

#### 2. Azure Computer Vision (Microsoft)

**Что это:**
- OCR от Microsoft
- Часть Azure Cognitive Services

**Преимущества:**
- ✅ Высокая точность
- ✅ Поддержка рукописного текста
- ✅ Интеграция с другими Azure сервисами

**Стоимость:**
- Первые 5000 транзакций/месяц — бесплатно
- Далее: $1.00 за 1000 изображений

**Официальный сайт:** https://azure.microsoft.com/en-us/products/ai-services/ai-vision

---

## Архитектура OCR в проекте

### Pipeline обработки документов

```
┌─────────────────────────────────────────────────────────────┐
│                  Загрузка документа                         │
│                  (PDF, JPG, PNG)                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Определение типа документа                      │
│              (PDF с текстом vs скан)                         │
│              └─ PyMuPDF                                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
┌──────────────────┐      ┌──────────────────────┐
│   PDF с текстом  │      │   PDF-скан           │
│   (текстовый)    │      │   (изображение)      │
└────────┬─────────┘      └──────────┬───────────┘
         │                           │
         │                           ▼
         │              ┌──────────────────────────┐
         │              │  Предобработка           │
         │              │  (OpenCV)                │
         │              │  - Улучшение качества    │
         │              │  - Выравнивание          │
         │              └──────────┬───────────────┘
         │                         │
         │                         ▼
         │              ┌──────────────────────────┐
         │              │  OCR                     │
         │              │  - Tesseract (быстро)    │
         │              │  OR                      │
         │              │  - GPT-4o Vision (точно) │
         │              └──────────┬───────────────┘
         │                         │
         └─────────────┬───────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Анализ и извлечение данных                     │
│              (LangChain + GPT-4o)                           │
│              - Тип документа                                │
│              - Реквизиты                                    │
│              - Даты, номера                                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Сохранение в базу данных                       │
│              (PostgreSQL + JSON)                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Рекомендуемая стратегия

### Гибридный подход (оптимальная стоимость + качество)

**1. Для хорошего качества сканов:**
```python
# Используем быстрый Tesseract
text = tesseract_ocr(image)

# Отправляем текст в GPT-4o (дешевле, чем изображение)
structured_data = gpt4o_extract_from_text(text)
```

**Экономия:** ~$0.007 на документ

**2. Для плохого качества сканов:**
```python
# Предобработка
improved_image = opencv_preprocess(image)

# Отправляем изображение напрямую в GPT-4o Vision
structured_data = gpt4o_vision_extract(improved_image)
```

**Стоимость:** ~$0.008 на документ

**3. Для критичных документов:**
```python
# Двойная проверка
tesseract_result = tesseract_ocr(image)
gpt4o_result = gpt4o_vision_extract(image)

# Сравнение результатов
if confidence_score(tesseract_result) < 0.8:
    final_result = gpt4o_result
else:
    final_result = tesseract_result
```

---

## Оценка стоимости OCR

### Пример: 100 документов качества на проект

**Вариант 1: Только Tesseract**
- Стоимость: $0 (бесплатно)
- Точность: ~85-90%

**Вариант 2: Только GPT-4o Vision**
- Стоимость: 100 × $0.008 = $0.80
- Точность: ~95-98%

**Вариант 3: Гибрид (рекомендуется)**
- 70% документов → Tesseract: $0
- 30% документов → GPT-4o Vision: 30 × $0.008 = $0.24
- Итого: **$0.24 на проект**
- Точность: ~92-95%

**При 100 проектах в месяц:** $24/месяц на OCR

---

## Классификация документов

### После OCR нужно классифицировать документы

**Типы документов качества:**
1. Сертификат соответствия
2. Декларация соответствия
3. Паспорт качества
4. Отказное письмо
5. Гарантийный талон
6. Информационное письмо
7. Руководство по эксплуатации
8. Свидетельство о регистрации

**Решение: LangChain + GPT-4o**

```python
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

llm = ChatOpenAI(model="gpt-4o", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", """Ты эксперт по классификации документов качества в строительстве.
    Определи тип документа из списка:
    - сертификат соответствия
    - декларация соответствия
    - паспорт качества
    - отказное письмо
    - гарантийный талон
    - информационное письмо
    - руководство по эксплуатации
    - свидетельство о регистрации

    Верни результат в JSON: {"type": "тип документа", "confidence": 0.0-1.0}
    """),
    ("human", "Текст документа:\n{text}")
])

chain = prompt | llm

result = chain.invoke({"text": ocr_text})
print(result.content)
```

---

## Рекомендации по использованию

### ✅ Лучшие практики

1. **Всегда предобрабатывайте плохие сканы:**
   - Увеличивает точность на 10-20%
   - Используйте OpenCV

2. **Кэшируйте результаты OCR:**
   - Не распознавайте один документ дважды
   - Храните результат в БД

3. **Используйте батчинг:**
   - Обрабатывайте документы пачками
   - Асинхронные запросы к API

4. **Валидация результатов:**
   - Проверяйте форматы дат
   - Проверяйте номера документов на паттерны

5. **Человек в цикле для критичных документов:**
   - Показывайте документы с низким confidence для проверки

---

### ❌ Чего избегать

1. **НЕ отправляйте огромные изображения:**
   - Сжимайте до разумного размера (max 2000px)

2. **НЕ используйте OCR для текстовых PDF:**
   - Сначала попробуйте извлечь текст напрямую

3. **НЕ игнорируйте confidence score:**
   - Результаты с низким confidence нужно проверять

---

## Следующие шаги

1. **Установить Tesseract OCR:**
   - Скачать с https://github.com/UB-Mannheim/tesseract/wiki

2. **Получить доступ к GPT-4o Vision:**
   - Через OpenAI API (тот же ключ)

3. **Создать тестовый pipeline:**
   - Загрузка документа → OCR → извлечение данных → сохранение

4. **Собрать тестовый датасет:**
   - 50-100 реальных документов качества
   - Протестировать точность

---

## Связь с другими компонентами

- **[AI/LLM](01-ai-llm.md):** GPT-4o Vision и классификация текста
- **[Backend](03-backend.md):** API для загрузки и обработки документов
- **[Database](08-database.md):** Хранение распознанного текста и метаданных
- **[Cloud](07-cloud.md):** Хранилище для изображений документов

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-13
