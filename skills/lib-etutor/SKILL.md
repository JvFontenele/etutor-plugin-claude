---
name: lib-etutor
description: Padrões e APIs da biblioteca etutor-lib (Vue 3) com exemplos reais do etutor-web. Use SEMPRE ao criar ou alterar telas do eTutor — CRUDs, DynamicForm, PageSearchTable, Table/DataTable, FormTable (inclusive standalone com manualEvents), selects com fnOptions/selectPagination/checklist, upload com formFileManager (S3), composables useFields/useFiltroFields/useRequests, serviços por back-end *-ws, rotas com Page + meta.title/meta.modulo, máscaras, validações, dialogs e relatórios PDF.
---

# etutor-lib — Guia de uso com exemplos reais

> **Contexto de diretório:** todos os paths (`src/...`) e comandos `npm` deste guia são relativos a
> `projeto-atualizado/front-end/etutor-web/` — entre nele antes de executar qualquer coisa.
> A lib fica ao lado, em `projeto-atualizado/front-end/lib-etutor/`.

Biblioteca de componentes Vue 3 + TypeScript do eTutor. Pacote: `etutor-lib`, consumido como
**tarball local** (`file:../lib-etutor/etutor-lib-0.0.1.tgz`). O `npm run build` da lib já gera o
tarball automaticamente (o script roda `vue-tsc -b && vite build && npm pack`) — não é preciso rodar
`npm pack` à parte depois de alterar a lib.

A `etutor-lib` é um fork da `somar-lib`: a API dos componentes é praticamente a mesma, mas o
**contexto mudou** — login OAuth2 próprio, sessão e perfis no `etutor-web`, menu declarado por
módulo (`modulo.ts`), vários back-ends `*-ws` atrás de um host único, e os componentes ligados a
pessoa do Somar não foram religados. Este guia é a fonte da verdade para o eTutor; padrões da
`somar-lib` que não constam aqui provavelmente **não funcionam**.

## Referências detalhadas

Leia o arquivo de referência da tarefa em mãos **antes** de escrever código:

| Arquivo | Consulte quando for mexer com |
|---|---|
| [references/http.md](references/http.md) | Back-ends `*-ws` e `services/<backend>/api.ts`, autenticação/cabeçalhos, `useRequests`, `useHttpRequest`, `execute` com callbacks por chamada, upload S3/download, relatórios em `Blob`, listas não paginadas |
| [references/formularios.md](references/formularios.md) | `DynamicForm`, `useFields`/`useFiltroFields`, tipos de campo (`is`), `FieldConfig`, selects com `fnOptions` (estático, reativo, cascata, `returnObject`), `selectPagination`, `checklist`, `formFileManager` |
| [references/formtable.md](references/formtable.md) | `formTable` como campo do `DynamicForm` **e** como componente standalone com `manualEvents` (o padrão do eTutor para listas com endpoint próprio) |
| [references/views-tabelas.md](references/views-tabelas.md) | Views (`index`/`cadastrar`/`editar`/`detalhar`), abas com `Tabs` × seções como rotas-filhas, `PageSearchTable`, `Table`/`DataTable` para GET não paginado, paginação local, relatórios PDF, dialogs, exportação |
| [references/config.md](references/config.md) | Rotas (`Page`, `meta.modulo`/`meta.codigo`/`meta.exigeSessao`, colapso de rotas aninhadas), `modulo.ts` e menu, sessão e `instituicaoId`, toast, plugins, `.env`, **utils prontas** (máscaras, validações, datas, download, transforms) |
| [references/dashboard-widgets.md](references/dashboard-widgets.md) | `DashboardGrid`/`WidgetRenderer`, widgets `chart`/`metric`/`table`, montagem de dashboards |

## Regras de ouro

1. **Formulários são sempre `DynamicForm` + `useFields`** — nunca componentes soltos (`<InputText>`, etc.) no template.
2. **Views finas** — sem lógica de busca na view de CRUD; o carregamento fica no composable `useFields(_id?, _disabled?)`.
   Views de **aba** e de lista com endpoint próprio são a exceção justificada: orquestram várias chamadas (ver `references/formtable.md`).
3. **Selects usam `fnOptions`** com a instância de `useHttpRequest` — sem `execute()` manual.
4. **Toda rota de tela precisa do `Page` da lib como rota-pai** — é ele que renderiza o
   `<div id="page-group-buttons">` onde o `PageGroupButtons` faz teleport. Sem o `Page`,
   os botões de Voltar/Salvar **não aparecem**.
