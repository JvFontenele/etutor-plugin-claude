# HTTP — back-ends, serviços e useRequests

## Os back-ends do eTutor

Todos os back-ends vivem no **mesmo host por ambiente** (`VITE_API_HOST`, ex.:
`https://dev.portaletutor.com.br`), distinguidos só pelo **context-path**. `main.ts` passa o host
como `baseUrl` de `etutorLib()` e cada cliente `useHttp({ path })` prefixa o seu context-path — a
chamada vai direto ao back-end, via CORS. Não há gateway único.

| `path` do `useHttp` | Back-end | Quem consome hoje |
|---|---|---|
| `/controle-acesso-ws` | `controle-acesso-ws` — usuários, perfis, instituições, domínios | `controle-acesso`, header, lib (`getUsuarioLogado`) |
| `/escola-ws` | `escola-ws` | `escola`, `aluno`, `gestor` |
| `/aluno-ws` | `aluno-ws` | `aluno`, `diario` |
| `/diario-ws/api` | `diario-ws` — inclusive `/instituicoes` e `/unidades` (lista inteira, para selects) | `diario`, `escola`, `gestor`, `controle-acesso`, header |
| `/gestao-documentos-ws` | `gestao-documentos-ws` — URL pré-assinada do S3, relatórios PDF | lib (`FormFileManager`), `aluno`, `diario`, `escola`, `gestor`, `professor` |
| `/gestao-indicadores-ws` | `gestao-indicadores-ws` — monitoramento, dashboard, veículos, transporte | `gestor`, `portal`, `core/sessao` |
| `/servidor-ws` | `servidor-ws` | `professor` |
| `/portal-aluno-ws` | `portal-aluno-ws` | `portal-aluno`, `core/sessao` |
| `/secretaria-api` | secretaria (matriz: mediação, modalidade, nível, etapa) | `gestor` (`services/secretaria-ws`) |
| `/apietutor-0.0.1` | `api-etutor` (legado) | `gestor` — só o que ainda não tem equivalente nos `*-ws` (ver `docs/gestor.md`) |

Código de cada `*-ws`: `projeto-atualizado/back-end/<ws>`. Contrato publicado:
`https://dev.portaletutor.com.br/<ws>/v3/api-docs`. **Confira o api-docs antes de assumir que uma
rota do repositório está no ar** — o deploy pode estar atrás do código (`docs/diario.md` tem um caso).

### Autenticação e cabeçalhos

Os `*-ws` são resource servers OAuth2 do mesmo issuer do `servidor-de-autenticacao`: sem
`Authorization: Bearer` devolvem 401. Quem injeta o Bearer é o interceptor do `AxiosPlugin` da lib,
**só quando há token**, junto com os cabeçalhos de contexto da sessão (`X-Instituicao-Id`,
`X-Escola-Id`, `X-Ano-Letivo-Id`, `X-Perfil-Id` — `cabecalhosDaSessao` em `src/core/sessao/session.ts`,
registrado por `setProvedorCabecalhos` no `main.ts`). Um 401 com refresh token guardado dispara o
refresh automaticamente. **Serviço nunca monta header de autenticação na mão.**

Rota pública (recuperação de senha, por exemplo) funciona deslogada porque, sem token, o
interceptor simplesmente não manda `Authorization`.

### Proxy do Vite

`vite.config.ts` tem proxy só para `/portal-aluno-ws` (sem CORS no back-end) e
`/gestao-documentos-ws`, com alvo fixo `https://dev.portaletutor.com.br`. Back-end novo **não** ganha
entrada no proxy: é chamado direto, por CORS.

## Serviços prontos na lib

A lib exporta os serviços que ela mesma usa ou que servem a mais de um lugar:

```ts
import {
  getUsuarioLogado, getUsuarioControleAcesso, achatarInstituicoes, // controle-acesso-ws (sessão)
  gerarUrlUpload, enviarArquivoParaS3, gerarUrlDownload,           // gestao-documentos-ws (S3)
} from "etutor-lib";
```

