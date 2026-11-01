# Clientes (admin) — só listar e editar

Pré-requisito: autenticação pronta (`Cliente`, `ClienteInput` e o
`clientesService` com `getMe`/`createMe` já existem).

## API

```
GET   /customers/?page=1&size=20   lista paginada (admin)
PATCH /customers/{id}/             { name?, phone?, document? }
(sem POST para o admin: o cliente cria o próprio perfil pelo app; sem DELETE)
```

- O `Customer` **não tem e-mail nem `is_active`** (isso fica no `User`,
  ligado pelo campo `user`).
- `document` (CPF) não é validado pela API (aceitou `"abc"`): a validação
  fica no front, com máscara e regra.

## Service — acrescentar ao `src/services/clientesService.ts`

```ts
// no topo, junto dos imports que já existem
import type { Paginated, PageParams } from '@/types/api'

// dentro do objeto clientesService, antes de getMe:
  // ---- Backoffice (admin) ----

  async listPage(params: PageParams) {
    const { data } = await api.get<Paginated<Cliente>>('/customers/', {
      params
    })
    return data
  },

  async update(id: string, patch: Partial<ClienteInput>) {
    const { data } = await api.patch<Cliente>(`/customers/${id}/`, patch)
    return data
  },

  // ---- App (o próprio cliente logado) ----
  // getMe / createMe (já existem)
```

Não crie `create` para o admin.

## Composable — `src/composables/useClientes.ts`

```ts
import { ref } from 'vue'
import { clientesService } from '@/services/clientesService'
import { getErrorMessage } from '@/services/helpers'
import type { PageParams } from '@/types/api'
import type { Cliente, ClienteInput } from '@/types/cliente'

// Página atual da tabela (paginação no servidor) + total na API
const clientes = ref<Cliente[]>([])
const totalClientes = ref(0)
const loading = ref(false)
const error = ref<string | null>(null)

async function fetchClientes(params: PageParams) {
  loading.value = true
  error.value = null
  try {
    const pagina = await clientesService.listPage(params)
    clientes.value = pagina.results
    totalClientes.value = pagina.total
  } catch (err) {
    error.value = getErrorMessage(err)
  } finally {
    loading.value = false
  }
}

// Só edição: quem cria o perfil é o próprio cliente, pelo app
async function update(id: string, patch: Partial<ClienteInput>) {
  const atualizado = await clientesService.update(id, patch)
  const index = clientes.value.findIndex(c => c.id === id)
  if (index !== -1) clientes.value[index] = atualizado
}

export function useClientes() {
  return { clientes, totalClientes, loading, error, fetchClientes, update }
}
```

## Formatar CPF e telefone — `src/utils/documento.ts`

```ts
// A API guarda só os dígitos (o form usa `unmasked-value`); a máscara é
// aplicada apenas na exibição. Valores fora do formato aparecem como vieram.

/** "12345678909" → "123.456.789-09" */
export function formatCpf(cpf: string) {
  return cpf.replace(/^(\d{3})(\d{3})(\d{3})(\d{2})$/, '$1.$2.$3-$4')
}

/** "11987654321" → "(11) 98765-4321" */
export function formatTelefone(telefone: string) {
  return telefone.replace(/^(\d{2})(\d{4,5})(\d{4})$/, '($1) $2-$3')
}
```

## Dialog — `ClienteFormDialog.vue`

Esqueleto do dialog de Categorias, com `reactive<ClienteInput>` e três
campos:

- Nome: regra `v => !!v.trim() || 'Informe o nome'`
- Telefone: `mask="(##) #####-####"` + `unmasked-value`
- CPF: `mask="###.###.###-##"` + `unmasked-value` + regra
  `v => v.length === 11 || 'CPF deve ter 11 dígitos'`

`unmasked-value` faz o `v-model` guardar só os dígitos — é o que a API
recebe.

## Page — `ClientesPage.vue`

Fica **mais simples** que o mock, para refletir a API:

- Sem botão "Novo Cliente", sem excluir e sem toggle de ativo.
- Sem coluna de e-mail (exigiria cruzar com `/users/`) e sem colunas que
  contavam pelo mock (ex.: "Pedidos" — volta na aula de Pedidos).
- Colunas de CPF e telefone com `format: formatCpf` / `format: formatTelefone`
  na definição da coluna (`QTableColumn`).
- Paginação no servidor como em Categorias; botão de editar abre o dialog.

## Testar

- [ ] Lista paginada vem da API, sem botão "Novo"
- [ ] Editar nome, telefone e CPF → PATCH 200; a tabela mostra
      `123.456.789-09` e `(11) 98765-4321`
- [ ] CPF incompleto → a regra do formulário bloqueia