5. **Lista com endpoint próprio → `FormTable` standalone com `manualEvents`**, gravando/removendo
   cada linha na hora e recarregando do back-end. Não acumule "adicionados/excluídos" para persistir no submit.
6. **Antes de criar qualquer função** (máscara, validação, data, download, upload, dialog, transform,
   helper de formatação etc.), verifique se ela já existe em `etutor-lib` e reaproveite — ver
   [references/config.md](references/config.md). Não recrie o que a lib já resolve.
7. **Fields são declarados de forma literal**, um por um, direto no objeto retornado por
   `useFields`/`useFiltroFields` — não crie funções pequenas para gerar/montar campos dinamicamente
   (ex.: `criarCampoSelect(nome, label, fn)`, `montarFieldsFromArray(lista)`). Fields quase sempre
   têm pequenas diferenças de `cols`, `label`, `isRequired`, `events` entre si, e abstrair isso cedo
   demais rende uma função genérica difícil de estender sem parâmetros extras. Prefira repetir a
   estrutura do campo em cada entrada — mais fácil de ler, revisar e ajustar um caso isolado sem
   afetar os outros.
8. **Fonte da verdade das APIs é o OpenAPI publicado do back-end**
   (`https://dev.portaletutor.com.br/<ws>/v3/api-docs`) e o código em
   `projeto-atualizado/back-end/<ws>`. Os tipos do front espelham o **OpenAPI publicado** — o DTO Java
   pode estar à frente do que está deployado. O legado (`projeto-legado/back-end`, `api-etutor`) só
   vale onde a funcionalidade ainda não migrou.
9. **Módulo nunca importa de outro módulo**, e código fora de `src/modules/` (core, router, layouts,
   header) nunca importa de dentro de um módulo. O compartilhado vive em `src/core` e `src/utils`;
   **não existe `src/services`**. Cada service fica com o dono (`modules/<m>/services/<backend>/`,
   `core/sessao/services/<backend>/`, `components/header/services/<backend>/`) — dois módulos que
   falam com o mesmo back-end têm, cada um, o seu `services/<backend>/api.ts`. Services da lib não
   são usados, exceto o de anexo.

## ⚠️ O que existe na lib mas NÃO funciona no eTutor

Estes componentes continuam no `componentsMap`/barrels, mas falam com contratos que o eTutor não
tem — não os use em código novo:

| Campo / componente | Por quê |
|---|---|
| `formPerson`, `formPersonSelect`, `formPersonSigilosa`, `botaoSigiloso`, `inputSigiloso`, `renderDialogFormPessoa` | `services/pessoa/*` da lib ainda chama `cadastro-geral-ws`/`gestao-institucional-ws` do Somar. No eTutor pessoa está no `servidor-ws` (módulo `professor`) e no `aluno-ws` |
| `formPessoa` **com os defaults** | Mesmo motivo. Só funciona com `sigiloso: false` e as buscas trocadas por adaptadores do módulo — ver abaixo |
| `selectGeneric` | Consulta `/api-commons` (modernização anterior). Domínios vêm do `*-ws` do módulo (ex.: `aluno/services/aluno-ws/dominio.ts` → `/api/referencias/*`) |
| `formFileIndividual` | Só guarda o `File` no model, não sobe nada. Anexo é `formFileManager` |
| Serviços de domínio/endereço/escola da lib (`getSexos`, `dominiosCommons`, `getSelectEstado`, `getDownloadArquivo`, `getPaginatedEscola`, ...) | Mesmos back-ends da modernização anterior; sem uso nas telas atuais |

