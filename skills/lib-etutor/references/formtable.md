# FormTable — tabelas com mini-formulário

O `FormTable` renderiza uma tabela com um mini-formulário de adição/edição. O mini-formulário abre em
um **modal** (`Dialog`): um botão "Adicionar" acima da tabela abre o modal vazio; a ação "Editar" de
uma linha abre o mesmo modal pré-preenchido com a linha. Submeter fecha o modal automaticamente.

No eTutor ele aparece de **duas formas**:

1. **Como componente standalone com `manualEvents`** — a lista tem endpoints próprios de
   inserção/edição/remoção e cada linha é gravada na hora. **É o padrão para lista com endpoint
   próprio.**
2. **Como campo do `DynamicForm`** (`is: "formTable"`) — a lista é uma propriedade do payload,
   persistida junto com o submit do formulário.

## Props

| Prop | Tipo | Descrição |
|---|---|---|
| `fields` | `FormField` | campos do mini-formulário (obrigatório; passe `{}` para tabela só de leitura) |
| `columns` | `ColumnDescriptor[]` | colunas da tabela (mesma API do `PageSearchTable`: `mask`, `cellClass`, `actions`) |
| `editable` | `boolean` | exibe ação de editar por linha |
| `deletable` | `boolean` | exibe ação de excluir por linha (com confirmação do catálogo) |
| `selectable` | `boolean` | exibe checkbox de seleção múltipla |
| `expanded` | `boolean` | linha expansível |
| `pagination` | `boolean` | paginação no cliente |
| `manualEvents` | `boolean` | o componente **não** mexe na lista — você grava e recarrega via eventos |
| `closeOnSubmit` | `boolean` (default `true`) | com `manualEvents`, `false` mantém o modal aberto após o submit |
| `disabled` | `boolean` | esconde o botão Adicionar e as ações (tabela somente leitura) |
| `label` | `string` | nome da entidade — prefixa botão e título: `label="Perfil"` → "Adicionar Perfil" / "Editar Perfil" |
| `addButtonLabel` / `addTitle` / `editTitle` | `string` | sobrepõem o `label` individualmente |
| `disabledDeleteItem` | `({ rowData }) => boolean` | desabilita o excluir por linha |
| `disabledEditableItem` | `({ rowData }) => boolean` | desabilita o editar por linha |
| `v-model` | `any[]` | itens da tabela |
| `v-model:data` | `object` | estado dos campos do mini-formulário |
| `v-model:selectedItems` | `any[]` | itens selecionados (com `selectable`) |

Expõe `openModal()`/`closeModal()` por `ref`. Slots: `#header` e `#footer` (`{ data }`) e
`#expanded` (`{ data, dataRow }`, com `expanded`). Como campo do `DynamicForm`, `name` é obrigatório.

## Eventos

| Evento | Payload | Quando dispara |
|---|---|---|
| `add` | valores do mini-formulário | submit em modo adição |
| `edit` | `{ data, index }` | submit em modo edição — `data` são os valores do form, `index` a posição da linha |
| `delete` | `{ index, ...linha }` | exclusão confirmada de uma linha |
| `initEdit` | `{ index, rowData }` | clique em editar, depois de copiar a linha para o mini-formulário |
| `clear` | — | sempre que o modal fecha (submit, Limpar, backdrop/Esc). É o único ponto de reset — use para zerar refs auxiliares |
| `update:data` / `update:selectedItems` | | mudança no mini-formulário / na seleção |

> **`edit` recebe `{ data, index }`, não a linha.** O `id` da linha não vem garantido em `data`;
> pegue-o de `lista[index]` (ver `onEditar` abaixo).

> **Sem `manualEvents`, edição é merge, não replace.** A linha final é
> `{ ...linhaAtual, ...valoresDoMiniFormulario }` — propriedades que não são campo do mini-formulário
> (tipicamente `id`) são preservadas. Isso permite **omitir** um campo inteiro no modo edição.

> **Botão "Limpar" só existe no modo adição.**

> **Campos `disabled` continuam sendo validados no submit.** O vee-validate não pula `rules` de campos
> desabilitados — uma regra "não repita este item" dispara contra a própria linha em edição se não
> checar o modo edição.

## Padrão eTutor — standalone com `manualEvents`

Cada linha é gravada/removida na hora no seu endpoint. O legado acumulava "adicionados/excluídos" e
só persistia no submit — a tela mentia até o usuário salvar e perdia tudo se o salvar falhasse. Não
reintroduza esse comportamento.

### Tabela de domínio escolhida num select

`controle-acesso/applications/manter-dominio/views/index.vue`: um `DynamicForm` com o select da
tabela e, embaixo, o `FormTable` dos itens daquela tabela; trocar o select recarrega a lista.

