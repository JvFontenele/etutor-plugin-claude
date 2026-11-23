---
name: nova-aplicacao
description: Scaffolding de uma nova aplicação (tela CRUD) ou módulo novo no etutor-web. Use quando pedirem para criar uma tela, funcionalidade, CRUD de entidade, aplicação ou módulo novo no eTutor — inclui modulo.ts (menu e perfis), rotas com Page + meta.title/meta.modulo, services por back-end *-ws, useRequests, useFields, views, abas e seções por rota.
---

# Nova aplicação / módulo — passo a passo (eTutor)

> **Contexto de diretório:** todos os paths (`src/...`) e comandos `npm` são relativos a
> `projeto-atualizado/front-end/etutor-web/` — entre nele antes de executar qualquer coisa.

Exemplos vivos para consultar durante o scaffolding:

- CRUD simples com `PageSearchTable` e sub-telas por rota: `src/modules/gestor/applications/manter-veiculo/`
- CRUD sobre `GET` não paginado: `src/modules/gestor/applications/manter-modelo-veiculo/` (`Table`) e
  `src/modules/controle-acesso/applications/manter-perfil/` (`DataTable` + `comoPagina`)
- Edição em abas com listas de endpoint próprio: `src/modules/controle-acesso/applications/manter-usuario/`
- Seções como rotas-filhas `:id/<seção>`: `src/modules/escola/applications/manter-escola/`,
  `src/modules/aluno/applications/manter-aluno/`
- Tabela de domínio (select + `FormTable` com `manualEvents`): `src/modules/controle-acesso/applications/manter-dominio/`

Detalhes de cada componente/composable: skill `etutor:lib-etutor` (e seus `references/`).

## Antes de começar

1. **Leia o contrato no OpenAPI publicado**, não no front legado:
   `https://dev.portaletutor.com.br/<ws>/v3/api-docs` + código em `projeto-atualizado/back-end/<ws>`.
   Rota que está no repo mas não no api-docs **não está no ar**. Os tipos espelham o OpenAPI
   publicado. O legado (`projeto-legado/front-end/web-*`, portal Angular, JSF) serve para o
   **layout** da tela (seções, rótulos, `cols`); `api-etutor` só onde não há `*-ws` equivalente.
2. **Acesso é por perfil no `modulo.ts`**, não por código de aplicação. Nada de
   `meta.application`, `/acesso-negado` ou cadastro de aplicação no `controle-acesso-ws`. Se você
   está prestes a escrever um código de 3 ou 6 dígitos numa rota, é padrão do Somar vazando.
3. **Descubra qual back-end `*-ws` atende a entidade** e o context-path dele (tabela em
   `references/http.md` da skill `lib-etutor`). Não há proxy a configurar: com `VITE_API_HOST` o
   cliente chama o back-end direto, por CORS.
4. **Regra de negócio e divergência de contrato** que você descobrir vão em `docs/<módulo>.md`
   (skill `etutor:comentarios`).

## Passo a passo — nova aplicação em módulo existente

Estrutura final (nada é registrado manualmente em routers — ver "Registro automático"):

```
modules/<modulo>/
├── modulo.ts                       # 6. item em `aplicacoes`
├── services/<backend>/
│   ├── api.ts                      # 2. cliente useHttp do back-end (um por back-end)
│   └── <entidade>.ts               # 2. funções de API
├── types/<entidade>.ts             # 1. tipagens de request e response
└── applications/<app>/
    ├── routes.ts                   # 5. rotas com Page + meta.modulo + meta.title
    ├── useRequests.ts              # 3. instâncias useHttpRequest
    ├── composables/
    │   ├── useFields.ts            # 4. campos do form principal
    │   ├── useFields<Secao>.ts     # 4. um por aba/seção, quando houver
    │   └── useFiltroFields.ts      # 4. campos do filtro da listagem
    └── views/
        ├── index.vue               # 7. listagem
        ├── cadastrar.vue           # 7. criação (só dados gerais)
        ├── editar.vue              # 7. form, ou Tabs com abas/
        ├── detalhar.vue            # 7. readonly
        └── abas/<Aba>.vue | <secao>.vue
```

Pastas que aparecem quando há necessidade (não crie vazias): `applications/<app>/components/`
(modais e pedaços de view), `applications/<app>/utils/`, e no módulo `enums/`, `components/`,
`utils/` — sempre do próprio módulo, nunca importados por outro.

1. **`types/<entidade>.ts`** — tipe **request e response separadamente**, conforme o api-docs. As
   chaves quase nunca coincidem (`modeloVeiculo.id` no detalhe vs `modeloVeiculoId` no request).