Os clientes `useHttp` da lib (`services/*/api.ts`) **não** são exportados. Os serviços de domínio,
endereço e escola que a lib ainda carrega (`getSexos`, `dominiosCommons`, `getSelectEstado`,
`getDownloadArquivo`, `getPaginatedEscola`, ...) apontam para `/api-commons` e `/api-escola`, da
modernização anterior, e nenhuma tela os usa — domínio vem do `*-ws` do módulo.

## Serviço do módulo

Um `services/<backend>/api.ts` **por back-end consumido**, dentro do módulo — mesmo que outro
módulo já tenha um cliente para o mesmo back-end (módulo não importa de outro módulo). Ex.: `gestor`,
`escola` e `diario` têm, cada um, o seu `services/diario-ws/api.ts`.

```ts
// src/modules/gestor/services/gestao-indicadores-ws/api.ts
import { useHttp } from "etutor-lib";

const api = useHttp({ path: "/gestao-indicadores-ws" });

export { api };
```

```ts
// src/modules/gestor/services/gestao-indicadores-ws/veiculo.ts (trecho)
import { api } from "./api";

export const getPaginatedVeiculos = api.getPaginated<VeiculoDetalhe>("/veiculo");

export const getVeiculoById = async (id: string) => await api.get<VeiculoDetalhe>(`/veiculo/${id}`);

export const postVeiculo = async (body: VeiculoRequest) => await api.post<VeiculoDetalhe>("/veiculo", body);

export const putVeiculo = async (id: string, body: VeiculoRequest) =>
  await api.put<VeiculoDetalhe>(`/veiculo/${id}`, body);

export const getSelectModeloVeiculo = async () => await api.get<DominioGestor[]>("/modeloVeiculo/select");
```

Métodos do `useHttp`: `get`, `getPaginated`, `post`, `put`, `del`. Todos aceitam um
`AxiosRequestConfig` no último argumento (`params`, `headers`, `responseType`).

Cliente usado **fora** de um módulo fica com o dono: `src/core/sessao/services/<backend>/api.ts`
(sessão) ou `src/components/header/services/<backend>/api.ts` (header). Não existe `src/services`.

### Validação do back-end que quebra o select

Constraints do Spring valem também para o estado inicial da tela. `@Size(min = 5)` em `nome` faz
uma string vazia (o select antes de o usuário digitar) devolver **400** — guarde localmente
(`src/modules/escola/services/escola-ws/escola.ts`):

```ts
export const getEscolasParaSelect = async (params: { nome?: string; page?: number; size?: number }) => {
  const nome = params.nome?.trim() ?? "";

  if (nome.length < 5) {
    return { content: [] as EscolaBusca[], totalPages: 0, totalElements: 0, number: 0, size: params.size ?? 20 };
  }

  return await api.get<PaginatedResponseDTO<EscolaBusca>>("/buscar/escola", {
    params: { nome, page: params.page ?? 0, size: params.size ?? 20 },
  });
};
```

Esse é o formato que o `selectPagination` consome: recebe `{ ...fnParams, nome, page, size }` num
objeto e devolve a página.

### GET que devolve lista, não página

Vários `*-ws` devolvem `List<T>` puro (todo o `diario-ws`, `GET /perfis` do `controle-acesso-ws`,
monitoramento do `gestao-indicadores-ws`). Dois adaptadores já existem — não crie um terceiro:

- **`comoPagina(itens)`** (`src/modules/controle-acesso/services/controle-acesso-ws/paginacao.ts`,
  idem em `diario/services/diario-ws/paginacao.ts`) — embrulha a lista num `PaginatedResponseDTO` de
  uma página, para `<DataTable :page>`.
