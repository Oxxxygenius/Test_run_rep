# Архитектура безопасности платформы

**Дата создания:** 2025-12-14
**Для кого:** Для разработчиков, DevOps инженеров и специалистов по информационной безопасности

---

## 🎯 Главная идея

Платформа работает с **конфиденциальными данными**:
- Проектная документация (коммерческая тайна)
- Персональные данные пользователей
- Сертификаты и паспорта качества

**Требования:**
- ✅ Данные должны быть защищены от утечек
- ✅ Доступ только у авторизованных пользователей
- ✅ Все действия логируются
- ✅ Соответствие требованиям GDPR и 152-ФЗ (о персональных данных)

---

## 🔐 Уровни безопасности

```
┌────────────────────────────────────────────────────┐
│              УРОВЕНЬ 1: Сеть                        │
│  - HTTPS/TLS шифрование                             │
│  - Firewall                                         │
│  - DDoS защита                                      │
└───────────────────┬────────────────────────────────┘
                    ▼
┌────────────────────────────────────────────────────┐
│         УРОВЕНЬ 2: Аутентификация                   │
│  - JWT токены                                       │
│  - Хеширование паролей (bcrypt)                     │
│  - 2FA (опционально)                                │
└───────────────────┬────────────────────────────────┘
                    ▼
┌────────────────────────────────────────────────────┐
│         УРОВЕНЬ 3: Авторизация                      │
│  - RBAC (Role-Based Access Control)                 │
│  - Проверка прав доступа к проектам                 │
└───────────────────┬────────────────────────────────┘
                    ▼
┌────────────────────────────────────────────────────┐
│         УРОВЕНЬ 4: Данные                           │
│  - Шифрование данных в БД                           │
│  - Приватные файлы в Object Storage                 │
│  - SQL Injection защита                             │
└───────────────────┬────────────────────────────────┘
                    ▼
┌────────────────────────────────────────────────────┐
│         УРОВЕНЬ 5: Аудит                            │
│  - Логирование всех действий                        │
│  - Мониторинг подозрительной активности             │
│  - Backup данных                                    │
└────────────────────────────────────────────────────┘
```

---

## 🌐 Уровень 1: Сетевая безопасность

### 1.1. HTTPS/TLS шифрование

**Обязательно:** Все данные передаются только по HTTPS

```nginx
# nginx.conf
server {
    listen 443 ssl http2;
    server_name pto-platform.com;

    # SSL сертификат (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/pto-platform.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pto-platform.com/privkey.pem;

    # Современные протоколы
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # HSTS (заставляет браузер всегда использовать HTTPS)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://backend:8000;
    }
}

# Редирект HTTP → HTTPS
server {
    listen 80;
    server_name pto-platform.com;
    return 301 https://$server_name$request_uri;
}
```

---

### 1.2. Firewall (межсетевой экран)

**Правила:**
```bash
# Разрешить только нужные порты
sudo ufw allow 22/tcp      # SSH (только для администраторов)
sudo ufw allow 80/tcp      # HTTP (для редиректа на HTTPS)
sudo ufw allow 443/tcp     # HTTPS

# Закрыть доступ к БД извне
sudo ufw deny 5432/tcp     # PostgreSQL (доступ только внутри сети)

# Закрыть доступ к Redis
sudo ufw deny 6379/tcp     # Redis

# Включить firewall
sudo ufw enable
```

---

### 1.3. Rate Limiting (защита от DDoS)

**Ограничение количества запросов:**

```python
# FastAPI middleware
from fastapi import Request, HTTPException
from collections import defaultdict
import time

# Хранилище запросов (в продакшене использовать Redis)
request_counts = defaultdict(list)

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    client_ip = request.client.host
    now = time.time()

    # Удаляем старые запросы (старше 1 минуты)
    request_counts[client_ip] = [
        req_time for req_time in request_counts[client_ip]
        if now - req_time < 60
    ]

    # Проверяем лимит (макс 100 запросов в минуту)
    if len(request_counts[client_ip]) >= 100:
        raise HTTPException(status_code=429, detail="Too many requests")

    # Добавляем текущий запрос
    request_counts[client_ip].append(now)

    response = await call_next(request)
    return response
```

**Альтернатива (Nginx):**
```nginx
# Ограничение на уровне Nginx
http {
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;

    server {
        location /api/ {
            limit_req zone=api_limit burst=20;
            proxy_pass http://backend:8000;
        }
    }
}
```

