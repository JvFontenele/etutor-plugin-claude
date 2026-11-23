# Views, abas e tabelas

## Views de CRUD

Views de CRUD são finas — sem lógica de busca. Tudo isso fica no composable.

### Listagem paginada (`index.vue`) — `PageSearchTable`

Quando o back-end tem `GET` paginado (`Pageable`):

```vue
<!-- src/modules/gestor/applications/manter-veiculo/views/index.vue -->
<script setup lang="ts">
import { PageSearchTable, type ColumnDescriptor } from "etutor-lib";
import { useRouter } from "vue-router";
import { useFiltroFields } from "../composables/useFiltroFields";
import { useRequests } from "../useRequests";
import type { VeiculoDetalhe } from "../../../types/veiculo";

const { fields, data } = useFiltroFields();
const { getItems } = useRequests();
const router = useRouter();

const columns: ColumnDescriptor<VeiculoDetalhe>[] = [
  { value: "placa", title: "Placa" },
  { value: "renavam", title: "Renavam" },
  { value: "modeloVeiculo.descricao", title: "Modelo" },
  {
    value: "actions",
    actions: [
      { title: "Editar",     icon: "SquarePen", link: ({ rowData }) => ({ name: "gestor-veiculos-editar",     params: { id: rowData.id } }) },
      { title: "Detalhar",   icon: "Search",    link: ({ rowData }) => ({ name: "gestor-veiculos-detalhar",   params: { id: rowData.id } }) },
      { title: "Manutenção", icon: "Wrench",    link: ({ rowData }) => ({ name: "gestor-veiculos-manutencao", params: { id: rowData.id } }) },
      { title: "Documentos", icon: "FileText",  link: ({ rowData }) => ({ name: "gestor-veiculos-documentos", params: { id: rowData.id } }) },
    ],
  },
];
</script>

<template>
  <PageSearchTable
    v-model:data="data"
    :columns="columns"
    addButtonLabel="Novo Veículo"
    :fields="fields"
    :queryFn="getItems"
    @add="router.push({ name: 'gestor-veiculos-cadastrar' })"
  />
</template>
```

> **Liste só o que a busca devolve.** A rota paginada costuma devolver um DTO mais enxuto que o
> detalhe; campos que só existem no detalhe apareceriam sempre vazios. Confira no api-docs.

> **⚠️ `sortable` com back-end Spring.** O `PageSearchTable` manda `sort=-campo` para descendente,
> formato que o `Pageable` do Spring não entende (esperaria `campo,desc`). Contra `Pageable`, deixe o
> `sortable` de fora. O `paginarLocal` (abaixo) entende `-campo`.

### GET não paginado

Quando o `GET` devolve a lista inteira (sem `Pageable`), há três padrões, todos reais:

| Situação | Componente | Exemplo |
|---|---|---|
| Lista curta, sem filtro | `Table` com `:pagination="false"` (ou `true` para paginar no cliente) | `gestor/applications/manter-modelo-veiculo/views/index.vue` |
| Lista sem filtro, mas no formato de página | `DataTable :page` + `comoPagina()` | `controle-acesso/applications/manter-perfil/views/index.vue` |
| Lista com filtro e paginação na tela | `PageSearchTable` com `queryFn` montada sobre `paginarLocal()` | `gestor/applications/monitorar-rede/useRequests.ts` |

```vue
<!-- src/modules/controle-acesso/applications/manter-perfil/views/index.vue (trecho) -->
<script setup lang="ts">
const { getItems } = useRequests();
const perfis = ref(comoPagina<Perfil>([]));

onMounted(async () => {
  perfis.value = comoPagina(await getItems.execute());
});
</script>

<template>
  <div class="space-y-4">
    <div class="flex justify-end">
      <AddButton text="Novo Perfil" @click="router.push({ name: 'manter-perfil-cadastrar' })" />
    </div>
    <DataTable :columns="columns" :page="perfis" />
  </div>
</template>
```

```vue
<!-- src/modules/gestor/applications/manter-modelo-veiculo/views/index.vue (trecho) -->
<template>
  <div class="flex justify-end mb-4">
    <AddButton type="button" @click="router.push({ name: 'gestor-modelos-veiculo-cadastrar' })">
      Novo Modelo
    </AddButton>
  </div>
  <Table :data="modelos" :columns="columns" :pagination="false" />
</template>
```