Lista de vínculos usa o **`FormTable` genérico** com o serviço do módulo
([references/formtable.md](references/formtable.md)). Já **escolher uma pessoa** (CPF + nome) é
`formPessoa` religado ao `servidor-ws` por adaptadores — ver
[references/formularios.md](references/formularios.md#formpessoa--religado-por-adaptadores) — e
**endereço** é o `formAddress` reescrito para o eTutor (CEP via viacep + mapa, sem selects) — ver
[references/formularios.md](references/formularios.md#formaddress).

`formAnexo`/`formTipoAnexo` foram **removidos** da lib. `useSSEService`/`useNotificationStore` existem
mas nenhuma tela usa.

Também mudou de contrato:

- **`formFileManager`** — adaptado ao eTutor: sobe direto no S3 por URL pré-assinada do
  `gestao-documentos-ws`; cada item do model é `{ name, type, size, url, identificador, file }` e o que se
  persiste é o `identificador`. Ver [references/http.md](references/http.md#upload-de-arquivo-s3).
- **`Page`** — breadcrumb e título vêm da hierarquia de rotas (`meta.title`/`meta.breadcrumb`),
  começando em "Início › Portal (`/modulos`)"; não há checagem de permissão por `meta.codigo` (acesso
  é o guard do `etutor-web`, ver [references/config.md](references/config.md)).
- **Mensagens** — o mecanismo de texto por código (`useMensagem("001")`, prop `codMessage`, loader e
  store de catálogo, `TipoMensagem`) **não existe mais**. Texto genérico vem do catálogo único da lib —
  `import { MENSAGENS } from "etutor-lib"` (ou `useMensagens()` em componente): `campoObrigatorio`,
  `confirmarExclusao`, `operacaoRealizada`, `minimoCaracteres(n)`, ... Texto de domínio fica na
  aplicação.

## Setup (`main.ts`)

`etutorLib()` recebe o host do ambiente (`VITE_API_HOST`) como `baseUrl`; cada serviço só conhece o
próprio context-path. O `AuthPlugin` só é instalado quando há host e client id; o
`setProvedorCabecalhos` faz o interceptor mandar o contexto da sessão em toda requisição autenticada.

```ts
// src/main.ts (trecho)
import etutorLib, { AuthPlugin, setProvedorCabecalhos } from "etutor-lib";
import { concluirLogin } from "@/core/sessao/posLogin";
import { atualizarInstituicaoSessao, cabecalhosDaSessao } from "@/core/sessao/session";

setProvedorCabecalhos(cabecalhosDaSessao);

const apiHost = import.meta.env.VITE_API_HOST;
const etutor = etutorLib({ baseUrl: apiHost });

app.use(QueryPlugin);
app.use(maps);
app.use(pinia);
app.use(router);
app.use(etutor);

const clientId = import.meta.env.VITE_OAUTH_CLIENT_ID;
if (apiHost && clientId) {
  app.use(
    AuthPlugin({
      apiBase: `${apiHost}/auth`,
      authorizeBase: `${apiHost}/auth`,
      clientId,
      clientSecret: import.meta.env.VITE_OAUTH_CLIENT_SECRET,
      responseType: import.meta.env.VITE_OAUTH_RESPONSE_TYPE || "code",
      scope: import.meta.env.VITE_OAUTH_SCOPE || "openid",
      destinoAposLogout: "/instituicao",
      aoAutenticar: async () => {
        const erro = await concluirLogin(router);
        if (erro) console.warn(erro);
      },
    }),
  );
}
```

Fluxo de login, sessão, seleção de perfil/instituição/escola/ano letivo e regras de acesso:
`docs/header.md` e `docs/core.md` do `etutor-web`.

## `App.vue` e menu

`App.vue` só escolhe o layout (`meta.layout`, default `LayoutEtutor`) e esconde a `RouterView`
enquanto um módulo com `exigeContextoEscolar` não tem escola na sessão. **Não há array de menu no
`App.vue`**: o `LayoutEtutor` lê `route.meta.modulo`, busca o `ModuloConfig` com `getModulo()` e
desenha no rail as `aplicacoes` visíveis para o perfil ativo (`aplicacoesVisiveis`). O portal
(`modules/portal/views/ModulosPage.vue`) lista os módulos com `modulosVisiveis`. Como declarar um
módulo e suas aplicações: [references/config.md](references/config.md#módulo-e-menu-modulots).

## Alterou a lib? Refaça o tarball

O `etutor-web` só enxerga o que está no **tarball**. Mudança na lib que não reflete no front é,
quase sempre, tarball desatualizado:

```bash
cd ../lib-etutor && npm run build   # já roda o npm pack no fim
cd ../etutor-web && npm i ../lib-etutor/etutor-lib-0.0.1.tgz
```

## Scaffolding de aplicação/módulo

A organização de pastas de uma feature (módulo, `modulo.ts`, application, `routes.ts`,
`useRequests`, `useFields`, views) está na skill `etutor:nova-aplicacao` — use-a para o passo a passo
completo. Testes: skill `etutor:testes`. Comentários e onde documentar regra: skill
`etutor:comentarios`.

## Ruído conhecido no console

`DynamicForm` faz `v-bind` de todo o objeto de configuração do campo no componente, então o Vue
avisa `Extraneous non-props attributes (cols, props, transform)` para cada campo. É só aviso de dev —
não tente "consertar" filtrando as chaves: mexeria no contrato de todos os inputs.
