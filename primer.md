# Пример приложения: «Мини-чат с авторизацией, Redux, WebSocket и SSR»

Ниже — сквозной пример, который покрывает весь заявленный стек. Приложение: **аутентификация → защищённый роут → список сообщений → WebSocket-чат → Redux → тесты**. Каждый блок снабжён пояснением.

---

## 1. Структура проекта

```
chat-app/
├── src/
│   ├── app/
│   │   ├── store.ts              # Redux store
│   │   ├── hooks.ts              # типизированные хуки
│   │   └── layout.tsx            # SSR-шаблон (Next.js)
│   ├── features/
│   │   ├── auth/
│   │   │   ├── authSlice.ts
│   │   │   ├── LoginForm.tsx
│   │   │   └── ProtectedRoute.tsx
│   │   └── chat/
│   │       ├── chatSlice.ts
│   │       ├── ChatRoom.tsx
│   │       └── useWebSocket.ts
│   ├── shared/
│   │   ├── types.ts
│   │   └── api.ts
│   └── pages/
│       ├── index.tsx             # SSR-страница
│       └── login.tsx
├── tests/
│   ├── ChatRoom.test.tsx
│   └── authSlice.test.ts
└── tsconfig.json
```

---

## 2. TypeScript: общие типы

**`src/shared/types.ts`**

```typescript
export interface User {
  id: number;
  name: string;
  role: "admin" | "user";
}

export interface Message {
  id: string;
  text: string;
  author: string;
  timestamp: number;
}

export interface AuthState {
  user: User | null;
  token: string | null;
  status: "idle" | "loading" | "authenticated" | "error";
}
```

---

## 3. Redux: срез аутентификации

**`src/features/auth/authSlice.ts`**

```typescript
import { createSlice, createAsyncThunk, PayloadAction } from "@reduxjs/toolkit";
import type { AuthState, User } from "@/shared/types";

const initialState: AuthState = { user: null, token: null, status: "idle" };

// Асинхронное действие (thunk)
export const login = createAsyncThunk(
  "auth/login",
  async ({ email, password }: { email: string; password: string }) => {
    const res = await fetch("/api/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password }),
    });
    if (!res.ok) throw new Error("Неверные учётные данные");
    return (await res.json()) as { user: User; token: string };
  }
);

const authSlice = createSlice({
  name: "auth",
  initialState,
  reducers: {
    logout(state) {
      state.user = null;
      state.token = null;
      state.status = "idle";
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(login.pending, (state) => { state.status = "loading"; })
      .addCase(login.fulfilled, (state, action: PayloadAction<{ user: User; token: string }>) => {
        state.user = action.payload.user;
        state.token = action.payload.token;
        state.status = "authenticated";
      })
      .addCase(login.rejected, (state) => { state.status = "error"; });
  },
});

export const { logout } = authSlice.actions;
export default authSlice.reducer;
```

**Что здесь важно:**
- `createAsyncThunk` автоматически генерирует три action: `pending`, `fulfilled`, `rejected`.
- `builder.addCase` — типизированный способ добавить обработчики.
- Мутации в reducer допустимы благодаря Immer внутри Redux Toolkit (он создаёт новый state за кулисами).

---

## 4. Redux store и типизированные хуки

**`src/app/store.ts`**

```typescript
import { configureStore } from "@reduxjs/toolkit";
import authReducer from "@/features/auth/authSlice";
import chatReducer from "@/features/chat/chatSlice";

export const store = configureStore({
  reducer: { auth: authReducer, chat: chatReducer },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

**`src/app/hooks.ts`**

```typescript
import { TypedUseSelectorHook, useDispatch, useSelector } from "react-redux";
import type { RootState, AppDispatch } from "./store";

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

**Зачем:** без типизированных хуков `useSelector` не знает форму state, а `dispatch` не подсказывает доступные thunk.

---

## 5. WebSocket: пользовательский хук

**`src/features/chat/useWebSocket.ts`**

