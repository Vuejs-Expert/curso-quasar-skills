# Usuários (admin) — criar em 2 chamadas

Mesmo padrão de Categorias. O que muda é o que a API permite.

Pré-requisito: autenticação pronta (o tipo `Usuario` e o `Role` já existem
em `src/types/usuario.ts`, e a store `useAuthStore` tem o usuário logado).

## API

```
GET   /users/?page=1&size=20    lista paginada (admin)
POST  /users/                   { email, password } — NÃO aceita role (nasce customer)
PATCH /users/{id}/              { email?, role?, is_active? }
(sem DELETE; a senha NÃO muda pelo PATCH)
```

- `role` só aceita `admin` ou `customer`. Perfis do mock como "Gerente" ou
  "Atendente" não existem, e o `User` da API **não tem nome**.
- `id` do usuário é **número** (os outros recursos usam uuid `string`).
- Senha: mínimo 8 caracteres (regra da API).

## Tipos — completar `src/types/usuario.ts`

`Role` e `Usuario` vieram da autenticação. Acrescente:

```ts
/** Corpo de POST /users/ — a API não aceita `role` no cadastro */
export interface NovoUsuario {
  email: string
  password: string
}

/** Campos que a API aceita em PATCH /users/{id}/ */
export interface UsuarioUpdate {
  email?: string
  role?: Role
  is_active?: boolean
}

/** O que o formulário de usuário do backoffice coleta */
export interface UsuarioForm extends NovoUsuario {
  role: Role
}
```

Se o mock tinha um tipo de "usuário logado" separado, ele some: a sessão usa
o mesmo `Usuario`.

## Rótulos — `src/constants/roles.ts`

```ts
import type { Role } from '@/types/usuario'

// O valor é da API (`admin`/`customer`); o rótulo em português é só da tela
export const ROLE_LABELS: Record<Role, string> = {
  admin: 'Administrador',
  customer: 'Cliente'
}

export const ROLE_OPTIONS = [
  { label: ROLE_LABELS.admin, value: 'admin' },
  { label: ROLE_LABELS.customer, value: 'customer' }
]
```

## Service — `src/services/usuariosService.ts`

```ts
import { api } from './api'
import type { Paginated, PageParams } from '@/types/api'
import type { NovoUsuario, Usuario, UsuarioUpdate } from '@/types/usuario'

/**
 * Sem DELETE: "excluir" um usuário é desativá-lo (`is_active: false`).
 * O cadastro (POST) não aceita `role` — sempre nasce `customer`; para criar
 * um admin é preciso um PATCH em seguida (feito no composable).
 */
export const usuariosService = {
  async listPage(params: PageParams) {
    const { data } = await api.get<Paginated<Usuario>>('/users/', { params })
    return data
  },

  async create(novo: NovoUsuario) {
    const { data } = await api.post<Usuario>('/users/', novo)
    return data
  },

  async update(id: number, patch: UsuarioUpdate) {
    const { data } = await api.patch<Usuario>(`/users/${id}/`, patch)
    return data
  }
}
```

## Composable — `src/composables/useUsuarios.ts`

Igual ao modelo (`usuarios`, `totalUsuarios`, `loading`, `error`,
`fetchUsuarios`, `update`, `toggleActive`). O `add` orquestra as 2
chamadas — **o service tem uma função por endpoint; quem combina é o
composable**:

```ts
/**
 * Criar usuário = 2 chamadas: a API cadastra todo mundo como `customer`,
 * então para criar um administrador fazemos um PATCH logo depois.
 * (Quem chamou recarrega a página da tabela.)
 */
async function add(novo: UsuarioForm) {
  const criado = await usuariosService.create({
    email: novo.email,
    password: novo.password
  })
  if (novo.role !== criado.role) {
    await usuariosService.update(criado.id, { role: novo.role })
  }
}
```

## Dialog — `UsuarioFormDialog.vue`

Mesmo esqueleto do dialog de Categorias, com `reactive<UsuarioForm>` e:

- campo **Senha só no cadastro** (`v-if="!usuario"`), regra
  `v => v.length >= 8 || 'Mínimo de 8 caracteres'`;
- `q-select` de perfil com `:options="ROLE_OPTIONS"`, `emit-value` e
  `map-options` (mostra "Administrador", guarda `'admin'`);
- perfil **travado para o próprio admin**:
  `:disable="usuario?.id === authStore.user?.id"` (senão ele perde o acesso);
- no `watch(modelValue)`: preencher `email` e `role` do usuário editado e
  zerar `password`.

## Page — `UsuariosPage.vue`

- Colunas: e-mail, perfil (`q-badge` com `ROLE_LABELS[props.row.role as Role]`),
  ativo (toggle), ações.
- Toggle de ativo **desabilitado na linha do próprio admin**
  (`:disable="props.row.id === authStore.user?.id"`).
- Sem botão de excluir (sem DELETE).
- `onSave`: na edição envie só `{ email, role }` (a senha não muda por
  PATCH); na criação chame `add(form)` e depois `reload()`.

## Testar

- [ ] Lista paginada vem da API, com o badge do perfil
- [ ] Na linha do próprio admin o toggle está desabilitado; no dialog dele o
      perfil está travado e não há campo de senha
- [ ] Desativar/reativar outra conta → PATCH 200
- [ ] Criar um administrador → POST 201 e depois PATCH com `role` (cria uma
      conta real na API: use um e-mail de teste)
- [ ] Senha com menos de 8 caracteres → regra do formulário bloqueia
