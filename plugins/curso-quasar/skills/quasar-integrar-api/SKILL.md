---
name: quasar-integrar-api
description: Padrão do curso para integrar a API (DRF + JWT) no front Quasar/Vue com TypeScript — base do axios, autenticação JWT (login, token, refresh, guard de rota) e recursos de CRUD (types-first, service HTTP sem estado, composable com ref no escopo do módulo, page com q-table paginado no servidor, dialog e Notify). Use sempre que for configurar o axios ou o login, integrar, criar ou alterar um endpoint, service, composable, tipo de API ou tela de CRUD, ou trocar um mock pela API real.
---

# Integrar um recurso da API

Esta skill define **um jeito único** de integrar a API do curso num projeto
Quasar 2 + Vue 3 + TypeScript. Siga-a à risca: o objetivo é código **simples,
explícito e igual em todas as telas** — é projeto de curso, então prefira o
óbvio a abstrações espertas.

Arquivos de apoio (leia quando o passo pedir):

- [references/infraestrutura.md](references/infraestrutura.md) — base que
  precisa existir antes de tudo (axios, proxy, erros, paginação).
- [references/autenticacao.md](references/autenticacao.md) — login JWT,
  tokens, interceptors, sessão e guard de rota.
- [references/modelos.md](references/modelos.md) — modelos de tipo, service,
  composable e page de um CRUD. **É a referência de estilo.**

## Ordem das etapas

A integração tem três etapas, sempre nesta ordem. Descubra em qual o
projeto está e faça **só a etapa pedida** (ou a primeira que faltar):

1. **Base** — axios, proxy, helpers de erro e paginação (Passo 0 abaixo).
2. **Autenticação** — login, tokens, interceptors, guard. Feita uma vez,
   seguindo [references/autenticacao.md](references/autenticacao.md) na
   ordem que ele define, um passo testável por vez.
3. **Recursos (CRUD)** — um por vez, pelo passo a passo abaixo.

Se o aluno pedir um CRUD de área protegida e a autenticação ainda não
existir, **não integre o CRUD**: diga que falta a etapa 2 e ofereça fazê-la
primeiro.

## Arquitetura (quem faz o quê)

```
Page (.vue)        → o que o usuário vê: onMounted, paginação do q-table, $q.notify
  └─ Composable    → estado compartilhado (ref no escopo do MÓDULO) + loading/error
       └─ Service  → só HTTP: URL, verbo, params tipados. Sem estado, sem tela
            └─ api → instância única do axios (src/services/api.ts)
Store Pinia        → SÓ estado transversal (auth, carrinho). Nunca dados de uma tela
```

Uma camada nunca pula a outra: page não importa `api` nem service (exceção:
`getErrorMessage` de `@/services/helpers`); service não importa `vue`,
`quasar` nem stores.

## Passo 0 — Conferir a base do projeto

Antes de integrar qualquer recurso, confira se estes arquivos existem:

- `src/services/api.ts`, `src/services/helpers.ts`, `src/types/api.ts`
- `devServer.proxy` para `/api` e os plugins `Notify` no `quasar.config.ts`
- `axios` instalado

O que faltar, crie **exatamente** como em
[references/infraestrutura.md](references/infraestrutura.md) e avise o aluno
do que foi criado. Se o projeto já tem algum recurso integrado, leia-o e
mantenha o mesmo estilo.

## Autenticação

Siga [references/autenticacao.md](references/autenticacao.md) exatamente:
uma store de sessão única (substitui as stores de auth falsas das áreas),
token no `LocalStorage` do Quasar, interceptor de request, guard com
`restoreSession` e, por último, o interceptor de refresh. Faça um passo por
vez e, ao fim de cada um, diga o que testar no navegador antes de seguir.
Use os nomes de rota que já existem no projeto.

## Passo a passo de um recurso (siga nesta ordem)

1. **Descubra o contrato real da API** — não invente campos. Consulte o
   Swagger (`/api/docs/`) ou faça uma chamada real. Anote: URL (com barra
   final), verbos existentes (normalmente **não há DELETE**), campos do GET,
   campos aceitos no POST/PATCH, se a listagem é paginada.
