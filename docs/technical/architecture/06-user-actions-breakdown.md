# Разбор всех функций платформы: Что происходит под капотом

**Дата создания:** 2025-12-14
**Для кого:** Для разработчиков и аналитиков, чтобы понять каждую функцию детально

---

## 📋 Содержание

1. [Регистрация и вход](#1-регистрация-и-вход)
2. [Создание проекта](#2-создание-проекта)
3. [Загрузка проектной документации](#3-загрузка-проектной-документации)
   - [3.1. Выбор шаблонов для генерации документов](#31-выбор-шаблонов-для-генерации-документов)
4. [Загрузка документов качества (сканы)](#4-загрузка-документов-качества-сканы)
5. [Загрузка исполнительной схемы](#5-загрузка-исполнительной-схемы)
6. [Генерация АОСР](#6-генерация-аоср)
7. [Редактирование АОСР](#7-редактирование-аоср)
8. [Поиск документов качества](#8-поиск-документов-качества)
9. [Проверка комплектности](#9-проверка-комплектности)
10. [Формирование финального комплекта ИД](#10-формирование-финального-комплекта-ид)
11. [Просмотр истории проекта](#11-просмотр-истории-проекта)
12. [Управление реестрами](#12-управление-реестрами)

---

## 1. Регистрация и вход

### 1.1. Регистрация нового пользователя

#### Что делает пользователь:
```
1. Открывает страницу регистрации
2. Заполняет форму:
   - Email: ivan@stroyprom.ru
   - Пароль: SecurePass123!
   - ФИО: Иванов Иван Иванович
   - Компания: ООО СтройПром
3. Нажимает "Зарегистрироваться"
```

#### Какие данные нужны:
```json
{
  "email": "ivan@stroyprom.ru",
  "password": "SecurePass123!",
  "full_name": "Иванов Иван Иванович",
  "company": "ООО СтройПром",
  "role": "user"
}
```

#### Что происходит под капотом:

**ШАГ 1: Фронтенд валидация**
```javascript
// React компонент
const handleRegister = async (formData) => {
  // 1. Валидация на клиенте
  if (!formData.email.includes('@')) {
    showError('Некорректный email');
    return;
  }

  if (formData.password.length < 8) {
    showError('Пароль должен быть минимум 8 символов');
    return;
  }

  // 2. Отправка на backend
  try {
    const response = await fetch('/api/v1/auth/register', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify(formData)
    });

    if (response.ok) {
      const data = await response.json();
      // Сохраняем JWT токен
      localStorage.setItem('access_token', data.access_token);
      // Редирект на главную
      navigate('/dashboard');
    }
  } catch (error) {
    showError('Ошибка регистрации');
  }
};
```

**ШАГ 2: Backend обработка**
```python
# FastAPI endpoint
from fastapi import APIRouter, HTTPException
from passlib.context import CryptContext
from sqlalchemy.orm import Session

router = APIRouter()
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class UserRegister(BaseModel):
    email: EmailStr
    password: str = Field(..., min_length=8)
    full_name: str
    company: str

@router.post("/auth/register")
def register(user_data: UserRegister, db: Session = Depends(get_db)):
    # 1. Проверка: существует ли email
    existing_user = db.query(User).filter(User.email == user_data.email).first()
    if existing_user:
        raise HTTPException(status_code=400, detail="Email уже зарегистрирован")

    # 2. Хеширование пароля
    password_hash = pwd_context.hash(user_data.password)

    # 3. Создание пользователя в БД
    new_user = User(
        email=user_data.email,
        password_hash=password_hash,
        full_name=user_data.full_name,
        company=user_data.company,
        role="user",
        created_at=datetime.utcnow()
    )

    db.add(new_user)
    db.commit()
    db.refresh(new_user)

    # 4. Создание JWT токена
    access_token = create_access_token(
        data={"sub": new_user.id, "email": new_user.email}
    )

    # 5. Возврат токена
    return {
        "access_token": access_token,
        "token_type": "bearer",
        "user": {
            "id": new_user.id,
            "email": new_user.email,
            "full_name": new_user.full_name
        }
    }
```

**ШАГ 3: Запись в БД**
```sql
INSERT INTO users (
    email,
    password_hash,
    full_name,
    company,
    role,
    created_at
) VALUES (
    'ivan@stroyprom.ru',
    '$2b$12$KIXQz7.5H5xK8hH6.gZvLOq...',  -- bcrypt hash
    'Иванов Иван Иванович',
    'ООО СтройПром',
    'user',
    '2024-12-14 10:00:00'
);
```

**ШАГ 4: Возврат результата**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "user": {
    "id": 1,
    "email": "ivan@stroyprom.ru",
    "full_name": "Иванов Иван Иванович"
  }
}
```

#### Возможные ошибки:
- `400 Bad Request` — Email уже существует
- `400 Bad Request` — Слабый пароль
- `500 Internal Server Error` — Ошибка БД

---

### 1.2. Вход в систему

#### Что делает пользователь:
```
1. Открывает страницу входа
2. Вводит:
   - Email: ivan@stroyprom.ru
   - Пароль: SecurePass123!
3. Нажимает "Войти"
```

#### Какие данные нужны:
```json
{
  "email": "ivan@stroyprom.ru",
  "password": "SecurePass123!"
}
```

#### Что происходит под капотом:

**ШАГ 1: Backend проверка**
```python
@router.post("/auth/login")
def login(credentials: UserLogin, db: Session = Depends(get_db)):
    # 1. Находим пользователя по email
    user = db.query(User).filter(User.email == credentials.email).first()

    if not user:
        raise HTTPException(status_code=401, detail="Неверный email или пароль")

    # 2. Проверяем пароль
    if not pwd_context.verify(credentials.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Неверный email или пароль")

    # 3. Создаём JWT токен
    access_token = create_access_token(
        data={"sub": user.id, "email": user.email}
    )

    # 4. Обновляем last_login
    user.last_login = datetime.utcnow()
    db.commit()

    return {
        "access_token": access_token,
        "token_type": "bearer",
        "user": {
            "id": user.id,
            "email": user.email,
            "full_name": user.full_name,
            "company": user.company
        }
    }
```

**ШАГ 2: Фронтенд сохраняет токен**
```javascript
// Сохранение в localStorage
localStorage.setItem('access_token', data.access_token);
localStorage.setItem('user', JSON.stringify(data.user));

// Настройка axios для последующих запросов
axios.defaults.headers.common['Authorization'] = `Bearer ${data.access_token}`;
```

#### Возможные ошибки:
- `401 Unauthorized` — Неверный email или пароль
- `429 Too Many Requests` — Слишком много попыток входа (защита от брутфорса)

---

## 2. Создание проекта

#### Что делает пользователь:
```
1. На главной странице нажимает "Создать проект"
2. Заполняет форму:
   - Название: ЖК Солнечный, корпус 1
   - Адрес: г. Москва, ул. Ленина, д. 1
   - Заказчик: ООО Стройинвест
   - Подрядчик: ООО СтройПром
   - Генподрядчик: ООО СтройГенПодряд
   - Застройщик: ООО Девелопмент
   - **Формат комплекта ИД**: Выбирает один из вариантов:
     [ ] С повторением документов качества (Вариант 1)
     [x] Со сквозной нумерацией (Вариант 2)
3. Нажимает "Создать"
```

#### Какие данные нужны:
```json
{
  "name": "ЖК Солнечный, корпус 1",
  "address": "г. Москва, ул. Ленина, д. 1",
  "client": "ООО Стройинвест",
  "contractor": "ООО СтройПром",
  "general_contractor": "ООО СтройГенПодряд",
  "developer": "ООО Девелопмент",
  "package_format": "unified"
}
```

**Варианты формата комплекта ИД:**
- `"repeated"` — Вариант 1: Документы качества повторяются для каждого АОСР
- `"unified"` — Вариант 2: Со сквозной нумерацией, документы качества в конце

#### Что происходит под капотом:

**ШАГ 1: Frontend отправка**
```javascript
const createProject = async (projectData) => {
  const response = await fetch('/api/v1/projects', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${localStorage.getItem('access_token')}`
    },
    body: JSON.stringify(projectData)
  });

  const project = await response.json();
  return project;
};
```

**ШАГ 2: Backend создание**
```python
from enum import Enum

class PackageFormat(str, Enum):
    """Формат формирования комплекта ИД"""
    REPEATED = "repeated"  # Вариант 1: документы к каждому АОСР
    UNIFIED = "unified"    # Вариант 2: со сквозной нумерацией

class ProjectCreate(BaseModel):
    name: str
    address: str
    client: str
    contractor: str
    general_contractor: str
    developer: str
    package_format: PackageFormat = PackageFormat.UNIFIED  # По умолчанию Вариант 2

@router.post("/projects", response_model=ProjectResponse)
def create_project(
    project_data: ProjectCreate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. Создаём проект
    new_project = Project(
        name=project_data.name,
        address=project_data.address,
        client=project_data.client,
        contractor=project_data.contractor,
        general_contractor=project_data.general_contractor,
        developer=project_data.developer,
        package_format=project_data.package_format.value,  # Сохраняем выбранный формат
        status="active",
        created_by=current_user.id,
        created_at=datetime.utcnow()
    )

    db.add(new_project)
    db.commit()
    db.refresh(new_project)

    # 2. Создаём папки в Object Storage
    create_project_folders(new_project.id)

    # 3. Логируем событие
    logger.info(f"User {current_user.id} created project {new_project.id} with package format: {project_data.package_format}")

    return new_project
```

**ШАГ 3: Создание структуры папок в Object Storage**
```python
def create_project_folders(project_id: int):
    """
    Создаёт структуру папок для проекта в облачном хранилище
    """
    folders = [
        f"projects/{project_id}/documents/rd/",       # Проектная документация
        f"projects/{project_id}/documents/quality/",  # Документы качества
        f"projects/{project_id}/documents/schemas/",  # Исполнительные схемы
        f"projects/{project_id}/aosr/",               # Сгенерированные АОСР
        f"projects/{project_id}/packages/",           # Финальные комплекты
    ]

    for folder in folders:
        # Создаём пустой файл .keep для создания папки
        s3_client.put_object(
            Bucket='pto-platform',
            Key=folder + '.keep',
            Body=b''
        )
```

**ШАГ 4: Запись в БД**
```sql
INSERT INTO projects (
    name,
    address,
    client,
    contractor,
    status,
    created_by,
    created_at
) VALUES (
    'ЖК Солнечный, корпус 1',
    'г. Москва, ул. Ленина, д. 1',
    'ООО Стройинвест',
    'ООО СтройПром',
    'active',
    1,  -- ID текущего пользователя
    '2024-12-14 10:05:00'
)
RETURNING id;  -- Получаем ID созданного проекта
```

**ШАГ 5: Возврат результата**
```json
{
  "id": 123,
  "name": "ЖК Солнечный, корпус 1",
  "address": "г. Москва, ул. Ленина, д. 1",
  "client": "ООО Стройинвест",
  "contractor": "ООО СтройПром",
  "status": "active",
  "created_at": "2024-12-14T10:05:00Z",
  "created_by": {
    "id": 1,
    "full_name": "Иванов Иван Иванович"
  }
}
```

---

## 3. Загрузка проектной документации

#### Что делает пользователь:
```
1. Открывает проект "ЖК Солнечный, корпус 1"
2. Нажимает "Загрузить проектную документацию"
3. Выбирает файл: "Раздел ОВ. Водоснабжение.pdf" (15 MB)
4. Нажимает "Загрузить"
```

#### Какие данные нужны:
```
Файл: Раздел ОВ. Водоснабжение.pdf
Метаданные:
  - project_id: 123
  - doc_type: "РД" (рабочая документация)
```

#### Что происходит под капотом:

**ШАГ 1: Frontend загрузка с прогрессом**
```javascript
const uploadDocument = async (file, projectId) => {
  // 1. Валидация
  if (file.size > 50 * 1024 * 1024) {
    throw new Error('Файл слишком большой (макс 50MB)');
  }

  if (!file.type.includes('pdf')) {
    throw new Error('Только PDF файлы');
  }

  // 2. Создание FormData
  const formData = new FormData();
  formData.append('file', file);
  formData.append('project_id', projectId);
  formData.append('doc_type', 'РД');

  // 3. Загрузка с отслеживанием прогресса
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();

    // Прогресс
    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) {
        const percentComplete = (e.loaded / e.total) * 100;
        updateProgressBar(percentComplete);
      }
    };

    // Успешная загрузка
    xhr.onload = () => {
      if (xhr.status === 200) {
        resolve(JSON.parse(xhr.responseText));
      } else {
        reject(new Error('Ошибка загрузки'));
      }
    };

    // Отправка
    xhr.open('POST', '/api/v1/documents/upload');
    xhr.setRequestHeader('Authorization', `Bearer ${token}`);
    xhr.send(formData);
  });
};
```

**ШАГ 2: Backend приём файла**
```python
from fastapi import UploadFile, File
import hashlib
import os

@router.post("/documents/upload")
async def upload_document(
    file: UploadFile = File(...),
    project_id: int = Form(...),
    doc_type: str = Form(...),
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. Проверка прав доступа к проекту
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")

    if project.created_by != current_user.id:
        raise HTTPException(status_code=403, detail="Нет доступа к проекту")

    # 2. Генерация уникального имени файла
    file_hash = hashlib.md5(file.filename.encode()).hexdigest()[:8]
    file_extension = os.path.splitext(file.filename)[1]
    unique_filename = f"rd_{file_hash}_{int(time.time())}{file_extension}"

    # 3. Сохранение в Object Storage
    file_content = await file.read()
    file_path = f"projects/{project_id}/documents/rd/{unique_filename}"

    s3_client.put_object(
        Bucket='pto-platform',
        Key=file_path,
        Body=file_content,
        ContentType=file.content_type
    )

    # 4. Создание записи в БД
    new_document = Document(
        project_id=project_id,
        filename=file.filename,
        file_path=file_path,
        doc_type=doc_type,
        mime_type=file.content_type,
        file_size=len(file_content),
        uploaded_by=current_user.id,
        created_at=datetime.utcnow()
    )

    db.add(new_document)
    db.commit()
    db.refresh(new_document)

    # 5. Запуск фоновой задачи анализа
    task = analyze_rd_task.delay(new_document.id)

    # 6. Создание записи о задаче
    generation_task = GenerationTask(
        task_id=task.id,
        task_type='analyze_rd',
        project_id=project_id,
        status='pending',
        created_at=datetime.utcnow()
    )
    db.add(generation_task)
    db.commit()

    return {
        "document_id": new_document.id,
        "filename": new_document.filename,
        "status": "uploaded",
        "task_id": task.id,
        "message": "Документ загружен, анализ начат"
    }
```

**ШАГ 3: Celery задача анализа РД**
```python
@celery_app.task(bind=True)
def analyze_rd_task(self, document_id: int):
    """
    Фоновая задача: анализ проектной документации
    """
    db = SessionLocal()

    try:
        # 1. Получаем документ
        document = db.query(Document).filter(Document.id == document_id).first()

        # 2. Обновляем статус задачи
        task = db.query(GenerationTask).filter(
            GenerationTask.task_id == self.request.id
        ).first()
        task.status = 'running'
        task.started_at = datetime.utcnow()
        db.commit()

        # 3. Скачиваем PDF из Object Storage
        response = s3_client.get_object(
            Bucket='pto-platform',
            Key=document.file_path
        )
        pdf_content = response['Body'].read()

        # 4. Сохраняем временно
        temp_path = f"/tmp/{document_id}.pdf"
        with open(temp_path, 'wb') as f:
            f.write(pdf_content)

        # 5. Извлекаем текст
        import fitz  # PyMuPDF
        doc = fitz.open(temp_path)
        full_text = ""
        for page in doc:
            full_text += page.get_text()

        # 6. Сохраняем текст в БД
        document.ocr_text = full_text
        db.commit()

        # 7. Отправляем в GPT-4o для анализа
        from agents.rd_analyzer import RDAnalyzerAgent

        analyzer = RDAnalyzerAgent()
        analysis_result = analyzer.analyze(full_text)

        # analysis_result = {
        #   "works": [
        #     {
        #       "type": "Монтаж трубопроводов",
        #       "materials": [...]
        #     }
        #   ]
        # }

        # 8. Создаём черновики АОСР
        for work in analysis_result['works']:
            aosr = AOSR(
                project_id=document.project_id,
                work_type=work['type'],
                work_description=work.get('description', ''),
                content=work,  # JSON
                status='draft',
                created_at=datetime.utcnow()
            )
            db.add(aosr)

        db.commit()

        # 9. Обновляем статус задачи
        task.status = 'completed'
        task.completed_at = datetime.utcnow()
        task.result = {
            'works_found': len(analysis_result['works']),
            'total_materials': sum(len(w['materials']) for w in analysis_result['works'])
        }
        db.commit()

        # 10. Удаляем временный файл
        os.remove(temp_path)

        return {
            'status': 'success',
            'works_found': len(analysis_result['works'])
        }

    except Exception as e:
        # Обработка ошибки
        task.status = 'failed'
        task.error_message = str(e)
        task.completed_at = datetime.utcnow()
        db.commit()

        logger.error(f"RD analysis failed: {e}", exc_info=True)
        raise

    finally:
        db.close()
```

**ШАГ 4: Агент анализа РД (с использованием LangChain)**
```python
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
import json

class RDAnalyzerAgent:
    """
    Агент анализа рабочей документации для подготовки АОСР

    ВАЖНО: Анализ фокусируется на данных, необходимых для АОСР согласно регламенту:
    docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку АОСР.xlsx
    """
    def __init__(self):
        self.llm = ChatOpenAI(
            model="gpt-4o",
            temperature=0.2,
            api_key=settings.OPENAI_API_KEY
        )

        self.prompt = ChatPromptTemplate.from_messages([
            ("system", """Ты эксперт по анализу строительной проектной документации для подготовки актов освидетельствования скрытых работ (АОСР).

ВАЖНО: Анализ выполняется согласно регламенту подготовки АОСР.

ОСНОВНАЯ ЗАДАЧА:
Извлечь из проектной документации ВСЕ данные, которые потребуются для заполнения АОСР.

МАКСИМАЛЬНОЕ ВНИМАНИЕ уделяй следующим данным:

1. **ВИДЫ СКРЫТЫХ РАБОТ** (те, которые требуют АОСР):
   - Монтаж фундаментов
   - Монтаж арматурных каркасов
   - Прокладка трубопроводов
   - Монтаж электропроводки в стенах/полах
   - Устройство гидроизоляции
   - Устройство теплоизоляции
   - Другие работы, которые будут скрыты последующими конструкциями

2. **МАТЕРИАЛЫ И ИЗДЕЛИЯ** (для таблицы в АОСР):
   - Точное наименование материала/изделия
   - Технические характеристики (диаметр, марка, класс прочности и т.д.)
   - ГОСТ/ТУ (обязательно!)
   - Количество с ТОЧНЫМИ единицами измерения (м, м², м³, шт, кг, т)

3. **ДОПОЛНИТЕЛЬНЫЕ ВАЖНЫЕ ДАННЫЕ**:
   - Номера позиций по спецификации/чертежам
   - Осевые привязки (для указания местоположения работ)
   - Отметки высот
   - Номера помещений/этажей
   - Ссылки на листы чертежей

4. **ДОКУМЕНТЫ КАЧЕСТВА** (требуемые сертификаты/паспорта):
   - Для каких материалов требуются сертификаты
   - Для каких материалов требуются паспорта качества
   - Для каких материалов требуются протоколы испытаний

Верни результат СТРОГО в JSON формате:
{{
  "works": [
    {{
      "type": "Точное название вида работ",
      "description": "Подробное описание работ с указанием осей/отметок/помещений",
      "location": "Местоположение работ (оси, этаж, помещение)",
      "drawings_references": ["№ чертежа 1", "№ чертежа 2"],
      "materials": [
        {{
          "name": "Полное наименование материала/изделия",
          "specification": "Детальные характеристики (диаметр, толщина, марка, класс)",
          "gost": "ГОСТ/ТУ номер (ОБЯЗАТЕЛЬНО!)",
          "quantity": точное число,
          "unit": "единица измерения (м, м², м³, шт, кг, т)",
          "position_number": "№ позиции по спецификации",
          "quality_docs_required": ["Сертификат", "Паспорт качества", "Протокол испытаний"]
        }}
      ],
      "testing_required": ["Какие испытания требуются (например: гидравлические, прочностные)"],
      "acceptance_criteria": "Критерии приёмки работ согласно СНиП/ГОСТ"
    }}
  ]
}}

НЕ пропускай материалы! Извлекай ВСЕ материалы из спецификаций и ведомостей.
Указывай ТОЧНЫЕ значения количества и единиц измерения.
ОБЯЗАТЕЛЬНО указывай ГОСТ/ТУ для каждого материала."""),
            ("human", "ДОКУМЕНТ:\n\n{text}")
        ])

    def analyze(self, text: str) -> dict:
        # 1. Если текст слишком большой → разбиваем на чанки
        if len(text) > 100000:  # ~100K символов
            text = text[:100000]  # Берём первые 100K

        # 2. Формируем промпт
        chain = self.prompt | self.llm

        # 3. Вызываем LLM
        response = chain.invoke({"text": text})

        # 4. Парсим JSON из ответа
        try:
            result = json.loads(response.content)
            return result
        except json.JSONDecodeError:
            # Если GPT вернул не валидный JSON → пробуем извлечь
            import re
            json_match = re.search(r'\{.*\}', response.content, re.DOTALL)
            if json_match:
                return json.loads(json_match.group())
            else:
                raise ValueError("LLM не вернул валидный JSON")
```

**ШАГ 5: Запись в БД (документ + АОСР черновики)**
```sql
-- Документ
INSERT INTO documents (
    project_id,
    filename,
    file_path,
    doc_type,
    mime_type,
    file_size,
    ocr_text,
    uploaded_by,
    created_at
) VALUES (
    123,
    'Раздел ОВ. Водоснабжение.pdf',
    'projects/123/documents/rd/rd_abc123_1702550400.pdf',
    'РД',
    'application/pdf',
    15728640,
    'ПРОЕКТНАЯ ДОКУМЕНТАЦИЯ\nРаздел: Отопление и вентиляция\n...',
    1,
    '2024-12-14 10:10:00'
);

-- АОСР (черновик)
INSERT INTO aosr (
    project_id,
    work_type,
    work_description,
    content,
    status,
    created_at
) VALUES (
    123,
    'Монтаж трубопроводов системы отопления',
    'Монтаж трубопроводов из труб ПНД',
    '{
      "type": "Монтаж трубопроводов системы отопления",
      "materials": [
        {"name": "Труба ПНД", "quantity": 150, "unit": "м"},
        {"name": "Фитинги", "quantity": 25, "unit": "шт"}
      ]
    }'::jsonb,
    'draft',
    '2024-12-14 10:11:30'
);
```

**ШАГ 6: Frontend получает уведомление**
```javascript
// WebSocket или polling для отслеживания статуса задачи
const checkTaskStatus = async (taskId) => {
  const response = await fetch(`/api/v1/tasks/${taskId}`);
  const task = await response.json();

  if (task.status === 'completed') {
    showNotification('✅ Анализ завершён! Найдено актов: ' + task.result.works_found);
    refreshProjectData();
  } else if (task.status === 'failed') {
    showNotification('❌ Ошибка анализа: ' + task.error_message);
  } else {
    // Ещё выполняется → проверяем через 5 секунд
    setTimeout(() => checkTaskStatus(taskId), 5000);
  }
};
```

#### Итоговый результат для пользователя:
```
✅ Документ "Раздел ОВ. Водоснабжение.pdf" загружен
⏳ Анализируем документацию... (1 мин)
✅ Анализ завершён!

Найдено актов для подготовки: 1
- АОСР №1: Монтаж трубопроводов системы отопления
  Материалы: 2 позиции
```

---

## 3.1. Выбор шаблонов для генерации документов

**ВАЖНО:** Сразу после завершения анализа РД платформа предлагает пользователю выбрать шаблоны для будущей генерации АОСР и реестров.

### Почему сразу после анализа РД?

После анализа рабочей документации платформа уже знает:
- Сколько будет АОСР
- Какие виды работ
- Какие материалы

Это подходящий момент, чтобы настроить шаблоны под конкретный объект.

### Что делает пользователь:

```
┌─────────────────────────────────────────────────────┐
│  ✅ Анализ завершён!                                 │
│                                                     │
│  Найдено актов: 1                                   │
│  - АОСР №1: Монтаж трубопроводов системы отопления  │
│                                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  📋 Выберите шаблоны для генерации документов       │
│                                                     │
│  Платформа будет использовать эти шаблоны для       │
│  создания АОСР и реестров                           │
│                                                     │
│  ○ Использовать стандартные шаблоны                 │
│     ├─ 04_Шаблон общего реестра.xlsx                │
│     ├─ 04_Шаблон реестра к АОСР.xlsx                │
│     └─ 04_Форма АОСР.xlsx                           │
│                                                     │
│  ○ Загрузить свои шаблоны под этот объект           │
│     ├─ [Загрузить общий реестр]  📎                 │
│     ├─ [Загрузить реестр ДК]     📎                 │
│     └─ [Загрузить форму АОСР]    📎                 │
│                                                     │
│  [Пропустить]              [Сохранить и продолжить] │
└─────────────────────────────────────────────────────┘
```

### Какие данные отправляются на backend:

#### Вариант 1: Использование стандартных шаблонов
```json
{
  "project_id": 123,
  "use_default_templates": true
}
```

#### Вариант 2: Загрузка пользовательских шаблонов
```javascript
const formData = new FormData();
formData.append('use_default_templates', false);
formData.append('general_registry_template', fileInput1.files[0]);
formData.append('quality_registry_template', fileInput2.files[0]);
formData.append('aosr_form_template', fileInput3.files[0]);

await fetch(`/api/v1/projects/123/select-templates`, {
  method: 'POST',
  body: formData
});
```

### Backend обработка:

**Endpoint:** `POST /api/v1/projects/{project_id}/select-templates`

```python
from fastapi import APIRouter, UploadFile, File, HTTPException
from typing import Optional, List
import shutil
import os

router = APIRouter()

@router.post("/projects/{project_id}/select-templates")
async def select_templates(
    project_id: int,
    use_default_templates: bool = True,
    general_registry_template: Optional[UploadFile] = File(None),
    quality_registry_template: Optional[UploadFile] = File(None),
    aosr_form_template: Optional[UploadFile] = File(None)
):
    """
    Выбор шаблонов для генерации АОСР и реестров

    Вызывается СРАЗУ ПОСЛЕ анализа РД, чтобы пользователь мог настроить
    шаблоны под конкретный объект перед началом работы с АОСР
    """
    from app.models import Project
    from app.database import get_db

    db = get_db()
    project = db.query(Project).filter(Project.id == project_id).first()

    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")

    # Создаём папку для шаблонов проекта
    template_dir = f"storage/projects/{project_id}/templates"
    os.makedirs(template_dir, exist_ok=True)

    if use_default_templates:
        # Копируем стандартные шаблоны
        default_templates = {
            'general_registry': 'docs/technical/info/04_Шаблоны_Формы/04_Шаблон общего реестра.xlsx',
            'quality_registry': 'docs/technical/info/04_Шаблоны_Формы/04_Шаблон реестра к АОСР.xlsx',
            'aosr_form': 'docs/technical/info/04_Шаблоны_Формы/04_Форма АОСР.xlsx'
        }

        templates_saved = {}
        for template_type, source_path in default_templates.items():
            if os.path.exists(source_path):
                dest_filename = os.path.basename(source_path)
                dest_path = os.path.join(template_dir, dest_filename)
                shutil.copy(source_path, dest_path)
                templates_saved[template_type] = dest_path

        # Сохраняем пути к шаблонам в базе
        project.custom_templates = templates_saved
        db.commit()

        return {
            "status": "success",
            "message": "Стандартные шаблоны успешно выбраны",
            "templates": templates_saved
        }

    else:
        # Загрузка пользовательских шаблонов
        if not all([general_registry_template, quality_registry_template, aosr_form_template]):
            raise HTTPException(
                status_code=400,
                detail="Необходимо загрузить все 3 шаблона: общий реестр, реестр ДК, форма АОСР"
            )

        uploaded_templates = {}

        # Сохраняем шаблон общего реестра
        general_path = os.path.join(template_dir, f"general_registry_{general_registry_template.filename}")
        with open(general_path, "wb") as f:
            shutil.copyfileobj(general_registry_template.file, f)
        uploaded_templates['general_registry'] = general_path

        # Сохраняем шаблон реестра ДК
        quality_path = os.path.join(template_dir, f"quality_registry_{quality_registry_template.filename}")
        with open(quality_path, "wb") as f:
            shutil.copyfileobj(quality_registry_template.file, f)
        uploaded_templates['quality_registry'] = quality_path

        # Сохраняем шаблон формы АОСР
        aosr_path = os.path.join(template_dir, f"aosr_form_{aosr_form_template.filename}")
        with open(aosr_path, "wb") as f:
            shutil.copyfileobj(aosr_form_template.file, f)
        uploaded_templates['aosr_form'] = aosr_path

        # Сохраняем пути к шаблонам в базе данных
        project.custom_templates = uploaded_templates
        db.commit()

        return {
            "status": "success",
            "message": "Пользовательские шаблоны успешно загружены",
            "templates": uploaded_templates
        }
```

### Что сохраняется в базе данных:

```sql
-- В таблице projects обновляется поле custom_templates
UPDATE projects
SET custom_templates = '{
  "general_registry": "storage/projects/123/templates/04_Шаблон общего реестра.xlsx",
  "quality_registry": "storage/projects/123/templates/04_Шаблон реестра к АОСР.xlsx",
  "aosr_form": "storage/projects/123/templates/04_Форма АОСР.xlsx"
}'::jsonb
WHERE id = 123;
```

### Frontend уведомление:

```javascript
// После успешного выбора шаблонов
const response = await selectTemplates(projectId, templateData);

if (response.status === 'success') {
  showNotification('✅ Шаблоны успешно настроены!');

  // Переходим к следующему этапу работы
  router.push(`/projects/${projectId}/quality-documents`);
}
```

### Итоговый результат:
```
✅ Анализ документации завершён
✅ Шаблоны настроены (стандартные)

Теперь можно:
- Загрузить документы качества
- Начать заполнение АОСР
```

**Примечание:** Если пользователь нажал "Пропустить", платформа автоматически использует стандартные шаблоны при генерации финального комплекта.

---

## 4. Загрузка документов качества (сканы)

#### Что делает пользователь:
```
1. В проекте открывает раздел "Документы качества"
2. Нажимает "Загрузить сканы"
3. Выбирает файлы:
   - Сертификат_труба_ПНД.pdf
   - Паспорт_фитинги.pdf
   - Декларация_соответствия.pdf
4. Нажимает "Загрузить"
```

#### Какие данные нужны:
```
Файлы: массив PDF файлов (сканы)
Метаданные:
  - project_id: 123
  - doc_type: "качество"
```

#### Что происходит под капотом:

**ШАГ 1: Множественная загрузка файлов**
```javascript
const uploadQualityDocuments = async (files, projectId) => {
  // Загружаем файлы параллельно
  const uploadPromises = Array.from(files).map(file => {
    return uploadSingleDocument(file, projectId, 'качество');
  });

  const results = await Promise.all(uploadPromises);
  return results;
};
```

**ШАГ 2: Backend сохранение каждого файла**
```python
# Аналогично загрузке РД, но doc_type = 'качество'
# После сохранения → запускается OCR задача
```

**ШАГ 3: OCR и извлечение метаданных**
```python
@celery_app.task
def process_quality_document_task(document_id: int):
    """
    OCR + извлечение реквизитов из документа качества
    """
    db = SessionLocal()

    try:
        document = db.query(Document).filter(Document.id == document_id).first()

        # 1. Скачиваем PDF
        pdf_path = download_from_storage(document.file_path)

        # 2. OCR через GPT-4o Vision (для сканов лучше чем Tesseract)
        from agents.ocr_agent import OCRAgent

        ocr_agent = OCRAgent()
        ocr_result = ocr_agent.process_document(pdf_path)

        # ocr_result = {
        #   "text": "полный текст документа",
        #   "confidence": 0.95,
        #   "metadata": {
        #     "doc_type": "сертификат",
        #     "document_number": "С-РФ.АЯ46.В.04659",
        #     "issue_date": "2024-01-15",
        #     "expiry_date": "2027-01-15",
        #     "manufacturer": "ООО Полипластик",
        #     "material_name": "Труба ПНД",
        #     "gost": "ГОСТ 18599-2001"
        #   }
        # }

        # 3. Сохраняем в БД
        document.ocr_text = ocr_result['text']
        document.ocr_confidence = ocr_result['confidence']
        document.metadata = ocr_result['metadata']
        document.doc_type = ocr_result['metadata'].get('doc_type', 'неизвестно')

        db.commit()

        return {'status': 'success', 'document_id': document_id}

    finally:
        db.close()
```

**ШАГ 4: OCR Agent (с GPT-4o Vision)**
```python
from langchain_openai import ChatOpenAI
from langchain.schema.messages import HumanMessage
import base64

class OCRAgent:
    def __init__(self):
        self.llm = ChatOpenAI(
            model="gpt-4o",  # Поддерживает vision
            temperature=0.1
        )

    def process_document(self, pdf_path: str) -> dict:
        # 1. Конвертируем PDF в изображения
        import fitz
        doc = fitz.open(pdf_path)

        # Берём первую страницу (обычно там все реквизиты)
        page = doc[0]
        pix = page.get_pixmap(dpi=300)
        img_bytes = pix.tobytes("png")

        # 2. Кодируем в base64
        img_base64 = base64.b64encode(img_bytes).decode('utf-8')

        # 3. Отправляем в GPT-4o Vision
        message = HumanMessage(
            content=[
                {
                    "type": "text",
                    "text": """Проанализируй этот документ и извлеки следующую информацию:

1. Тип документа (сертификат, паспорт, декларация)
2. Номер документа
3. Дата выдачи
4. Срок действия (если есть)
5. Производитель
6. Наименование материала/изделия
7. ГОСТ/ТУ

Также распознай ВЕСЬ текст документа.

Верни результат в JSON:
{
  "doc_type": "...",
  "document_number": "...",
  "issue_date": "YYYY-MM-DD",
  "expiry_date": "YYYY-MM-DD",
  "manufacturer": "...",
  "material_name": "...",
  "gost": "...",
  "full_text": "весь распознанный текст"
}"""
                },
                {
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:image/png;base64,{img_base64}"
                    }
                }
            ]
        )

        response = self.llm.invoke([message])

        # 4. Парсим JSON
        import json
        result = json.loads(response.content)

        return {
            "text": result.get('full_text', ''),
            "confidence": 0.95,  # GPT-4o Vision обычно очень точен
            "metadata": {
                "doc_type": result.get('doc_type'),
                "document_number": result.get('document_number'),
                "issue_date": result.get('issue_date'),
                "expiry_date": result.get('expiry_date'),
                "manufacturer": result.get('manufacturer'),
                "material_name": result.get('material_name'),
                "gost": result.get('gost')
            }
        }
```

**ШАГ 5: Запись в БД**
```sql
UPDATE documents
SET
    ocr_text = 'СЕРТИФИКАТ СООТВЕТСТВИЯ\nНомер: С-РФ.АЯ46.В.04659...',
    ocr_confidence = 0.95,
    metadata = '{
      "doc_type": "сертификат",
      "document_number": "С-РФ.АЯ46.В.04659",
      "issue_date": "2024-01-15",
      "expiry_date": "2027-01-15",
      "manufacturer": "ООО Полипластик",
      "material_name": "Труба ПНД",
      "gost": "ГОСТ 18599-2001"
    }'::jsonb,
    doc_type = 'сертификат'
WHERE id = 456;
```

#### Итоговый результат для пользователя:
```
✅ Загружено 3 документа
⏳ Распознавание...
✅ Распознавание завершено!

Документы качества:
1. Сертификат соответствия
   - Материал: Труба ПНД
   - ГОСТ: ГОСТ 18599-2001
   - Действителен до: 15.01.2027

2. Паспорт качества
   - Материал: Фитинги соединительные
   - Производитель: ООО ТехПром

3. Декларация соответствия
   - Материал: Изоляция трубная
```

---

## 5. Загрузка исполнительной схемы

#### Что делает пользователь:
```
1. В АОСР №1 нажимает "Прикрепить исполнительную схему"
2. Выбирает файл: "Схема_монтаж_труб.pdf"
3. Нажимает "Загрузить"
```

#### Какие данные нужны:
```
Файл: Схема_монтаж_труб.pdf (чертёж из AutoCAD/Revit, экспортированный в PDF)
Метаданные:
  - project_id: 123
  - aosr_id: 789
  - doc_type: "схема"
```

#### Что происходит под капотом:

**ШАГ 1: Загрузка схемы**
```python
@router.post("/documents/upload-schema")
async def upload_schema(
    file: UploadFile = File(...),
    aosr_id: int = Form(...),
    db: Session = Depends(get_db)
):
    # 1. Получаем АОСР
    aosr = db.query(AOSR).filter(AOSR.id == aosr_id).first()
    if not aosr:
        raise HTTPException(status_code=404, detail="АОСР не найден")

    # 2. Сохраняем файл в Object Storage
    file_content = await file.read()
    file_path = f"projects/{aosr.project_id}/documents/schemas/schema_{aosr_id}.pdf"

    s3_client.put_object(
        Bucket='pto-platform',
        Key=file_path,
        Body=file_content,
        ContentType='application/pdf'
    )

    # 3. Создаём документ
    schema_doc = Document(
        project_id=aosr.project_id,
        filename=file.filename,
        file_path=file_path,
        doc_type='схема',
        mime_type='application/pdf',
        file_size=len(file_content),
        created_at=datetime.utcnow()
    )
    db.add(schema_doc)
    db.commit()
    db.refresh(schema_doc)

    # 4. Привязываем к АОСР
    aosr.schema_document_id = schema_doc.id
    db.commit()

    return {
        "document_id": schema_doc.id,
        "aosr_id": aosr_id,
        "message": "Схема прикреплена к АОСР"
    }
```

**ШАГ 2: Запись в БД**
```sql
-- Создание документа
INSERT INTO documents (
    project_id,
    filename,
    file_path,
    doc_type,
    mime_type,
    file_size,
    created_at
) VALUES (
    123,
    'Схема_монтаж_труб.pdf',
    'projects/123/documents/schemas/schema_789.pdf',
    'схема',
    'application/pdf',
    2457600,
    '2024-12-14 10:20:00'
);

-- Обновление АОСР
UPDATE aosr
SET schema_document_id = 567
WHERE id = 789;
```

#### Итоговый результат:
```
✅ Исполнительная схема прикреплена к АОСР №1
📄 Файл: Схема_монтаж_труб.pdf (2.4 MB)
```

---

## 6. Генерация АОСР

Это один из самых важных процессов! Разберём подробно.

#### Что делает пользователь:
```
1. Видит список черновиков АОСР (созданных после анализа РД)
2. Выбирает АОСР №1: "Монтаж трубопроводов"
3. Нажимает "Сгенерировать АОСР"
4. Заполняет дополнительную форму:
   - Дата работ: 10.12.2024
   - Ответственный от подрядчика: Иванов И.И., инженер
   - Ответственный от заказчика: Петров П.П., технический надзор
   - Примечания: (опционально)
5. Нажимает "Создать акт"
```

#### Какие данные нужны:
```json
{
  "aosr_id": 789,
  "work_date": "2024-12-10",
  "responsible_persons": {
    "contractor": {
      "name": "Иванов И.И.",
      "position": "Инженер ПТО"
    },
    "client": {
      "name": "Петров П.П.",
      "position": "Технический надзор"
    },
    "supervisor": {
      "name": "Сидоров С.С.",
      "position": "Представитель стройконтроля"
    }
  },
  "notes": ""
}
```

#### Что происходит под капотом:

**ШАГ 1: Backend получает запрос**
```python
class AOSRGenerateRequest(BaseModel):
    aosr_id: int
    work_date: date
    responsible_persons: dict
    notes: Optional[str] = ""

@router.post("/aosr/generate")
def generate_aosr(
    request: AOSRGenerateRequest,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """
    Генерация PDF файла АОСР

    ВАЖНО: Генерация выполняется согласно регламенту:
    docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку АОСР.xlsx

    Процесс:
    1. Валидация данных АОСР (все обязательные поля заполнены)
    2. Обновление метаданных (дата работ, ответственные лица)
    3. Запуск асинхронной задачи генерации PDF через Celery
    4. Возврат task_id для отслеживания статуса
    """
    # 1. Получаем АОСР
    aosr = db.query(AOSR).filter(AOSR.id == request.aosr_id).first()
    if not aosr:
        raise HTTPException(status_code=404, detail="АОСР не найден")

    # 2. Обновляем данные
    aosr.work_date = request.work_date
    content = aosr.content  # JSON
    content['responsible_persons'] = request.responsible_persons
    content['notes'] = request.notes
    aosr.content = content
    aosr.status = 'generating'
    db.commit()

    # 3. Запускаем Celery задачу генерации (согласно регламенту)
    task = generate_aosr_pdf_task.delay(aosr.id)

    # 4. Создаём запись о задаче
    generation_task = GenerationTask(
        task_id=task.id,
        task_type='generate_aosr',
        project_id=aosr.project_id,
        aosr_id=aosr.id,
        status='pending',
        created_at=datetime.utcnow()
    )
    db.add(generation_task)
    db.commit()

    return {
        "task_id": task.id,
        "status": "generating",
        "message": "Генерация АОСР начата"
    }
```

**ШАГ 2: Celery задача генерации PDF**
```python
@celery_app.task(bind=True)
def generate_aosr_pdf_task(self, aosr_id: int):
    """
    Асинхронная задача генерации PDF файла АОСР

    ВАЖНО: Генерация выполняется согласно регламенту:
    docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку АОСР.xlsx

    Процесс:
    1. Извлечение данных АОСР из БД
    2. Подготовка данных согласно структуре регламента
    3. Генерация PDF с использованием AOSRGeneratorAgent
    4. Загрузка PDF в Object Storage
    5. Обновление статуса АОСР в БД

    Args:
        aosr_id: ID акта освидетельствования скрытых работ

    Returns:
        dict: {'status': 'success', 'aosr_id': int}
    """
    db = SessionLocal()

    try:
        # 1. Получаем АОСР с проектом
        aosr = db.query(AOSR).filter(AOSR.id == aosr_id).first()
        project = aosr.project

        # 2. Подготавливаем данные для генерации (согласно регламенту)
        data = {
            "aosr_number": f"АОСР-{aosr.id}",
            "project": {
                "name": project.name,
                "address": project.address,
                "client": project.client,
                "contractor": project.contractor
            },
            "work": {
                "type": aosr.work_type,
                "description": aosr.work_description,
                "date": aosr.work_date.strftime("%d.%m.%Y")
            },
            "materials": aosr.content.get('materials', []),
            "responsible_persons": aosr.content.get('responsible_persons', {}),
            "notes": aosr.content.get('notes', '')
        }

        # 3. Используем агента генерации АОСР
        from agents.aosr_generator import AOSRGeneratorAgent

        generator = AOSRGeneratorAgent()
        pdf_path = generator.generate_pdf(data)

        # 4. Загружаем PDF в Object Storage
        storage_path = f"projects/{project.id}/aosr/aosr_{aosr_id}.pdf"

        with open(pdf_path, 'rb') as f:
            s3_client.put_object(
                Bucket='pto-platform',
                Key=storage_path,
                Body=f.read(),
                ContentType='application/pdf'
            )

        # 5. Обновляем АОСР
        aosr.status = 'generated'
        aosr.generated_pdf_path = storage_path
        aosr.updated_at = datetime.utcnow()
        db.commit()

        # 6. Обновляем статус задачи
        task = db.query(GenerationTask).filter(
            GenerationTask.task_id == self.request.id
        ).first()
        task.status = 'completed'
        task.completed_at = datetime.utcnow()
        db.commit()

        # 7. Удаляем временный файл
        os.remove(pdf_path)

        return {'status': 'success', 'aosr_id': aosr_id}

    except Exception as e:
        # Обработка ошибки
        aosr.status = 'error'
        db.commit()

        task = db.query(GenerationTask).filter(
            GenerationTask.task_id == self.request.id
        ).first()
        task.status = 'failed'
        task.error_message = str(e)
        db.commit()

        raise

    finally:
        db.close()
```

**ШАГ 3: Агент генерации АОСР (ReportLab)**
```python
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
from reportlab.lib.units import mm
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont

class AOSRGeneratorAgent:
    """
    Агент для генерации актов освидетельствования скрытых работ (АОСР)

    ВАЖНО: При создании АОСР необходимо руководствоваться регламентом:
    docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку АОСР.xlsx

    Регламент содержит:
    - Обязательные поля и разделы АОСР
    - Правила заполнения таблиц материалов
    - Требования к подписям ответственных лиц
    - Структуру документа согласно ГОСТ
    """
    def __init__(self):
        # Регистрируем русский шрифт
        pdfmetrics.registerFont(TTFont('DejaVuSans', 'DejaVuSans.ttf'))

    def generate_pdf(self, data: dict) -> str:
        """
        Генерирует PDF файл АОСР согласно регламенту

        Регламент: docs/technical/info/02_Регламенты_Процессы/02_ТЗ на подготовку АОСР.xlsx

        Args:
            data: Словарь с данными АОСР, включающий:
                - aosr_number: Номер акта
                - project: Информация о проекте (name, address, client, contractor)
                - work: Информация о работах (type, description)
                - materials: Список материалов с их характеристиками
                - participants: Ответственные лица

        Returns:
            str: Путь к сгенерированному PDF файлу
        """
        # 1. Создаём временный файл
        import tempfile
        temp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.pdf')
        pdf_path = temp_file.name
        temp_file.close()

        # 2. Создаём PDF
        c = canvas.Canvas(pdf_path, pagesize=A4)
        width, height = A4

        # 3. Настройки
        c.setFont('DejaVuSans', 10)

        # 4. Рисуем содержимое
        y = height - 30*mm

        # Заголовок
        c.setFont('DejaVuSans', 14)
        c.drawCentredString(width/2, y, "АКТ ОСВИДЕТЕЛЬСТВОВАНИЯ СКРЫТЫХ РАБОТ")
        y -= 10*mm

        c.setFont('DejaVuSans', 12)
        c.drawCentredString(width/2, y, f"№ {data['aosr_number']}")
        y -= 15*mm

        # Информация об объекте
        c.setFont('DejaVuSans', 10)
        c.drawString(20*mm, y, f"Объект: {data['project']['name']}")
        y -= 5*mm
        c.drawString(20*mm, y, f"Адрес: {data['project']['address']}")
        y -= 5*mm
        c.drawString(20*mm, y, f"Заказчик: {data['project']['client']}")
        y -= 5*mm
        c.drawString(20*mm, y, f"Подрядчик: {data['project']['contractor']}")
        y -= 10*mm

        # Информация о работах
        c.setFont('DejaVuSans-Bold', 11)
        c.drawString(20*mm, y, "НАИМЕНОВАНИЕ РАБОТ:")
        y -= 5*mm

        c.setFont('DejaVuSans', 10)
        c.drawString(20*mm, y, data['work']['type'])
        y -= 5*mm
        c.drawString(20*mm, y, data['work']['description'])
        y -= 10*mm

        # Таблица материалов
        c.setFont('DejaVuSans-Bold', 11)
        c.drawString(20*mm, y, "ПРИМЕНЁННЫЕ МАТЕРИАЛЫ И ИЗДЕЛИЯ:")
        y -= 7*mm

        # Заголовки таблицы
        c.setFont('DejaVuSans', 9)
        c.drawString(20*mm, y, "№")
        c.drawString(30*mm, y, "Наименование")
        c.drawString(100*mm, y, "Кол-во")
        c.drawString(130*mm, y, "Ед. изм.")
        c.drawString(160*mm, y, "ГОСТ/ТУ")
        y -= 5*mm

        # Линия
        c.line(20*mm, y, width - 20*mm, y)
        y -= 5*mm

        # Материалы
        for i, material in enumerate(data['materials'], 1):
            if y < 50*mm:  # Если место кончилось → новая страница
                c.showPage()
                c.setFont('DejaVuSans', 9)
                y = height - 30*mm

            c.drawString(20*mm, y, str(i))
            c.drawString(30*mm, y, material['name'])
            c.drawString(100*mm, y, str(material['quantity']))
            c.drawString(130*mm, y, material['unit'])
            c.drawString(160*mm, y, material.get('gost', '-'))
            y -= 5*mm

        y -= 10*mm

        # Ответственные лица
        c.setFont('DejaVuSans-Bold', 11)
        c.drawString(20*mm, y, "ОТВЕТСТВЕННЫЕ ЛИЦА:")
        y -= 7*mm

        c.setFont('DejaVuSans', 10)
        contractor = data['responsible_persons'].get('contractor', {})
        c.drawString(20*mm, y, f"Подрядчик: {contractor.get('name', '')} ({contractor.get('position', '')})")
        c.drawString(140*mm, y, "________________")
        y -= 7*mm

        client = data['responsible_persons'].get('client', {})
        c.drawString(20*mm, y, f"Заказчик: {client.get('name', '')} ({client.get('position', '')})")
        c.drawString(140*mm, y, "________________")
        y -= 7*mm

        supervisor = data['responsible_persons'].get('supervisor', {})
        c.drawString(20*mm, y, f"Стройконтроль: {supervisor.get('name', '')} ({supervisor.get('position', '')})")
        c.drawString(140*mm, y, "________________")
        y -= 15*mm

        # Дата
        c.drawString(20*mm, y, f"Дата освидетельствования: {data['work']['date']}")

        # Примечания
        if data.get('notes'):
            y -= 10*mm
            c.setFont('DejaVuSans-Bold', 10)
            c.drawString(20*mm, y, "Примечания:")
            y -= 5*mm
            c.setFont('DejaVuSans', 9)
            c.drawString(20*mm, y, data['notes'])

        # 5. Сохраняем
        c.save()

        return pdf_path
```

**ШАГ 4: Обновление БД**
```sql
UPDATE aosr
SET
    status = 'generated',
    generated_pdf_path = 'projects/123/aosr/aosr_789.pdf',
    updated_at = '2024-12-14 10:25:00'
WHERE id = 789;
```

#### Итоговый результат для пользователя:
```
✅ АОСР №1 успешно создан!
📄 Файл: aosr_789.pdf (250 KB)

[Кнопка: Скачать PDF]
[Кнопка: Предпросмотр]
[Кнопка: Редактировать]
```

---

## 7. Редактирование АОСР

#### Что делает пользователь:
```
1. Открывает сгенерированный АОСР №1
2. Нажимает "Редактировать"
3. Изменяет данные:
   - Количество материала "Труба ПНД": 150 м → 175 м (скорректировали)
   - Добавляет новый материал: "Изоляция трубная - 175 м"
   - Изменяет дату работ: 10.12.2024 → 12.12.2024
4. Нажимает "Сохранить и пересоздать PDF"
```

#### Какие данные нужны:
```json
{
  "aosr_id": 789,
  "updates": {
    "work_date": "2024-12-12",
    "materials": [
      {
        "name": "Труба ПНД",
        "quantity": 175,
        "unit": "м",
        "gost": "ГОСТ 18599-2001"
      },
      {
        "name": "Изоляция трубная",
        "quantity": 175,
        "unit": "м",
        "gost": "ГОСТ 30732-2006"
      },
      {
        "name": "Фитинги",
        "quantity": 25,
        "unit": "шт"
      }
    ]
  }
}
```

#### Что происходит под капотом:

**ШАГ 1: Backend обновление**
```python
@router.put("/aosr/{aosr_id}")
def update_aosr(
    aosr_id: int,
    updates: AOSRUpdateRequest,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. Получаем АОСР
    aosr = db.query(AOSR).filter(AOSR.id == aosr_id).first()
    if not aosr:
        raise HTTPException(status_code=404, detail="АОСР не найден")

    # 2. Проверка прав доступа
    if aosr.project.created_by != current_user.id:
        raise HTTPException(status_code=403, detail="Нет доступа")

    # 3. Сохраняем старую версию для истории
    old_version = aosr.content.copy()

    # 4. Обновляем данные
    if updates.work_date:
        aosr.work_date = updates.work_date

    if updates.materials:
        content = aosr.content
        content['materials'] = updates.materials
        aosr.content = content

    aosr.status = 'draft'  # Возвращаем в черновик
    aosr.updated_at = datetime.utcnow()

    # 5. Сохраняем историю изменений
    history_entry = AOSRHistory(
        aosr_id=aosr_id,
        previous_version=old_version,
        changed_by=current_user.id,
        change_type='manual_edit',
        created_at=datetime.utcnow()
    )
    db.add(history_entry)

    db.commit()

    # 6. Запускаем пересоздание PDF
    task = generate_aosr_pdf_task.delay(aosr_id)

    return {
        "aosr_id": aosr_id,
        "status": "updated",
        "task_id": task.id,
        "message": "АОСР обновлён, PDF пересоздаётся"
    }
```

**ШАГ 2: Запись в БД (с историей)**
```sql
-- Обновление АОСР
UPDATE aosr
SET
    work_date = '2024-12-12',
    content = '{
      "materials": [
        {"name": "Труба ПНД", "quantity": 175, "unit": "м"},
        {"name": "Изоляция трубная", "quantity": 175, "unit": "м"},
        {"name": "Фитинги", "quantity": 25, "unit": "шт"}
      ],
      ...
    }'::jsonb,
    status = 'draft',
    updated_at = '2024-12-14 10:30:00'
WHERE id = 789;

-- История изменений
INSERT INTO aosr_history (
    aosr_id,
    previous_version,
    changed_by,
    change_type,
    created_at
) VALUES (
    789,
    '{...старая версия...}'::jsonb,
    1,
    'manual_edit',
    '2024-12-14 10:30:00'
);
```

**ШАГ 3: Автоматический поиск документов на новый материал**
```python
# После обновления АОСР → проверяем новые материалы
@celery_app.task
def check_new_materials_task(aosr_id: int):
    """
    Проверяет, есть ли новые материалы без документов качества
    """
    db = SessionLocal()

    try:
        aosr = db.query(AOSR).filter(AOSR.id == aosr_id).first()
        materials = aosr.content.get('materials', [])

        # Для каждого материала проверяем наличие документов
        for material in materials:
            # Ищем документы качества для этого материала
            existing_docs = db.query(AOSRQualityDocument).join(Document).filter(
                AOSRQualityDocument.aosr_id == aosr_id,
                Document.metadata['material_name'].astext.ilike(f"%{material['name']}%")
            ).count()

            if existing_docs == 0:
                # Нет документов → запускаем поиск
                search_quality_doc_task.delay(aosr_id, material)

    finally:
        db.close()
```

#### Итоговый результат:
```
✅ АОСР №1 обновлён
⏳ Пересоздаём PDF...
⏳ Ищем документы качества для нового материала "Изоляция трубная"...

✅ PDF пересоздан!
✅ Найден сертификат для материала "Изоляция трубная"
```

---

## 8. Поиск документов качества

Это одна из самых сложных и важных функций!

#### Что делает пользователь:
```
1. Открывает АОСР №1
2. Видит список материалов:
   - Труба ПНД ✅ (документ найден)
   - Изоляция трубная ❌ (документ не найден)
   - Фитинги ❌ (документ не найден)
3. Нажимает "Найти недостающие документы"
```

#### Какие данные нужны:
```json
{
  "aosr_id": 789,
  "materials_to_search": [
    {
      "name": "Изоляция трубная",
      "gost": "ГОСТ 30732-2006",
      "manufacturer": null
    },
    {
      "name": "Фитинги",
      "gost": null,
      "manufacturer": null
    }
  ]
}
```

#### Что происходит под капотом:

**ШАГ 1: Backend запускает поиск**
```python
@router.post("/aosr/{aosr_id}/search-quality-docs")
def search_quality_documents(
    aosr_id: int,
    db: Session = Depends(get_db)
):
    # 1. Получаем АОСР
    aosr = db.query(AOSR).filter(AOSR.id == aosr_id).first()

    # 2. Получаем список материалов без документов
    materials = aosr.content.get('materials', [])
    materials_to_search = []

    for material in materials:
        # Проверяем, есть ли уже документ
        existing_doc = db.query(AOSRQualityDocument).join(Document).filter(
            AOSRQualityDocument.aosr_id == aosr_id,
            Document.metadata['material_name'].astext.ilike(f"%{material['name']}%")
        ).first()

        if not existing_doc:
            materials_to_search.append(material)

    # 3. Запускаем параллельный поиск для каждого материала
    from celery import group

    search_tasks = group(
        search_quality_doc_task.s(aosr_id, material)
        for material in materials_to_search
    )

    result = search_tasks.apply_async()

    return {
        "materials_count": len(materials_to_search),
        "task_ids": result.id,
        "message": f"Запущен поиск для {len(materials_to_search)} материалов"
    }
```

**ШАГ 2: Celery задача поиска для одного материала**
```python
@celery_app.task(bind=True, max_retries=2)
def search_quality_doc_task(self, aosr_id: int, material: dict):
    """
    Ищет документ качества для конкретного материала

    Стратегия поиска:
    1. Локальная база загруженных документов проекта
    2. Специализированные сайты (santech.ru, petrovich.ru и т.д.)
    3. Общий поиск в интернете через Google
    4. Генерация паспорта качества (если ничего не найдено)
    """
    db = SessionLocal()

    try:
        aosr = db.query(AOSR).filter(AOSR.id == aosr_id).first()

        # ЭТАП 1: Поиск в локальной базе
        local_result = search_in_local_db(aosr.project_id, material)

        if local_result:
            # Найден в локальной базе → привязываем к АОСР
            link_document_to_aosr(db, aosr_id, local_result['document_id'], material)
            return {
                'status': 'found_local',
                'document_id': local_result['document_id'],
                'source': 'local_database'
            }

        # ЭТАП 2: Поиск на специализированных сайтах
        from agents.document_search_agent import DocumentSearchAgent

        search_agent = DocumentSearchAgent()

        # Поиск на санитарно-технических сайтах
        online_result = search_agent.search_on_specialized_sites(material)

        if online_result and online_result['confidence'] > 0.8:
            # Найден подходящий документ → скачиваем и сохраняем
            doc_id = save_found_document(
                db,
                aosr.project_id,
                online_result['file_content'],
                online_result['filename'],
                material
            )

            link_document_to_aosr(db, aosr_id, doc_id, material)

            return {
                'status': 'found_online',
                'document_id': doc_id,
                'source': online_result['source_url']
            }

        # ЭТАП 3: Общий поиск в Google
        google_result = search_agent.search_via_google(material)

        if google_result and google_result['confidence'] > 0.7:
            doc_id = save_found_document(
                db,
                aosr.project_id,
                google_result['file_content'],
                google_result['filename'],
                material
            )

            link_document_to_aosr(db, aosr_id, doc_id, material)

            return {
                'status': 'found_google',
                'document_id': doc_id,
                'source': google_result['source_url']
            }

        # ЭТАП 4: Генерация паспорта качества
        from agents.document_generator_agent import DocumentGeneratorAgent

        generator = DocumentGeneratorAgent()
        generated_doc = generator.generate_passport(aosr, material)

        # Сохраняем сгенерированный документ
        doc_id = save_generated_document(
            db,
            aosr.project_id,
            generated_doc['file_path'],
            generated_doc['filename'],
            material
        )

        link_document_to_aosr(db, aosr_id, doc_id, material)

        return {
            'status': 'generated',
            'document_id': doc_id,
            'source': 'auto_generated'
        }

    except Exception as e:
        logger.error(f"Document search failed: {e}", exc_info=True)

        # Сохраняем в лог неудачного поиска
        search_log = DocumentSearchHistory(
            material_name=material['name'],
            manufacturer=material.get('manufacturer'),
            gost=material.get('gost'),
            found_documents=None,
            error_message=str(e),
            created_at=datetime.utcnow()
        )
        db.add(search_log)
        db.commit()

        raise

    finally:
        db.close()
```

**ШАГ 3: Агент поиска документов (детально)**
```python
from playwright.sync_api import sync_playwright
from langchain_openai import ChatOpenAI
import requests

class DocumentSearchAgent:
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0.1)

        # Список специализированных сайтов
        self.specialized_sites = [
            {
                'url': 'https://www.santech.ru/',
                'name': 'SanTech',
                'search_selector': 'input[name="q"]',
                'result_selector': '.product-item'
            },
            {
                'url': 'https://petrovich.ru/',
                'name': 'Петрович',
                'search_selector': 'input[type="search"]',
                'result_selector': '.catalog-item'
            }
        ]

    def search_on_specialized_sites(self, material: dict) -> dict:
        """
        Поиск на специализированных сайтах
        """
        search_query = self._build_search_query(material)

        with sync_playwright() as p:
            browser = p.chromium.launch(headless=True)

            for site in self.specialized_sites:
                try:
                    result = self._search_on_site(browser, site, search_query, material)

                    if result:
                        browser.close()
                        return result

                except Exception as e:
                    logger.warning(f"Search failed on {site['name']}: {e}")
                    continue

            browser.close()
            return None

    def _search_on_site(self, browser, site_config: dict, query: str, material: dict):
        """
        Поиск на конкретном сайте
        """
        page = browser.new_page()

        try:
            # 1. Открываем сайт
            page.goto(site_config['url'])
            page.wait_for_load_state('networkidle')

            # 2. Вводим поисковый запрос
            search_input = page.query_selector(site_config['search_selector'])
            if not search_input:
                return None

            search_input.fill(query)
            search_input.press('Enter')
            page.wait_for_load_state('networkidle')

            # 3. Ищем PDF документы в результатах
            pdf_links = page.query_selector_all('a[href$=".pdf"]')

            if not pdf_links:
                return None

            # 4. Берём первые 3 результата и проверяем каждый
            for pdf_link in pdf_links[:3]:
                pdf_url = pdf_link.get_attribute('href')

                # Делаем абсолютный URL
                if not pdf_url.startswith('http'):
                    from urllib.parse import urljoin
                    pdf_url = urljoin(site_config['url'], pdf_url)

                # 5. Скачиваем PDF
                response = requests.get(pdf_url, timeout=30)
                if response.status_code != 200:
                    continue

                # 6. Проверяем релевантность через LLM
                relevance = self._check_document_relevance(
                    response.content,
                    material
                )

                if relevance['is_relevant'] and relevance['confidence'] > 0.8:
                    return {
                        'file_content': response.content,
                        'filename': pdf_url.split('/')[-1],
                        'source_url': pdf_url,
                        'confidence': relevance['confidence'],
                        'site': site_config['name']
                    }

            return None

        finally:
            page.close()

    def _check_document_relevance(self, pdf_content: bytes, material: dict) -> dict:
        """
        Проверяет, подходит ли найденный документ для материала
        """
        # 1. Извлекаем текст из PDF
        import fitz
        import tempfile

        with tempfile.NamedTemporaryFile(suffix='.pdf', delete=False) as tmp:
            tmp.write(pdf_content)
            tmp_path = tmp.name

        doc = fitz.open(tmp_path)
        text = ""
        for page in doc:
            text += page.get_text()

        os.unlink(tmp_path)

        # 2. Спрашиваем LLM
        prompt = f"""Проверь, подходит ли этот документ для материала.

МАТЕРИАЛ:
- Наименование: {material['name']}
- ГОСТ/ТУ: {material.get('gost', 'не указан')}
- Производитель: {material.get('manufacturer', 'не указан')}

ДОКУМЕНТ (первые 2000 символов):
{text[:2000]}

Ответь в JSON:
{{
  "is_relevant": true/false,
  "confidence": 0.0-1.0,
  "reason": "почему подходит или не подходит",
  "document_type": "сертификат/паспорт/декларация",
  "found_material_name": "наименование из документа",
  "found_gost": "ГОСТ из документа"
}}"""

        response = self.llm.invoke(prompt)

        import json
        result = json.loads(response.content)
        return result

    def _build_search_query(self, material: dict) -> str:
        """
        Формирует поисковый запрос
        """
        parts = [material['name']]

        if material.get('gost'):
            parts.append(material['gost'])

        if material.get('manufacturer'):
            parts.append(material['manufacturer'])

        parts.append('сертификат')

        return ' '.join(parts)

    def search_via_google(self, material: dict) -> dict:
        """
        Поиск через Google Search API
        """
        # Используем Custom Search API от Google
        # (требует API ключ)

        search_query = self._build_search_query(material) + ' filetype:pdf'

        # Вызов Google Custom Search API
        # ... (реализация аналогична search_on_specialized_sites)

        pass
```

**ШАГ 4: Сохранение найденного документа**
```python
def save_found_document(
    db: Session,
    project_id: int,
    file_content: bytes,
    filename: str,
    material: dict
) -> int:
    """
    Сохраняет найденный документ в проект
    """
    # 1. Сохраняем в Object Storage
    file_path = f"projects/{project_id}/documents/quality/found_{int(time.time())}_{filename}"

    s3_client.put_object(
        Bucket='pto-platform',
        Key=file_path,
        Body=file_content,
        ContentType='application/pdf'
    )

    # 2. OCR для извлечения метаданных
    from agents.ocr_agent import OCRAgent

    # Сохраняем временно для OCR
    temp_path = f"/tmp/{filename}"
    with open(temp_path, 'wb') as f:
        f.write(file_content)

    ocr_agent = OCRAgent()
    ocr_result = ocr_agent.process_document(temp_path)

    os.unlink(temp_path)

    # 3. Создаём запись в БД
    document = Document(
        project_id=project_id,
        filename=filename,
        file_path=file_path,
        doc_type=ocr_result['metadata'].get('doc_type', 'сертификат'),
        mime_type='application/pdf',
        file_size=len(file_content),
        ocr_text=ocr_result['text'],
        ocr_confidence=ocr_result['confidence'],
        metadata=ocr_result['metadata'],
        created_at=datetime.utcnow()
    )

    db.add(document)
    db.commit()
    db.refresh(document)

    return document.id

def link_document_to_aosr(
    db: Session,
    aosr_id: int,
    document_id: int,
    material: dict
):
    """
    Привязывает документ качества к АОСР
    """
    link = AOSRQualityDocument(
        aosr_id=aosr_id,
        document_id=document_id,
        material_name=material['name'],
        relevance_score=0.9,
        created_at=datetime.utcnow()
    )

    db.add(link)
    db.commit()
```

**ШАГ 5: Если ничего не найдено → Генерация паспорта**
```python
class DocumentGeneratorAgent:
    """
    Агент для программной генерации документов качества

    ВАЖНО: Программно создаются ТОЛЬКО документы качества (паспорта, сертификаты).
    Реестры ВСЕГДА используют шаблоны - если шаблона нет, запрашиваем у пользователя.
    """

    def generate_passport(self, aosr: AOSR, material: dict) -> dict:
        """
        Генерирует паспорт качества на материал программно

        Используется только когда:
        - Документ не найден в базе
        - Документ не найден через поиск в интернете
        - Пользователь не загрузил документ вручную
        """
        # 1. Собираем данные для паспорта
        data = {
            'supplier': aosr.project.contractor,
            'client': aosr.project.client,
            'object_address': aosr.project.address,
            'material_name': material['name'],
            'characteristics': material.get('specification', ''),
            'gost': material.get('gost', ''),
            'quantity': material['quantity'],
            'unit': material['unit'],
            'delivery_date': (aosr.work_date - timedelta(days=7)).strftime('%d.%m.%Y'),
            'manufacture_date': (aosr.work_date - timedelta(days=30)).strftime('%d.%m.%Y'),
            'warranty_period': '12 месяцев'
        }

        # 2. Генерируем PDF
        pdf_path = self._generate_passport_pdf(data)

        return {
            'file_path': pdf_path,
            'filename': f"passport_{material['name'].replace(' ', '_')}.pdf"
        }

    def _generate_passport_pdf(self, data: dict) -> str:
        """
        Создаёт PDF паспорта качества
        """
        from reportlab.lib.pagesizes import A4
        from reportlab.pdfgen import canvas

        temp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.pdf')
        pdf_path = temp_file.name
        temp_file.close()

        c = canvas.Canvas(pdf_path, pagesize=A4)
        width, height = A4

        c.setFont('DejaVuSans', 12)

        y = height - 30

        # Заголовок
        c.setFont('DejaVuSans-Bold', 14)
        c.drawCentredString(width/2, y, "ПАСПОРТ КАЧЕСТВА")
        y -= 20

        # Данные
        c.setFont('DejaVuSans', 10)
        c.drawString(50, y, f"Поставщик: {data['supplier']}")
        y -= 15
        c.drawString(50, y, f"Заказчик: {data['client']}")
        y -= 15
        c.drawString(50, y, f"Объект: {data['object_address']}")
        y -= 25

        c.setFont('DejaVuSans-Bold', 11)
        c.drawString(50, y, "НАИМЕНОВАНИЕ МАТЕРИАЛА:")
        y -= 15
        c.setFont('DejaVuSans', 10)
        c.drawString(50, y, data['material_name'])
        y -= 15
        c.drawString(50, y, f"ГОСТ/ТУ: {data['gost']}")
        y -= 25

        c.drawString(50, y, f"Количество: {data['quantity']} {data['unit']}")
        y -= 15
        c.drawString(50, y, f"Дата поставки: {data['delivery_date']}")
        y -= 15
        c.drawString(50, y, f"Дата изготовления: {data['manufacture_date']}")
        y -= 15
        c.drawString(50, y, f"Гарантийный срок: {data['warranty_period']}")
        y -= 40

        c.drawString(50, y, "Поставщик: ________________")

        c.save()

        return pdf_path
```

#### Итоговый результат:
```
⏳ Поиск документов качества для 2 материалов...

Материал: "Изоляция трубная"
├─ Поиск в локальной базе... ❌
├─ Поиск на SanTech.ru... ✅ Найден сертификат!
└─ Скачан и привязан к АОСР

Материал: "Фитинги"
├─ Поиск в локальной базе... ❌
├─ Поиск на SanTech.ru... ❌
├─ Поиск на Petrovich.ru... ❌
├─ Поиск в Google... ❌
└─ Создан паспорт качества ✅

✅ Все документы найдены/созданы!
```

---

## 9. Проверка комплектности

#### Что делает пользователь:
```
1. В АОСР №1 нажимает "Проверить комплектность"
2. Платформа анализирует весь комплект
```

#### Какие данные нужны:
```json
{
  "aosr_id": 789
}
```

#### Что происходит под капотом:

**ШАГ 1: Backend запуск проверки**
```python
@router.post("/aosr/{aosr_id}/validate")
def validate_aosr(
    aosr_id: int,
    db: Session = Depends(get_db)
):
    # Запускаем Celery задачу валидации
    task = validation_agent_task.delay(aosr_id)

    return {
        "task_id": task.id,
        "status": "validating",
        "message": "Проверка запущена"
    }
```

**ШАГ 2: Celery задача валидации**
```python
@celery_app.task
def validation_agent_task(aosr_id: int):
    """
    Проверяет комплектность и корректность АОСР
    """
    db = SessionLocal()

    try:
        aosr = db.query(AOSR).filter(AOSR.id == aosr_id).first()

        from agents.validation_agent import ValidationAgent

        validator = ValidationAgent()
        validation_result = validator.validate_aosr(aosr, db)

        # validation_result = {
        #   "is_valid": true/false,
        #   "issues": [
        #     {
        #       "type": "warning/error",
        #       "message": "...",
        #       "field": "..."
        #     }
        #   ],
        #   "checks_performed": [...]
        # }

        # Сохраняем результат
        aosr.validation_result = validation_result
        aosr.validation_status = 'valid' if validation_result['is_valid'] else 'has_issues'
        aosr.validated_at = datetime.utcnow()
        db.commit()

        return validation_result

    finally:
        db.close()
```

**ШАГ 3: Агент валидации (детально)**
```python
from datetime import datetime, timedelta

class ValidationAgent:
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0.1)

    def validate_aosr(self, aosr: AOSR, db: Session) -> dict:
        """
        Полная проверка АОСР
        """
        issues = []

        # 1. Проверка наличия всех обязательных документов
        issues.extend(self._check_required_documents(aosr, db))

        # 2. Проверка дат
        issues.extend(self._check_dates_consistency(aosr, db))

        # 3. Проверка соответствия материалов проекту
        issues.extend(self._check_materials_compliance(aosr, db))

        # 4. Проверка полноты данных
        issues.extend(self._check_data_completeness(aosr))

        # 5. Проверка через LLM (логическая согласованность)
        issues.extend(self._check_logical_consistency(aosr, db))

        # Определяем валидность
        critical_issues = [i for i in issues if i['type'] == 'error']
        is_valid = len(critical_issues) == 0

        return {
            "is_valid": is_valid,
            "issues": issues,
            "checks_performed": [
                "required_documents",
                "dates_consistency",
                "materials_compliance",
                "data_completeness",
                "logical_consistency"
            ],
            "summary": {
                "total_issues": len(issues),
                "errors": len(critical_issues),
                "warnings": len([i for i in issues if i['type'] == 'warning'])
            }
        }

    def _check_required_documents(self, aosr: AOSR, db: Session) -> list:
        """
        Проверка наличия документов на все материалы
        """
        issues = []
        materials = aosr.content.get('materials', [])

        for material in materials:
            # Ищем документ качества для этого материала
            doc_count = db.query(AOSRQualityDocument).join(Document).filter(
                AOSRQualityDocument.aosr_id == aosr.id,
                Document.metadata['material_name'].astext.ilike(f"%{material['name']}%")
            ).count()

            if doc_count == 0:
                issues.append({
                    'type': 'error',
                    'category': 'missing_document',
                    'message': f"Отсутствует документ качества для материала '{material['name']}'",
                    'field': 'quality_documents',
                    'material': material['name']
                })

        # Проверка исполнительной схемы
        if not aosr.schema_document_id:
            issues.append({
                'type': 'warning',
                'category': 'missing_schema',
                'message': "Не прикреплена исполнительная схема",
                'field': 'schema_document'
            })

        return issues

    def _check_dates_consistency(self, aosr: AOSR, db: Session) -> list:
        """
        КРИТИЧНО: Проверка дат
        """
        issues = []
        work_date = aosr.work_date

        # Получаем все документы качества
        quality_docs = db.query(Document).join(AOSRQualityDocument).filter(
            AOSRQualityDocument.aosr_id == aosr.id
        ).all()

        for doc in quality_docs:
            metadata = doc.metadata or {}

            # 1. Проверка даты поставки
            delivery_date_str = metadata.get('delivery_date') or metadata.get('issue_date')
            if delivery_date_str:
                try:
                    delivery_date = datetime.strptime(delivery_date_str, '%Y-%m-%d').date()

                    # Дата поставки должна быть РАНЬШЕ даты работ
                    if delivery_date > work_date:
                        issues.append({
                            'type': 'error',
                            'category': 'date_inconsistency',
                            'message': f"Дата поставки ({delivery_date_str}) позже даты работ ({work_date}). Материал ещё не поставлен!",
                            'field': 'dates',
                            'document': doc.filename
                        })

                    # Предупреждение, если слишком большой разрыв (>1 года)
                    if (work_date - delivery_date).days > 365:
                        issues.append({
                            'type': 'warning',
                            'category': 'date_warning',
                            'message': f"Большой разрыв между поставкой ({delivery_date_str}) и работами ({work_date})",
                            'field': 'dates',
                            'document': doc.filename
                        })

                except ValueError:
                    issues.append({
                        'type': 'warning',
                        'category': 'invalid_date',
                        'message': f"Некорректный формат даты в документе {doc.filename}",
                        'field': 'dates'
                    })

            # 2. Проверка срока действия сертификата
            expiry_date_str = metadata.get('expiry_date')
            if expiry_date_str:
                try:
                    expiry_date = datetime.strptime(expiry_date_str, '%Y-%m-%d').date()

                    # Сертификат должен действовать на момент работ
                    if expiry_date < work_date:
                        issues.append({
                            'type': 'error',
                            'category': 'expired_certificate',
                            'message': f"Сертификат {doc.filename} недействителен на дату работ (истёк {expiry_date_str})",
                            'field': 'dates',
                            'document': doc.filename
                        })

                except ValueError:
                    pass

        return issues

    def _check_materials_compliance(self, aosr: AOSR, db: Session) -> list:
        """
        Проверка соответствия материалов проекту
        """
        issues = []

        # Получаем проектную документацию
        rd_docs = db.query(Document).filter(
            Document.project_id == aosr.project_id,
            Document.doc_type == 'РД'
        ).all()

        if not rd_docs:
            # Нет РД для сравнения
            return issues

        # Используем LLM для сопоставления
        materials_in_aosr = aosr.content.get('materials', [])

        for rd_doc in rd_docs:
            if not rd_doc.ocr_text:
                continue

            # Спрашиваем LLM: соответствуют ли материалы в АОСР проекту?
            prompt = f"""Сравни материалы из АОСР с проектной документацией.

МАТЕРИАЛЫ В АОСР:
{json.dumps(materials_in_aosr, ensure_ascii=False, indent=2)}

ПРОЕКТНАЯ ДОКУМЕНТАЦИЯ (фрагмент):
{rd_doc.ocr_text[:3000]}

Найди несоответствия:
1. Материалы в АОСР, которых нет в проекте
2. Различия в характеристиках (диаметр, марка, ГОСТ)
3. Значительные расхождения в количестве

Ответь в JSON:
{{
  "has_issues": true/false,
  "issues": [
    {{
      "material": "название материала",
      "issue_type": "not_in_project/spec_mismatch/quantity_mismatch",
      "description": "описание проблемы"
    }}
  ]
}}"""

            response = self.llm.invoke(prompt)
            result = json.loads(response.content)

            if result.get('has_issues'):
                for issue in result.get('issues', []):
                    issues.append({
                        'type': 'warning',
                        'category': 'material_compliance',
                        'message': f"Материал '{issue['material']}': {issue['description']}",
                        'field': 'materials',
                        'rd_document': rd_doc.filename
                    })

        return issues

    def _check_data_completeness(self, aosr: AOSR) -> list:
        """
        Проверка полноты заполнения данных
        """
        issues = []

        # Проверка даты работ
        if not aosr.work_date:
            issues.append({
                'type': 'error',
                'category': 'missing_data',
                'message': "Не указана дата работ",
                'field': 'work_date'
            })

        # Проверка ответственных лиц
        responsible = aosr.content.get('responsible_persons', {})

        required_roles = ['contractor', 'client', 'supervisor']
        for role in required_roles:
            if not responsible.get(role, {}).get('name'):
                issues.append({
                    'type': 'warning',
                    'category': 'missing_data',
                    'message': f"Не указан ответственный от {role}",
                    'field': 'responsible_persons'
                })

        # Проверка материалов
        materials = aosr.content.get('materials', [])
        if not materials:
            issues.append({
                'type': 'error',
                'category': 'missing_data',
                'message': "Отсутствуют материалы в АОСР",
                'field': 'materials'
            })

        return issues

    def _check_logical_consistency(self, aosr: AOSR, db: Session) -> list:
        """
        Проверка логической согласованности через LLM
        """
        issues = []

        # Собираем все данные АОСР
        full_data = {
            'work_type': aosr.work_type,
            'work_date': aosr.work_date.isoformat() if aosr.work_date else None,
            'materials': aosr.content.get('materials', []),
            'responsible_persons': aosr.content.get('responsible_persons', {}),
            'project': {
                'name': aosr.project.name,
                'address': aosr.project.address
            }
        }

        # Получаем информацию о документах качества
        quality_docs_info = []
        quality_docs = db.query(Document).join(AOSRQualityDocument).filter(
            AOSRQualityDocument.aosr_id == aosr.id
        ).all()

        for doc in quality_docs:
            quality_docs_info.append({
                'filename': doc.filename,
                'type': doc.doc_type,
                'metadata': doc.metadata
            })

        # Спрашиваем LLM о логической согласованности
        prompt = f"""Проанализируй АОСР на логическую согласованность.

ДАННЫЕ АОСР:
{json.dumps(full_data, ensure_ascii=False, indent=2)}

ДОКУМЕНТЫ КАЧЕСТВА:
{json.dumps(quality_docs_info, ensure_ascii=False, indent=2)}

Проверь:
1. Логичны ли указанные объёмы работ?
2. Соответствуют ли материалы виду работ?
3. Нет ли очевидных ошибок или противоречий?
4. Достаточно ли документов качества?

Ответь в JSON:
{{
  "has_issues": true/false,
  "issues": [
    {{
      "severity": "error/warning",
      "description": "описание проблемы"
    }}
  ]
}}"""

        response = self.llm.invoke(prompt)
        result = json.loads(response.content)

        if result.get('has_issues'):
            for issue in result.get('issues', []):
                issues.append({
                    'type': issue.get('severity', 'warning'),
                    'category': 'logical_inconsistency',
                    'message': issue['description'],
                    'field': 'general'
                })

        return issues
```

**ШАГ 4: Запись результата в БД**
```sql
UPDATE aosr
SET
    validation_result = '{
      "is_valid": true,
      "issues": [
        {
          "type": "warning",
          "message": "Большой разрыв между поставкой и работами"
        }
      ],
      "summary": {
        "total_issues": 1,
        "errors": 0,
        "warnings": 1
      }
    }'::jsonb,
    validation_status = 'valid',
    validated_at = '2024-12-14 10:40:00'
WHERE id = 789;
```

#### Итоговый результат:
```
✅ Проверка завершена!

Результат: Комплект готов к сдаче ✅

Найдено замечаний: 1
└─ ⚠️ Предупреждение: Большой разрыв между поставкой и работами
   (Сертификат выдан 01.01.2023, работы выполнены 12.12.2024)

Все критические проверки пройдены:
✅ Все документы качества присутствуют
✅ Даты согласованы
✅ Сертификаты действительны
✅ Материалы соответствуют проекту
✅ Все обязательные поля заполнены
```

---

## 10. Формирование финального комплекта ИД

Это финальный этап — сборка всех документов в один PDF файл!

#### Что делает пользователь:
```
1. Открывает проект "ЖК Солнечный, корпус 1"
2. Видит список АОСР:
   - АОСР №1: Монтаж трубопроводов ✅ Готов
   - АОСР №2: Монтаж вентиляции ✅ Готов
3. Нажимает "Сформировать комплект ИД"
4. Платформа собирает все документы в один PDF
```

#### Какие данные нужны:
```json
{
  "project_id": 123,
  "include_options": {
    "title_page": true,
    "general_registry": true,
    "aosr_list": true,
    "schemas": true,
    "quality_documents": true,
    "notes": "Дополнительные примечания (опционально)"
  }
}
```

#### Что происходит под капотом:

**ШАГ 1: Backend запуск формирования**
```python
@router.post("/projects/{project_id}/create-package")
def create_final_package(
    project_id: int,
    options: PackageCreateRequest,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. Проверка прав доступа
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project or project.created_by != current_user.id:
        raise HTTPException(status_code=403, detail="Нет доступа")

    # 2. Проверка готовности АОСР
    aosr_list = db.query(AOSR).filter(
        AOSR.project_id == project_id,
        AOSR.status == 'generated',
        AOSR.validation_status == 'valid'
    ).all()

    if not aosr_list:
        raise HTTPException(
            status_code=400,
            detail="Нет готовых АОСР для формирования комплекта"
        )

    # 3. Запускаем Celery задачу
    task = create_package_task.delay(project_id, options.dict())

    # 4. Создаём запись о задаче
    generation_task = GenerationTask(
        task_id=task.id,
        task_type='create_package',
        project_id=project_id,
        status='pending',
        created_at=datetime.utcnow()
    )
    db.add(generation_task)
    db.commit()

    return {
        "task_id": task.id,
        "status": "creating",
        "message": "Формирование комплекта ИД начато",
        "aosr_count": len(aosr_list)
    }
```

**ШАГ 2: Celery задача сборки комплекта**
```python
@celery_app.task(bind=True)
def create_package_task(self, project_id: int, options: dict):
    """
    Формирует финальный комплект ИД
    """
    db = SessionLocal()

    try:
        project = db.query(Project).filter(Project.id == project_id).first()

        # 1. Собираем все файлы, которые нужно включить
        from services.package_creator import PackageCreator

        creator = PackageCreator(project, db)
        file_list = creator.collect_files(options)

        # file_list = [
        #   {'type': 'title_page', 'path': None, 'title': 'Титульный лист'},
        #   {'type': 'general_registry', 'path': None, 'title': 'Общий реестр'},
        #   {'type': 'aosr', 'path': 'projects/123/aosr/aosr_789.pdf', 'title': 'АОСР №1'},
        #   {'type': 'schema', 'path': 'projects/123/schemas/schema_789.pdf', 'title': 'Схема к АОСР №1'},
        #   {'type': 'quality_doc', 'path': '...', 'title': 'Сертификат ...'},
        #   ...
        # ]

        # 2. Создаём финальный PDF
        final_pdf_path = creator.create_final_pdf(file_list, options)

        # 3. Загружаем в Object Storage
        storage_path = f"projects/{project_id}/packages/package_{int(time.time())}.pdf"

        with open(final_pdf_path, 'rb') as f:
            s3_client.put_object(
                Bucket='pto-platform',
                Key=storage_path,
                Body=f.read(),
                ContentType='application/pdf'
            )

        # 4. Создаём запись о комплекте в БД
        package = ExecutivePackage(
            project_id=project_id,
            file_path=storage_path,
            file_size=os.path.getsize(final_pdf_path),
            aosr_count=len([f for f in file_list if f['type'] == 'aosr']),
            total_documents=len(file_list),
            created_by=project.created_by,
            created_at=datetime.utcnow()
        )
        db.add(package)
        db.commit()
        db.refresh(package)

        # 5. Обновляем статус задачи
        task = db.query(GenerationTask).filter(
            GenerationTask.task_id == self.request.id
        ).first()
        task.status = 'completed'
        task.completed_at = datetime.utcnow()
        task.result = {
            'package_id': package.id,
            'file_path': storage_path,
            'file_size_mb': round(package.file_size / (1024 * 1024), 2),
            'total_documents': package.total_documents
        }
        db.commit()

        # 6. Удаляем временный файл
        os.remove(final_pdf_path)

        return {
            'status': 'success',
            'package_id': package.id,
            'file_path': storage_path
        }

    except Exception as e:
        task = db.query(GenerationTask).filter(
            GenerationTask.task_id == self.request.id
        ).first()
        task.status = 'failed'
        task.error_message = str(e)
        db.commit()

        logger.error(f"Package creation failed: {e}", exc_info=True)
        raise

    finally:
        db.close()
```

**ШАГ 3: Сервис создания комплекта (детально)**
```python
from PyPDF2 import PdfMerger, PdfReader
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
import tempfile

class PackageCreator:
    def __init__(self, project: Project, db: Session):
        self.project = project
        self.db = db

    def collect_files(self, options: dict) -> list:
        """
        Собирает список всех файлов для комплекта
        """
        files = []

        # 1. Титульный лист (генерируем на лету)
        if options.get('title_page', True):
            files.append({
                'type': 'title_page',
                'path': None,  # Генерируем динамически
                'title': 'Титульный лист',
                'data': self._prepare_title_page_data()
            })

        # 2. Общий реестр (генерируем на лету)
        if options.get('general_registry', True):
            files.append({
                'type': 'general_registry',
                'path': None,
                'title': 'Общий реестр исполнительной документации',
                'data': self._prepare_registry_data()
            })

        # 3. Все АОСР (из БД)
        if options.get('aosr_list', True):
            aosr_list = self.db.query(AOSR).filter(
                AOSR.project_id == self.project.id,
                AOSR.status == 'generated'
            ).order_by(AOSR.id).all()

            for aosr in aosr_list:
                # АОСР
                files.append({
                    'type': 'aosr',
                    'path': aosr.generated_pdf_path,
                    'title': f"АОСР №{aosr.id}: {aosr.work_type}",
                    'aosr_id': aosr.id
                })

                # Исполнительная схема к АОСР
                if options.get('schemas', True) and aosr.schema_document_id:
                    schema_doc = self.db.query(Document).filter(
                        Document.id == aosr.schema_document_id
                    ).first()

                    if schema_doc:
                        files.append({
                            'type': 'schema',
                            'path': schema_doc.file_path,
                            'title': f"Исполнительная схема к АОСР №{aosr.id}"
                        })

                # Реестр документов качества для этого АОСР
                files.append({
                    'type': 'quality_registry',
                    'path': None,
                    'title': f"Реестр документов качества к АОСР №{aosr.id}",
                    'data': self._prepare_quality_registry_data(aosr.id)
                })

                # Документы качества
                if options.get('quality_documents', True):
                    quality_docs = self.db.query(Document).join(AOSRQualityDocument).filter(
                        AOSRQualityDocument.aosr_id == aosr.id
                    ).all()

                    for doc in quality_docs:
                        files.append({
                            'type': 'quality_doc',
                            'path': doc.file_path,
                            'title': f"{doc.doc_type}: {doc.filename}"
                        })

        return files

    def create_final_pdf(self, file_list: list, options: dict) -> str:
        """
        Создаёт финальный PDF файл
        """
        # Создаём временный файл для результата
        temp_output = tempfile.NamedTemporaryFile(delete=False, suffix='.pdf')
        output_path = temp_output.name
        temp_output.close()

        # Используем PdfMerger для объединения
        merger = PdfMerger()

        try:
            for item in file_list:
                if item['path'] is None:
                    # Генерируем PDF на лету (титульный лист, реестры)
                    generated_pdf = self._generate_special_page(item)
                    merger.append(generated_pdf)
                    os.unlink(generated_pdf)  # Удаляем временный
                else:
                    # Скачиваем из Object Storage
                    local_pdf = self._download_from_storage(item['path'])
                    merger.append(local_pdf)
                    os.unlink(local_pdf)  # Удаляем временный

                # Добавляем bookmark для навигации
                merger.add_outline_item(
                    item['title'],
                    len(merger.pages) - 1
                )

            # Сохраняем финальный PDF
            merger.write(output_path)
            merger.close()

            return output_path

        except Exception as e:
            merger.close()
            if os.path.exists(output_path):
                os.unlink(output_path)
            raise

    def _generate_special_page(self, item: dict) -> str:
        """
        Генерирует специальные страницы (титульник, реестры)
        """
        temp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.pdf')
        pdf_path = temp_file.name
        temp_file.close()

        c = canvas.Canvas(pdf_path, pagesize=A4)
        width, height = A4

        if item['type'] == 'title_page':
            self._draw_title_page(c, width, height, item['data'])
        elif item['type'] == 'general_registry':
            self._draw_general_registry(c, width, height, item['data'])
        elif item['type'] == 'quality_registry':
            self._draw_quality_registry(c, width, height, item['data'])

        c.save()
        return pdf_path

    def _draw_title_page(self, c, width, height, data):
        """
        Рисует титульный лист
        """
        c.setFont('DejaVuSans-Bold', 16)
        y = height - 100

        c.drawCentredString(width/2, y, "ИСПОЛНИТЕЛЬНАЯ ДОКУМЕНТАЦИЯ")
        y -= 40

        c.setFont('DejaVuSans', 12)
        c.drawCentredString(width/2, y, f"Объект: {data['project_name']}")
        y -= 20
        c.drawCentredString(width/2, y, f"Адрес: {data['project_address']}")
        y -= 60

        c.setFont('DejaVuSans', 10)
        c.drawString(50, y, f"Заказчик: {data['client']}")
        y -= 15
        c.drawString(50, y, f"Подрядчик: {data['contractor']}")
        y -= 40

        c.drawString(50, y, f"Количество актов: {data['aosr_count']}")
        y -= 15
        c.drawString(50, y, f"Количество документов: {data['total_documents']}")
        y -= 40

        c.drawString(50, y, f"Дата формирования: {datetime.now().strftime('%d.%m.%Y')}")

    def _draw_general_registry(self, c, width, height, data):
        """
        Рисует общий реестр
        """
        c.setFont('DejaVuSans-Bold', 14)
        y = height - 50

        c.drawCentredString(width/2, y, "ОБЩИЙ РЕЕСТР")
        c.drawCentredString(width/2, y - 15, "ИСПОЛНИТЕЛЬНОЙ ДОКУМЕНТАЦИИ")
        y -= 50

        # Таблица
        c.setFont('DejaVuSans-Bold', 9)
        c.drawString(30, y, "№")
        c.drawString(60, y, "Наименование документа")
        c.drawString(400, y, "Страницы")
        y -= 15

        c.line(30, y, width - 30, y)
        y -= 10

        # Заполняем таблицу
        c.setFont('DejaVuSans', 8)
        for i, item in enumerate(data['items'], 1):
            if y < 50:  # Новая страница
                c.showPage()
                c.setFont('DejaVuSans', 8)
                y = height - 50

            c.drawString(30, y, str(i))
            c.drawString(60, y, item['title'])
            c.drawString(400, y, str(item['page_number']))
            y -= 12

    def _draw_quality_registry(self, c, width, height, data):
        """
        Рисует реестр документов качества для АОСР
        """
        c.setFont('DejaVuSans-Bold', 12)
        y = height - 50

        c.drawString(50, y, f"РЕЕСТР ДОКУМЕНТОВ КАЧЕСТВА")
        y -= 15
        c.setFont('DejaVuSans', 10)
        c.drawString(50, y, f"к АОСР №{data['aosr_id']}: {data['work_type']}")
        y -= 30

        # Таблица
        c.setFont('DejaVuSans-Bold', 8)
        c.drawString(30, y, "№")
        c.drawString(50, y, "Материал")
        c.drawString(200, y, "Документ")
        c.drawString(350, y, "Номер")
        c.drawString(450, y, "Действителен до")
        y -= 12

        c.line(30, y, width - 30, y)
        y -= 10

        c.setFont('DejaVuSans', 7)
        for i, doc in enumerate(data['documents'], 1):
            if y < 50:
                c.showPage()
                c.setFont('DejaVuSans', 7)
                y = height - 50

            c.drawString(30, y, str(i))
            c.drawString(50, y, doc['material_name'][:25])
            c.drawString(200, y, doc['doc_type'])
            c.drawString(350, y, doc.get('doc_number', '-')[:15])
            c.drawString(450, y, doc.get('expiry_date', '-'))
            y -= 10

    def _download_from_storage(self, file_path: str) -> str:
        """
        Скачивает файл из Object Storage во временную папку
        """
        response = s3_client.get_object(
            Bucket='pto-platform',
            Key=file_path
        )

        temp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.pdf')
        temp_file.write(response['Body'].read())
        temp_file.close()

        return temp_file.name

    def _prepare_title_page_data(self) -> dict:
        """
        Подготавливает данные для титульного листа
        """
        aosr_count = self.db.query(AOSR).filter(
            AOSR.project_id == self.project.id,
            AOSR.status == 'generated'
        ).count()

        total_docs = self.db.query(Document).filter(
            Document.project_id == self.project.id
        ).count()

        return {
            'project_name': self.project.name,
            'project_address': self.project.address,
            'client': self.project.client,
            'contractor': self.project.contractor,
            'aosr_count': aosr_count,
            'total_documents': total_docs
        }

    def _prepare_registry_data(self) -> dict:
        """
        Подготавливает данные для общего реестра
        """
        items = []
        current_page = 1

        # Титульный лист
        items.append({'title': 'Титульный лист', 'page_number': current_page})
        current_page += 1

        # Общий реестр
        items.append({'title': 'Общий реестр', 'page_number': current_page})
        current_page += 1

        # АОСР и документы
        aosr_list = self.db.query(AOSR).filter(
            AOSR.project_id == self.project.id,
            AOSR.status == 'generated'
        ).all()

        for aosr in aosr_list:
            items.append({
                'title': f"АОСР №{aosr.id}: {aosr.work_type}",
                'page_number': current_page
            })
            # Примерно 3 страницы на АОСР
            current_page += 3

            # Схема
            if aosr.schema_document_id:
                items.append({
                    'title': f"Схема к АОСР №{aosr.id}",
                    'page_number': current_page
                })
                current_page += 1

            # Реестр документов качества
            items.append({
                'title': f"Реестр документов качества к АОСР №{aosr.id}",
                'page_number': current_page
            })
            current_page += 1

            # Документы качества
            quality_docs_count = self.db.query(AOSRQualityDocument).filter(
                AOSRQualityDocument.aosr_id == aosr.id
            ).count()

            items.append({
                'title': f"Документы качества ({quality_docs_count} шт.)",
                'page_number': current_page
            })
            current_page += quality_docs_count * 2  # Примерно 2 страницы на документ

        return {'items': items}

    def _prepare_quality_registry_data(self, aosr_id: int) -> dict:
        """
        Подготавливает данные для реестра документов качества
        """
        aosr = self.db.query(AOSR).filter(AOSR.id == aosr_id).first()

        documents = []
        quality_docs = self.db.query(Document, AOSRQualityDocument).join(
            AOSRQualityDocument
        ).filter(
            AOSRQualityDocument.aosr_id == aosr_id
        ).all()

        for doc, link in quality_docs:
            metadata = doc.metadata or {}

            documents.append({
                'material_name': link.material_name,
                'doc_type': doc.doc_type,
                'doc_number': metadata.get('document_number', ''),
                'expiry_date': metadata.get('expiry_date', '')
            })

        return {
            'aosr_id': aosr_id,
            'work_type': aosr.work_type,
            'documents': documents
        }
```

**ШАГ 4: Запись в БД**
```sql
-- Новая таблица для комплектов
CREATE TABLE executive_packages (
    id SERIAL PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id),
    file_path VARCHAR(500) NOT NULL,
    file_size BIGINT,
    aosr_count INTEGER,
    total_documents INTEGER,
    created_by INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Запись
INSERT INTO executive_packages (
    project_id,
    file_path,
    file_size,
    aosr_count,
    total_documents,
    created_by,
    created_at
) VALUES (
    123,
    'projects/123/packages/package_1702551234.pdf',
    47185920,  -- ~45 MB
    2,
    15,
    1,
    '2024-12-14 10:50:00'
);
```

#### Итоговый результат:
```
✅ Комплект ИД сформирован!

📦 Файл: package_1702551234.pdf
📏 Размер: 45.0 MB
📄 Страниц: 250
📋 Содержимое:
   ├─ Титульный лист (1 стр)
   ├─ Общий реестр (2 стр)
   ├─ АОСР №1: Монтаж трубопроводов (3 стр)
   ├─ Исполнительная схема к АОСР №1 (1 стр)
   ├─ Реестр документов качества к АОСР №1 (1 стр)
   ├─ Сертификат: Труба ПНД (2 стр)
   ├─ Паспорт: Изоляция трубная (1 стр)
   ├─ Паспорт: Фитинги (1 стр)
   ├─ АОСР №2: Монтаж вентиляции (3 стр)
   └─ ...

[Кнопка: Скачать комплект]
[Кнопка: Предпросмотр]
[Кнопка: Создать новый комплект]
```

---

## 11. Просмотр истории проекта

#### Что делает пользователь:
```
1. Открывает проект "ЖК Солнечный, корпус 1"
2. Нажимает вкладку "История"
3. Видит все действия по проекту
```

#### Какие данные нужны:
```json
{
  "project_id": 123,
  "filters": {
    "date_from": "2024-12-01",
    "date_to": "2024-12-14",
    "event_types": ["document_upload", "aosr_generated", "validation"],
    "user_id": null
  },
  "pagination": {
    "page": 1,
    "per_page": 20
  }
}
```

#### Что происходит под капотом:

**ШАГ 1: Backend запрос истории**
```python
@router.get("/projects/{project_id}/history")
def get_project_history(
    project_id: int,
    date_from: Optional[str] = None,
    date_to: Optional[str] = None,
    event_type: Optional[str] = None,
    page: int = 1,
    per_page: int = 20,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. Проверка прав
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project or project.created_by != current_user.id:
        raise HTTPException(status_code=403)

    # 2. Собираем события из разных источников
    from services.history_aggregator import HistoryAggregator

    aggregator = HistoryAggregator(project_id, db)
    events = aggregator.get_timeline(
        date_from=date_from,
        date_to=date_to,
        event_type=event_type,
        page=page,
        per_page=per_page
    )

    return {
        "events": events['items'],
        "total": events['total'],
        "page": page,
        "per_page": per_page,
        "pages": events['pages']
    }
```

**ШАГ 2: Агрегатор истории**
```python
class HistoryAggregator:
    """
    Собирает события из разных таблиц в единую хронологию
    """

    def __init__(self, project_id: int, db: Session):
        self.project_id = project_id
        self.db = db

    def get_timeline(
        self,
        date_from=None,
        date_to=None,
        event_type=None,
        page=1,
        per_page=20
    ) -> dict:
        """
        Получает хронологию событий проекта
        """
        events = []

        # 1. События загрузки документов
        events.extend(self._get_document_events(date_from, date_to))

        # 2. События генерации АОСР
        events.extend(self._get_aosr_events(date_from, date_to))

        # 3. События валидации
        events.extend(self._get_validation_events(date_from, date_to))

        # 4. События поиска документов
        events.extend(self._get_search_events(date_from, date_to))

        # 5. События создания комплектов
        events.extend(self._get_package_events(date_from, date_to))

        # 6. События изменения АОСР
        events.extend(self._get_aosr_history_events(date_from, date_to))

        # 7. Фильтруем по типу события
        if event_type:
            events = [e for e in events if e['type'] == event_type]

        # 8. Сортируем по дате (новые сначала)
        events.sort(key=lambda x: x['timestamp'], reverse=True)

        # 9. Пагинация
        total = len(events)
        start = (page - 1) * per_page
        end = start + per_page
        items = events[start:end]

        return {
            'items': items,
            'total': total,
            'pages': (total + per_page - 1) // per_page
        }

    def _get_document_events(self, date_from, date_to) -> list:
        """
        События загрузки документов
        """
        query = self.db.query(Document).filter(
            Document.project_id == self.project_id
        )

        if date_from:
            query = query.filter(Document.created_at >= date_from)
        if date_to:
            query = query.filter(Document.created_at <= date_to)

        documents = query.all()

        events = []
        for doc in documents:
            events.append({
                'type': 'document_upload',
                'icon': '📄',
                'title': f"Загружен документ: {doc.filename}",
                'description': f"Тип: {doc.doc_type}, Размер: {self._format_size(doc.file_size)}",
                'timestamp': doc.created_at,
                'user': self._get_user_name(doc.uploaded_by) if hasattr(doc, 'uploaded_by') else None,
                'metadata': {
                    'document_id': doc.id,
                    'doc_type': doc.doc_type
                }
            })

        return events

    def _get_aosr_events(self, date_from, date_to) -> list:
        """
        События генерации АОСР
        """
        query = self.db.query(AOSR).filter(
            AOSR.project_id == self.project_id,
            AOSR.status == 'generated'
        )

        if date_from:
            query = query.filter(AOSR.created_at >= date_from)
        if date_to:
            query = query.filter(AOSR.created_at <= date_to)

        aosr_list = query.all()

        events = []
        for aosr in aosr_list:
            events.append({
                'type': 'aosr_generated',
                'icon': '✅',
                'title': f"Создан АОСР №{aosr.id}",
                'description': f"{aosr.work_type}",
                'timestamp': aosr.created_at,
                'user': self._get_user_name(aosr.created_by) if hasattr(aosr, 'created_by') else None,
                'metadata': {
                    'aosr_id': aosr.id,
                    'work_type': aosr.work_type
                }
            })

        return events

    def _get_validation_events(self, date_from, date_to) -> list:
        """
        События валидации
        """
        query = self.db.query(AOSR).filter(
            AOSR.project_id == self.project_id,
            AOSR.validated_at.isnot(None)
        )

        if date_from:
            query = query.filter(AOSR.validated_at >= date_from)
        if date_to:
            query = query.filter(AOSR.validated_at <= date_to)

        aosr_list = query.all()

        events = []
        for aosr in aosr_list:
            status_icon = '✅' if aosr.validation_status == 'valid' else '⚠️'
            status_text = 'Прошёл проверку' if aosr.validation_status == 'valid' else 'Есть замечания'

            events.append({
                'type': 'validation',
                'icon': status_icon,
                'title': f"Проверка АОСР №{aosr.id}: {status_text}",
                'description': self._format_validation_summary(aosr.validation_result),
                'timestamp': aosr.validated_at,
                'metadata': {
                    'aosr_id': aosr.id,
                    'validation_status': aosr.validation_status,
                    'issues_count': len(aosr.validation_result.get('issues', []))
                }
            })

        return events

    def _get_package_events(self, date_from, date_to) -> list:
        """
        События создания комплектов
        """
        query = self.db.query(ExecutivePackage).filter(
            ExecutivePackage.project_id == self.project_id
        )

        if date_from:
            query = query.filter(ExecutivePackage.created_at >= date_from)
        if date_to:
            query = query.filter(ExecutivePackage.created_at <= date_to)

        packages = query.all()

        events = []
        for pkg in packages:
            events.append({
                'type': 'package_created',
                'icon': '📦',
                'title': f"Создан комплект ИД",
                'description': f"Размер: {self._format_size(pkg.file_size)}, АОСР: {pkg.aosr_count}, Документов: {pkg.total_documents}",
                'timestamp': pkg.created_at,
                'user': self._get_user_name(pkg.created_by),
                'metadata': {
                    'package_id': pkg.id,
                    'file_path': pkg.file_path
                }
            })

        return events

    def _get_aosr_history_events(self, date_from, date_to) -> list:
        """
        События изменения АОСР
        """
        # Если есть таблица истории изменений
        if not hasattr(self.db.query, 'AOSRHistory'):
            return []

        query = self.db.query(AOSRHistory).join(AOSR).filter(
            AOSR.project_id == self.project_id
        )

        if date_from:
            query = query.filter(AOSRHistory.created_at >= date_from)
        if date_to:
            query = query.filter(AOSRHistory.created_at <= date_to)

        history_entries = query.all()

        events = []
        for entry in history_entries:
            events.append({
                'type': 'aosr_edited',
                'icon': '✏️',
                'title': f"Изменён АОСР №{entry.aosr_id}",
                'description': self._describe_changes(entry.previous_version, entry.aosr.content),
                'timestamp': entry.created_at,
                'user': self._get_user_name(entry.changed_by),
                'metadata': {
                    'aosr_id': entry.aosr_id,
                    'change_type': entry.change_type
                }
            })

        return events

    def _get_search_events(self, date_from, date_to) -> list:
        """
        События поиска документов качества
        """
        query = self.db.query(DocumentSearchHistory).filter(
            DocumentSearchHistory.material_name.in_(
                self.db.query(AOSR.content).filter(
                    AOSR.project_id == self.project_id
                )
            )
        )

        # Упрощённо — в реальности нужна более сложная связь

        return []  # Пропускаем для краткости

    def _format_size(self, size_bytes: int) -> str:
        """Форматирует размер файла"""
        if size_bytes < 1024:
            return f"{size_bytes} B"
        elif size_bytes < 1024 * 1024:
            return f"{size_bytes / 1024:.1f} KB"
        else:
            return f"{size_bytes / (1024 * 1024):.1f} MB"

    def _get_user_name(self, user_id: int) -> str:
        """Получает имя пользователя"""
        user = self.db.query(User).filter(User.id == user_id).first()
        return user.full_name if user else "Неизвестно"

    def _format_validation_summary(self, validation_result: dict) -> str:
        """Форматирует итог валидации"""
        if not validation_result:
            return ""

        summary = validation_result.get('summary', {})
        errors = summary.get('errors', 0)
        warnings = summary.get('warnings', 0)

        parts = []
        if errors > 0:
            parts.append(f"Ошибок: {errors}")
        if warnings > 0:
            parts.append(f"Предупреждений: {warnings}")

        return ", ".join(parts) if parts else "Замечаний нет"

    def _describe_changes(self, old_version: dict, new_version: dict) -> str:
        """Описывает изменения между версиями"""
        changes = []

        # Сравниваем материалы
        old_materials = set(m['name'] for m in old_version.get('materials', []))
        new_materials = set(m['name'] for m in new_version.get('materials', []))

        added = new_materials - old_materials
        removed = old_materials - new_materials

        if added:
            changes.append(f"Добавлено материалов: {len(added)}")
        if removed:
            changes.append(f"Удалено материалов: {len(removed)}")

        return ", ".join(changes) if changes else "Незначительные изменения"
```

#### Итоговый результат:
```
ИСТОРИЯ ПРОЕКТА "ЖК Солнечный, корпус 1"

┌─────────────────────────────────────────────────────────┐
│ 14.12.2024 10:50 | 📦 Создан комплект ИД               │
│ Иванов И.И.      | Размер: 45 MB, АОСР: 2, Документов: 15 │
├─────────────────────────────────────────────────────────┤
│ 14.12.2024 10:40 | ✅ Проверка АОСР №1: Прошёл проверку │
│ Система          | Предупреждений: 1                   │
├─────────────────────────────────────────────────────────┤
│ 14.12.2024 10:30 | ✏️ Изменён АОСР №1                   │
│ Иванов И.И.      | Добавлено материалов: 1             │
├─────────────────────────────────────────────────────────┤
│ 14.12.2024 10:25 | ✅ Создан АОСР №1                    │
│ Система          | Монтаж трубопроводов системы отопления │
├─────────────────────────────────────────────────────────┤
│ 14.12.2024 10:20 | 📄 Загружен документ: Схема_монтаж_труб.pdf │
│ Иванов И.И.      | Тип: схема, Размер: 2.4 MB          │
├─────────────────────────────────────────────────────────┤
│ 14.12.2024 10:15 | 📄 Загружен документ: Сертификат_труба_ПНД.pdf │
│ Иванов И.И.      | Тип: сертификат, Размер: 1.2 MB     │
└─────────────────────────────────────────────────────────┘

[Фильтры: Тип события ▼ | Дата ▼ | Пользователь ▼]
[Показано: 6 из 24 событий]
```

---

## 12. Управление реестрами

#### Что делает пользователь:
```
1. Открывает АОСР №1
2. Нажимает "Реестр материалов"
3. Видит/редактирует реестр применённых материалов
```

#### Какие данные нужны:
```json
{
  "aosr_id": 789
}
```

#### Что происходит под капотом:

**ШАГ 1: Backend получение реестра**
```python
@router.get("/aosr/{aosr_id}/materials-registry")
def get_materials_registry(
    aosr_id: int,
    db: Session = Depends(get_db)
):
    # 1. Получаем АОСР
    aosr = db.query(AOSR).filter(AOSR.id == aosr_id).first()
    if not aosr:
        raise HTTPException(status_code=404)

    # 2. Формируем реестр
    from services.registry_generator import RegistryGenerator

    generator = RegistryGenerator(aosr, db)
    registry = generator.generate_materials_registry()

    return registry
```

**ШАГ 2: Генератор реестра**
```python
class RegistryGenerator:
    def __init__(self, aosr: AOSR, db: Session):
        self.aosr = aosr
        self.db = db

    def generate_materials_registry(self) -> dict:
        """
        Генерирует реестр применённых материалов
        """
        materials = self.aosr.content.get('materials', [])

        registry_items = []

        for material in materials:
            # Находим документ качества для этого материала
            doc_link = self.db.query(AOSRQualityDocument, Document).join(
                Document
            ).filter(
                AOSRQualityDocument.aosr_id == self.aosr.id,
                AOSRQualityDocument.material_name == material['name']
            ).first()

            if doc_link:
                link, doc = doc_link
                metadata = doc.metadata or {}

                item = {
                    'material_name': material['name'],
                    'specification': material.get('specification', ''),
                    'gost': material.get('gost', ''),
                    'quantity': material['quantity'],
                    'unit': material['unit'],
                    'quality_document': {
                        'type': doc.doc_type,
                        'number': metadata.get('document_number', ''),
                        'issue_date': metadata.get('issue_date', ''),
                        'expiry_date': metadata.get('expiry_date', ''),
                        'manufacturer': metadata.get('manufacturer', '')
                    }
                }
            else:
                item = {
                    'material_name': material['name'],
                    'specification': material.get('specification', ''),
                    'gost': material.get('gost', ''),
                    'quantity': material['quantity'],
                    'unit': material['unit'],
                    'quality_document': None
                }

            registry_items.append(item)

        return {
            'aosr_id': self.aosr.id,
            'work_type': self.aosr.work_type,
            'total_materials': len(registry_items),
            'items': registry_items
        }

    def generate_pdf_registry(self) -> str:
        """
        Создаёт PDF реестра материалов
        """
        registry = self.generate_materials_registry()

        temp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.pdf')
        pdf_path = temp_file.name
        temp_file.close()

        c = canvas.Canvas(pdf_path, pagesize=A4)
        width, height = A4

        c.setFont('DejaVuSans-Bold', 12)
        y = height - 50

        c.drawString(50, y, "РЕЕСТР ПРИМЕНЁННЫХ МАТЕРИАЛОВ И ИЗДЕЛИЙ")
        y -= 15
        c.setFont('DejaVuSans', 10)
        c.drawString(50, y, f"к АОСР №{registry['aosr_id']}: {registry['work_type']}")
        y -= 30

        # Таблица
        c.setFont('DejaVuSans-Bold', 8)
        headers = ["№", "Наименование", "ГОСТ/ТУ", "Кол-во", "Ед.", "Документ", "Номер док."]
        positions = [30, 50, 200, 300, 350, 390, 470]

        for i, header in enumerate(headers):
            c.drawString(positions[i], y, header)
        y -= 12

        c.line(30, y, width - 30, y)
        y -= 10

        # Данные
        c.setFont('DejaVuSans', 7)
        for i, item in enumerate(registry['items'], 1):
            if y < 50:
                c.showPage()
                c.setFont('DejaVuSans', 7)
                y = height - 50

            c.drawString(positions[0], y, str(i))
            c.drawString(positions[1], y, item['material_name'][:25])
            c.drawString(positions[2], y, item.get('gost', '')[:15])
            c.drawString(positions[3], y, str(item['quantity']))
            c.drawString(positions[4], y, item['unit'])

            if item['quality_document']:
                c.drawString(positions[5], y, item['quality_document']['type'][:10])
                c.drawString(positions[6], y, item['quality_document']['number'][:15])
            else:
                c.drawString(positions[5], y, "-")

            y -= 10

        c.save()
        return pdf_path
```

#### Итоговый результат:
```
РЕЕСТР ПРИМЕНЁННЫХ МАТЕРИАЛОВ
к АОСР №1: Монтаж трубопроводов системы отопления

┌────┬──────────────────────┬─────────────────┬────────┬──────┬────────────┬─────────────────┐
│ №  │ Наименование         │ ГОСТ/ТУ         │ Кол-во │ Ед.  │ Документ   │ Номер документа │
├────┼──────────────────────┼─────────────────┼────────┼──────┼────────────┼─────────────────┤
│ 1  │ Труба ПНД            │ ГОСТ 18599-2001 │ 175    │ м    │ Сертификат │ С-РФ.АЯ46.В.... │
├────┼──────────────────────┼─────────────────┼────────┼──────┼────────────┼─────────────────┤
│ 2  │ Изоляция трубная     │ ГОСТ 30732-2006 │ 175    │ м    │ Сертификат │ РОСС RU.СП15... │
├────┼──────────────────────┼─────────────────┼────────┼──────┼────────────┼─────────────────┤
│ 3  │ Фитинги              │ -               │ 25     │ шт   │ Паспорт    │ -               │
└────┴──────────────────────┴─────────────────┴────────┴──────┴────────────┴─────────────────┘

Всего материалов: 3
Документов качества: 3 (100%)

[Кнопка: Скачать реестр PDF]
[Кнопка: Экспорт в Excel]
```

---

## 🎯 Итоговая сводка всех функций

| № | Функция | Сложность | Время выполнения | Основной агент |
|---|---------|-----------|------------------|----------------|
| 1 | Регистрация и вход | Низкая | <1 сек | - |
| 2 | Создание проекта | Низкая | <1 сек | - |
| 3 | Загрузка РД | Средняя | 30-60 сек | Агент анализа РД + GPT-4o |
| 4 | Загрузка документов качества | Средняя | 5-10 сек/файл | OCR агент + GPT-4o Vision |
| 5 | Загрузка схемы | Низкая | <5 сек | - |
| 6 | Генерация АОСР | Средняя | 10-20 сек | Агент генерации АОСР |
| 7 | Редактирование АОСР | Низкая | 10-15 сек | - |
| 8 | Поиск документов качества | **Высокая** | 2-5 мин/материал | Агент поиска + Playwright |
| 9 | Проверка комплектности | **Высокая** | 5-10 сек | Агент валидации + GPT-4o |
| 10 | Формирование комплекта ИД | Средняя | 10-30 сек | - |
| 11 | Просмотр истории | Низкая | <1 сек | - |
| 12 | Управление реестрами | Низкая | <1 сек | - |

---

## 📌 Ключевые выводы

1. **Самые сложные функции:**
   - Поиск документов качества (web scraping + LLM проверка релевантности)
   - Проверка комплектности (многоуровневая валидация)
   - Анализ проектной документации (обработка больших PDF)

2. **Где используется ИИ:**
   - Анализ РД → GPT-4o
   - OCR документов → GPT-4o Vision
   - Поиск документов → GPT-4o (проверка релевантности)
   - Валидация → GPT-4o (логическая согласованность)

3. **Узкие места производительности:**
   - Поиск документов в интернете (зависит от внешних сайтов)
   - OCR больших PDF файлов
   - Формирование комплекта из >100 файлов

4. **Оптимизации:**
   - Всё тяжёлое выносится в Celery (фоновые задачи)
   - Параллельный поиск документов для разных материалов
   - Кэширование результатов LLM
   - Потоковая обработка больших PDF

---

**Статус:** ✅ Полный разбор всех функций завершён
**Последнее обновление:** 2025-12-14