---

## 🔑 Уровень 2: Аутентификация

### 2.1. Хеширование паролей (bcrypt)

**Никогда не храним пароли в открытом виде!**

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# При регистрации
def register_user(email, password):
    # Хешируем пароль
    password_hash = pwd_context.hash(password)

    # Сохраняем в БД
    user = User(
        email=email,
        password_hash=password_hash  # НЕ password!
    )
    db.add(user)
    db.commit()

# При входе
def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)
```

**Результат:**
```
Пароль: SecurePass123!
Hash: $2b$12$KIXQz7.5H5xK8hH6.gZvLOqT3Yz1K...

Даже если кто-то украдёт БД → пароли невозможно восстановить
```

---

### 2.2. JWT токены

**Как работает:**
```
Пользователь вводит email + password
    ↓
Backend проверяет пароль
    ↓
Если OK → создаёт JWT токен
    ↓
Пользователь хранит токен в браузере (localStorage)
    ↓
При каждом запросе отправляет токен в заголовке:
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
    ↓
Backend проверяет токен → разрешает доступ
```

**Код:**
```python
from datetime import datetime, timedelta
from jose import JWTError, jwt

SECRET_KEY = "your-secret-key-keep-it-safe"  # ВАЖНО: хранить в .env
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def verify_token(token: str):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return user_id
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

**Защита от кражи токенов:**
```python
# 1. Короткий срок жизни (30 минут)
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# 2. Refresh токен (для продления сессии)
REFRESH_TOKEN_EXPIRE_DAYS = 7

# 3. Хранить токен в httpOnly cookie (защита от XSS)
response.set_cookie(
    key="access_token",
    value=token,
    httponly=True,  # JavaScript не может прочитать
    secure=True,    # Только HTTPS
    samesite="lax"  # Защита от CSRF
)
```

---

### 2.3. Двухфакторная аутентификация (2FA)

**Опциональная защита для критичных операций:**

```python
import pyotp

# Генерация секретного ключа (при включении 2FA)
def enable_2fa(user_id):
    secret = pyotp.random_base32()
    user.totp_secret = secret
    db.commit()

    # QR-код для Google Authenticator
    totp_uri = pyotp.totp.TOTP(secret).provisioning_uri(
        name=user.email,
        issuer_name="PTO Platform"
    )
    return totp_uri

# Проверка кода при входе
def verify_2fa_code(user, code):
    totp = pyotp.TOTP(user.totp_secret)
    return totp.verify(code, valid_window=1)
```

---

## 🛡️ Уровень 3: Авторизация (доступ к данным)

### 3.1. RBAC (Role-Based Access Control)

**Роли пользователей:**

| Роль | Права |
|------|-------|
| `admin` | Полный доступ ко всей платформе |
| `manager` | Видит все проекты своей компании |
| `user` | Видит только свои проекты |
| `guest` | Только чтение (без редактирования) |

**Код:**
```python
from enum import Enum

class UserRole(str, Enum):
    ADMIN = "admin"
    MANAGER = "manager"
    USER = "user"
    GUEST = "guest"

# Декоратор для проверки роли
def require_role(required_role: UserRole):
    def decorator(func):
        async def wrapper(current_user: User, *args, **kwargs):
            if current_user.role != required_role and current_user.role != UserRole.ADMIN:
                raise HTTPException(status_code=403, detail="Insufficient permissions")
            return await func(current_user, *args, **kwargs)
        return wrapper
    return decorator

# Использование
@app.delete("/api/v1/projects/{project_id}")
@require_role(UserRole.MANAGER)
def delete_project(project_id: int, current_user: User = Depends(get_current_user)):
    # Только менеджеры и админы могут удалять проекты
    ...
```

---

### 3.2. Проверка доступа к проектам

**Правило:** Пользователь видит только свои проекты

```python
@app.get("/api/v1/projects/{project_id}")
def get_project(project_id: int, current_user: User = Depends(get_current_user)):
    project = db.query(Project).filter(Project.id == project_id).first()

    if not project:
        raise HTTPException(status_code=404, detail="Project not found")

    # ВАЖНО: Проверяем, что проект принадлежит пользователю
    if project.user_id != current_user.id and current_user.role != UserRole.ADMIN:
        raise HTTPException(status_code=403, detail="Access denied")

    return project
```

---