2. **Types-first** (`src/types/<entidade>.ts`): a interface espelha o JSON
   **exatamente** — `snake_case`, `is_active`, ids `uuid` como `string`.
   Sem mapper/tradução: o front se adapta à API. Crie também
   `<Entidade>Input` com os campos aceitos na escrita. Rode o typecheck
   (`pnpm typecheck` / `vue-tsc --noEmit`): **cada erro é um item do
   checklist** da integração. Faça uma entidade por vez.
3. **Service** (`src/services/<entidades>Service.ts`): objeto
   `export const xService = { listPage, listAll?, create, update }`, cada
   função `async`, desestrutura `{ data }` e usa o genérico do axios
   (`api.get<Paginated<X>>`). `params` **sempre** tipado como `PageParams`
   (o axios aceita qualquer objeto — o typecheck não te salva aqui).
4. **Composable** (`src/composables/use<Entidades>.ts`): `ref`s declarados
   **fora** da função (estado compartilhado entre telas). Se havia mock,
   remova o import do mock. Leitura grava o erro em `error` (não lança);
   escrita **deixa o erro subir** para a page. Atualize o item no array com a
   resposta do PATCH em vez de recarregar tudo.
5. **Page**: `q-table` com paginação no servidor (`v-model:pagination` com
   `rowsNumber`, `@request`), `q-banner` de erro com "Tentar novamente",
   dialog de formulário com prop `saving`, e `$q.notify` com
   `getErrorMessage(err)` em todo `catch`.
6. **Verifique**: typecheck e lint sem erros **e** teste a tela no
   navegador — os slots do `q-table` tipam `row` como `any`, então o
   compilador não pega campo renomeado lá dentro.

## Regras que não se negociam

- `<script setup lang="ts">`, Composition API, `defineProps<{...}>()` /
  `defineEmits<{...}>()` na forma de tipo. Pinia só em setup-store.
- Tipos importados com `import type` de `@/types/...`. Alias `@` = `src/`.
- Nada de `any`. Nada de `try/catch` vazio. Nada de `console.log` esquecido.
- Erro para o usuário sempre via `getErrorMessage(err)` — nunca
  `err.response.data` direto na tela.
- Listagem de admin = paginação no servidor (`listPage`). Lista pequena de
  apoio (select, chips) = `listAll()` com `size: 100` (máximo da API).
- "Excluir" = alternar `is_active` (a API não tem DELETE). Só use DELETE se o
  Swagger mostrar que existe.
- Token e refresh vivem só em `src/boot/axios.ts`: nunca coloque header
  `Authorization` nem lógica de refresh em service ou page.
- Não crie camada genérica (`createCrudService`, `useCrud`, tabela de rotas):
  repetir o padrão explícito é intencional neste curso.
- Não formate à mão contra o formatter do projeto; rode o lint dele.
- Comentários em português, explicando o **porquê**.

## Componentes Quasar

Se o MCP `quasar` estiver disponível, **consulte-o antes de usar uma prop,
slot ou evento** de componente/plugin (`get_api` para `QTable`, `QSelect`,
`Notify`…; `search_docs`/`get_page` para guias). Ele responde com a versão
instalada no projeto — não confie na memória para a API do Quasar.

## Quando parar e perguntar

- O contrato da API está ambíguo ou o endpoint não existe/retorna diferente
  do Swagger → mostre o que encontrou e pergunte, não invente campo.
- A tarefa parece pedir Pinia para dados de uma única tela → proponha
  composable e explique o porquê.
- A API exige mais de uma chamada para uma operação (ex.: criar admin =
  POST + PATCH role) → faça no composable e comente o motivo.
- A estrutura do projeto é muito diferente da descrita aqui (sem TypeScript,
  Options API, outro client HTTP) → descreva a diferença e pergunte antes.

## Ao terminar, responda com

1. Arquivos criados/alterados (um por linha, com o papel de cada um).
2. Resultado do typecheck e do lint.
3. O que ainda precisa ser testado no navegador (roteiro curto de cliques).
4. Qualquer divergência encontrada entre Swagger e API real.