`paginarLocal(itens, params)` (`src/modules/gestor/utils/paginacaoLocal.ts`) devolve um
`PaginatedResponseDTO` a partir da lista inteira, respeitando `page`, `size` e `sort`. A `queryFn`
recebe os filtros em `params`, chama o serviço e pagina no front:

```ts
// src/modules/gestor/applications/monitorar-rede/useRequests.ts (trecho)
const queryTurmas = async ({ params }: QueryFnPaginatedParans) => {
  const idAnoLetivo = Number(params.idAnoLetivo);
  if (!idAnoLetivo) return paginarLocal([], params);

  return paginarLocal(await getTurmasMonitoramento(idAnoLetivo), params);
};
```

```vue
<PageSearchTable v-model:data="data" :columns="columns" :fields="fields" :queryFn="queryTurmas" />
```

`paginarLocal` está em `gestor/utils`: outro módulo que precise dele não importa de `gestor` —
mova para `src/utils` (compartilhado) antes de reutilizar.

### Cadastrar (`cadastrar.vue`)

```vue
<!-- src/modules/gestor/applications/manter-veiculo/views/cadastrar.vue -->
<script setup lang="ts">
import { ref } from "vue";
import { DynamicForm, PageGroupButtons } from "etutor-lib";
import { useRouter } from "vue-router";
import { useFieldsVeiculo } from "../composables/useFieldsVeiculo";
import { useRequests } from "../useRequests";
import { getInstituicaoId } from "@/core/sessao/instituicao";
import type { VeiculoRequest } from "../../../types/veiculo";

const router = useRouter();
const { postVeiculo } = useRequests();
const formRef = ref<InstanceType<typeof DynamicForm> | null>(null);
const { fields, data } = useFieldsVeiculo();          // sem parâmetros = modo cadastro
const salvando = ref(false);

async function onSubmit() {
  if (salvando.value) return;
  salvando.value = true;
  try {
    const result = await formRef.value?.validate();
    if (!result?.valid) return;
    const payload = {
      ...(formRef.value?.transformData() as unknown as VeiculoRequest),
      instituicaoId: getInstituicaoId(),
    };
    await postVeiculo.execute(payload);                // onSuccess do useRequests volta para a listagem
  } finally {
    salvando.value = false;
  }
}
</script>

<template>
  <DynamicForm ref="formRef" :fields="fields" v-model="data" />
  <PageGroupButtons @save="onSubmit()" @back="router.push({ name: 'gestor-veiculos' })" />
</template>
```

`PageGroupButtons`: o Voltar sempre faz `router.back()` e, depois, emite `back` — passe `@back`
quando a tela precisa de um destino fixo (a listagem). `clear` habilita o botão Limpar (que recarrega
a página); `title-save` troca o rótulo do Salvar; o slot default recebe botões extras. O Salvar só
aparece com listener `@save` ou com `type-save="submit"` (botão `type=submit` de um `<form>`).

O `Page` tem ainda o alvo `#page-before-title` para teleportar botões ao lado do título, e
`meta.breadcrumb` sobrepõe `meta.title` só no breadcrumb.

### Detalhar (`detalhar.vue`)

```vue
<script setup lang="ts">
// src/modules/gestor/applications/manter-veiculo/views/detalhar.vue
const route = useRoute();
const router = useRouter();
const id = route.params.id as string;
const { fields, data } = useFieldsVeiculo(id, true);   // true = _disabled em cada campo
</script>

<template>
  <DynamicForm :fields="fields" v-model="data" />
  <PageGroupButtons @back="router.push({ name: 'gestor-veiculos' })" />
</template>
```

O padrão atual desabilita **campo a campo** com `disabled: _disabled` dentro do `useFields` — assim
um campo pode continuar ativo no detalhar (ex.: um botão de visualizar). `:disabled="true"` no
`DynamicForm` ainda funciona e desabilita tudo, mas as telas novas não usam.

`PageGroupButtons` também aceita `:pdf="{ fields, data, config }"` para exibir "Exportar PDF" do
detalhe (usa `exportFormToPDF`).

## Entidade com seções

A entidade selecionada é **sempre o `:id` da rota**, nunca uma store global (recarregar a página não
pode perder a seleção). Há dois jeitos de dividir as seções, ambos em uso:

### 1. Abas com `Tabs` (seções na mesma rota)

`editar.vue` é só o `Tabs`; cada aba é um componente em `views/abas/` que recebe o id por prop e
carrega seus próprios dados. Sem `manter-conteudo`, o `Tabs` renderiza **só o slot ativo**.

