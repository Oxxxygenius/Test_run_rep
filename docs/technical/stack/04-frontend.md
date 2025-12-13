# Frontend (Интерфейс пользователя)

**Дата создания:** 2025-12-13
**Приоритет:** ⭐⭐⭐⭐ (Высокий)

---

## Назначение

Frontend — это то, что видит и с чем работает пользователь:
- Дашборд проектов
- Загрузка документов
- Генерация актов
- Просмотр комплектов ИД
- Мониторинг задач

---

## Рекомендуемая связка

### React + TypeScript + Vite + TanStack Query

```
┌─────────────────────────────────────────────────────┐
│                  Frontend Stack                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [Framework]                                        │
│  └─ React 18 + TypeScript                           │
│                                                     │
│  [Build Tool]                                       │
│  └─ Vite (быстрая сборка)                           │
│                                                     │
│  [State Management]                                 │
│  └─ TanStack Query (React Query)                    │
│                                                     │
│  [UI Components]                                    │
│  └─ shadcn/ui + Tailwind CSS                        │
│                                                     │
│  [Routing]                                          │
│  └─ React Router v6                                 │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Конкретные инструменты

### 1. React + TypeScript (UI Framework)

**Установка (через Vite):**
```bash
npm create vite@latest pto-platform-frontend -- --template react-ts
cd pto-platform-frontend
npm install
```

**Пример компонента:**
```typescript
// src/components/ProjectList.tsx
import { useQuery } from '@tanstack/react-query';
import { fetchProjects } from '@/api/projects';

interface Project {
  id: number;
  name: string;
  address: string;
  status: string;
}

export function ProjectList() {
  const { data: projects, isLoading, error } = useQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects
  });

  if (isLoading) return <div>Загрузка...</div>;
  if (error) return <div>Ошибка загрузки проектов</div>;

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      {projects?.map((project: Project) => (
        <div key={project.id} className="border rounded-lg p-4">
          <h3 className="font-bold">{project.name}</h3>
          <p className="text-sm text-gray-600">{project.address}</p>
          <span className="badge">{project.status}</span>
        </div>
      ))}
    </div>
  );
}
```

---

### 2. TanStack Query (React Query) — Управление серверным состоянием

**Почему TanStack Query:**
- ✅ Автоматический кэш
- ✅ Автоматическая перезагрузка данных
- ✅ Оптимистичные обновления
- ✅ Легкая работа с async операциями

**Установка:**
```bash
npm install @tanstack/react-query
```

**Настройка:**
```typescript
// src/main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 минут
      retry: 1,
    },
  },
});

root.render(
  <QueryClientProvider client={queryClient}>
    <App />
  </QueryClientProvider>
);
```

**Пример mutation:**
```typescript
// src/hooks/useGenerateAOSR.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { generateAOSR } from '@/api/aosr';

export function useGenerateAOSR() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: generateAOSR,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['aosr'] });
    },
  });
}

// Использование в компоненте
function AOSRGenerator() {
  const { mutate, isPending } = useGenerateAOSR();

  const handleGenerate = () => {
    mutate({ projectId: 1, workType: 'Монтаж труб' });
  };

  return (
    <button onClick={handleGenerate} disabled={isPending}>
      {isPending ? 'Генерируем...' : 'Сгенерировать АОСР'}
    </button>
  );
}
```

---

### 3. shadcn/ui + Tailwind CSS (UI компоненты)

**Что это:**
- shadcn/ui — коллекция готовых компонентов (не библиотека, а копируемые компоненты)
- Tailwind CSS — utility-first CSS фреймворк

**Установка:**
```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# Установка shadcn/ui
npx shadcn-ui@latest init
```

**Добавление компонентов:**
```bash
npx shadcn-ui@latest add button
npx shadcn-ui@latest add card
npx shadcn-ui@latest add table
npx shadcn-ui@latest add dialog
npx shadcn-ui@latest add form
```

**Пример использования:**
```typescript
import { Button } from "@/components/ui/button";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";

