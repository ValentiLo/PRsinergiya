# Конспект: Современная разработка на React + TypeScript (стек Middle+/Senior)

Ниже — структурированный конспект по всем заявленным темам. Материал рассчитан на уровень, где нужно не просто знать синтаксис, а понимать **почему** и **когда** применять каждый инструмент.

---

## 1. React: компонентный подход

### 1.1. Суть компонентного подхода

React строится на идее, что интерфейс — это **дерево компонентов**. Каждый компонент — независимая единица, которая:
- принимает входные данные (**props**) от родителя;
- хранит собственное состояние (**state**);
- возвращает описание того, что должно быть на экране (JSX).

Компоненты бывают **функциональными** (современный стандарт) и классовыми (legacy). Функциональные компоненты + хуки полностью вытеснили классовые в новом коде.

**Ключевые принципы:**
- **Однонаправленный поток данных**: props передаются сверху вниз, дети не мутируют props родителя.
- **Композиция вместо наследования**: переиспользование достигается через вложение компонентов и `children`, а не через классовые иерархии.
- **Чистота**: компонент должен быть чистой функцией от props и state — одинаковый вход даёт одинаковый выход.

### 1.2. JSX

**JSX** — синтаксический сахар над `React.createElement`. Компилятор (Babel или tsc) превращает `<Button label="OK" />` в вызов функции.

**Что важно знать:**
- JSX не является HTML. Атрибуты пишутся в camelCase (`className`, `onClick`, `htmlFor`).
- Выражения встраиваются через `{}`: `<div>{user.name}</div>`.
- Условный рендеринг: `{isLoading && <Spinner />}` или тернарник `{isLoading ? <Spinner /> : <Content />}`.
- Фрагменты `<>...</>` позволяют вернуть несколько элементов без лишнего DOM-узла.

### 1.3. Виртуальный DOM

**Виртуальный DOM (VDOM)** — это лёгкое in-memory представление реального DOM в виде JavaScript-объектов.

**Как работает процесс обновления:**

1. При изменении state или props React создаёт **новое дерево VDOM**.
2. Выполняется **reconciliation** (согласование): React сравнивает новое дерево со старым (diff-алгоритм).
3. Вычисляется **минимальный набор изменений**.
4. Изменения применяются к **реальному DOM** пакетно (batching).

**Зачем это нужно:** прямое манипулирование DOM медленное. VDOM позволяет минимизировать дорогие операции с реальным DOM и абстрагировать разработчика от ручного управления.

**Важно понимать:** VDOM — не серебряная пуля. Его основная ценность — **декларативность**: вы описываете «что должно быть», а React решает «как это обновить».

---

## 2. Управление состоянием в React

### 2.1. Локальное состояние (useState)

`useState` — базовый хук для локального состояния компонента.

```typescript
const [count, setCount] = useState<number>(0);
```

**Ключевые моменты:**
- `setCount` **асинхронен** в смысле батчинга: несколько вызовов в одном обработчике объединяются.
- Если новое значение зависит от предыдущего — используйте **функциональную форму**: `setCount(prev => prev + 1)`.
- Мутация объекта в state не вызовет ререндер. Нужно создавать новый объект: `setUser({ ...user, name: "Новое" })`.

### 2.2. Побочные эффекты (useEffect)

`useEffect` запускает побочный код после рендера: запросы к API, подписки, работа с DOM.

```typescript
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then(res => res.json())
    .then(setUser)
    .catch(err => { if (err.name !== "AbortError") console.error(err); });

  return () => controller.abort(); // cleanup
}, [id]); // массив зависимостей
```

**Правила:**
- **Пустой массив `[]`** — эффект выполнится один раз после монтирования (аналог `componentDidMount`).
- **Массив с зависимостями** — эффект перезапускается при изменении любой зависимости.
- **Без массива** — эффект выполняется после каждого рендера (обычно не нужно).
- **Cleanup-функция** вызывается перед следующим запуском эффекта и при размонтировании.

**Частая ошибка:** забыть зависимость → эффект работает со «старым» замыканием. ESLint-правило `react-hooks/exhaustive-deps` помогает это отслеживать.

### 2.3. Контекст (useContext)

`useContext` решает проблему **prop drilling** — передачи данных через много уровней вложенности.

