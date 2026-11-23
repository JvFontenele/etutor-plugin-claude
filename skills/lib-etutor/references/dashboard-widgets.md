# Dashboard e Widgets

Grid arrastável/redimensionável para montar dashboards a partir de uma lista de widgets tipados.
Baseado em `gridstack` (`^13.2.0`) e `echarts` (`^6.1.0`), ambos dependências da `etutor-lib` — não
é preciso instalá-las no `etutor-web`.

## Componentes exportados

Todos exportados diretamente de `etutor-lib`:

| Componente | Descrição |
|---|---|
| `DashboardGrid` | Grid principal — recebe a lista de widgets e monta o layout |
| `WidgetRenderer` | Despacha o card certo por `widget.type` (uso interno do `DashboardGrid`, raramente usado direto) |
| `ChartWidget` | Card de gráfico (echarts) |
| `MetricWidget` | Card de métrica/KPI |
| `TableWidget` | Card de tabela |

## Tipos

Também exportados de `etutor-lib` (`components/lib/dashboard/dashboard.types.ts`):

```ts
type WidgetLayout = {
  id: string;
  title?: string;
  lastUpdate?: string;
  x: number;
  y: number;
  w: number;
  h: number;
};

type WidgetConfig = ChartWidgetConfig | MetricWidgetConfig | TableWidgetConfig;

type ChartWidgetConfig = WidgetLayout & {
  type: "chart";
  data: ChartData;   // estende EChartsOption + { categories?, series? }
};

type MetricWidgetConfig = WidgetLayout & {
  type: "metric";
  value: number;
  formatter?: (value: number) => string;
};

type TableWidgetConfig = WidgetLayout & {
  type: "table";
  value: any[];
  headers?: TableHeader[];   // { text, value }
  subtitle?: string;
};
```

## Props do `DashboardGrid`

| Prop | Tipo | Descrição |
|---|---|---|
| `widgets` | `WidgetConfig[]` | Lista de widgets a renderizar (obrigatório) |
| `version` | `number` | Incremente para forçar re-render do **conteúdo** dos widgets já montados (ex.: chegaram dados novos). Nunca recarregue a lista `widgets` para atualizar dados — ver regra abaixo |
| `modify` | `boolean` (default `false`) | `false`: grid somente leitura (sem mover/redimensionar), sempre carrega o layout de `widgets`. `true`: habilita drag/resize e persiste o layout do usuário em `localStorage` (chave `location.pathname + "-layout"`), restaurando-o nas visitas seguintes em vez de `widgets` |

Slot default do item do grid: `<slot :widget="widget">` — permite customizar o card por fora do
`WidgetRenderer` padrão.

## Regras de uso críticas

- **Monte `<DashboardGrid>` só depois que `widgets` já tiver o tamanho final** (padrão:
  `v-if="!isLoading"`). O carregamento interno (`grid.load()`) roda uma única vez, no mount.
- **Trocar a prop `widgets` depois de montado não serve para atualizar dados** — o `v-for` do
  template e o `grid.load()` interno disputam a posse do DOM (sintomas: títulos duplicados, nós
  órfãos vazios). Para atualizar o conteúdo de widgets já existentes, incremente `version`.

## Exemplo

Uso real: `src/modules/gestor/applications/dashboard/` — `useDashboard.ts` monta `widgets` e
incrementa `version` a cada carga; a view renderiza
`<DashboardGrid v-if="widgets.length" :widgets="widgets" :version="version" />`. O `ChartWidget` só
redesenha quando `version` muda (ver `docs/gestor.md`). Esqueleto genérico:

```vue
<script setup lang="ts">
import { DashboardGrid, type WidgetConfig } from "etutor-lib";
import { ref, onMounted } from "vue";

const isLoading = ref(true);
const version = ref(0);

const widgets = ref<WidgetConfig[]>([
  { id: "total-alunos", type: "metric", x: 0, y: 0, w: 3, h: 2, title: "Total de alunos", value: 0 },
  {
    id: "matriculas-por-mes",
    type: "chart",
    x: 3, y: 0, w: 6, h: 4,
    title: "Matrículas por mês",
    data: { categories: ["Jan", "Fev", "Mar"], series: [{ type: "bar", data: [] }] },
  },
  {
    id: "ultimas-turmas",
    type: "table",
    x: 0, y: 4, w: 9, h: 4,
    title: "Últimas turmas",
    value: [],
    headers: [
      { text: "Turma", value: "descricao" },
      { text: "Turno", value: "turno" },
    ],
  },
]);

onMounted(async () => {
  // ... busca os dados e popula widgets.value com o tamanho final ...
  isLoading.value = false;
});

// Dados novos depois de montado: atualize o conteúdo e incremente version
function recarregar() {
  version.value++;
}
</script>

<template>
  <DashboardGrid v-if="!isLoading" :widgets="widgets" :version="version" modify />
</template>
```
