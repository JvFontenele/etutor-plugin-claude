# Rotas, módulo, sessão, Toast, Plugins, .env e Utils

## Rotas

O registro é automático em dois níveis via `import.meta.glob` (`src/router/index.ts` agrega
`modules/**/router/index.ts`; cada módulo agrega `applications/**/routes.ts`) — **nunca registre rota
manualmente**. Não existem `meta.application` nem `/acesso-negado`: o acesso é decidido pelo guard
de `src/router/index.ts` a partir do `modulo.ts` (ver abaixo).

### Router do módulo (`modules/<modulo>/router/index.ts`)

Exporta um **array** de rotas (as applications já declaram o path absoluto):

```ts
import type { RouteRecordRaw } from "vue-router";

const applicationRoutes = import.meta.glob<RouteRecordRaw | RouteRecordRaw[]>(
  "../applications/**/routes.ts",
  { eager: true, import: "default" },
);

const routes: RouteRecordRaw[] = [];
for (const mod of Object.values(applicationRoutes)) {
  Array.isArray(mod) ? routes.push(...mod) : routes.push(mod);
}

export default routes;
```

### Rotas da application (`applications/<app>/routes.ts`)

```ts
// src/modules/controle-acesso/applications/manter-usuario/routes.ts
import type { RouteRecordRaw } from "vue-router";
import { Page } from "etutor-lib";
import LayoutEtutor from "@/layouts/LayoutEtutor.vue";

const routes: RouteRecordRaw[] = [
  {
    path: "/controle-acesso/usuarios",
    component: Page,
    meta: { layout: LayoutEtutor, title: "Usuários", modulo: "controle-acesso" },
    children: [
      { path: "",           name: "manter-usuario",           meta: { title: "Consultar Usuário" }, component: () => import("./views/index.vue") },
      { path: "cadastrar",  name: "manter-usuario-cadastrar", meta: { title: "Cadastrar Usuário" }, component: () => import("./views/cadastrar.vue") },
      { path: ":id/editar", name: "manter-usuario-editar",    meta: { title: "Editar Usuário" },    component: () => import("./views/editar.vue") },
    ],
  },
];

export default routes;
```

Não são opcionais:

1. **`component: Page`** na rota-pai. Além do breadcrumb e do título, é o `Page` que renderiza o
   `<div id="page-group-buttons">` alvo do teleport do `PageGroupButtons`. Sem ele os botões de
   Voltar/Salvar **não aparecem**.
2. **`meta.modulo`** na rota-pai — o **nome da pasta** do módulo. É por ele que o `LayoutEtutor`
   mostra o rail do módulo e que o guard checa os `perfis` do `modulo.ts`. Rota sem `meta.modulo`
   não passa pela checagem de perfil.
3. **`meta.layout`** — o `App.vue` lê `currentRoute.meta.layout` (com `LayoutEtutor` como default).
   `meta: { layout: "div" }` para tela sem shell (as do portal); o Portal do Aluno usa
   `LayoutPortalAluno`.
4. **`meta.title`** em cada filho — alimenta o breadcrumb e o `document.title`. O `title` da rota-pai
   (`route.matched[0]`) vira o título do módulo no header.

O **`name`** das rotas segue o prefixo da application (`manter-usuario`, `manter-usuario-editar`): o
rail marca como ativo o item cujo `rota` é igual ao nome atual ou prefixo dele (`<rota>-...`).

`meta` opcionais:

| Campo | Efeito |
|---|---|
| `codigo` | Só persiste o filtro do `PageSearchTable` entre navegações da mesma application (`useFilterStore`); trocar de `codigo` limpa os filtros. Não é controle de acesso. Ex.: `aluno/applications/manter-aluno/routes.ts` (`codigo: "manter-aluno"`) |
| `perfis` | Restringe a rota a perfis específicos, além do módulo (`diario/applications/manter-diario/routes.ts`, `somenteDocente`) |
| `exigeSessao: false` | Rota pública — o guard não exige login (portal, `/callback`, `/recuperar-senha`) |
| `tipoAmbiente: "aluno"` | Rota do Portal do Aluno (`portal-aluno`) |