```vue
<script setup lang="ts">
const { getTipos, getItems, postItem, deleteItem, ... } = useRequests();

const tipos = ref<string[]>([]);
const itens = ref<UnidadeMedida[]>([]);

const { fields, data } = useFiltroFields(() => tipos.value);          // select "Tabela"
const ehUnidadeMedida = computed(() => data.value.tipo === TIPO_UNIDADE_MEDIDA);
const tabela = useDominioTable(() => ehUnidadeMedida.value);          // { fields, data, columns }

onMounted(async () => {
  tipos.value = [...((await getTipos.execute()) ?? []), TIPO_UNIDADE_MEDIDA];
  data.value.tipo = tipos.value[0];
});

watch(() => data.value.tipo, recarregar);

async function recarregar() {
  if (!data.value.tipo) return;
  itens.value = (ehUnidadeMedida.value ? await getUnidades.execute() : await getItems.execute(data.value.tipo)) ?? [];
}

async function onAdd(values: any) {
  await postItem.execute(data.value.tipo!, { descricao: values.descricao });
  await recarregar();
}

async function onDelete({ id }: UnidadeMedida) {
  await deleteItem.execute(data.value.tipo!, id);
  await recarregar();
}
</script>

<template>
  <DynamicForm :fields="fields" v-model="data" />

  <FormTable
    v-if="data.tipo"
    name="dominios"
    label="Item"
    :fields="tabela.fields.value"
    :columns="tabela.columns.value"
    v-model="itens"
    v-model:data="tabela.data.value"
    editable
    deletable
    manualEvents
    @add="onAdd"
    @edit="onEdit"
    @delete="onDelete"
  />
</template>
```

O composable da tabela devolve `fields`/`columns` como `computed`, para o mini-formulário mudar de
forma conforme a tabela escolhida (`sigla` só existe para unidade de medida):

```ts
// controle-acesso/applications/manter-dominio/composables/useFields.ts (trecho)
export function useDominioTable(comSigla: () => boolean) {
  const data = ref<Record<string, any>>({});

  const fields = computed<FormField>(() => ({
    descricao: { is: "input", name: "descricao", label: "Descrição", cols: comSigla() ? 9 : 12, isRequired: true },
    ...(comSigla() && {
      sigla: { is: "input", name: "sigla", label: "Sigla", cols: 3, isRequired: true },
    }),
  }));

  const columns = computed<ColumnDescriptor<UnidadeMedida>[]>(() => [
    { value: "descricao", title: "Descrição" },
    ...(comSigla() ? [{ value: "sigla", title: "Sigla" } as const] : []),
  ]);

  return { fields, data, columns };
}
```

### Adicionar, editar e excluir atualizando a lista local

`gestor/applications/manter-veiculo/views/manutencao.vue` — o `POST`/`PUT` devolvem a linha salva,
então a lista é atualizada com o retorno, sem recarregar:

```ts
async function onAdicionar(valores: ManutencaoFormData) {
  const salvo = await postManutencao.execute(veiculoId, valores as unknown as ManutencaoVeiculoRequest);
  if (!salvo) return;
  linhas.value = [...linhas.value, salvo];
}

async function onEditar({ data: valores, index }: { data: ManutencaoFormData; index: number }) {
  const linhaAtual = linhas.value[index];
  if (!linhaAtual?.id) return;
  const salvo = await putManutencao.execute(String(linhaAtual.id), valores as unknown as ManutencaoVeiculoRequest);
  if (!salvo) return;
  linhas.value = linhas.value.map((linha, i) => (i === index ? salvo : linha));
}

async function onExcluir({ index, id }: { index: number; id?: number }) {
  if (!id) return;
  await deleteManutencao.execute(String(id));
  linhas.value = linhas.value.filter((_, i) => i !== index);
}

// O FormTable copia a linha crua (data ISO) para o form; o InputDate espera dd/MM/yyyy
function onInitEdit({ rowData }: { rowData: ManutencaoVeiculoDetalhe }) {
  data.value.data = isoParaBr(rowData.data);
}
```

```vue
<FormTable
  name="manutencoes"
  label="manutenção"
  addButtonLabel="Adicionar"
  :fields="fields"
  :columns="columns"
  v-model="linhas"
  v-model:data="data"
  editable
  deletable
  manualEvents
  @add="onAdicionar"
  @edit="onEditar"
  @delete="onExcluir"
  @initEdit="onInitEdit"
/>
```