## 🗄️ Уровень 4: Безопасность данных

### 4.1. Защита от SQL Injection

**Плохо (уязвимо):**
```python
# НИКОГДА ТАК НЕ ДЕЛАЙТЕ!
def search_documents(query: str):
    sql = f"SELECT * FROM documents WHERE filename LIKE '%{query}%'"
    result = db.execute(sql)
    # Если query = "'; DROP TABLE documents; --"
    # → SQL Injection атака!
```

**Хорошо (безопасно):**
```python
# Используем параметризованные запросы
def search_documents(query: str):
    result = db.query(Document).filter(
        Document.filename.ilike(f"%{query}%")
    ).all()
    # SQLAlchemy автоматически экранирует query
```

---

### 4.2. Шифрование чувствительных данных

**Шифрование в БД (для особо важных полей):**

```python
from cryptography.fernet import Fernet

# Генерация ключа (хранить в .env!)
ENCRYPTION_KEY = Fernet.generate_key()
cipher = Fernet(ENCRYPTION_KEY)

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String)
    phone_encrypted = Column(LargeBinary)  # Зашифрованный телефон

    @property
    def phone(self):
        if self.phone_encrypted:
            return cipher.decrypt(self.phone_encrypted).decode()
        return None

    @phone.setter
    def phone(self, value):
        if value:
            self.phone_encrypted = cipher.encrypt(value.encode())
```

---

### 4.3. Безопасное хранение файлов

**Файлы в Object Storage должны быть приватными:**

```python
import boto3

s3_client = boto3.client(
    's3',
    endpoint_url='https://storage.yandexcloud.net',
    aws_access_key_id=AWS_ACCESS_KEY,
    aws_secret_access_key=AWS_SECRET_KEY
)

# Загружаем файл (приватный доступ)
s3_client.upload_file(
    'document.pdf',
    'pto-platform-private',  # Приватный bucket
    'projects/123/document.pdf',
    ExtraArgs={'ACL': 'private'}  # НЕ публичный!
)

# Генерация временной ссылки (срок действия 1 час)
def generate_presigned_url(file_path, expiration=3600):
    url = s3_client.generate_presigned_url(
        'get_object',
        Params={'Bucket': 'pto-platform-private', 'Key': file_path},
        ExpiresIn=expiration
    )
    return url

# Пользователь получает временную ссылку
@app.get("/api/v1/documents/{document_id}/download")
def download_document(document_id: int, current_user: User = Depends(get_current_user)):
    doc = db.query(Document).filter(Document.id == document_id).first()

    # Проверяем права доступа
    if doc.project.user_id != current_user.id:
        raise HTTPException(status_code=403)

    # Генерируем временную ссылку (действует 1 час)
    download_url = generate_presigned_url(doc.file_path, expiration=3600)
    return {"url": download_url}
```

---

### 4.4. Валидация загружаемых файлов

**Защита от загрузки вредоносных файлов:**

```python
from fastapi import UploadFile
import magic  # python-magic

ALLOWED_MIME_TYPES = [
    'application/pdf',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document',  # .docx
    'image/jpeg',
    'image/png'
]

MAX_FILE_SIZE = 50 * 1024 * 1024  # 50 MB

@app.post("/api/v1/documents/upload")
async def upload_document(file: UploadFile):
    # 1. Проверка размера
    file_size = 0
    content = await file.read()
    file_size = len(content)

    if file_size > MAX_FILE_SIZE:
        raise HTTPException(status_code=400, detail="File too large")

    # 2. Проверка MIME типа (по содержимому, а не расширению!)
    mime = magic.from_buffer(content, mime=True)
    if mime not in ALLOWED_MIME_TYPES:
        raise HTTPException(status_code=400, detail=f"File type {mime} not allowed")

    # 3. Сканирование на вирусы (опционально, через ClamAV)
    if contains_virus(content):
        raise HTTPException(status_code=400, detail="Malware detected")

    # 4. Генерация безопасного имени файла
    import uuid
    safe_filename = f"{uuid.uuid4()}.pdf"

    # Загружаем файл
    ...
```

---

## 📝 Уровень 5: Аудит и логирование

### 5.1. Логирование всех действий

**Что логировать:**
- ✅ Вход/выход пользователя
- ✅ Создание/удаление проектов
- ✅ Загрузка/скачивание документов
- ✅ Изменение настроек
- ✅ Ошибки и исключения

