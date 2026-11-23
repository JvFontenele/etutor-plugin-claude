---
name: testes
description: Padrão de testes do etutor-web (Vitest + @vue/test-utils + MSW + Playwright). Use ao criar ou alterar testes — useRequests.spec, useFields.spec, specs de página .vue e de aba, sessão/guard/login, testes de integração ou E2E — ou quando pedirem para cobrir uma application com testes.
---

# Testes — padrão do eTutor

> **Contexto de diretório:** todos os paths (`src/...`, `e2e/...`) e comandos `npm` são relativos a
> `projeto-atualizado/front-end/etutor-web/` — entre nele antes de executar qualquer coisa.

## Estado atual — leia antes de começar

A infraestrutura está pronta (Vitest, `@vue/test-utils`, happy-dom, MSW, Playwright, coverage v8) e
há **10 specs**, concentradas em sessão, guard e login OAuth2:

| Spec | Cobre |
|---|---|
| `src/core/sessao/session.spec.ts`, `contexto-cabecalhos.spec.ts` | sessão e cabeçalhos de contexto |
| `src/router/guard.spec.ts` | guard de rota com `AuthPlugin` real |
| `src/modules/portal/{auth-fluxo,authorize-host,login-popup,sessao-token}.spec.ts` | fluxo OAuth2 |
| `src/modules/portal/modulos.spec.ts` | `modulosVisiveis`/acesso por perfil (contrato dos `modulo.ts`) |
| `src/modules/portal-aluno/.../acompanhamento/useFaltas.spec.ts` | regra pura de composable |
| `src/utils/disciplina.spec.ts` | util compartilhado |

`escola`, `aluno`, `gestor`, `diario`, `controle-acesso`, `professor` e os layouts ainda não têm
spec. O que ainda falta na infra:

- `src/test/msw/` (handlers + `setup.ts`) — exigido por `vitest.integration.config.ts`
- `src/integration/feature/` — alvo do `test:integration:feature`
- `e2e/` — `testDir` do Playwright
- `src/integration/swagger-api-docs.spec.ts`, alvo do script `test:integration` — **não existe**.
  Ou crie a spec, ou não use esse script.

Já existe: `src/test/setup.ts` (silencia `console` e limpa mocks entre testes), registrado nos dois
configs de Vitest, e `src/test-utils/login.ts` (`urlDoLogin`: captura a URL do authorize aberta em
popup pelo `auth.login()`).

## Onde criar cada teste

| Tipo | Local |
|---|---|
| Regra de negócio de uma application | `src/modules/<mod>/applications/<app>/__tests__/*.spec.ts` (ou ao lado do composable) |
| Sessão, perfis, guard | ao lado do código: `src/core/sessao/*.spec.ts`, `src/router/*.spec.ts` |
| Contrato dos módulos (perfis, `modulosVisiveis`, `aplicacoesVisiveis`) | `src/modules/portal/modulos.spec.ts` — **estenda, não duplique** |
| Integração feature (router + pinia + MSW) | `src/integration/feature/*.integration.spec.ts` |
| E2E (Playwright) | `e2e/` |

A fronteira de módulos vale para teste também: spec de um módulo não importa de outro módulo.

Ambiente: `node` por padrão (`vitest.config.ts`); specs que montam componentes/composables Vue ou
usam `localStorage`/`window` declaram happy-dom no **topo do arquivo**
(`// @vitest-environment happy-dom` ou o docblock `/** @vitest-environment happy-dom */`, como as
specs existentes).

`vitest.config.ts` inclui `src/**/*.spec.ts` e exclui `src/integration/feature/**` — a suíte de
integração roda só via `npm run test:integration:feature`.

## O que priorizar

1. Ação principal da tela (submit, add/edit/delete em `FormTable`, navegação)
2. Fluxo de sucesso **e** de erro (no erro: não navegar, não exibir sucesso — lembre que `execute`
   devolve `null` em vez de lançar)
3. **Payload gerado** — request e response têm chaves diferentes, e os `transform` fazem conversão
   (`undefined` para sair do payload, `toUpperCase`, identificador do `formFileManager`)
4. Regras que existem só no front por causa de constraint do back-end (ex.: mínimo de 5 caracteres
   antes de chamar a busca)
5. Acesso: perfis do `modulo.ts` e `meta.perfis` (`modulos.spec.ts`, `guard.spec.ts`)

## Anatomia 1 — `useRequests.spec.ts` (ambiente node)

