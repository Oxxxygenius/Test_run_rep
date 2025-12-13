# Web Scraping (Поиск документов качества)

**Дата создания:** 2025-12-13
**Приоритет:** ⭐⭐⭐⭐⭐ (Критический — уникальная фича)

---

## Назначение

Автоматический поиск документов качества (сертификаты, паспорта, декларации) в интернете по:
- Наименованию материала
- Производителю
- ГОСТ/ТУ
- Артикулу

---

## Рекомендуемая связка

### Playwright + BeautifulSoup + LLM

```
┌─────────────────────────────────────────────────────┐
│            Web Scraping Stack                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [Browser Automation]                               │
│  └─ Playwright (динамический контент)               │
│                                                     │
│  [HTML Parsing]                                     │
│  └─ BeautifulSoup4                                  │
│                                                     │
│  [Search Strategy]                                  │
│  └─ LangChain Agent + Google Search                 │
│                                                     │
│  [Download Management]                              │
│  └─ requests + aiohttp                              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Конкретные инструменты

### 1. Playwright (Браузерная автоматизация)

**Что это:**
- Современная альтернатива Selenium
- Поддерживает JavaScript-рендеринг
- Быстрее и надежнее

**Установка:**
```bash
pip install playwright
playwright install chromium
```

**Пример использования:**
```python
from playwright.async_api import async_playwright
import asyncio

async def search_document_on_site(material_name: str, manufacturer: str):
    """Поиск документа на конкретном сайте"""
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()

        # Переход на сайт
        await page.goto('https://www.santech.ru/')

        # Поиск
        await page.fill('input[name="search"]', f"{material_name} {manufacturer}")
        await page.click('button[type="submit"]')

        # Ожидание результатов
        await page.wait_for_selector('.search-results')

        # Извлечение ссылок на документы
        links = await page.eval_on_selector_all(
            'a.document-link',
            '(elements) => elements.map(e => e.href)'
        )

        await browser.close()
        return links

# Использование
asyncio.run(search_document_on_site("Труба ПНД", "Полипластик"))
```

**Официальный сайт:** https://playwright.dev/python/

---

### 2. BeautifulSoup4 (Парсинг HTML)

**Что это:**
- Библиотека для парсинга HTML/XML
- Для статических страниц

**Установка:**
```bash
pip install beautifulsoup4 lxml
```

**Пример парсинга:**
```python
import requests
from bs4 import BeautifulSoup

def scrape_documents_from_page(url: str):
    """Парсинг страницы с документами"""
    response = requests.get(url)
    soup = BeautifulSoup(response.content, 'lxml')

    documents = []

    # Поиск ссылок на PDF документы
    for link in soup.find_all('a', href=True):
        href = link['href']
        if href.endswith('.pdf'):
            documents.append({
                'title': link.get_text(strip=True),
                'url': href,
                'type': classify_document_type(link.get_text())
            })

    return documents

def classify_document_type(title: str) -> str:
    """Определение типа документа по названию"""
    title_lower = title.lower()
    if 'сертификат' in title_lower:
        return 'certificate'
    elif 'паспорт' in title_lower:
        return 'passport'
    elif 'декларация' in title_lower:
        return 'declaration'
    else:
        return 'unknown'
```

---

### 3. LangChain Agent для умного поиска

**Что это:**
- AI-агент, который умеет искать в интернете
- Использует Google Search + анализирует результаты

**Установка:**
```bash
pip install langchain langchain-community google-search-results
```

**Пример агента:**
```python
from langchain_openai import ChatOpenAI
from langchain.agents import AgentType, initialize_agent, Tool
from langchain_community.utilities import GoogleSerperAPIWrapper

# Настройка поиска Google
search = GoogleSerperAPIWrapper(serper_api_key="YOUR_SERPER_KEY")

def search_quality_document(query: str) -> str:
    """Поиск документа качества"""
    return search.run(query)

# Создание агента
llm = ChatOpenAI(model="gpt-4o", temperature=0)

tools = [
    Tool(
        name="Google Search",
        func=search_quality_document,
        description="Используй для поиска документов качества в интернете"
    )
]

agent = initialize_agent(
    tools,
    llm,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True
)

# Использование
result = agent.run(
    """Найди сертификат соответствия на трубу ПНД ПЭ100 SDR11 Ø110мм
    производителя Полипластик. Верни прямую ссылку на PDF."""
)
print(result)
```

**Получение Serper API ключа:** https://serper.dev/

---

### 4. Стратегия поиска на конкретных сайтах

```python
from typing import List, Optional
import aiohttp
from dataclasses import dataclass

@dataclass
class DocumentSource:
    name: str
    base_url: str
    search_url: str
    selectors: dict

# Список источников
DOCUMENT_SOURCES = [
    DocumentSource(
        name="Сантехника.ру",
        base_url="https://www.santech.ru",
        search_url="https://www.santech.ru/search?q={query}",
        selectors={
            'results': '.product-item',
            'document_link': 'a.document-link'
        }
    ),
    DocumentSource(
        name="Петрович",
        base_url="https://petrovich.ru",
        search_url="https://petrovich.ru/search/?query={query}",
        selectors={
            'results': '.catalog-item',
            'document_link': 'a[href*="certificate"]'
        }
    ),
    DocumentSource(
        name="Исполнительная документация РФ",
        base_url="https://исполнительнаядокументация.рф",
        search_url="https://исполнительнаядокументация.рф/?s={query}",
        selectors={
            'results': '.doc-item',
            'document_link': 'a.download'
        }
    )
]