Prefira **import nomeado** (`import { Page } from "etutor-lib"`) a
`() => import("etutor-lib").then((m) => m.Page)`: a lib já é importada estaticamente por
`App.vue`/`main.ts`, então o import dinâmico não gera chunk separado — só um aviso do Rollup.

### Guard

`src/router/index.ts`: toda rota exige token OAuth2 (`useAuthStore().isAuthenticated`), salvo
`meta.exigeSessao: false`; sem token chama `auth.login(to.fullPath)`. Autenticado, redireciona para
`portal-modulos` se não há perfil ativo ou se o perfil não está em `modulo.perfis`, `meta.perfis` ou
`aplicacao.perfis`. Regras completas: `docs/header.md`.

> ⚠️ `aplicacao.perfis` só é conferido quando `to.name === aplicacao.rota` — **a rota de entrada**.
> As sub-rotas (`-cadastrar`, `-editar`, `:id/...`) escapam. Aplicação mais restrita que o módulo
> precisa também de `meta.perfis` na rota-pai (o `Page`), que vale para todos os filhos.

### `Page` — colapso automático de rotas aninhadas

Por padrão o `Page` **colapsa a cadeia de rotas aninhadas**: se a rota tem uma folha mais profunda
que a view imediatamente filha (`Page` → view intermediária → sub-view), o `<RouterView />` interno
do `Page` monta a **folha diretamente**, pulando os níveis do meio — a view intermediária não
precisa declarar seu próprio `<RouterView />` para a sub-rota aparecer.

Caso real: em `escola/applications/manter-escola/routes.ts`, `cadastrar`, `:id/editar`,
`:id/arquivos`, ... são **filhas** da rota da listagem (`index.vue`, sem `<RouterView />`), e mesmo
assim a seção aparece sozinha na tela.

| Prop | Tipo | Descrição |
|---|---|---|
| `aninharFilhos` | `boolean` (default `false`) | Desliga o colapso e volta ao `<RouterView />` plano (um nível por vez). Use quando a view intermediária precisa desenhar algo em volta do filho (abas, cabeçalho de registro) — aí é ela que declara o próprio `<RouterView />` |
| `title` / `items` / `rootLabel` / `rootHref` | | Título e breadcrumb explícitos (sem eles, derivados das rotas casadas) |
| `noRouterView` | `boolean` | Renderiza o slot em vez de `<RouterView />` |

Com import nomeado, a prop vai no registro da rota: `{ path: "/x", component: Page, props: { aninharFilhos: true }, ... }`.

### Fallback 404

Fica só no módulo `portal` (`modules/portal/router/index.ts`), como rota global — não repita por módulo:

```ts
{
  path: "/:pathMatch(.*)*",
  name: "global-not-found",
  meta: { layout: LayoutEtutor, exigeSessao: false },
  component: PageNotFound,
}
```

## Módulo e menu (`modulo.ts`)

Não existe lista central de módulos nem menu no `App.vue`. Cada módulo exporta um `ModuloConfig` em
`src/modules/<modulo>/modulo.ts`; `src/core/modulos/menus.ts` agrega todos via
`import.meta.glob("../../modules/*/modulo.ts")` (o id do módulo é o nome da pasta) e expõe
`getModulo`, `aplicacoesVisiveis` e `modulosVisiveis`. Módulo novo aparece no portal e no rail
sozinho — não edite `menus.ts`, o portal nem o `modulo.ts` de outro módulo.

```ts
// src/modules/controle-acesso/modulo.ts
import type { ModuloConfig } from "@/core/modulos/modulo.types";
import { Perfil } from "@/core/sessao/perfis";

const modulo: ModuloConfig = {
  nome: "Controle de Acesso",
  descricao: "Usuários, perfis, instituições e tabelas de domínio.",
  icon: "ShieldCheck",
  perfis: [Perfil.ANALISTA, Perfil.ADMINISTRADOR],

  aplicacoes: [
    { titulo: "Usuários", icon: "Users", rota: "manter-usuario" },
    { titulo: "Perfis", icon: "ShieldCheck", rota: "manter-perfil" },
    { titulo: "Instituições", icon: "Building2", rota: "manter-instituicao" },
    { titulo: "Domínios", icon: "Table", rota: "manter-dominio" },
  ],
};

export default modulo;
```