2. **`services/<backend>/`** — um `api.ts` por back-end consumido pelo módulo:

   ```ts
   // services/gestao-indicadores-ws/api.ts
   import { useHttp } from "etutor-lib";
   const api = useHttp({ path: "/gestao-indicadores-ws" });   // context-path do back-end
   export { api };

   // services/gestao-indicadores-ws/veiculo.ts
   export const getPaginatedVeiculos = api.getPaginated<VeiculoDetalhe>("/veiculo");
   export const getVeiculoById = async (id: string) => await api.get<VeiculoDetalhe>(`/veiculo/${id}`);
   export const postVeiculo    = async (body: VeiculoRequest) => await api.post<VeiculoDetalhe>("/veiculo", body);
   export const putVeiculo     = async (id: string, body: VeiculoRequest) => await api.put<VeiculoDetalhe>(`/veiculo/${id}`, body);
   export const deleteDocumentoVeiculo = async (documentoId: string) => await api.del(`/veiculo/documento/${documentoId}`);
   ```

   Se o módulo já tem `services/<backend>/api.ts`, reaproveite-o. Se só **outro** módulo tem, crie
   o seu — nunca importe de outro módulo. Anexo em formulário é `formFileManager` (sobe sozinho);
   upload manual usa o `services/gestao-documentos-ws/arquivo.ts` do próprio módulo — ver
   `references/http.md`.

3. **`useRequests.ts`** — todas as instâncias `useHttpRequest` da application: `getItems` (paginado,
   passado direto), `getItem`, mutações com `onSuccess`/`onError`, selects **sem** callbacks e, se
   as abas usarem, o `toast` reexportado. Ver `references/http.md`.

4. **`composables/`** — `useFields(_id?, _disabled?)` serve às views de cadastrar/editar/detalhar e
   carrega o dado no `onMounted` interno, traduzindo o detalhe numa função de aplicar; `useFiltroFields`
   **só com os filtros que o back-end aceita**. Ver `references/formularios.md`.

5. **`routes.ts`** — path absoluto, `Page` da lib como componente da rota-pai, `meta.layout`,
   `meta.title` e **`meta.modulo`** (nome da pasta do módulo) na rota-pai, `meta.title` em cada filho:

   ```ts
   import type { RouteRecordRaw } from "vue-router";
   import { Page } from "etutor-lib";
   import LayoutEtutor from "@/layouts/LayoutEtutor.vue";

   const routes: RouteRecordRaw[] = [
     {
       path: "/gestor",
       component: Page,
       meta: { layout: LayoutEtutor, title: "Gestor", modulo: "gestor" },
       children: [
         { path: "veiculos",            name: "gestor-veiculos",           meta: { title: "Veículos" },        component: () => import("./views/index.vue") },
         { path: "veiculos/cadastrar",  name: "gestor-veiculos-cadastrar", meta: { title: "Novo Veículo" },    component: () => import("./views/cadastrar.vue") },
         { path: "veiculos/:id/editar", name: "gestor-veiculos-editar",    meta: { title: "Editar Veículo" },  component: () => import("./views/editar.vue") },
         { path: "veiculos/:id",        name: "gestor-veiculos-detalhar",  meta: { title: "Detalhar Veículo" },component: () => import("./views/detalhar.vue") },
       ],
     },
   ];

   export default routes;
   ```

   - **Sem o `Page` como rota-pai, os botões de Voltar/Salvar não aparecem** (o `PageGroupButtons`
     teleporta para o `<div id="page-group-buttons">` que só o `Page` renderiza).
   - **Sem `meta.modulo`**, o rail não mostra o menu do módulo e o guard não checa os perfis.
   - Os `name` das sub-telas começam com o `name` de entrada (`gestor-veiculos-...`) — é assim que o
     rail marca o item ativo.
   - `meta.codigo` (opcional) só faz o `PageSearchTable` lembrar o filtro ao voltar para a listagem.

6. **`modulo.ts`** — adicione a application em `aplicacoes`, com `rota` = `name` da rota de entrada:

   ```ts
   { titulo: "Veículos", icon: "Car", rota: "gestor-veiculos" },
   ```

   `perfis` na aplicação só se ela for **mais restrita** que o módulo — e aí repita em `meta.perfis`
   na rota-pai: o guard só confere `aplicacao.perfis` na rota de entrada, não nas sub-rotas. Sem a
   entrada em `aplicacoes` a tela existe, mas não aparece no rail.

7. **Views** — escolha a listagem pelo contrato do `GET` (`references/views-tabelas.md`):
   `PageSearchTable` para `Pageable`; `Table`, `DataTable` + `comoPagina` ou `PageSearchTable` +
   `paginarLocal` para lista inteira. `cadastrar.vue` com `useFields()`; `detalhar.vue` com
   `useFields(id, true)` (o `_disabled` vai em cada campo). Submits usam `formRef.value?.validate()` +
   `formRef.value?.transformData()` dentro da trava `salvando` (`if (salvando.value) return` +
   `try/finally`) contra duplo envio; `PageGroupButtons` com `@back` para a listagem.
   `instituicaoId` vem de `getInstituicaoId()` (`@/core/sessao/instituicao`) quando o request exigir —
   no body, ou em query param via `getItems` próprio (ver `references/http.md`).