```typescript
const ThemeContext = createContext<"light" | "dark">("light");

function ThemeToggle() {
  const theme = useContext(ThemeContext);
  return <button>{theme === "light" ? "🌙" : "☀️"}</button>;
}
```

**Когда использовать:** тема, локаль, текущий пользователь, настройки — то, что нужно многим компонентам на разных уровнях.

**Когда не использовать:** контекст не заменяет глобальный стейт-менеджер. При частых изменениях контекста все потребители ререндерятся. Для сложного состояния — Redux, Zustand и т.п.

### 2.4. useReducer

`useReducer` — альтернатива `useState` для **сложной логики состояния**, где важна предсказуемость переходов.

```typescript
type State = { count: number; step: number };
type Action = { type: "increment" } | { type: "decrement" } | { type: "setStep"; payload: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment": return { ...state, count: state.count + state.step };
    case "decrement": return { ...state, count: state.count - state.step };
    case "setStep":   return { ...state, step: action.payload };
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });
```

**Когда использовать:** несколько связанных полей, сложные переходы, логика которых проще выразить через `switch` по action.

### 2.5. useCallback и useMemo

Оба хука — для **оптимизации производительности**, но решают разные задачи.

**`useMemo`** мемоизирует **значение**:

```typescript
const sortedItems = useMemo(
  () => items.slice().sort((a, b) => a.price - b.price),
  [items]
);
```

**`useCallback`** мемоизирует **функцию**:

```typescript
const handleClick = useCallback(() => {
  setCount(c => c + 1);
}, []); // функция не пересоздаётся между рендерами
```

**Главное правило:** не оптимизируйте преждевременно. `useMemo` и `useCallback` сами по себе имеют накладные расходы. Применяйте, когда:
- функция передаётся в `React.memo`-компонент;
- вычисление действительно тяжёлое;
- функция используется в зависимостях других хуков.

---

## 3. Жизненный цикл компонентов

В функциональных компонентах жизненный цикл моделируется через `useEffect`.

| Классовый метод | Хук-аналог |
|-----------------|-----------|
| `componentDidMount` | `useEffect(() => {...}, [])` |
| `componentDidUpdate` | `useEffect(() => {...}, [deps])` |
| `componentWillUnmount` | `useEffect(() => { return () => {...} }, [])` |

**Три фазы:**
1. **Монтирование (Mount)** — компонент создаётся и вставляется в DOM.
2. **Обновление (Update)** — props или state изменились, происходит ререндер.
3. **Размонтирование (Unmount)** — компонент удаляется из DOM, вызываются cleanup-функции.

---

## 4. TypeScript: типы и интерфейсы

### 4.1. Базовые типы

`boolean`, `number`, `string`, `null`, `undefined`, `symbol`, `bigint`, `any`, `unknown`, `never`, `void`.

**`any` vs `unknown`:** `any` отключает проверку типов полностью. `unknown` — безопасная альтернатива: присвоить можно что угодно, но **использовать** значение без сужения типа нельзя.

```typescript
let value: unknown = "текст";
// value.toUpperCase(); // ошибка
if (typeof value === "string") value.toUpperCase(); // OK
```

### 4.2. Интерфейсы и типы

**`interface`** — описывает форму объекта. Поддерживает **слияние объявлений** (declaration merging).

**`type`** — псевдоним для любого типа: примитива, объединения, кортежа, функции.

**Правило выбора:** `interface` по умолчанию; `type` — когда нужны объединения, отображённые типы или слияние невозможно.

### 4.3. Типизация компонентов

```typescript
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: "primary" | "secondary";
}

function Button({ label, onClick, variant = "primary" }: ButtonProps) {
  return <button className={variant} onClick={onClick}>{label}</button>;
}
```

**Обобщённые компоненты:**

```typescript
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}

function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map(renderItem)}</ul>;
}
```

**Типизация событий:** `React.ChangeEvent<HTMLInputElement>`, `React.MouseEvent<HTMLButtonElement>`.

### 4.4. Миграция с JavaScript