**Код:**
```python
import logging
from datetime import datetime

# Настройка логгера
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[
        logging.FileHandler("app.log"),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

# Middleware для логирования всех запросов
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = datetime.now()

    # Обрабатываем запрос
    response = await call_next(request)

    # Логируем
    duration = (datetime.now() - start_time).total_seconds()
    logger.info(
        f"{request.method} {request.url.path} - "
        f"Status: {response.status_code} - "
        f"Duration: {duration:.3f}s - "
        f"User: {getattr(request.state, 'user_id', 'anonymous')}"
    )

    return response

# Логирование критичных действий
def log_security_event(event_type: str, user_id: int, details: dict):
    logger.warning(
        f"SECURITY EVENT: {event_type} | "
        f"User: {user_id} | "
        f"Details: {details}"
    )

# Пример использования
@app.delete("/api/v1/projects/{project_id}")
def delete_project(project_id: int, current_user: User = Depends(get_current_user)):
    log_security_event(
        event_type="PROJECT_DELETED",
        user_id=current_user.id,
        details={"project_id": project_id}
    )
    # Удаляем проект
    ...
```

---

### 5.2. Мониторинг подозрительной активности

**Автоматическое обнаружение аномалий:**

```python
# Пример: Обнаружение брутфорса (подбора паролей)
from collections import defaultdict

failed_login_attempts = defaultdict(int)

@app.post("/api/v1/auth/login")
def login(credentials: LoginCredentials):
    user = db.query(User).filter(User.email == credentials.email).first()

    if not user or not verify_password(credentials.password, user.password_hash):
        # Увеличиваем счётчик неудачных попыток
        failed_login_attempts[credentials.email] += 1

        # Если > 5 попыток за 5 минут → блокируем
        if failed_login_attempts[credentials.email] >= 5:
            log_security_event(
                event_type="BRUTE_FORCE_DETECTED",
                user_id=None,
                details={"email": credentials.email}
            )
            raise HTTPException(
                status_code=429,
                detail="Too many failed attempts. Account locked for 30 minutes."
            )

        raise HTTPException(status_code=401, detail="Invalid credentials")

    # Успешный вход → сбрасываем счётчик
    failed_login_attempts[credentials.email] = 0
    ...
```

---

### 5.3. Backup и восстановление

**Ежедневный backup базы данных:**

```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_DIR="/backups"
DB_NAME="pto_platform"

# Создаём backup
pg_dump -U postgres $DB_NAME | gzip > $BACKUP_DIR/backup_$DATE.sql.gz

# Загружаем в облачное хранилище
aws s3 cp $BACKUP_DIR/backup_$DATE.sql.gz s3://pto-backups/

# Удаляем старые backups (старше 30 дней)
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +30 -delete

echo "Backup completed: backup_$DATE.sql.gz"
```

**Автоматизация (cron):**
```bash
# Запуск каждый день в 3:00 ночи
0 3 * * * /scripts/backup.sh
```

---

## 🚨 Checklist безопасности

Перед запуском в продакшн:

- [ ] **Сеть:**
  - [ ] HTTPS включен (с валидным SSL сертификатом)
  - [ ] Firewall настроен (закрыты лишние порты)
  - [ ] Rate limiting включен

- [ ] **Аутентификация:**
  - [ ] Пароли хешируются (bcrypt)
  - [ ] JWT токены с коротким сроком действия
  - [ ] SECRET_KEY хранится в .env (не в коде!)

- [ ] **Авторизация:**
  - [ ] RBAC настроен
  - [ ] Проверка прав доступа к проектам работает

- [ ] **Данные:**
  - [ ] SQL Injection защита (параметризованные запросы)
  - [ ] Файлы в Object Storage приватные
  - [ ] Валидация загружаемых файлов

- [ ] **Аудит:**
  - [ ] Логирование всех действий
  - [ ] Мониторинг подозрительной активности
  - [ ] Автоматический backup БД

- [ ] **Обновления:**
  - [ ] Регулярные обновления зависимостей
  - [ ] Сканирование на уязвимости (Dependabot, Snyk)

---

## 🔗 Связанные документы

- [04-scaling-strategy.md](04-scaling-strategy.md) — Безопасное масштабирование
- [06-user-actions-breakdown.md](06-user-actions-breakdown.md) — Детали реализации аутентификации

---

**Статус:** ✅ Актуально
**Последнее обновление:** 2025-12-14
