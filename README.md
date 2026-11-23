# etutor-plugin-claude

Plugin do Claude Code com as skills do projeto **eTutor** (Tout Soluções): os padrões de
implementação do `etutor-web` e da `etutor-lib`.

Derivado do [`somar-plugin-claude`](https://github.com/JvFontenele/somar-plugin-claude). A
`etutor-lib` é um fork da `somar-lib` e a API dos componentes é a mesma, mas o contexto é outro:
login OAuth2 único com sessão e perfis do próprio `etutor-web`, menu declarado por módulo, vários
back-ends `*-ws` atrás de um host único e os componentes de pessoa do Somar não religados.

## Instalação

```
/plugin marketplace add JvFontenele/etutor-plugin-claude
/plugin install etutor@etutor-plugin-claude
```

As skills passam a ser invocadas como `etutor:lib-etutor`, `etutor:nova-aplicacao`, `etutor:testes` e
`etutor:comentarios`.

## Skills

| Skill | Uso |
|---|---|
| `etutor:lib-etutor` | Padrões e APIs da `etutor-lib`: `DynamicForm`/`useFields`, `PageSearchTable`, `Table`/`DataTable` para GET não paginado, `FormTable` (campo e standalone com `manualEvents`), selects com `fnOptions`/`selectPagination`/`checklist`, upload S3 com `formFileManager`, `useHttp`/`useHttpRequest` com um cliente por back-end `*-ws`, rotas com `Page` + `meta.modulo`, máscaras, validações, datas, dialogs, relatórios PDF e exportação. Inclui o que **não** funciona no eTutor. |
| `etutor:nova-aplicacao` | Scaffolding de uma nova aplicação (CRUD, com abas ou seções por rota) ou módulo novo no `etutor-web`, incluindo o `modulo.ts`. |
| `etutor:testes` | Padrão de testes do `etutor-web` (Vitest + `@vue/test-utils` + MSW + Playwright), inclusive o que ainda falta na infra. |
| `etutor:comentarios` | Regra do projeto sobre comentários no código: regra de negócio e bug de back-end vão em `docs/<módulo>.md`, não em comentário; `CLAUDE.md` cobre só estrutura. |

## O que mudou em relação ao plugin do Somar

- `somar-lib` → `etutor-lib`; `etutorLib({ baseUrl: VITE_API_HOST })` — um host por ambiente, e
  cada back-end é só um context-path nesse host (`/controle-acesso-ws`, `/escola-ws`,
  `/diario-ws/api`, ...), chamado direto via CORS.
- Autenticação **voltou, em outro formato**: `AuthPlugin`/`useAuthStore` da lib fazem OAuth2
  authorization code contra o `servidor-de-autenticacao` (`${VITE_API_HOST}/auth`); o interceptor
  manda `Authorization: Bearer` e os cabeçalhos de contexto (`X-Instituicao-Id`, `X-Escola-Id`,
  `X-Ano-Letivo-Id`, `X-Perfil-Id`). Sessão, perfis e guard de rota vivem no `etutor-web`
  (`src/core/sessao`, `src/router`), não na lib.
- Menu e acesso: sem API de menu nem `meta.application`/`/acesso-negado`. Cada módulo declara
  `src/modules/<modulo>/modulo.ts` (`ModuloConfig` com `perfis` e `aplicacoes`), agregado por
  `core/modulos/menus.ts`; a rota-raiz leva `meta.modulo`. `meta.codigo` é opcional e só persiste o
  filtro do `PageSearchTable`.
- Layouts vivem no `etutor-web`, não na lib: `LayoutEtutor` (header + rail de ícones do módulo
  ativo) para todos os módulos e `LayoutPortalAluno` só para o Portal do Aluno. `utils/fixedValues` e a mensagem por código (`useMensagem`,
  `codMessage`) também saíram; texto genérico vem do catálogo `MENSAGENS`.
- Fora como padrão: `formPerson*`, `botaoSigiloso`, `inputSigiloso` (contratos do Somar) e
  `selectGeneric` (aponta para back-end da modernização anterior). `formPessoa` só com
  `sigiloso: false` e buscas trocadas por adaptadores do módulo; `formAddress` foi reescrito para o
  eTutor (viacep + mapa). `formAnexo`/`formTipoAnexo` foram removidos da lib.
- Não existem na `etutor-lib` (não copie do plugin do Somar): modelos de exportação PDF
  (`ListaModelosPdf`, `FonteRelatorio`/JSONata), `utils/fixedValues`, `useMensagem("001")`.
- Dentro: `services/<backend>/api.ts` por back-end dentro de cada módulo (módulo não importa de
  outro módulo); `Page` obrigatório como rota-pai; `FormTable` standalone com `manualEvents`;
  `FormFileManager` com upload direto no S3 via `gestao-documentos-ws`; `selectPagination`,
  `checklist`; `execute(..., { onSuccess })`; adaptadores de lista não paginada (`comoPagina`,
  `paginarLocal`); relatórios PDF em `Blob`; utils de data, download e remoção de máscara;
  publicação por módulo (`npm run modulos:sync`).

## Referência de contexto

O plugin documenta os padrões; o contexto de cada repositório fica nos `CLAUDE.md` e em
`docs/*.md` de `projeto-atualizado/front-end/etutor-web/` e no `CLAUDE.md` de
`projeto-atualizado/front-end/lib-etutor/`.

## Origem e licenças

`etutor:lib-etutor`, `etutor:nova-aplicacao`, `etutor:testes` e `etutor:comentarios` são de autoria
interna do projeto eTutor.