8. **Entidade com seções** — dois padrões, ambos com a entidade = **`:id` da rota**, nunca store global:
   - **abas** (`Tabs` em `editar.vue`, cada aba em `views/abas/` recebendo o id por prop e
     carregando seus próprios dados) — `controle-acesso/manter-usuario`;
   - **seções por rota** (`:id/<seção>`, cada uma com sua view, acessadas por ações da listagem) —
     `escola/manter-escola`, `aluno/manter-aluno`.

   Cadastro é só dos dados gerais — as outras seções dependem de um id já existente.

9. **Listas com endpoint próprio → `FormTable` standalone com `manualEvents`**, gravando/removendo
   cada linha na hora. Ver `references/formtable.md`.

10. **Testes** — siga a skill `etutor:testes`.

### Registro automático — NÃO edite routers na mão

- `modules/<mod>/router/index.ts` agrega `applications/**/routes.ts` via `import.meta.glob`.
- `src/router/index.ts` agrega `modules/**/router/index.ts` também via `import.meta.glob`.
- `src/core/modulos/menus.ts` agrega `modules/*/modulo.ts` via `import.meta.glob`.

Criar os arquivos no lugar certo é suficiente.

## Passo extra — módulo novo

1. **`modules/<modulo>/router/index.ts`** — só o agregador de applications, sem fallback próprio (o
   404 global vive no módulo `portal`):

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

2. **`modules/<modulo>/modulo.ts`** — o `ModuloConfig` do módulo. Ele aparece sozinho no portal
   (`modulosVisiveis`) e no rail; não edite `menus.ts`, `App.vue`, o portal nem o `modulo.ts` de
   outro módulo.

   ```ts
   import { Perfil, type ModuloConfig } from "@/core/modulos/modulo.types";

   const modulo: ModuloConfig = {
     nome: "Gestão Integrada",
     descricao: "Monitoramento da rede, indicadores e relatórios.",
     icon: "ChartNoAxesColumn",
     perfis: [Perfil.ANALISTA, Perfil.ADMINISTRADOR],
     aplicacoes: [
       { titulo: "Dashboard", icon: "LayoutDashboard", rota: "gestor-dashboard" },
       // ...
     ],
   };

   export default modulo;
   ```

   - **`perfis` é obrigatório**: lista vazia/ausente bloqueia todo mundo. `Perfil.ANALISTA` primeiro.
   - `exigeContextoEscolar: true` só se as telas dependem de escola/ano letivo da sessão (hoje só
     `diario`).
   - `meta.modulo` de todas as rotas-raiz das applications = nome da pasta do módulo.

3. **Repositório do módulo** — cada módulo também é publicado num repositório próprio no Gitea
   (`etutor-web-modulo-<nome>`) e o build escolhe a versão de cada um. Depois do merge, publique com
   `npm run modulos:sync -- <modulo>` (`git subtree push` de `src/modules/<modulo>`). O
   `modulos:list` lê as pastas de `src/modules`; para módulo novo, confirme que o repositório
   `etutor-web-modulo-<nome>` existe no Gitea antes do primeiro sync (o `GUIA_MODULOS.md` ainda não
   lista `portal-aluno` nem `professor`). O `sync` publica na **branch atual**; para outro ambiente,
   `-- <modulo> --branch <amb>`. Fluxo
   completo em `GUIA_MODULOS.md` (não use mais tags `modulo/versao` nem `release/environments/*.json`).

4. **`docs/<modulo>.md`** — crie e indexe em `docs/README.md` quando houver regra de negócio ou
   divergência de back-end a registrar.

## Checklist final

- [ ] Contrato conferido no OpenAPI publicado do `*-ws` (não no front legado nem só no DTO Java)
- [ ] `Page` como rota-pai; `meta.layout`, `meta.title` e `meta.modulo` preenchidos; nenhum `meta.application`
- [ ] Application listada em `aplicacoes` do `modulo.ts` (e `perfis` do módulo preenchido, se módulo novo)
- [ ] `services/<backend>/api.ts` com o context-path correto, dentro do próprio módulo
- [ ] Nenhum import de outro módulo nem de services da lib (exceto anexo); services só em `modules/<m>/services/<backend>/`
- [ ] Selects via `fnOptions` (sem `execute()` manual); lista grande via `selectPagination` com instância `useHttpRequest`
- [ ] Nenhum `formPerson*`/`selectGeneric`; `formPessoa` só com `sigiloso: false` + adaptadores do módulo; anexo com `formFileManager`
- [ ] Views de CRUD sem lógica de busca (carregamento no composable); seções com id pela rota
- [ ] Submits com trava `salvando`; `@back` com destino fixo
- [ ] Listas com endpoint próprio em `FormTable` + `manualEvents`, persistindo na hora
- [ ] `instituicaoId` de `getInstituicaoId()`, quando o request exigir
- [ ] Utils da `etutor-lib` reutilizadas (máscaras/validações/datas/download) — não recriadas; anexo em form via `formFileManager`
- [ ] Regras de negócio novas em `docs/<módulo>.md`, não em comentário
- [ ] Testes da application (skill `etutor:testes`)
- [ ] `npm run build` passando
