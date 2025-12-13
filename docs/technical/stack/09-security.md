# Безопасность

**Дата создания:** 2025-12-13
**Приоритет:** ⭐⭐⭐⭐⭐ (Критический)

---

## Назначение

Обеспечение безопасности платформы:
- Аутентификация и авторизация
- Защита API
- Безопасное хранение данных
- Защита от основных уязвимостей

---

## Основные компоненты

### 1. Аутентификация (JWT)

**Установка:**
```bash
pip install python-jose[cryptography] passlib[bcrypt]
```

**Реализация:**
```python
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

SECRET_KEY = "your-secret-key-here-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60 * 24 * 7  # 7 дней

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def verify_token(token: str):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        return None
```

---

### 2. Защита API endpoints

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    payload = verify_token(token)
    if payload is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return payload

# Использование в endpoint
@app.get("/projects", dependencies=[Depends(get_current_user)])
def get_projects():
    return []
```

---

### 3. CORS (Cross-Origin Resource Sharing)

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://pto-platform.ru"],  # Только ваш домен
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

### 4. Rate Limiting (защита от DDoS)

```bash
pip install slowapi
```

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/generate-aosr")
@limiter.limit("10/minute")  # Максимум 10 запросов в минуту
async def generate_aosr(request: Request):
    pass
```

---

### 5. Валидация загружаемых файлов

```python
from fastapi import UploadFile, HTTPException

ALLOWED_EXTENSIONS = {'.pdf', '.jpg', '.jpeg', '.png', '.docx', '.xlsx'}
MAX_FILE_SIZE = 50 * 1024 * 1024  # 50MB

async def validate_file(file: UploadFile):
    # Проверка расширения
    file_ext = Path(file.filename).suffix.lower()
    if file_ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(400, f"File type {file_ext} not allowed")

    # Проверка размера
    content = await file.read()
    await file.seek(0)  # Reset для последующего чтения

    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(400, f"File too large (max {MAX_FILE_SIZE} bytes)")

    # Проверка MIME type
    import magic
    mime_type = magic.from_buffer(content, mime=True)

    allowed_mimes = {
        'application/pdf',
        'image/jpeg',
        'image/png',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
    }

    if mime_type not in allowed_mimes:
        raise HTTPException(400, f"Invalid file type: {mime_type}")

    return True
```

---

### 6. SQL Injection Protection

**✅ Используйте ORM (SQLAlchemy)** — автоматическая защита

**❌ Никогда так:**
```python
# ПЛОХО - уязвимо к SQL injection
query = f"SELECT * FROM users WHERE email = '{email}'"
```

**✅ Всегда так:**
```python
# ХОРОШО - параметризованный запрос
session.query(User).filter(User.email == email).first()
```

---

### 7. XSS Protection

**Backend:**
- Не возвращайте raw HTML в API
- Всегда используйте JSON

**Frontend:**
- React автоматически экранирует (escapes) данные
- Избегайте `dangerouslySetInnerHTML`

---

### 8. Безопасное хранение секретов

**Environment Variables (.env):**
```bash
# .env
DATABASE_URL=postgresql://user:password@localhost/db
OPENAI_API_KEY=sk-...
SECRET_KEY=your-secret-key-here
REDIS_URL=redis://localhost:6379/0
```

**Никогда не коммитьте .env в Git:**
```bash
# .gitignore
.env
.env.local
*.env
```

---

### 9. HTTPS (SSL/TLS)

**Обязательно используйте HTTPS в продакшене**

**С Nginx:**
```nginx
server {
    listen 443 ssl;
    server_name pto-platform.ru;

    ssl_certificate /etc/letsencrypt/live/pto-platform.ru/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pto-platform.ru/privkey.pem;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Получение бесплатного SSL (Let's Encrypt):**
```bash
sudo certbot --nginx -d pto-platform.ru
```

---

### 10. Безопасное логирование

```python
import logging

logger = logging.getLogger(__name__)

# ❌ ПЛОХО - пароли в логах
logger.info(f"User login: {email}, password: {password}")

# ✅ ХОРОШО - без чувствительных данных
logger.info(f"User login attempt: {email}")
```

---

## Checklist безопасности

### Обязательно перед продакшеном

- [ ] JWT токены с коротким временем жизни
- [ ] HTTPS everywhere
- [ ] Rate limiting на критичных endpoints
- [ ] Валидация всех входных данных
- [ ] CORS настроен правильно (только нужные домены)
- [ ] Секреты в environment variables
- [ ] Логирование (но без паролей/токенов)
- [ ] Регулярные бэкапы БД
- [ ] Мониторинг безопасности (Sentry)
- [ ] Обновление зависимостей (проверка на уязвимости)

---

## Проверка уязвимостей

### Safety (проверка Python зависимостей)

```bash
pip install safety
safety check
```

### Bandit (статический анализ кода)

```bash
pip install bandit
bandit -r ./app
```

---

## Связь с другими компонентами

- **[Backend](03-backend.md):** JWT middleware
- **[Frontend](04-frontend.md):** Token storage
- **[Database](08-database.md):** Хранение паролей (bcrypt)

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-13