Mocks com `vi.hoisted`, `vi.mock` de `etutor-lib`/`vue-router`/services, e **import do módulo testado
depois dos mocks**. As opções passadas ao `useHttpRequest` são inspecionadas via `mock.calls`.
Exemplo sobre `gestor/applications/manter-veiculo/useRequests.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from "vitest";

const {
  useHttpRequestMock, useRouterMock, useToastMock, pushMock, showSuccessMock,
  getPaginatedVeiculosMock, postVeiculoMock, getSelectModeloVeiculoMock,
} = vi.hoisted(() => ({
  useHttpRequestMock: vi.fn(),
  useRouterMock: vi.fn(),
  useToastMock: vi.fn(),
  pushMock: vi.fn(),
  showSuccessMock: vi.fn(),
  getPaginatedVeiculosMock: vi.fn(),
  postVeiculoMock: vi.fn(),
  getSelectModeloVeiculoMock: vi.fn(),
}));

vi.mock("etutor-lib", () => ({
  useHttpRequest: useHttpRequestMock,
  useToast: useToastMock,
  MENSAGENS: { operacaoRealizada: "Operação realizada com sucesso!" },
}));
vi.mock("vue-router", () => ({ useRouter: useRouterMock }));
vi.mock("@/modules/gestor/services/gestao-indicadores-ws/veiculo", () => ({
  getPaginatedVeiculos: getPaginatedVeiculosMock,
  postVeiculo: postVeiculoMock,
  getSelectModeloVeiculo: getSelectModeloVeiculoMock,
  getVeiculoById: vi.fn(),
  putVeiculo: vi.fn(),
  // ... todas as demais funções importadas pelo useRequests, com vi.fn()
}));

import { useRequests } from "@/modules/gestor/applications/manter-veiculo/useRequests";

describe("useRequests de gestor/manter-veiculo", () => {
  beforeEach(() => {
    pushMock.mockReset();
    useRouterMock.mockReturnValue({ push: pushMock });
    useToastMock.mockReturnValue({ showSuccess: showSuccessMock, showQueryError: vi.fn() });
    useHttpRequestMock.mockImplementation((fn, options) => ({ fn, options, execute: vi.fn() }));
  });

  it("mapeia getItems para o serviço paginado", () => {
    expect(useRequests().getItems).toBe(getPaginatedVeiculosMock);
  });

  it("volta para a listagem no sucesso do cadastro", () => {
    useRequests();
    const options = useHttpRequestMock.mock.calls.find(([fn]) => fn === postVeiculoMock)?.[1];
    options.onSuccess();
    expect(showSuccessMock).toHaveBeenCalledWith("Operação realizada com sucesso!");
    expect(pushMock).toHaveBeenCalledWith({ name: "gestor-veiculos" });
  });

  it("select fica sem callbacks", () => {
    useRequests();
    const options = useHttpRequestMock.mock.calls.find(([fn]) => fn === getSelectModeloVeiculoMock)?.[1];
    expect(options).toBeUndefined();
  });
});
```

> O `vi.mock` de um service precisa listar **todas** as funções que o `useRequests` importa dele,
> senão o import real roda e tenta montar o axios. Mocke o módulo inteiro com `vi.fn()`.

## Anatomia 2 — `useFields.spec.ts` (happy-dom)

Composables com `onMounted` precisam de um componente host montado:

```ts
// @vitest-environment happy-dom
import { describe, expect, it, vi } from "vitest";
import { defineComponent } from "vue";
import { mount } from "@vue/test-utils";

const { getVeiculoExecuteMock } = vi.hoisted(() => ({ getVeiculoExecuteMock: vi.fn() }));

vi.mock("@/modules/gestor/applications/manter-veiculo/useRequests", () => ({
  useRequests: () => ({
    getVeiculo: { execute: getVeiculoExecuteMock },
    getSelectModeloVeiculo: { execute: vi.fn() },
    getSelectTransporteEscolar: { execute: vi.fn() },
  }),
}));
vi.mock("@/core/sessao/session", () => ({ getInstituicaoSessao: () => ({ id: 2, nome: "Rede X" }) }));

import { useFieldsVeiculo } from "@/modules/gestor/applications/manter-veiculo/composables/useFieldsVeiculo";

const flushPromises = () => new Promise((resolve) => setTimeout(resolve, 0));

function setup(...args: Parameters<typeof useFieldsVeiculo>) {
  let result!: ReturnType<typeof useFieldsVeiculo>;
  const Host = defineComponent({
    setup() {
      result = useFieldsVeiculo(...args);
      return () => null;
    },
  });
  mount(Host);
  return () => result;
}

it("traduz o detalhe do back-end para as chaves do formulário", async () => {
  getVeiculoExecuteMock.mockResolvedValue({ placa: "ABC1D23", modeloVeiculo: { id: 7 }, transporteEscolar: { id: 3 } });

  const getResult = setup("10");
  await flushPromises();

  const { data } = getResult();
  expect(data.value.modeloVeiculoId).toBe(7);
  expect(data.value.transporteEscolarId).toBe(3);
});

it("instituição é só exibição: sai do payload", () => {
  const { fields } = setup()();
  expect(fields.value.instituicao.transform?.("Rede X")).toBeUndefined();
});
```

Teste o **contrato dos fields**: `render`/`disabled` condicionais, `key` da cascata, `transform`
(inclusive os que devolvem `undefined`), `fnParams` reativo e a tradução do detalhe.

## Anatomia 3 — spec de página / aba (happy-dom)