```vue
<!-- src/modules/controle-acesso/applications/manter-usuario/views/editar.vue -->
<script setup lang="ts">
import { computed, ref } from "vue";
import { Tabs } from "etutor-lib";
import { useRoute } from "vue-router";
import InformacoesGerais from "./abas/InformacoesGerais.vue";
import Vinculos from "./abas/Vinculos.vue";

const route = useRoute();
const usuarioId = computed(() => route.params.id?.toString() ?? "");

const abaAtiva = ref("informacoes-gerais");

const tabs = [
  { label: "Informações Gerais", value: "informacoes-gerais" },
  { label: "Perfis e Lotações", value: "vinculos" },
];
</script>

<template>
  <Tabs :tabs="tabs" v-model="abaAtiva">
    <template #informacoes-gerais><InformacoesGerais :usuario-id="usuarioId" /></template>
    <template #vinculos><Vinculos :usuario-id="usuarioId" /></template>
  </Tabs>
</template>
```

Aba de formulário — salva e **não navega para fora**:

```vue
<!-- src/modules/controle-acesso/applications/manter-usuario/views/abas/InformacoesGerais.vue -->
<script setup lang="ts">
const props = defineProps<{ usuarioId: string }>();

const { putItem } = useRequests();
const router = useRouter();
const formRef = ref<InstanceType<typeof DynamicForm> | null>(null);
const { fields, data } = useFields(props.usuarioId);

const salvando = ref(false);

async function onSubmit() {
  if (salvando.value) return;
  salvando.value = true;
  try {
    const result = await formRef.value?.validate();
    if (!result?.valid) return;
    await putItem.execute(props.usuarioId, formRef.value?.transformData());
  } finally {
    salvando.value = false;
  }
}
</script>

<template>
  <DynamicForm ref="formRef" :fields="fields" v-model="data" />
  <PageGroupButtons @save="onSubmit()" @back="router.push({ name: 'manter-usuario' })" />
</template>
```

Aba de lista com endpoint próprio: `views/abas/Vinculos.vue` — ver [formtable.md](formtable.md).

**Um único submit para todas as abas:** `<Tabs manter-conteudo>` mantém todas as abas montadas
(`v-show`); cada aba expõe `getValues()` via `defineExpose` e a view junta os payloads num só
`POST` (`escola/applications/manter-estrutura-escola/views/manter.vue`).

### 2. Seções como rotas-filhas (`:id/<seção>`)

A listagem oferece uma ação por seção, cada uma levando a uma rota própria; cada seção tem sua view
(e composable) e carrega o próprio dado a partir do `:id`. É o padrão de `escola`
(`manter-escola`: `:id/editar`, `:id/compartilhamento`, `:id/dados-educacionais`, `:id/arquivos`, ...)
e `aluno` (`manter-aluno`: `:id/editar`, `:id/necessidades-especiais`, `:id/anotacoes`,
`:id/ficha-antropometrica`).

```ts
// src/modules/escola/applications/manter-escola/views/index.vue (trecho das actions)
{ title: "Arquivos", icon: "Files", link: ({ rowData }) => ({ name: "manter-escola-arquivos", params: { id: rowData.id } }) },
```

```ts
// src/modules/escola/utils/useIdsEscola.ts — id da rota para as seções do módulo
export function useIdsEscola() {
  const route = useRoute();
  return computed<{ id: number | string }>(() => route.params as any);
}
```