```typescript
import { useEffect, useRef, useCallback } from "react";
import { useAppDispatch } from "@/app/hooks";
import { messageReceived } from "./chatSlice";
import type { Message } from "@/shared/types";

export function useWebSocket(url: string) {
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectAttempts = useRef(0);
  const dispatch = useAppDispatch();

  const connect = useCallback(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => { reconnectAttempts.current = 0; };

    ws.onmessage = (event) => {
      const msg: Message = JSON.parse(event.data);
      dispatch(messageReceived(msg));
    };

    ws.onclose = () => {
      const delay = Math.min(1000 * 2 ** reconnectAttempts.current, 30_000);
      reconnectAttempts.current++;
      setTimeout(connect, delay); // экспоненциальный backoff
    };

    ws.onerror = () => ws.close();
  }, [url, dispatch]);

  useEffect(() => {
    connect();
    return () => wsRef.current?.close();
  }, [connect]);

  const send = useCallback((msg: Omit<Message, "id" | "timestamp">) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(msg));
    }
  }, []);

  return { send };
}
```

**Ключевые решения:**
- **Cleanup** закрывает соединение при размонтировании.
- **Экспоненциальный backoff** при разрыве — не спамим сервер переподключениями.
- **`send` через `useCallback`** — стабильная ссылка, чтобы не пересоздавать зависимые компоненты.

---

## 6. Redux: срез чата

**`src/features/chat/chatSlice.ts`**

```typescript
import { createSlice, PayloadAction } from "@reduxjs/toolkit";
import type { Message } from "@/shared/types";

interface ChatState { messages: Message[] }

const chatSlice = createSlice({
  name: "chat",
  initialState: { messages: [] } as ChatState,
  reducers: {
    messageReceived(state, action: PayloadAction<Message>) {
      state.messages.push(action.payload);
    },
  },
});

export const { messageReceived } = chatSlice.actions;
export default chatSlice.reducer;
```

---

## 7. Компонент чата

**`src/features/chat/ChatRoom.tsx`**

```typescript
import { useState, useMemo, useCallback } from "react";
import { useAppSelector } from "@/app/hooks";
import { useWebSocket } from "./useWebSocket";

export function ChatRoom() {
  const [input, setInput] = useState("");
  const messages = useAppSelector((s) => s.chat.messages);
  const user = useAppSelector((s) => s.auth.user);
  const { send } = useWebSocket("wss://example.com/ws");

  // Мемоизируем список — не пересортировываем при каждом нажатии клавиши
  const sortedMessages = useMemo(
    () => messages.slice().sort((a, b) => a.timestamp - b.timestamp),
    [messages]
  );

  const handleSend = useCallback(() => {
    if (!input.trim() || !user) return;
    send({ text: input, author: user.name });
    setInput("");
  }, [input, user, send]);

  return (
    <div>
      <ul>
        {sortedMessages.map((m) => (
          <li key={m.id}><b>{m.author}:</b> {m.text}</li>
        ))}
      </ul>
      <input value={input} onChange={(e) => setInput(e.target.value)} />
      <button onClick={handleSend}>Отправить</button>
    </div>
  );
}
```

**Что демонстрирует:**
- `useAppSelector` — типизированный доступ к state.
- `useMemo` — сортировка только при изменении `messages`.
- `useCallback` — стабильный обработчик.
- Контролируемый input через `useState`.

---

## 8. Защита маршрутов

**`src/features/auth/ProtectedRoute.tsx`**

```typescript
import { Navigate } from "react-router-dom";
import { useAppSelector } from "@/app/hooks";
import type { ReactNode } from "react";

export function ProtectedRoute({ children }: { children: ReactNode }) {
  const status = useAppSelector((s) => s.auth.status);
  if (status !== "authenticated") return <Navigate to="/login" replace />;
  return <>{children}</>;
}
```

**Помните:** это только UX-защита. Сервер проверяет JWT на каждом запросе.

---

## 9. SSR: страница Next.js

**`src/pages/index.tsx`**

```typescript
import { GetServerSideProps } from "next";
import { ChatRoom } from "@/features/chat/ChatRoom";
import { ProtectedRoute } from "@/features/auth/ProtectedRoute";

interface Props { initialMessages: { id: string; text: string; author: string; timestamp: number }[] }

export default function HomePage({ initialMessages }: Props) {
  return (
    <ProtectedRoute>
      <ChatRoom />
    </ProtectedRoute>
  );
}

// SSR: сообщения рендерятся на сервере, краулер видит контент
export const getServerSideProps: GetServerSideProps<Props> = async (ctx) => {
  const token = ctx.req.cookies.token;
  if (!token) return { redirect: { destination: "/login", permanent: false } };

  const res = await fetch("https://api.example.com/messages", {
    headers: { Authorization: `Bearer ${token}` },
  });
  const initialMessages = await res.json();

  return { props: { initialMessages } };
};
```