async def search_on_all_sources(material: dict) -> List[dict]:
    """Поиск документов на всех источниках параллельно"""
    query = f"{material['name']} {material['manufacturer']} {material.get('gost', '')}"

    async with aiohttp.ClientSession() as session:
        tasks = [
            search_on_source(session, source, query)
            for source in DOCUMENT_SOURCES
        ]
        results = await asyncio.gather(*tasks)

    # Объединение результатов
    all_documents = []
    for source_results in results:
        all_documents.extend(source_results)

    return all_documents

async def search_on_source(session, source: DocumentSource, query: str) -> List[dict]:
    """Поиск на конкретном источнике"""
    from playwright.async_api import async_playwright

    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()

        try:
            url = source.search_url.format(query=query)
            await page.goto(url, timeout=10000)

            # Ожидание загрузки результатов
            await page.wait_for_selector(source.selectors['results'], timeout=5000)

            # Извлечение ссылок
            documents = []
            elements = await page.query_selector_all(source.selectors['document_link'])

            for element in elements:
                href = await element.get_attribute('href')
                text = await element.inner_text()

                if href and href.endswith('.pdf'):
                    documents.append({
                        'source': source.name,
                        'url': href if href.startswith('http') else source.base_url + href,
                        'title': text
                    })

            await browser.close()
            return documents

        except Exception as e:
            print(f"Ошибка при поиске на {source.name}: {e}")
            await browser.close()
            return []
```

---

## Скачивание и проверка документов

```python
import aiohttp
import aiofiles
from pathlib import Path

async def download_document(url: str, save_path: Path) -> bool:
    """Скачивание документа"""
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url, timeout=30) as response:
                if response.status == 200:
                    content = await response.read()

                    # Проверка, что это PDF
                    if content[:4] != b'%PDF':
                        return False

                    # Сохранение файла
                    async with aiofiles.open(save_path, 'wb') as f:
                        await f.write(content)

                    return True
        return False
    except Exception as e:
        print(f"Ошибка скачивания {url}: {e}")
        return False

async def verify_document_relevance(pdf_path: Path, material: dict) -> float:
    """
    Проверка релевантности документа материалу
    Возвращает score от 0 до 1
    """
    from services.ocr import extract_text_from_pdf
    from langchain_openai import ChatOpenAI

    # Извлечение текста из PDF
    text = extract_text_from_pdf(pdf_path)

    # Проверка через LLM
    llm = ChatOpenAI(model="gpt-4o", temperature=0)

    prompt = f"""
    Материал: {material['name']}
    Производитель: {material['manufacturer']}
    ГОСТ/ТУ: {material.get('gost', 'не указан')}

    Текст документа:
    {text[:2000]}  # Первые 2000 символов

    Это документ качества (сертификат/паспорт/декларация) именно на этот материал?
    Верни только число от 0 до 1, где:
    - 1 = точное совпадение
    - 0.7-0.9 = высокая вероятность
    - 0.5-0.7 = возможно подходит
    - 0-0.5 = не подходит
    """

    response = llm.invoke(prompt)
    try:
        score = float(response.content.strip())
        return min(max(score, 0), 1)
    except:
        return 0.0
```

---

## Полный workflow поиска документа

```python
async def find_quality_document_for_material(material: dict) -> Optional[dict]:
    """
    Полный цикл поиска документа качества для материала
    """
    # 1. Поиск на всех источниках
    documents = await search_on_all_sources(material)

    if not documents:
        print(f"Документы не найдены для {material['name']}")
        return None

    # 2. Скачивание и проверка документов
    best_document = None
    best_score = 0

    for doc in documents[:5]:  # Проверяем топ-5
        # Скачивание
        filename = f"temp_{hash(doc['url'])}.pdf"
        save_path = Path(f"./temp/{filename}")

        if await download_document(doc['url'], save_path):
            # Проверка релевантности
            score = await verify_document_relevance(save_path, material)

            if score > best_score:
                best_score = score
                best_document = {
                    **doc,
                    'score': score,
                    'local_path': save_path
                }

            # Если нашли точное совпадение, прекращаем поиск
            if score >= 0.9:
                break

    if best_document and best_score >= 0.5:
        return best_document
    else:
        return None
```

---

## Альтернативные инструменты

### 1. Selenium (если нужна поддержка старых браузеров)

**Установка:**
```bash
pip install selenium webdriver-manager
```

---

### 2. Scrapy (если нужен полноценный crawler)

**Когда использовать:**
- Для массового скрапинга сайтов
- Когда нужна очередь задач

---

## Рекомендации

### ✅ Лучшие практики

1. **Уважайте robots.txt**
2. **Добавляйте задержки между запросами (rate limiting)**
3. **Используйте User-Agent**
4. **Кэшируйте результаты поиска**
5. **Retry механизм для отказоустойчивости**
6. **Логирование всех поисков**

### ❌ Чего избегать

1. **НЕ делайте слишком частые запросы (DDoS)**
2. **НЕ игнорируйте ошибки сети**
3. **НЕ доверяйте найденным документам на 100%**

---

## Связь с другими компонентами

- **[AI/LLM](01-ai-llm.md):** Проверка релевантности найденных документов
- **[OCR](02-ocr.md):** Анализ скачанных документов
- **[Backend](03-backend.md):** API для запуска поиска через Celery

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-13