As rotas-filhas podem ficar aninhadas sob a da listagem sem `<RouterView />` nela — o `Page` monta a
folha direto (ver [config.md](config.md#page--colapso-automático-de-rotas-aninhadas)).

**Cadastro é só dos dados gerais.** As demais seções dependem de um id já existente nos endpoints.

## Relatórios (PDF)

Padrão do `gestor` (`applications/relatorios`): uma `Tabs` por recorte (Instituição, Escola, Turmas,
...), cada aba com um `DynamicForm` de parâmetros e uma lista de relatórios; gerar abre o PDF em nova
aba.

```ts
// src/modules/gestor/applications/relatorios/views/abas/Instituicao.vue (trecho)
const { gerar, gerando } = useGerarRelatorio();

const itens = computed<ItemRelatorio[]>(() => [
  {
    chave: "totalizacao",
    titulo: "Total de Matrículas das Escolas",
    acao: () =>
      gerar(
        "totalizacao",
        `total-matriculas-${data.value.ano}.pdf`,
        () => getRelatorioTotalizacaoMatriculas(data.value.ano!),
        { 404: "Instituição não encontrada!" },     // mensagem por status, sobrepõe a padrão
      ),
  },
]);
```

- `useGerarRelatorio` (`relatorios/composables/useGerarRelatorio.ts`) — guarda qual relatório está
  sendo gerado (`gerando`), chama o serviço (que devolve `Blob`) e traduz o status de erro em toast.
- `abrirPdfEmNovaAba(blob, nome)` (`gestor/utils/pdf.ts`) — reembala em `application/pdf`, abre em
  nova aba e, se o popup for bloqueado, cai em `baixarBlob`; revoga a URL só depois de 60s.

Os dois vivem no módulo `gestor`; para outro módulo, mova para `src/utils` em vez de importar de
`gestor`. O `escola` tem o seu próprio `abrirRelatorio` em `escola/utils/arquivos.ts`.

## Dialog com formulário — `DialogFields`

Formulário em modal é o `DialogFields`: ele já valida e emite `save` com o `transformData()`.
Usado em `escola/applications/manter-caixa-escolar/views/index.vue`, `components/header/ModalSelecao.vue`,
`diario/.../ModalAvaliacao`.

```vue
<DialogFields
  v-model:open="modalConselho"
  :title="conselhoEditando ? 'Editar Conselho' : 'Adicionar Conselho'"
  :fields="conselhoFields"
  v-model="conselho"
  @save="salvarConselho"
/>
<!-- salvarConselho(payload) recebe o payload já transformado -->

```

O botão Voltar emite `back`. `renderDialogForm` (programático) está **comentado na lib** — não abre
nada; use o componente.

### `Dialog` cru

Só quando o modal não é um formulário único (conteúdo composto, várias tabelas):

```vue
<script setup lang="ts">
import { Dialog, DynamicForm } from "etutor-lib";

const formModal = ref<InstanceType<typeof DynamicForm> | null>(null);

async function onSaveModal() {
  const result = await formModal.value?.validate();
  if (!result?.valid) return;
  salvarItemModal();
}
</script>

<template>
  <!-- o Dialog da lib emite `back`, não `close` (diferente do Somar) -->
  <Dialog class="min-w-[1000px]" v-model:open="openDialog" title="Registrar" @back="openDialog = false" @save="onSaveModal">
    <DynamicForm ref="formModal" class="px-4" :fields="fieldsModal" v-model="itemSelecionado" />
  </Dialog>
</template>
```

### Confirmação programática

```ts
import { renderDialogConfirmation } from "etutor-lib";

const confirmou = await renderDialogConfirmation({
  title: "Confirmar exclusão",
  message: "Deseja realmente excluir este registro?",
});
if (!confirmou) return;
await deleteItem.execute(id);
```

## PageSearchTable

Integra filtros (via `DynamicForm`), tabela paginada e ordenação. A view `index.vue` só configura
colunas e passa as funções.

| Prop | Tipo | Descrição |
|---|---|---|
| `v-model:data` | `ref` | objeto de filtros (controlado por `useFiltroFields`) |
| `:columns` | `ColumnDescriptor[]` | definição das colunas |
| `:fields` | `FormField` | campos do formulário de busca |
| `:queryFn` | `api.getPaginated(...)` ou função `({ params }) => Promise<PaginatedResponseDTO>` | passe direto, sem `useHttpRequest` |
| `addButtonLabel` | `string` | label do botão de criação (omitir esconde o botão) |
| `initialPageSize` | `number` | padrão 10 |
| `autoQuery` | `boolean` | busca ao montar, sem esperar o submit do filtro |
| `extraParams` | `Record<string, any>` | parâmetros fixos somados aos filtros |
| `selectable` / `expanded` / `showActions` | `boolean` | seleção múltipla / linha expansível / coluna de ações (padrão `true`) |
| `showPdfExport` | `boolean` | botão de exportar a tabela em PDF (opt-in) |
| `pdfConfig` | `ExportToPDFOptions` | título, orientação, logo, fonte do PDF |
| `@add` / `@clear` | evento | botão de criação / limpar filtros |

Com `meta.codigo` na rota, o filtro é guardado e restaurado ao voltar para a listagem (ver
[config.md](config.md#rotas-da-application-applicationsapproutests)). O filtro aplicado é congelado
no submit e vira o cabeçalho do PDF exportado (`resumirFiltros`).

A busca inicial já aguarda o `whenReady()` do `DynamicForm` interno antes de disparar — se um campo
do filtro usa `fixedValue` assíncrono, a primeira busca só sai depois dele resolver. Ver
[`fixedValue` assíncrono e `whenReady()`](formularios.md#fixedvalue-assíncrono-e-whenready).

### `ColumnDescriptor`

```ts
interface ColumnDescriptor<T = any> {
  value: string;                              // propriedade no objeto (suporta ponto: "modeloVeiculo.descricao")
  title?: string;                             // omitir na coluna de actions
  sortable?: boolean;                         // ver a ressalva do Pageable acima
  sortByValue?: string;                       // campo de ordenação diferente do value
  defaultSort?: "asc" | "desc";
  mask?: (value: any) => string;              // formata o valor exibido
  largura?: number | string;                  // largura no desktop: número = px, string = CSS ("20%")
  alinhamento?: "left" | "center" | "right";  // cabeçalho + célula no desktop (PDF usa pdf.alignment)
  cellClass?: string | ((row: any) => string);// classe CSS (condicional) — recebe a linha (row.original)
  cellRenderer?: (row: Row<T>) => VNode;      // recebe o Row do TanStack: o dado está em row.original
  actions?: Actions[];                        // coluna de ações (value: "actions")
  show?: boolean;                             // oculta a coluna
  hideOnMobile?: boolean;
  enableHiding?: boolean;
  exportable?: boolean;                       // participa de XLSX/PDF (padrão true)
  pdf?: ColumnPdfOptions;                     // largura, alignment, titulo, noWrap, elemento
}
```

### `Actions` — ações de linha

```ts
interface Actions {
  title?: string | ((event: EventProps) => string);              // rótulo/tooltip
  icon?: IconName;                                                // ícone lucide ("SquarePen", "Trash", "Search", "Download")

  link?: (event: EventProps) => string | RouteLocationRaw;        // navegação
  onClick?: (event: EventProps) => void;                          // handler (pode ser async)
  disable?: (event: EventProps) => boolean;

  confirmation?: boolean;                                         // diálogo de confirmação antes do onClick
  titleConfirmation?: string | ((event: EventProps) => string);
  message?: (event: EventProps) => string;                        // omitido: MENSAGENS.confirmarAcao
}

// EventProps: { rowData, formActions?, index? } — formActions.refetch() recarrega a tabela
```

Use `link` **ou** `onClick`, não os dois. Em `onClick` que altera dados, chame
`formActions?.refetch()` para atualizar a tabela:

```ts
{
  title: "Excluir",
  icon: "Trash",
  confirmation: true,
  titleConfirmation: MENSAGENS.tituloExcluir,
  message: () => MENSAGENS.confirmarExclusao,
  onClick: async ({ rowData, formActions }) => {
    await deleteItem.execute(rowData.id);
    formActions?.refetch();
  },
},
```

### Máscaras em colunas

```ts
{ value: "cpf",          title: "CPF",              mask: maskCPF },
{ value: "instituicao",  title: "Exige instituição", mask: (v) => maskBoolean(v) },
{ value: "tamanho",      title: "Tamanho (M2)",     mask: maskDecimal },
{ value: "lotacoes",     title: "Lotações",         mask: (v) => (v ?? []).map((l: any) => l.unidadeEnsinoNome).join(", ") || "—" },
```

## Exportação

Além do `showPdfExport` do `PageSearchTable` (e de `showXlsxExport`/`showPdfExport` no `Table`), a
lib exporta utilitários para uso direto:

| Função / componente | Uso |
|---|---|
| `exportToXLSX(...)` | exporta linhas para planilha |
| `exportToPDF(...)` | exporta tabela para PDF |
| `exportFormToPDF(...)` | exporta um formulário/detalhe para PDF |
| `gerarPdf`, `cabecalhoPadrao`, `blocoFiltros`, `rodapePadrao`, `estilosPadrao` | montar documento pdfmake sob medida |
| `createColumns`, `getExportableColumns`, `resumirFiltros`, `fetchAllPages` | apoio de colunas/filtros/paginação |
| `<DownloadXlsx />`, `<DownloadPdf />`, `<DownloadFormPdf />` | botões prontos |

Relatório oficial (boletim, ata, totalização...) não é gerado no front: vem pronto em PDF do
`gestao-documentos-ws` — ver [Relatórios](#relatórios-pdf).