export function Dashboard() {
  return (
    <div className="container mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Дашборд проектов</h1>

      <Card>
        <CardHeader>
          <CardTitle>Статистика</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="grid grid-cols-3 gap-4">
            <div>
              <p className="text-sm text-gray-500">Всего проектов</p>
              <p className="text-2xl font-bold">24</p>
            </div>
            <div>
              <p className="text-sm text-gray-500">В работе</p>
              <p className="text-2xl font-bold">12</p>
            </div>
            <div>
              <p className="text-sm text-gray-500">Завершено</p>
              <p className="text-2xl font-bold">12</p>
            </div>
          </div>
        </CardContent>
      </Card>

      <Button className="mt-4">Создать новый проект</Button>
    </div>
  );
}
```

**Официальный сайт:** https://ui.shadcn.com/

---

### 4. React Router (Навигация)

**Установка:**
```bash
npm install react-router-dom
```

**Пример роутинга:**
```typescript
// src/App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Dashboard } from './pages/Dashboard';
import { ProjectPage } from './pages/ProjectPage';
import { AOSRPage } from './pages/AOSRPage';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/projects/:id" element={<ProjectPage />} />
        <Route path="/projects/:id/aosr" element={<AOSRPage />} />
      </Routes>
    </BrowserRouter>
  );
}
```

---

## Структура проекта

```
frontend/
├── public/
│   └── favicon.ico
│
├── src/
│   ├── api/                    # API клиенты
│   │   ├── client.ts           # Axios instance
│   │   ├── projects.ts
│   │   ├── documents.ts
│   │   └── aosr.ts
│   │
│   ├── components/             # Переиспользуемые компоненты
│   │   ├── ui/                 # shadcn/ui компоненты
│   │   ├── ProjectCard.tsx
│   │   ├── DocumentUploader.tsx
│   │   └── AOSRGenerator.tsx
│   │
│   ├── pages/                  # Страницы
│   │   ├── Dashboard.tsx
│   │   ├── ProjectPage.tsx
│   │   ├── AOSRPage.tsx
│   │   └── DocumentsPage.tsx
│   │
│   ├── hooks/                  # Custom hooks
│   │   ├── useProjects.ts
│   │   ├── useDocuments.ts
│   │   └── useAOSR.ts
│   │
│   ├── types/                  # TypeScript типы
│   │   ├── project.ts
│   │   ├── document.ts
│   │   └── aosr.ts
│   │
│   ├── utils/                  # Утилиты
│   │   ├── formatDate.ts
│   │   └── downloadFile.ts
│   │
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
│
├── package.json
├── tsconfig.json
├── vite.config.ts
└── tailwind.config.js
```

---

## API клиент (Axios)

```typescript
// src/api/client.ts
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8000/api/v1',
  headers: {
    'Content-Type': 'application/json',
  },
});

// Interceptor для добавления токена
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// src/api/projects.ts
import { apiClient } from './client';

export async function fetchProjects() {
  const { data } = await apiClient.get('/projects');
  return data;
}

export async function createProject(project: CreateProjectDTO) {
  const { data } = await apiClient.post('/projects', project);
  return data;
}
```

---

## Загрузка файлов

```typescript
// src/components/DocumentUploader.tsx
import { useState } from 'react';
import { useMutation } from '@tanstack/react-query';
import { Button } from '@/components/ui/button';

export function DocumentUploader({ projectId }: { projectId: number }) {
  const [file, setFile] = useState<File | null>(null);

  const uploadMutation = useMutation({
    mutationFn: async (file: File) => {
      const formData = new FormData();
      formData.append('file', file);
      formData.append('project_id', String(projectId));

      const response = await fetch('/api/v1/documents/upload', {
        method: 'POST',
        body: formData,
      });
      return response.json();
    },
  });

  const handleUpload = () => {
    if (file) {
      uploadMutation.mutate(file);
    }
  };

  return (
    <div>
      <input
        type="file"
        accept=".pdf,.jpg,.png"
        onChange={(e) => setFile(e.target.files?.[0] || null)}
      />
      <Button onClick={handleUpload} disabled={!file || uploadMutation.isPending}>
        {uploadMutation.isPending ? 'Загрузка...' : 'Загрузить'}
      </Button>
    </div>
  );
}
```

---

## Альтернативные варианты

### Next.js (если нужен SSR)

**Когда использовать:**
- Если нужна SEO оптимизация
- Если нужен Server-Side Rendering

**Для нашего проекта:** НЕ нужен (это внутренняя B2B платформа)

---

### Vue.js + TypeScript

**Когда использовать:**
- Если команда знакома с Vue
- Для более простой кривой обучения

---

## Рекомендации

### ✅ Лучшие практики

1. **Используйте TypeScript для type safety**
2. **Разделяйте логику и UI (custom hooks)**
3. **Используйте React Query для API**
4. **Добавьте loading и error states**
5. **Оптимизируйте производительность (React.memo, useMemo)**

---

## Связь с другими компонентами

- **[Backend](03-backend.md):** REST API + WebSocket
- **[Security](09-security.md):** JWT авторизация

---

**Статус:** ✅ Готово к использованию
**Последнее обновление:** 2025-12-13