- **`paginarLocal(itens, params)`** (`src/modules/gestor/utils/paginacaoLocal.ts`) — fatia e ordena no
  front respeitando `page`/`size`/`sort` (`-campo` = desc), para servir de `queryFn` do
  `PageSearchTable`.

Uso de cada um: [views-tabelas.md](views-tabelas.md#get-não-paginado).

### Upload de arquivo (S3)

O upload não passa pelo back-end de domínio: o front pede uma URL pré-assinada ao
`gestao-documentos-ws`, sobe o arquivo direto no S3 e guarda o **`identificador`** devolvido no
registro de domínio. Não há endpoint de confirmação depois do `PUT`.

- **Caso comum: `is: "formFileManager"`** — o componente já faz o fluxo inteiro. Persista só o
  identificador com `transform`:

  ```ts
  // src/modules/gestor/applications/manter-veiculo/composables/useFieldsDocumento.ts
  caminho: {
    is: "formFileManager",
    name: "caminho",
    label: "Arquivo",
    cols: 12,
    isRequired: true,
    props: { deletable: true, maxFiles: 1 },
    transform: (val: Array<{ identificador: string }>) => val?.[0]?.identificador,
  },
  ```

  Na edição, devolva o identificador salvo como item do model:
  `data.value.logo = result.identificadorLogo ? [{ name: "Logo", identificador: result.identificadorLogo }] : []`
  (`controle-acesso/applications/manter-instituicao/composables/useFields.ts`) — o componente busca a
  URL de leitura sozinho.

- **Fluxo manual** (ex.: foto do aluno): o módulo tem o próprio
  `services/gestao-documentos-ws/arquivo.ts` (`gerarUrlUpload`, `gerarUrlDownload`,
  `enviarArquivoParaS3`) — service fica com o dono, como qualquer outro. Modelo:
  `aluno/services/gestao-documentos-ws/arquivo.ts`. Fluxo: `gerarUrlUpload(nome, contentType)` →
  `enviarArquivoParaS3(url, file, contentType)` → manda o `identificador` ao back-end de domínio.
  O `contentType` do `PUT` tem que ser **idêntico** ao enviado em `gerarUrlUpload` (faz parte da
  assinatura) e o `PUT` vai **sem** `Authorization` (por isso é um `fetch` à parte, fora do cliente
  `useHttp`). A lib também exporta essas três funções — são as que o `FormFileManager` usa por dentro.

### Download / visualização

`gerarUrlDownload(identificador)` devolve `{ url }` pré-assinada, que serve direto em
`window.open(url)` ou `<img :src>` — sem `Blob`, sem Bearer (foto do aluno em
`aluno/applications/manter-aluno`).

### Relatórios (PDF em `Blob`)

Relatório é `GET` com `responseType: "blob"`:

```ts
// src/modules/gestor/services/gestao-documentos-ws/relatorios.ts
const relatorio = async (url: string, params?: Record<string, unknown>) =>
  await api.get<Blob>(url, { params, responseType: "blob" });

export const getRelatorioTurmasPorEscola = async (anoLetivoId: number) =>
  await relatorio(`/relatorios/escola/turmas/${anoLetivoId}`);
```

Como abrir e tratar erro na tela: [views-tabelas.md](views-tabelas.md#relatórios-pdf).

## useRequests — gerenciar chamadas HTTP

Uma `useRequests()` por application, agregando **todas** as instâncias `useHttpRequest` que as views
e os composables da application usam. Cada chamada de `useRequests()` cria instâncias novas (estado
próprio de `data`/`loading`), então o `useFields` e a view têm cada um as suas — é o padrão da lib.

```ts
// src/modules/gestor/applications/manter-veiculo/useRequests.ts (trecho)
import { useHttpRequest, useToast, MENSAGENS } from "etutor-lib";
import { useRouter } from "vue-router";
import { getPaginatedVeiculos, getVeiculoById, postVeiculo, putVeiculo, getSelectModeloVeiculo, ... }
  from "../../services/gestao-indicadores-ws/veiculo";

export function useRequests() {
  const router = useRouter();
  const toast = useToast();

  function onError(e: unknown) {
    toast.showQueryError(e);
  }

  function handleSuccessSalvar() {
    toast.showSuccess(MENSAGENS.operacaoRealizada);
    router.push({ name: "gestor-veiculos" });
  }

  return {
    getItems: getPaginatedVeiculos,                                        // direto para PageSearchTable

    getVeiculo: useHttpRequest(getVeiculoById, { onError }),
    postVeiculo: useHttpRequest(postVeiculo, { onSuccess: handleSuccessSalvar, onError }),
    putVeiculo: useHttpRequest(putVeiculo, { onSuccess: handleSuccessSalvar, onError }),

    getSelectModeloVeiculo: useHttpRequest(getSelectModeloVeiculo),        // selects sem callbacks

    getManutencoes: useHttpRequest(getManutencoesVeiculo, { onError }),    // listas: cada view trata o sucesso
    postManutencao: useHttpRequest(postManutencaoVeiculo, { onError }),
    deleteManutencao: useHttpRequest(deleteManutencaoVeiculo, { onError }),
  };
}
```

Pontos de atenção:

- `getItems` (paginado) é passado **direto** para `PageSearchTable` via `:queryFn` — não envolva em `useHttpRequest`.
  Quando a busca exige `instituicaoId` em query param, `getItems` vira uma função que o acrescenta
  (`aluno/applications/manter-aluno/useRequests.ts`, `professor/applications/dados-funcionais`):

  ```ts
  function getItems(query: QueryFnPaginatedParans) {
    return getPaginatedAlunos({ ...query, params: { ...query.params, instituicaoId: getInstituicaoId() } });
  }
  ```
- Selects (`getSelect*`) ficam **sem `onSuccess`** — são repassados como `fnOptions` e o componente
  chama `.execute()` sozinho. `{ onError }` é aceitável (vários `useRequests` de `escola` e `diario`
  o passam) para o select vazio não falhar em silêncio. Isso vale também para `selectPagination`: passe a **instância**
  `useHttpRequest`, não a função crua (o componente chama `fnOptions.execute(...)`).
- Mutações de cadastro recebem `onSuccess` (toast com `MENSAGENS.operacaoRealizada` + volta para a
  listagem) e `onError` (`toast.showQueryError`).
- Em tela de abas, a mutação de update fica com um `onSuccess` **só de toast, sem navegação**
  (salvar uma aba não sai da tela) — ex.: `putItem: useHttpRequest(putUsuario, { onSuccess: onSuccessGravacao, onError })`
  em `controle-acesso/applications/manter-usuario/useRequests.ts`. Edição em form único (sem abas)
  volta para a listagem, como o cadastro (`manter-instituicao`).
- Reexportar o `toast` no retorno evita repetir `useToast()` em cada aba.

### Callbacks por chamada — `execute(..., { onSuccess })`

O `execute` aceita um objeto de opções como **último argumento**, além dos argumentos da função de
serviço. Quando ele traz `onSuccess`, o `onSuccess` global do `useHttpRequest` **não** roda (o local
substitui, não soma); `onError` local e global rodam os dois.

```ts
// src/components/header/ModalTrocarSenha.vue
await alterarSenhaLogado.execute(form.senha, {
  onSuccess: async () => {
    toast.showSuccess("Senha alterada com sucesso! Faça login novamente.");
    model.value = false;
    encerrarSessao();
    await useAuthStore().logout();
  },
});
```

### Retorno e erro

`execute` devolve o resultado ou **`null`** em caso de erro (não relança). Por isso o padrão nas
telas é `(await getX.execute(id)) ?? []` / `if (!result) return`, ou checar
`putItem.error.value` depois. `data`, `loading` e `error` também ficam disponíveis na instância — e
`loading` já alimenta o `AppLoading` global via `useLoading`.
