# Облачная инфраструктура

**Дата создания:** 2025-12-13
**Приоритет:** ⭐⭐⭐⭐ (Высокий)

---

## Назначение

Облачная инфраструктура для:
- Хостинг backend и frontend
- Хранение файлов (PDF, изображения)
- Базы данных
- Очереди задач (Redis)
- Масштабирование под нагрузкой

---

## Рекомендуемая связка

### Yandex Cloud (российский провайдер)

```
┌─────────────────────────────────────────────────────┐
│            Cloud Infrastructure                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [Compute]                                          │
│  └─ Yandex Compute Cloud (VMs)                      │
│     └─ 2-4 vCPU, 8-16GB RAM                         │
│                                                     │
│  [Database]                                         │
│  └─ Yandex Managed PostgreSQL                       │
│                                                     │
│  [Storage]                                          │
│  └─ Yandex Object Storage (S3-compatible)           │
│                                                     │
│  [Cache/Queue]                                      │
│  └─ Yandex Managed Redis                            │
│                                                     │
│  [Load Balancer]                                    │
│  └─ Yandex Application Load Balancer                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Конкретные сервисы

### 1. Compute (Виртуальные машины)

**Yandex Compute Cloud:**

**Конфигурация для MVP:**
- **Backend сервер:** 2 vCPU, 8GB RAM, 50GB SSD
- **Стоимость:** ~5000 руб/месяц

**Установка:**
```bash
# Создание VM через Yandex Cloud CLI
yc compute instance create \
  --name pto-backend \
  --zone ru-central1-a \
  --cores 2 \
  --memory 8 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2204-lts,size=50 \
  --network-interface subnet-name=default,nat-ip-version=ipv4
```

---

### 2. Object Storage (Файловое хранилище)

**Yandex Object Storage (S3-compatible):**

**Для чего:**
- Загруженные документы
- Сгенерированные PDF
- Сканы документов качества

**Стоимость:**
- Хранение: 1.53 руб/ГБ/месяц
- Передача данных: 1.53 руб/ГБ

**Python SDK:**
```bash
pip install boto3
```

**Пример использования:**
```python
import boto3

# Настройка клиента
s3 = boto3.client(
    's3',
    endpoint_url='https://storage.yandexcloud.net',
    aws_access_key_id='YOUR_ACCESS_KEY',
    aws_secret_access_key='YOUR_SECRET_KEY'
)

# Загрузка файла
s3.upload_file(
    'aosr_123.pdf',
    'pto-platform-documents',
    'projects/1/aosr/aosr_123.pdf'
)

# Скачивание файла
s3.download_file(
    'pto-platform-documents',
    'projects/1/aosr/aosr_123.pdf',
    'local_aosr_123.pdf'
)

# Получение ссылки (expires in 3600 sec)
url = s3.generate_presigned_url(
    'get_object',
    Params={
        'Bucket': 'pto-platform-documents',
        'Key': 'projects/1/aosr/aosr_123.pdf'
    },
    ExpiresIn=3600
)
```

---

### 3. Database (Managed PostgreSQL)

**Yandex Managed Service for PostgreSQL:**

**Конфигурация:**
- PostgreSQL 15
- 2 vCPU, 8GB RAM
- 50GB SSD
- **Стоимость:** ~6000 руб/месяц

**Преимущества:**
- ✅ Автоматические бэкапы
- ✅ Мониторинг
- ✅ Высокая доступность

---

### 4. Redis (Кэш + очереди Celery)

**Yandex Managed Service for Redis:**

**Конфигурация:**
- Redis 7.0
- 2GB RAM
- **Стоимость:** ~2000 руб/месяц

---

## Деплой приложения

### Docker + Docker Compose

**Dockerfile (backend):**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Установка зависимостей системы
RUN apt-get update && apt-get install -y \
    tesseract-ocr \
    tesseract-ocr-rus \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Установка Python зависимостей
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копирование кода
COPY . .

# Запуск
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/pto_db
      - REDIS_URL=redis://redis:6379/0
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - db
      - redis

  celery_worker:
    build: ./backend
    command: celery -A celery_app worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/pto_db
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis
      - db

  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend

  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=pto_db
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

## CI/CD (GitHub Actions)

**`.github/workflows/deploy.yml`:**
```yaml
name: Deploy to Yandex Cloud

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Copy files to server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "."
          target: "/opt/pto-platform"

      - name: Deploy with Docker Compose
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/pto-platform
            docker-compose down
            docker-compose pull
            docker-compose up -d --build
```

---

## Альтернативные провайдеры

### AWS (Amazon Web Services)

**Когда использовать:**
- Если нужна максимальная надежность
- Если бюджет позволяет (дороже)

**Стоимость (примерно):**
- EC2 t3.medium: $30/месяц
- RDS PostgreSQL: $40/месяц
- S3 Storage: $0.023/GB

---

### VK Cloud (бывший Mail.ru Cloud)

**Российский провайдер:**
- Цены сопоставимы с Yandex Cloud
- Меньше сервисов

---

## Оценка стоимости (MVP)

| Сервис | Конфигурация | Стоимость/мес |
|--------|-------------|---------------|
| Compute (Backend) | 2 vCPU, 8GB RAM | 5000 руб |
| PostgreSQL | 2 vCPU, 8GB RAM, 50GB | 6000 руб |
| Redis | 2GB RAM | 2000 руб |
| Object Storage | 100GB | 150 руб |
| Трафик | ~500GB | 750 руб |
| **ИТОГО** | | **~14 000 руб/мес** |

---

## Мониторинг и логи

### Sentry (для ошибок)

```bash
pip install sentry-sdk
```

```python
import sentry_sdk

sentry_sdk.init(
    dsn="https://...@sentry.io/...",
    traces_sample_rate=1.0,
)
```

---

## Связь с другими компонентами

- **[Backend](03-backend.md):** Деплой на сервер
- **[Database](08-database.md):** Managed PostgreSQL
- **[Documents](05-documents.md):** Object Storage для PDF

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-13