Постепенный процесс:
1. Переименовать `.js` → `.ts`, `.tsx`.
2. Настроить `tsconfig.json` с `strict: true`.
3. Исправлять ошибки от реальных багов к недостающим типам.
4. Установить `@types/библиотека` для сторонних зависимостей.
5. **Типы стираются при компиляции** — поведение программы не меняется.

---

## 5. Redux: управление состоянием

### 5.1. Три принципа Redux

1. **Единственный источник истины** — всё состояние в одном store.
2. **Состояние только для чтения** — изменения через action (описание «что произошло»).
3. **Изменения через чистые функции** — reducers возвращают новый state.

### 5.2. Actions, reducers, store

```typescript
// Action
const increment = () => ({ type: "counter/increment" } as const);

// Reducer
function counterReducer(state = { value: 0 }, action: { type: string }) {
  switch (action.type) {
    case "counter/increment": return { value: state.value + 1 };
    default: return state;
  }
}

// Store
const store = configureStore({ reducer: { counter: counterReducer } });
```

### 5.3. react-redux

```typescript
function Counter() {
  const count = useSelector((state: RootState) => state.counter.value);
  const dispatch = useDispatch();
  return <button onClick={() => dispatch(increment())}>{count}</button>;
}
```

- `useSelector` подписывается на часть state (селектор должен быть **стабильным**).
- `useDispatch` даёт функцию `dispatch`.

### 5.4. Асинхронные действия (redux-thunk)

Thunk — это функция, которая принимает `dispatch` и `getState` и может выполнять асинхронную логику.

```typescript
export const fetchUser = (id: number) => async (dispatch: AppDispatch) => {
  dispatch(userLoading());
  try {
    const res = await fetch(`/api/users/${id}`);
    const data = await res.json();
    dispatch(userLoaded(data));
  } catch (err) {
    dispatch(userFailed(err.message));
  }
};
```

**Альтернативы:** Redux Toolkit (RTK Query), redux-saga, zustand для простых случаев.

### 5.5. Отладка Redux

- **Redux DevTools** — просмотр всех action, состояния до/после, time-travel debugging.
- **`redux-logger`** — логирование в консоль.
- **Именование action** по шаблону `domain/event` (`user/fetched`) упрощает фильтрацию.

---

## 6. WebSocket

### 6.1. Что это

**WebSocket** — протокол, обеспечивающий **двунаправленное постоянное соединение** между клиентом и сервером поверх TCP. В отличие от HTTP, соединение не закрывается после каждого запроса, и сервер может **отправлять данные клиенту без запроса**.

**Установление соединения (handshake):**

1. Клиент отправляет HTTP-запрос с заголовком `Upgrade: websocket`.
2. Сервер отвечает `101 Switching Protocols`.
3. Соединение переключается на WebSocket-протокол (фреймы).

### 6.2. Работа с WebSocket в React

```typescript
useEffect(() => {
  const ws = new WebSocket("wss://example.com/chat");

  ws.onopen = () => console.log("Соединение установлено");
  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    setMessages(prev => [...prev, data]);
  };
  ws.onerror = (err) => console.error("Ошибка", err);
  ws.onclose = () => console.log("Соединение закрыто");

  return () => ws.close(); // cleanup
}, []);
```

### 6.3. Обработка событий и сообщений

- **События:** `open`, `message`, `error`, `close`.
- **Формат сообщений:** чаще всего JSON, но возможны бинарные данные (`Blob`, `ArrayBuffer`).
- **Heartbeat / ping-pong:** периодическая отправка «пинга» для проверки живости соединения. Если ответа нет — переподключение.
- **Переподключение:** при `close` использовать экспоненциальный backoff (1с, 2с, 4с…).

---

## 7. SSR (Server-Side Rendering)

### 7.1. Зачем SSR

**Проблема CSR (Client-Side Rendering):** браузер получает пустой HTML и ждёт, пока загрузится JS, выполнится React и нарисует интерфейс. Пользователь видит белый экран, поисковики плохо индексируют.

**SSR** рендерит HTML **на сервере** и отдаёт браузеру уже готовую разметку. Затем клиент **гидратирует** её — присоединяет обработчики событий.

### 7.2. Преимущества и недостатки

| Плюсы | Минусы |
|-------|--------|
| Быстрый First Contentful Paint | Нагрузка на сервер |
| SEO (краулеры видят контент) | Сложность инфраструктуры |
| Работа без JS (частично) | Hydration mismatch (ошибки при расхождении) |