**Что происходит:**
1. Пользователь запрашивает `/`.
2. Сервер проверяет cookie с токеном.
3. Сервер запрашивает сообщения у API и рендерит HTML с готовым списком.
4. Браузер получает HTML с контентом, затем **гидратирует** его React-кодом.

---

## 10. Тестирование

### 10.1. Модульный тест reducer

**`tests/authSlice.test.ts`**

```typescript
import reducer, { logout, login } from "@/features/auth/authSlice";

test("logout очищает пользователя", () => {
  const state = { user: { id: 1, name: "Иван", role: "user" as const }, token: "abc", status: "authenticated" as const };
  const next = reducer(state, logout());
  expect(next.user).toBeNull();
  expect(next.status).toBe("idle");
});

test("login.fulfilled сохраняет данные", () => {
  const state = { user: null, token: null, status: "idle" as const };
  const action = login.fulfilled({ user: { id: 1, name: "Иван", role: "user" }, token: "abc" }, "", { email: "", password: "" });
  const next = reducer(state, action);
  expect(next.user?.name).toBe("Иван");
  expect(next.status).toBe("authenticated");
});
```

### 10.2. Компонентный тест

**`tests/ChatRoom.test.tsx`**

```typescript
import { render, screen, fireEvent } from "@testing-library/react";
import { Provider } from "react-redux";
import { configureStore } from "@reduxjs/toolkit";
import { ChatRoom } from "@/features/chat/ChatRoom";
import authReducer from "@/features/auth/authSlice";
import chatReducer from "@/features/chat/chatSlice";

test("сообщение добавляется в список", () => {
  const store = configureStore({
    reducer: { auth: authReducer, chat: chatReducer },
    preloadedState: {
      auth: { user: { id: 1, name: "Иван", role: "user" }, token: "x", status: "authenticated" },
      chat: { messages: [] },
    },
  });

  render(<Provider store={store}><ChatRoom /></Provider>);

  fireEvent.change(screen.getByRole("textbox"), { target: { value: "Привет" } });
  fireEvent.click(screen.getByRole("button", { name: /отправить/i }));

  // Проверяем поведение — что пользователь видит
  expect(screen.getByRole("textbox")).toHaveValue("");
});
```

**Ключевое:** тест ищет элементы по роли (`getByRole`), а не по классу или id. Это устойчиво к изменениям разметки.

---

## 11. Что покрывает этот пример

| Тема | Где реализовано |
|------|-----------------|
| Компонентный подход | `ChatRoom`, `ProtectedRoute`, `LoginForm` |
| JSX и VDOM | Все компоненты возвращают JSX; React обновляет DOM через reconciliation |
| `useState` | `input` в `ChatRoom` |
| `useEffect` | Подключение WebSocket |
| `useCallback` | `handleSend`, `send` |
| `useMemo` | `sortedMessages` |
| Redux: actions, reducers, store | `authSlice`, `chatSlice`, `store.ts` |
| `react-redux` | `useAppSelector`, `useAppDispatch` |
| Async + thunk | `login` через `createAsyncThunk` |
| Отладка Redux | Подключить Redux DevTools (в `configureStore` — по умолчанию) |
| WebSocket | `useWebSocket`: open, message, close, reconnect |
| TypeScript | `User`, `Message`, `AuthState`, типизация props и событий |
| SSR | `getServerSideProps` в Next.js |
| Тестирование | Модульный тест reducer + компонентный тест |
| Аутентификация | JWT в заголовке, `ProtectedRoute`, проверка на сервере |

---

## 12. Как запустить

```bash
npx create-next-app@latest chat-app --typescript
cd chat-app
npm install @reduxjs/toolkit react-redux
npm install -D @testing-library/react @testing-library/jest-dom vitest
```

Затем скопируйте файлы из примера в соответствующие директории и запустите `npm run dev`.

---

## Итог

Этот пример — **скелет реального приложения**. Он не идеален (в реальном проекте добавились бы: refresh token, обработка ошибок сети, оптимизация подписок Redux, тесты интеграции с MSW), но он показывает, **как все части стека соединяются вместе**.

Каждый инструмент решает конкретную задачу:
- **React** — рендеринг интерфейса.
- **TypeScript** — безопасность типов.
- **Redux** — предсказуемое управление состоянием.
- **WebSocket** — реальное время.
- **SSR** — быстрая первая отрисовка и SEO.
- **Тесты** — уверенность в изменениях.