- `rota` é o **nome** da rota de entrada da application; `icon` é um ícone lucide.
- `perfis` do módulo é **obrigatório na prática**: lista vazia ou ausente **bloqueia** todo mundo
  (`temAcesso` exige lista não vazia). Convenção: `Perfil.ANALISTA` primeiro.
- `aplicacoes[].perfis` só restringe mais; ausente herda o módulo.
- `exigeContextoEscolar: true` faz o header abrir a seleção de escola/ano letivo e esconder a tela
  até haver escola na sessão (hoje só `diario`).
- Módulo sem nenhuma aplicação visível não aparece no portal.

`Perfil` (`src/core/sessao/perfis.ts`) espelha os ids de `ca03_perfil` do `controle-acesso-ws`
(`ANALISTA`, `DOCENTE`, `DIRETOR`, `ADMINISTRADOR`, `ALUNO`) — acesso é decidido **por id**, nunca
pela descrição.

## Sessão e `instituicaoId`

A sessão é app-level, em `src/core/sessao/` (fora dos módulos). Use as funções, nunca leia o
`localStorage` direto:

| Função | Uso |
|---|---|
| `getInstituicaoId()` (`core/sessao/instituicao.ts`) | `instituicaoId` exigido pelos cadastros (escola, veículo, transporte, calendário...) — ex.: `gestor/applications/manter-veiculo/views/cadastrar.vue` |
| `getSessao()` / `getInstituicaoSessao()` (`core/sessao/session.ts`) | sessão reativa (usuário, perfil, instituição, unidade, ano letivo) e instituição selecionada |
| `getContextoEscolar()` / `getPerfilSessao()` | contexto escolar e perfil ativos |

Não existe mais `INSTITUICAO_ID` em `constants.ts` de módulo: o valor vem da sessão (default
`VITE_INSTITUICAO_ID` ou `2` para quem entra numa rota sem passar pelo portal).

### Contexto do diário (`InfoDiario`)

O `Page` renderiza sozinho o `InfoDiario` (turma/disciplina/período selecionados) nas rotas
`/diario/.../disciplinas|horario|material`. Quem alimenta é o contexto da lib, persistido em
`localStorage` (`etutor:contexto-diario`):

```ts
import { setContextoDiario, editContextoDiario, getContextoDiario } from "etutor-lib";

setContextoDiario({ turma, disciplina, show: true }); // substitui (null limpa)
editContextoDiario({ periodoAvaliativo });             // merge parcial
const ctx = getContextoDiario();                        // Ref<DiarioCtx | null>
```

Limpe com `setContextoDiario(null)` ao sair da sessão (já feito no header e no portal do aluno).

## Toast

```ts
import { useToast, MENSAGENS } from "etutor-lib";

const toast = useToast();
toast.showSuccess(MENSAGENS.operacaoRealizada);
toast.showWarning("Este perfil já está vinculado a esta instituição.");
toast.showError("Não foi possível gerar o relatório.");
toast.showQueryError(error);   // extrai e exibe a mensagem de erro da API
```

O padrão é instanciar em `useRequests()` e reexportar o `toast` no retorno, para as views e abas não
repetirem `useToast()`.

## Plugins

- **`AuthPlugin`** (da lib) — instalado no `main.ts` quando há `VITE_API_HOST` e
  `VITE_OAUTH_CLIENT_ID` (ver `SKILL.md`).
- **`plugins/vueQuery.ts`** — `QueryClient` sem retry para erro 4xx.
- **`plugins/maps.ts`** — `MapsPlugin({ apiKey, mapId })` da lib com `VITE_GOOGLE_MAP`/`VITE_GOOGLE_MAP_ID`.
  Componentes que dependem dele: `GoogleMap`, `MultGoogleMapPins`, `inputCoordinate`,
  `renderDialogGoogleMaps`; fora deles, leia a config com `useMapsConfig()`.

## `.env`

Arquivos: `.env`, `.env.dev`, `.env.homolog`, `.env.prod` (`npm run dev` usa `--mode dev`).