Anexo dentro do mini-formulário: `views/documentos.vue` da mesma application usa um campo
`formFileManager` com `transform` que devolve o `identificador` (ver
[http.md](http.md#upload-de-arquivo-s3)).

### Dois FormTable dependentes (mestre-detalhe)

`controle-acesso/applications/manter-usuario/views/abas/Vinculos.vue`: a primeira tabela
(`selectable`) escolhe o vínculo perfil+instituição; a segunda mostra as lotações dele. Validação que
o back-end devolveria como erro cru fica no `onAdd`, com `toast.showWarning`:

```ts
async function onAddVinculo(values: any) {
  const perfilId = Number(values.perfilId);
  const instituicaoId = Number(values.instituicaoId);

  const repetido = vinculos.value.some((v) => v.perfilId === perfilId && v.instituicaoId === instituicaoId);
  if (repetido) {
    toast.showWarning("Este perfil já está vinculado a esta instituição.");
    return;
  }

  await postVinculo.execute(props.usuarioId, { perfilId, instituicaoId });
  await recarregar();
}
```

```vue
<Title text="Perfis" type="subtitle" />
<FormTable
  name="vinculos"
  label="Perfil"
  :fields="vinculoTable.fields.value"
  :columns="vinculoTable.columns"
  v-model="vinculos"
  v-model:data="vinculoTable.data.value"
  v-model:selectedItems="selecionados"
  selectable
  deletable
  manualEvents
  @add="onAddVinculo"
  @delete="onDeleteVinculo"
/>

<FormTable
  v-if="vinculoAtual"
  name="lotacoes"
  label="Lotação"
  :fields="lotacaoTable.fields.value"
  :columns="lotacaoTable.columns"
  :modelValue="lotacoes"
  v-model:data="lotacaoTable.data.value"
  deletable
  manualEvents
  @add="onAddLotacao"
  @delete="onDeleteLotacao"
/>
```

Um composable por tabela (`useVinculoTable`, `useLotacaoTable` em
`composables/useFieldsVinculo.ts`), devolvendo `{ fields, data, columns }`.

Pontos do padrão:

- `manualEvents` + `@add`/`@edit`/`@delete`, cada um chamando o seu endpoint e depois recarregando
  (ou aplicando o retorno do back-end na lista).
- O `delete` recebe `{ index, ...linha }`: desestruture a chave que o endpoint precisa e saia cedo se
  ela não vier.
- Cuidado com **qual id** o endpoint espera (o da linha filha, não o da entidade da rota).

### Tabela somente leitura

`disabled` esconde o botão Adicionar e as ações; `fields` pode ser `{}`:

```vue
<FormTable name="unidadesFilhas" :fields="{}" :columns="colunas" v-model="unidadesFilhas" disabled />
```

## Como campo do `DynamicForm`

Quando a lista faz parte do payload da entidade (não tem endpoint próprio). Exemplo real:
`escola/applications/manter-estrutura-escola/views/tabs/dependencias.vue`, persistido no `POST` único
da estrutura:

```ts
dependenciaEscola: {
  is: "formTable",
  name: "dependenciaEscola",
  events: {
    add: (item: any) => add(item),
    delete: ({ index }: any) => deletar(index),
    edit: (item: any) => edit(item.index, item.data),
    initEdit: ({ rowData }) => initEdit(rowData),
  },
  props: {
    deletable: !props.disabled,
    editable: !props.disabled,
    manualEvents: true,
    disabled: props.disabled,
    columns: [
      { value: "dependenciaDescricao", title: "Descrição" },
      { value: "quantidade", title: "Quantidade" },
      { value: "tamanho", title: "Tamanho Individual (M2)", mask: maskDecimal },
    ],
    fields: {
      dependencia: {
        is: "select", name: "dependencia", label: "Dependência", isRequired: true, cols: 6,
        flatten: "merge",
        transform(val: GenericSelect) {
          if (!val) return val;
          return { dependenciaId: val.id, dependenciaDescricao: val.descricao, dependencia: val };
        },
        props: {
          fnOptions: getSelectTipoDependencia, returnObject: true, optionLabel: "descricao", optionValue: "id",
          // não repetir dependência já na lista, exceto a da linha em edição
          optionDisabled: (item) =>
            (item.id !== itemEdit.value?.dependenciaId &&
              model.value.dependenciaEscola.some(({ dependenciaId }) => dependenciaId === item.id)) || false,
        },
      },
      // ...
    },
  },
},
```

Aqui o `manualEvents` serve só para a tela controlar a lista local (`add`/`edit`/`deletar` mexem no
`model`); nada vai ao back-end até o salvar da estrutura.

### Modo edição com campos diferentes

`initEdit` liga a flag; os `fields` reagem a ela; `clear` desliga:

```ts
events: {
  initEdit({ rowData }) { idItemEdicao.value = rowData?.id; editItem.value = true; },
  clear() { dataItens.value = {}; editItem.value = false; idItemEdicao.value = ""; },
},
```

### Seleção múltipla

```ts
props: { editable: false, deletable: false, selectable: true, selectedItems: selectedItems.value, columns: [...] },
events: { "update:selectedItems"(items: any[]) { selectedItems.value = items; } },
```