Stub dos componentes da `etutor-lib` com `defineComponent` (declarando `props`/`emits` usados), mock
do `useRequests` da application e de `vue-router`, interação via `findComponent`. Exemplo sobre
`gestor/applications/manter-veiculo/views/manutencao.vue` (FormTable com `manualEvents`):

```ts
// @vitest-environment happy-dom
vi.mock("etutor-lib", () => ({
  FormTable: defineComponent({
    name: "FormTable",
    props: ["fields", "columns", "modelValue", "data", "editable", "deletable", "manualEvents", "label"],
    emits: ["add", "edit", "delete", "initEdit", "update:modelValue", "update:data"],
    template: "<div />",
  }),
  Title: defineComponent({ name: "Title", props: ["text", "type"], template: "<div />" }),
  PageGroupButtons: defineComponent({ name: "PageGroupButtons", emits: ["save", "back"], template: "<div />" }),
  isoParaBr: (v: string) => v,
}));
vi.mock("vue-router", () => ({
  useRoute: () => ({ params: { id: "10" } }),
  useRouter: () => ({ push: vi.fn() }),
}));
vi.mock("@/modules/gestor/applications/manter-veiculo/useRequests", () => ({
  useRequests: () => ({
    getManutencoes: { execute: getManutencoesMock },
    postManutencao: { execute: postManutencaoMock },
    putManutencao: { execute: putManutencaoMock },
    deleteManutencao: { execute: deleteManutencaoMock },
  }),
}));
// ... mock do composable useFieldsManutencao, se ele chamar useRequests/selects

import Manutencao from "@/modules/gestor/applications/manter-veiculo/views/manutencao.vue";

it("edita pelo id da linha, não pelo índice", async () => {
  getManutencoesMock.mockResolvedValue([{ id: 55, observacao: "a" }]);
  putManutencaoMock.mockResolvedValue({ id: 55, observacao: "b" });

  const wrapper = mount(Manutencao);
  await flushPromises();
  await wrapper.findComponent({ name: "FormTable" }).vm.$emit("edit", { data: { observacao: "b" }, index: 0 });

  expect(putManutencaoMock).toHaveBeenCalledWith("55", { observacao: "b" });
});
```

Padrões úteis nas telas do eTutor:

- Aba recebe o id por prop: `mount(Aba, { props: { usuarioId: "10" } })`; seção por rota lê o id de
  `useRoute()` — mocke `vue-router`.
- Lista com `manualEvents`: emita `add`/`edit`/`delete` no `FormTable` stub e verifique **qual
  endpoint** foi chamado, **com qual id** e se a lista foi recarregada/atualizada. Lembre os payloads:
  `edit` → `{ data, index }`, `delete` → `{ index, ...linha }`.
- Validação no `onAdd` (ex.: vínculo repetido em `manter-usuario/views/abas/Vinculos.vue`): avisa com
  `toast.showWarning` e **não** chama a API.
- Coluna de ações: pegue `props("columns")` e chame `onClick({ rowData })` direto.

## Sessão, guard e login

- Monte um app mínimo com `createPinia()` + `AuthPlugin({...})` real (ver `src/router/guard.spec.ts`)
  e limpe `localStorage`/`sessionStorage` no `beforeEach` — sessão e tokens vivem lá.
- Para afirmar sobre o redirect de login, use `urlDoLogin(auth, destino)` de `src/test-utils/login.ts`.
- Regra de acesso por perfil: estenda `src/modules/portal/modulos.spec.ts`
  (`describe("acesso por perfil")`).

## Integração (MSW) e E2E

- **Integração feature**: `npm run test:integration:feature` usa `vitest.integration.config.ts`, que
  carrega `src/test/setup.ts` **e** `src/test/msw/setup.ts`. Crie o `setup.ts` do MSW
  (`setupServer` + `beforeAll/afterEach/afterAll`) e os handlers por back-end, casando pelo
  context-path (`*/gestao-indicadores-ws/veiculo`, `*/controle-acesso-ws/usuarios/logado`, ...) — no
  app real o axios usa `VITE_API_HOST` como `baseURL`, então não dependa de path relativo. Use para
  fluxo com `router` + `pinia` + HTTP mockado.
- **E2E (Playwright)**: jornadas críticas em `e2e/`. `baseURL` é `http://localhost:8080`
  (`PLAYWRIGHT_BASE_URL` sobrepõe) e, sem essa variável, o Playwright sobe o `npm run dev` sozinho.
  Toda rota fora do portal exige login OAuth2 (popup no `servidor-de-autenticacao`) e perfil ativo
  na sessão — a jornada começa pelo login, ou o teste injeta sessão/token. O `npm run test:e2e`
  ignora `@real-scan`: marque com essa tag qualquer teste que dependa de back-end real.

## Comandos

```bash
npm test                            # vitest run (unit/component)
npm run test:watch
npm run test:coverage
npm run test:integration:feature    # MSW
npm run test:e2e                    # Playwright (ignora @real-scan)
```