```
VITE_API_HOST=https://dev.portaletutor.com.br   # host único de todos os back-ends
VITE_OAUTH_CLIENT_ID=etutor
VITE_OAUTH_RESPONSE_TYPE=code
VITE_OAUTH_SCOPE=openid
VITE_GOOGLE_MAP=
VITE_GOOGLE_MAP_ID=
VITE_INSTITUICAO_ID=                             # opcional: instituição default (senão 2)
```

`VITE_OAUTH_CLIENT_SECRET` também é lido pelo `main.ts`. Sem `VITE_API_HOST`/`VITE_OAUTH_CLIENT_ID`
(hoje `homolog`/`prod` estão vazios) o `AuthPlugin` não é instalado e as rotas protegidas ficam
inacessíveis. O `docker/Dockerfile` recebe `BUILD_MODE`, `VITE_API_HOST`, os `VITE_OAUTH_*` e os
`VITE_GOOGLE_MAP*` como build args e gera o `.env.${BUILD_MODE}` no build — variável nova de `.env`
também precisa entrar lá.

## Utils — funções utilitárias (`etutor-lib`)

> **Regra:** antes de criar qualquer utilitário (máscara, validação, transformação, data, download,
> upload, dialog), **verifique se já existe**. Importe de `etutor-lib` em vez de recriar.

```ts
import { maskCPF, removeMaskCPF, validateCPF, isoParaBr, baixarBlob, RemoveEmptyProperties } from "etutor-lib";
```

### Máscaras (`utils/masks`)

| Função | Assinatura | Descrição |
|---|---|---|
| `maskCPF` | `(cpf: string) => string` | `"11122233344"` → `"111.222.333-44"` |
| `maskCNPJ` | `(cnpj: string) => string` | `"12345678901234"` → `"12.345.678/9012-34"` |
| `maskCEP` | `(cep: string) => string` | `"12345678"` → `"12345-678"` |
| `maskCPF_CNPJ` | `(cpfCnpj: string) => string` | detecta CPF (11) ou CNPJ (14) e aplica a máscara correta |
| `maskMatricula` | `(data: string) => string` | matrícula com pontos a cada 3 dígitos |
| `maskTelefone` | `(tel: string) => string` | telefone formatado |
| `maskMoeda` | `(valor: number \| string) => string` | BRL: `1234.56` → `"R$ 1.234,56"` |
| `maskBoolean` | `(value?: string \| null) => string` | `"true"` → `"Sim"`, `"false"` → `"Não"` |
| `maskDecimal` | `(value?: number \| string) => string` | número pt-BR com 2–6 decimais |

Inversas (tiram a máscara antes de mandar ao back-end): `removeMaskCPF`, `removeMaskCNPJ`,
`removeMaskCEP`, `removeMaskCPF_CNPJ`, `removeMaskMatricula`, `removeMaskTelefone`,
`removeMaskMoeda`, `removeMaskBoolean`, `removeMaskDecimal`.

Para máscara **de digitação** no input, use `props: { mask: "..." }` (Maska), que aceita array de
padrões: `props: { mask: ["(##) ####-####", "(##) #####-####"] }`.

### Validações (`utils/validations`)

| Função | Descrição |
|---|---|
| `isRequired(value)` | obrigatório (rejeita null, `""`, undefined, array/objeto vazio) |
| `isEmail(value)` | formato de e-mail |
| `minLength(length)` | fábrica de validador de tamanho mínimo |
| `validarMatricula(value)` | exatamente 6 dígitos |
| `validateCPF(value)` / `validateCNPJ(value)` | mod-11 |
| `validarDocumento(value)` | auto-detecta CPF ou CNPJ e valida |
| `isValidRangeDate(dataInicio, dataFim)` | devolve validador: `dataFim >= dataInicio` (`dd/mm/yyyy`) |
| `isValidPercentual(value)` | percentual entre 0 e 100, com casas decimais; vazio passa — combine com `isRequired` se for obrigatório |
| `validarPlaca(value)` / `validarRenavam(value)` / `validateChassis(value)` | veículo (usados em `gestor/applications/manter-veiculo`) |

`isValidaCepMarica` (CEP do município de Maricá) é herança do Somar — não use.