### 7.3. Инструменты

- **Next.js** — стандарт де-факто для SSR в React. App Router поддерживает Server Components, стриминг, кэширование.
- **Remix** — альтернатива с фокусом на веб-стандарты.
- **Vite SSR** — ручная настройка для нестандартных сценариев.

**Ключевая проблема:** SSR и CSR должны рендерить **одинаковый** HTML. Если сервер и клиент генерируют разную разметку (например, из-за `Date.now()` или `Math.random()`), React выдаст hydration mismatch.

---

## 8. Тестирование React-приложений

### 8.1. Три уровня тестов

| Уровень | Что проверяет | Инструменты |
|---------|---------------|-------------|
| **Модульные (unit)** | Отдельные функции, хуки, редьюсеры | Jest, Vitest |
| **Компонентные** | Поведение одного компонента в изоляции | React Testing Library, Jest |
| **Интеграционные** | Взаимодействие нескольких компонентов и API | React Testing Library + MSW, Cypress |

### 8.2. Что тестировать

**Не тестируйте реализацию — тестируйте поведение.** React Testing Library пропагандирует подход «как пользователь»:

```typescript
test("кнопка увеличивает счётчик", () => {
  render(<Counter />);
  const button = screen.getByRole("button", { name: /увеличить/i });
  fireEvent.click(button);
  expect(screen.getByText(/счётчик: 1/i)).toBeInTheDocument();
});
```

**Ключевые запросы:** `getByRole`, `getByLabelText`, `getByText` — они отражают то, что видит пользователь.

### 8.3. Мокирование API

MSW (Mock Service Worker) перехватывает сетевые запросы на уровне Service Worker, не требуя мокать `fetch`:

```typescript
rest.get("/api/user", (req, res, ctx) => res(ctx.json({ name: "Иван" })));
```

---

## 9. Аутентификация и авторизация в SPA

### 9.1. Различие

- **Аутентификация** — «кто ты?» (проверка личности).
- **Авторизация** — «что тебе можно?» (проверка прав).

### 9.2. Механизмы

**JWT (JSON Web Token)** — самый распространённый в SPA:

1. Клиент отправляет логин/пароль на сервер.
2. Сервер возвращает JWT (подписанный токен с payload).
3. Клиент сохраняет токен и отправляет его в заголовке `Authorization: Bearer <token>`.
4. Сервер проверяет подпись и срок действия.

**Хранение токена:**
- **localStorage** — уязвим к XSS (если злоумышленник выполнит JS, он прочитает токен).
- **HttpOnly cookie** — недоступен для JS, но требует защиты от CSRF.

**Refresh token:** access token короткоживущий (15 мин), refresh token долгоживущий (7 дней) и используется только для получения нового access token.

### 9.3. Защита маршрутов в React

```typescript
function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated } = useAuth();
  if (!isAuthenticated) return <Navigate to="/login" replace />;
  return <>{children}</>;
}
```

**Важно:** защита на клиенте — только **UX**. Сервер **обязан** проверять права на каждом запросе. Клиентские проверки легко обойти через DevTools.

### 9.4. Роли и права

Используйте контекст или Redux для хранения роли. Компоненты проверяют права перед рендером:

```typescript
{hasPermission("admin:users") && <AdminPanel />}
```

---

## Итог

| Тема | Ключевой навык |
|------|----------------|
| **React** | Компоненты, JSX, VDOM, хуки — понимать, когда какой хук применять |
| **TypeScript** | Типизация props, state, событий; постепенная миграция |
| **Redux** | Actions → reducers → store; thunk для async; DevTools |
| **WebSocket** | Установка соединения, обработка событий, переподключение |
| **SSR** | Next.js, hydration, баланс между сервером и клиентом |
| **Тестирование** | Поведение, а не реализация; RTL + MSW |
| **Auth** | JWT, защита маршрутов, серверная проверка прав |

**Главное правило:** каждый инструмент решает конкретную проблему. Не тащите Redux в приложение с одним счётчиком, не используйте `useMemo` везде подряд, не пишите SSR ради моды. Понимание **проблемы** важнее знания **API**.
