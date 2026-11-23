# Formulários — DynamicForm, useFields e selects

O padrão principal de formulário é o `DynamicForm` com `FormField`. **Não** se usam componentes
individuais como `<InputText>` no template.

Os composables ficam em `composables/`. Cada `useFields` pode receber parâmetros opcionais
(`_id`, `_disabled`) e serve para as views de cadastrar/editar/detalhar. O carregamento do dado é
feito **dentro do composable** via `onMounted`.

Uma application com abas ou seções tem **um composable por aba/seção** e composables auxiliares só
para as tabelas (ex.: `useVinculoTable()`/`useLotacaoTable()` em
`controle-acesso/applications/manter-usuario/composables/useFieldsVinculo.ts`, devolvendo
`{ fields, data, columns }`).

## Tipos de campo (`is`)

Lista do `componentsMap` (`lib-etutor/src/components/lib/fields/FormTypes.ts`):

| Valor | Componente | Observação |
|---|---|---|
| `"input"` | InputText | aceita `props.mask` (Maska): `mask: "#####-###"`, ou array de máscaras |
| `"textArea"` | InputTextArea | |
| `"inputCode"` | InputCode | |
| `"inputCPF"` | InputCPF | |
| `"inputCNPJ"` | InputCNPJ | |
| `"inputCPFCNPJ"` | InputCPFCNPJ | |
| `"inputMatricula"` | InputMatricula | |
| `"inputCurrency"` | InputCurrency | |
| `"inputQuantity"` | InputQuantity | `props.precision: 0` para `Integer` no back-end |
| `"inputDate"` | InputDate | trabalha em `dd/MM/yyyy`; back-ends em `yyyy-MM-dd` → use `isoParaBr`/`brParaIso` |
| `"inputDateRange"` | InputDateRange | |
| `"inputCoordinate"` | InputCoordinate | |
| `"inputIcon"` | InputIcon | **seletor de ícone** lucide (o valor é o nome do ícone). Ícone clicável dentro do input é `is: "input"` com `props: { icon: "Search" }` + `events: { "click:icon": ... }` |
| `"inputFile"` | InputFile | o valor é um `File` — não sobe nada sozinho |
| `"inputImagePreview"` | InputImagePreview | upload com prévia de imagem |
| `"select"` | SelectField | |
| `"selectPagination"` | SelectPaginationField | select com busca por texto + scroll infinito, para listas grandes |
| `"combobox"` | SelectCombobox | |
| `"checkbox"` | CheckboxField | um booleano |
| `"checklist"` | ChecklistField | lista de checkboxes com `fnOptions` — o valor é um **array de ids** |
| `"button"` | Button | botão inline no formulário |
| `"titulo"` | Title | separador visual / subtítulo (`props: { text, type: "subtitle" }`) |
| `"formTable"` | FormTable | tabela com mini-formulário em modal — ver [formtable.md](formtable.md) |
| `"formFileManager"` | FormFileManager | anexos com upload direto no S3 (`gestao-documentos-ws`) — ver [http.md](http.md#upload-de-arquivo-s3) |
| `"tablePagination"` | FormTablePagination | tabela inline com paginação |
| `"slot"` | SlotFields | conteúdo customizado via slot |
| `"generic"` | GenericField | campo genérico (renderiza componente arbitrário) |

Tipos que existem no mapa mas **não funcionam ou não se usam no eTutor** — não use em código novo:
`formPerson`, `formPersonSelect`, `formPersonSigilosa`, `botaoSigiloso`, `inputSigiloso` (contratos
do Somar), `selectGeneric` (consulta `/api-commons`, da modernização anterior) e `formFileIndividual`
(não sobe arquivo). `formPessoa` e `formAddress` funcionam, com o contrato descrito abaixo. Ver o
`SKILL.md`.

## `formPessoa` — religado por adaptadores

Escolher uma pessoa já cadastrada (CPF + busca paginada por nome) é `formPessoa` com
`sigiloso: false` (não há revelação de dado sigiloso no eTutor) e as duas buscas **trocadas** por
funções do módulo — sem elas, o componente chama o `cadastro-geral-ws` do Somar.

```ts
// src/modules/professor/applications/dados-funcionais/views/cadastrar.vue (trecho)
pessoa: {
  is: "formPessoa", name: "pessoa", cols: 12, isRequired: true,
  flatten: "merge",
  transform: (val: any) => ({ pessoaId: val?.id }),
  props: {
    titulo: "Servidor",
    selected: true,             // nome vira busca paginada + CPF busca ao digitar
    sigiloso: false,
    cadastrarNome: false,
    tipoDocumento: "cpf",
    labelNome: "Nome",
    fnBuscarPessoa: buscarPessoaPorCpf,   // ({ cpfCnpj }) => pessoa | null
    fnBuscarPorNome: buscarPessoaPorNome, // (params) => página de pessoas
    extraFields: { nome: { order: 20 }, cpfCnpj: { order: 30 } },
  },
},
```

Os adaptadores (`professor/services/servidor-ws/pessoa.ts`) chamam o back-end do módulo e devolvem
cada pessoa no formato que o componente espera: `{ id, nome, cpfCnpj, telefone, email }`. Eles
recebem os params **direto** (não o wrapper `{ params }` do `getPaginated`).

## `formAddress`

Reescrito para o eTutor: coordenada (mapa) + CEP, logradouro, número, complemento e bairro, em texto
livre. CEP consulta o viacep; um ponto no mapa preenche CEP/logradouro/número/bairro. Estado e
cidade, quando a tela tiver, são campos próprios com o service do módulo. Precisa do `MapsPlugin`.

```ts
// src/modules/controle-acesso/applications/manter-instituicao/composables/useFields.ts
endereco: {
  is: "formAddress", name: "endereco", disabled: _disabled,
  flatten: "merge",
  transform: ({ coordenadas, cep, ...resto }: Record<string, any> = {}) => ({
    ...resto,
    cep: removeMaskCEP(cep) || undefined,
    latitude: coordenadas ? String(coordenadas.lat) : undefined,
    longitude: coordenadas ? String(coordenadas.lng) : undefined,
  }),
},
```

`props.requiredFields` (`{ cep: true, numero: true, ... }`) marca os subcampos obrigatórios. Ao
aplicar o detalhe, monte `data.value.endereco = { coordenadas: { lat, lng }, cep, logradouro, ... }`.

## Propriedades disponíveis em todo campo (`FieldConfig`)

```ts
interface FieldConfig {
  is: Fields;                                 // tipo do campo (obrigatório)
  name?: string;                              // nome da chave no objeto de dados
  label?: string;
  cols?: number;                              // colunas no grid (1–12)
  disabled?: boolean;
  isRequired?: boolean | Partial<Record<string, boolean>>;
  placeholder?: string;
  options?: any[];                            // opções estáticas para select
  props?: Partial<PropsFields>;               // props extras passadas ao componente
  events?: Record<string, Function>;          // listeners (ex.: "update:modelValue")
  rules?: Array<(val: any) => true | string>; // validações customizadas
  transform?: (val: any) => any;              // transforma o valor no transformData()
  fixedValue?: unknown | (() => unknown | Promise<unknown>); // valor fixo/pré-selecionado (pode ser assíncrono)
  flatten?: boolean | "merge";                // funde sub-objeto no objeto pai
  render?: boolean;                           // exibe/oculta o campo — só boolean (ver abaixo)
  order?: number;                             // posição em forms com extraFields
  class?: string;                             // classe CSS no componente do campo
  classField?: string;                        // classe CSS na <div> do grid que envolve o campo
  key?: string | number;                      // forçar re-mount do campo
  slot?: string;                              // nome do slot (is: "slot") — no DynamicForm o slot é `${name}:${slot}`
  suffix?: string;
}
```

> **`render` só aceita boolean.** O `DynamicForm` testa `field.render !== false`; uma função nunca é
> chamada e o campo fica sempre visível. Como `useFields` devolve um `computed`, escreva
> `render: data.value.estadoCivil === "CASADO"` — o boolean é recalculado a cada mudança.

As linhas do grid somam 12 colunas: o projeto ajusta `cols` para cada linha ocupar as 12 (commits de
`aluno` e `professor`).

### `transform` e `transformData()`

`formRef.value?.transformData()` monta o payload aplicando o `transform` de cada campo:

- `transform: () => undefined` → o campo **sai** do payload (o JSON do axios descarta `undefined`).
  É como se declara campo auxiliar de tela (ex.: `estado`, que só cascateia a cidade; `instituicao`
  só exibido no form de veículo) e relação com endpoint próprio.
- `flatten: "merge"` → funde as chaves do sub-objeto direto no pai.
- `flatten: true` → funde com prefixo camelCase (`dataContrato` + `inicio` → `dataContratoInicio`).

Conversões de tipo ditadas pelo back-end:

```ts
// gestor/applications/manter-veiculo/composables/useFieldsVeiculo.ts
placa: {
  is: "input", name: "placa", label: "Placa", cols: 3, isRequired: true,
  rules: [validarPlaca],
  transform: (val: string) => val?.toUpperCase() || undefined,
},

// anexo: persiste só o identificador do S3
caminho: {
  is: "formFileManager", name: "caminho", label: "Arquivo", cols: 12, isRequired: true,
  props: { deletable: true, maxFiles: 1 },
  transform: (val: Array<{ identificador: string }>) => val?.[0]?.identificador,
},
```

Boolean primitivo no request (não aceita `null`) → `is: "checkbox"` com `transform: (v) => !!v`, não
select Sim/Não de três estados.

Para tirar máscara também há `removeMaskCPF`, `removeMaskCNPJ`, `removeMaskCEP`, ... na lib.

## `composables/useFields.ts` — padrão

```ts
// src/modules/gestor/applications/manter-veiculo/composables/useFieldsVeiculo.ts (trecho)
import { computed, onMounted, ref } from "vue";
import { validarPlaca, validarRenavam, type FormField } from "etutor-lib";
import { useRequests } from "../useRequests";
import { getInstituicaoSessao } from "@/core/sessao/session";
import type { VeiculoDetalhe, VeiculoRequest } from "../../../types/veiculo";

export function useFieldsVeiculo(_id?: string, _disabled?: boolean) {
  const { getVeiculo, getSelectModeloVeiculo, getSelectTransporteEscolar } = useRequests();

  const data = ref<Partial<VeiculoRequest>>({});

  const fields = computed<FormField>(() => ({
    instituicao: {
      is: "input", name: "instituicao", label: "Instituição", cols: 12, disabled: true,
      fixedValue: () => getInstituicaoSessao()?.nome,   // só exibição; o id vai no payload pela view
      transform: () => undefined,
    },
    placa: {
      is: "input", name: "placa", label: "Placa", cols: 3, isRequired: true, disabled: _disabled,
      rules: [validarPlaca],
      transform: (val: string) => val?.toUpperCase() || undefined,
    },
    modeloVeiculoId: {
      is: "select", name: "modeloVeiculoId", label: "Modelo", cols: 6, isRequired: true, disabled: _disabled,
      props: { fnOptions: getSelectModeloVeiculo, optionLabel: "descricao", optionValue: "id" },
    },
    // ...
  }));

  /** Traduz o detalhe do back-end para as chaves do formulário. */
  function aplicarVeiculo(veiculo: VeiculoDetalhe) {
    Object.assign(data.value, veiculo, {
      modeloVeiculoId: veiculo.modeloVeiculo?.id,
      transporteEscolarId: veiculo.transporteEscolar?.id,
      instituicao: getInstituicaoSessao()?.nome,
    });
  }

  onMounted(async () => {
    if (!_id) return;
    const resultado = await getVeiculo.execute(_id);
    if (resultado) aplicarVeiculo(resultado);
  });

  return { fields, data };
}
```

> **Request e response quase nunca têm as mesmas chaves.** O detalhe traz objetos aninhados
> (`modeloVeiculo.id`) e o request espera ids achatados (`modeloVeiculoId`); no `aluno-ws` o CPF vem
> na raiz na listagem e em `documentacao.cpf` no detalhe. Tipe request e response separadamente
> (conferindo no api-docs) e centralize a tradução numa função de aplicar o detalhe. Divergências
> que não são óbvias vão em `docs/<módulo>.md` (skill `etutor:comentarios`).

### Campo condicional

`render` some com o campo; `disabled` mantém visível. O critério pode ler o `data` do select já
carregado (`getSelectX.data.value`):

```ts
// src/modules/professor/applications/dados-pessoais/composables/useFieldsPessoa.ts (trecho)
const isCasado = computed(() => {
  const opcoes = getSelectEstadoCivil.data.value ?? [];
  const selecionado = opcoes.find((o) => o.id === data.value.estadoCivilId);
  const descricao = selecionado?.descricao?.toLowerCase() ?? "";
  return descricao.includes("casado") || descricao.includes("uni");
});

conjuge: {
  is: "select", name: "conjuge", label: "Cônjuge", cols: 3, disabled: _disabled,
  render: isCasado.value,
  options: OPCOES_SIM_NAO,
  props: { optionLabel: "descricao", optionValue: "id" },
},
```

Regra que depende da **descrição** de um domínio, como esta, é frágil — registre em
`docs/<módulo>.md` (este caso está em `docs/professor.md`).

## `composables/useFiltroFields.ts` — busca/listagem

Sem `isRequired` (salvo quando o back-end exige o filtro). **Só coloque no filtro o que o back-end
aceita** — parâmetro que o back-end ignora dá a impressão de filtro que não filtra:

```ts
// src/modules/gestor/applications/manter-veiculo/composables/useFiltroFields.ts
export function useFiltroFields() {
  const { getSelectModeloVeiculo } = useRequests();

  const data = ref<{ placa?: string; renavam?: string; modeloVeiculoId?: number }>({});

  const fields = computed<FormField>(() => ({
    placa: { is: "input", name: "placa", label: "Placa", cols: 4 },
    renavam: { is: "input", name: "renavam", label: "Renavam", cols: 4, props: { mask: "###########" } },
    modeloVeiculoId: {
      is: "select", name: "modeloVeiculoId", label: "Modelo", cols: 4,
      props: { fnOptions: getSelectModeloVeiculo, optionLabel: "descricao", optionValue: "id" },
    },
  }));

  return { fields, data };
}
```

## `fixedValue` assíncrono e `whenReady()`

`fixedValue` aceita uma função (inclusive assíncrona, `() => Promise<unknown>`) — útil quando o valor
fixo vem da sessão ou de uma consulta. O `DynamicForm` resolve, aplica o valor no campo e o trava.

O `DynamicForm` expõe (via `defineExpose`) o método **`whenReady(): Promise<void>`**, que resolve
quando a primeira aplicação de `fixValues()` termina — inclusive os `fixedValue` assíncronos. Quem
dispara uma busca automática ao montar, **fora** do fluxo do `PageSearchTable`, precisa aguardar
`whenReady()` antes de chamar `validate()`/disparar a query:

```ts
async function buscarComFiltro() {
  await formRef.value?.whenReady?.();     // aguarda o fixedValue assíncrono resolver
  const result = await formRef.value?.validate();
  if (!result?.valid) return;
  onSubmit(formRef.value?.transformData());
}
```

> O `PageSearchTable` já aguarda `whenReady()` internamente antes da primeira busca. As funções
> prontas de valor fixo do Somar (`utils/fixedValues`) **não existem na `etutor-lib`** — o valor vem
> da sessão (`src/core/sessao`).

## Select — `fnOptions`

O campo `is: "select"` busca opções automaticamente via `fnOptions` em `props`. A instância
`useHttpRequest` fica em `useRequests`; o composable apenas a desestrutura e repassa — **sem
`execute()` manual**.

| Prop | Tipo | Descrição |
|---|---|---|
| `fnOptions` | `UseHttpRequestReturn` | instância `useHttpRequest` — o componente chama `.execute()` sozinho |
| `optionLabel` | `string` | propriedade usada como rótulo (suporta ponto: `"pessoa.nome"`) |
| `optionValue` | `string` | propriedade usada como valor armazenado |
| `optionDisabled` | `(item) => boolean` | desabilita opções (ex.: já presentes na lista) |
| `fnParams` | `any` | **um único argumento** do `execute` (array **não** é espalhado). Não é observado: mudou → troque a `key` |
| `returnObject` | `boolean` | armazena o objeto inteiro em vez de só `optionValue` |
| `multiple` | `boolean` | seleção múltipla |

Opções fixas vão em `options` (ou `props.options`) sem `fnOptions` — ex.: o select "Tabela" de
`controle-acesso/applications/manter-dominio/composables/useFields.ts`.

### Básico

```ts
// useRequests.ts — select sem callbacks
getSelectModeloVeiculo: useHttpRequest(getSelectModeloVeiculo),

// useFields.ts
modeloVeiculoId: {
  is: "select", name: "modeloVeiculoId", label: "Modelo", cols: 6, isRequired: true,
  props: { fnOptions: getSelectModeloVeiculo, optionLabel: "descricao", optionValue: "id" },
},
```

### Cascata (select dependente) — `key` + `fnParams`

O `SelectField` carrega no `onMounted`, então **quem dispara o recarregamento é o re-mount**: mude a
`key` quando o parâmetro mudar.

```ts
// src/modules/gestor/applications/monitorar-rede/composables/useFiltroFieldsTurmas.ts (trecho)
ano: {
  is: "select", name: "ano", label: "Ano Letivo", cols: 3, isRequired: true,
  options: anosLetivos.value,
  props: { optionLabel: "descricao", optionValue: "id" },
},
idAnoLetivo: {
  is: "select", name: "idAnoLetivo", label: "Escola", cols: 6, isRequired: true,
  disabled: !data.value.ano,
  key: data.value.ano ?? "sem-ano",                 // remonta ao trocar o ano
  props: {
    fnOptions: getSelectEscolasDoAno,
    fnParams: data.value.ano,
    optionLabel: "nome",
    optionValue: "ano",
  } as any,
},
```

Quando o endpoint dependente não deve ser chamado enquanto a dependência não existe, guarde na
função de serviço (`if (!ano) return []`) ou troque a própria função:
`fnOptions: data.value.pai ? getSelectFilho : async () => []`.

### `returnObject: true`

Quando o formulário precisa de vários campos do item selecionado. Combine com `flatten: "merge"` (e
um `transform` que monte as chaves) para fundir no objeto pai — ex.: o campo `dependencia` em
`escola/applications/manter-estrutura-escola/views/tabs/dependencias.vue` (ver
[formtable.md](formtable.md#como-campo-do-dynamicform)).

### `selectPagination` — lista grande com busca

O componente chama `fnOptions.execute({ ...fnParams, nome, page, size })` e lê `content` da página
devolvida. Passe uma **instância `useHttpRequest`** — função crua não tem `.execute` e o select fica
vazio.

```ts
// src/modules/gestor/applications/relatorios/composables/useFieldsRelatorios.ts (trecho)
idServidor: {
  is: "selectPagination",
  name: "idServidor",
  label: "Professor",
  cols: 6,
  isRequired: true,
  disabled: !data.value.ano,
  key: data.value.ano ?? "sem-ano",
  props: {
    fnOptions: getSelectProfessores,            // useHttpRequest(getProfessoresParaSelect)
    fnParams: { ano: data.value.ano, idInstituicao: instituicaoId() ?? undefined },
    optionLabel: "nome_servidor",
    optionValue: "codigo_servidor",
  } as any,
},
```

Se o back-end rejeita o termo vazio (`@Size(min = 5)`), a função de serviço devolve página vazia —
ver [http.md](http.md#validação-do-back-end-que-quebra-o-select).

### `checklist` — relação N:N

O valor é um **array de ids**. Quando a relação tem endpoint próprio, ela não vai no PUT da entidade
(`transform: () => undefined`). Forma do campo (ilustrativa):

```ts
programas: {
  is: "checklist",
  name: "programas",
  label: "Programas",
  cols: 6,
  disabled: _disabled,
  props: { fnOptions: getSelectPrograma, optionLabel: "descricao", optionValue: "id" },
  transform: () => undefined,          // gravada por endpoint próprio
},
```

Props extras do `ChecklistField` para quando o model não é uma lista de ids crus:
`itemValue` (caminho do valor dentro de cada item, ex.: `"capacidadeFuncao.id"`) e `itemDefaults`
(campos extras gravados no item ao marcar a opção).

O layout do checklist é `flex-wrap` fixo; com muitas opções o `aluno` usa uma grade própria
(`aluno/applications/manter-aluno/views/components/RecursosInepChecklist.vue`).

### Dentro de `formTable`

`fnOptions` funciona igual nos campos do mini-formulário — ver [formtable.md](formtable.md).
