---
name: comentarios
description: Regra do projeto eTutor sobre comentários no código-fonte. Use SEMPRE antes de escrever, revisar ou decidir se um comentário vai no código — ao criar/editar arquivos em qualquer repositório do eTutor (etutor-web, etutor-lib, back-ends), ao portar lógica de outro back-end, ou ao explicar uma decisão não óbvia.
---

# Comentários no código — regra do eTutor

Decisão do projeto (2026-09-23, commit "docs: extrai comentários de código para docs/ e limpa o
código-fonte" em `etutor-web`): regras de negócio, restrições técnicas, bugs conhecidos do
back-end/legado e decisões não óbvias **não vivem como comentário no código-fonte**. Vivem em
`docs/<módulo>.md` no repositório do front (`projeto-atualizado/front-end/etutor-web/docs/`), um
arquivo por módulo, indexados em `docs/README.md`.

## O que fazer

1. **Regra de negócio, bug de back-end, divergência de contrato, decisão de arquitetura não óbvia**
   → vai em `docs/<módulo>.md`, não em comentário. Se o arquivo do módulo ainda não existe, crie.
2. **CLAUDE.md** (de cada repositório) cobre só estrutura e convenções mecânicas — como as pastas se
   organizam, como registrar uma rota, onde fica cada back-end. Nunca a regra de negócio em si; só um
   ponteiro para o `docs/*.md` correspondente quando fizer sentido.
3. Comentário no código só quando o **WHY não é óbvio nem pelo nome das coisas nem pela leitura do
   arquivo** — e mesmo assim, curto (uma linha, não um bloco). Prefira `// Ver docs/<módulo>.md.` a
   reexplicar a regra inline.
4. **Nunca** deixe comentário tipo changelog ("migrado de X", "antes isso vinha de Y", "adicionado em
   tal data") no código — isso é histórico de decisão, pertence ao `docs/*.md` ou à mensagem de commit,
   não ao arquivo vivo.
5. **Código comentado não fica no arquivo.** Bloco desativado "para usar depois" é apagado — o git
   guarda a versão anterior. (Ex. a não repetir: `escola/.../manter-estrutura-escola/composables/useEstrutura.ts`
   inteiro comentado.)

## O que evitar (visto em revisões reais do projeto)

- Docblock explicando por que um endpoint mudou de contrato → devia estar em `docs/gestor.md` (ou
  módulo equivalente), não como comentário acima da função.
- Comentário `⚠️ Gap conhecido do back-end: ...` deixado no service do front para reportar um bug de
  back-end → reporte no `docs/<módulo>.md` (seção de gaps conhecidos) ou fora do código; o código não
  é o canal de comunicação com o time de back-end.
- Comentário explicando a inversão de ordem de campos num formulário, motivada por uma regra de API →
  a regra vai no doc; no código, no máximo `// Ver docs/gestor.md.`

## Onde documentar cada tipo de conteúdo

| Conteúdo | Onde |
|---|---|
| Estrutura de pastas, convenção de rotas, registro automático | `CLAUDE.md` do repositório |
| Regra de negócio, bug de back-end, divergência de contrato | `docs/<módulo>.md` |
| Decisão de arquitetura ampla (login único, sem menu central) | `CLAUDE.md`, seção curta + ponteiro pro `docs/*.md` se houver detalhe |
| WHY pontual e não óbvio de uma linha de código específica | Comentário de uma linha, no local exato |
| Histórico de mudança, data, "antes vs depois" | Mensagem de commit — nunca no código nem no CLAUDE.md |