### Datas (`utils/dates`)

| Função | Descrição |
|---|---|
| `isoParaBr(data?)` | `"2024-03-01"` → `"01/03/2024"` |
| `brParaIso(data?)` | `"01/03/2024"` → `"2024-03-01"` |
| `parseBrDate(dateString)` | `"dd/mm/yyyy"` → `Date` (ou `null`) |
| `formatarDataBR(input)` | qualquer data → `"dd/mm/yyyy"` |
| `months` / `shortMonths` | nomes dos meses em português |

`isoParaBr`/`brParaIso` existem porque o `InputDate` trabalha em `dd/MM/yyyy` e os back-ends em
`yyyy-MM-dd` — use nos `transform` e ao aplicar o detalhe. Confira antes se o back-end já não devolve
a data formatada (houve formatação redundante removida em `escola` e `aluno`).

### Download (`utils/download`)

| Função | Descrição |
|---|---|
| `baixarBlob(blob, filename)` | dispara o download de um `Blob` (relatórios — ver `gestor/utils/pdf.ts`) |
| `nomeDoArquivoPeloPath(path, fallback?)` | extrai o nome do arquivo de um path/key |

Arquivo guardado no S3 não precisa de `Blob`: `gerarUrlDownload(identificador)` devolve uma URL
pré-assinada — ver [http.md](http.md#download--visualização).

### Transformações (`utils/transforms`)

| Função | Descrição |
|---|---|
| `RemoveEmptyProperties(obj)` | remove propriedades `null`, `undefined` ou `""` |
| `dataPattern(value, pattern)` | transforma string de dígitos por template |
| `padLeft(value, maxLength?)` | zeros à esquerda (padrão 3) |
| `splitBase64(base64)` | separa Data URL em conteúdo e tipo MIME |
| `toBase64(file)` | `File` → base64 Data URL |
| `normalizarTexto(texto)` | remove acentos, minúsculas e trim (busca) |

### Dialogs programáticos (`utils/render`)

| Função | Descrição |
|---|---|
| `renderDialogConfirmation(opt)` | confirmação → `Promise<boolean>` |
| `renderDialogCorrelative(opt)` | sucesso com número de protocolo |
| `renderDialogGoogleMaps(coord)` | mapa com coordenadas |
| `renderDialogPhotos(opt)` | carrossel de fotos/anexos |
| ~~`renderDialogForm(opt)`~~ | **comentado na lib, não abre nada** — use o componente `DialogFields` ([views-tabelas.md](views-tabelas.md#dialog-com-formulário--dialogfields)) |

`renderDialogFormPessoa` depende das APIs de pessoa do Somar — não funciona no eTutor.

### Outros utilitários exportados

| Função | Descrição |
|---|---|
| `generateUUID()` | id local (linha nova antes de existir no back-end) |
| `getThemeColor(cssVar)` / `getThemeColorHex(cssVar, fallback)` | cor do tema — para gráficos (`echarts`) seguirem a identidade da lib |
| `mergeFormFields(base, extras?)` | mescla fields extras respeitando `order` (é o que dá sentido ao `FieldConfig.order`) |

### Tabelas, exportação e PDF

`ColumnDescriptor`, `createColumns`, `getExportableColumns`, `resumirFiltros`, `fetchAllPages`,
`exportToXLSX`, `exportToPDF`, `exportFormToPDF`, `gerarPdf`, `cabecalhoPadrao`, `blocoFiltros`,
`rodapePadrao`, `estilosPadrao` — ver [views-tabelas.md](views-tabelas.md#exportação).

### Removidas no fork

`utils/fixedValues` (`fixedUnidade`, `fixedUnidadeNucleo`, `fixedOrgaoUsuario`) **não existe** na
`etutor-lib`. O mecanismo de `fixedValue` do campo continua funcionando — inclusive assíncrono, ver
[`fixedValue` assíncrono e `whenReady()`](formularios.md#fixedvalue-assíncrono-e-whenready) —, e o
valor vem da sessão do `etutor-web` (ex.: `fixedValue: () => getInstituicaoSessao()?.nome` em
`gestor/applications/manter-veiculo/composables/useFieldsVeiculo.ts`).
