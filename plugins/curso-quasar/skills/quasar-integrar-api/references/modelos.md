# Modelos — integração de um recurso

Exemplo fictício: recurso `/suppliers/` → entidade **Fornecedor**. Troque
nomes, URL e campos pelos do Swagger; mantenha a estrutura.

## 1. Tipo — `src/types/fornecedor.ts`

```ts
/** Formato exato de `Supplier` na API (GET /suppliers/). */
export interface Fornecedor {
  id: string
  name: string
  is_active: boolean
  created_at: string
  updated_at: string
}

/** Campos que a API aceita em POST/PATCH /suppliers/ */
export interface FornecedorInput {
  name: string
  is_active?: boolean
}
```

## 2. Service — `src/services/fornecedoresService.ts`

```ts
import { api } from './api'
import type { Paginated, PageParams } from '@/types/api'
import type { Fornecedor, FornecedorInput } from '@/types/fornecedor'

/**
 * Service = só HTTP. Sem estado, sem tela.
 * Sem DELETE na API: "excluir" = `is_active: false` via `update`.
 */
export const fornecedoresService = {
  async listPage(params: PageParams) {
    const { data } = await api.get<Paginated<Fornecedor>>('/suppliers/', {
      params
    })
    return data
  },

  async create(input: FornecedorInput) {
    const { data } = await api.post<Fornecedor>('/suppliers/', input)
    return data
  },

  async update(id: string, patch: Partial<FornecedorInput>) {
    const { data } = await api.patch<Fornecedor>(`/suppliers/${id}/`, patch)
    return data
  }
}
```

## 3. Composable — `src/composables/useFornecedores.ts`

```ts
import { ref } from 'vue'
import { fornecedoresService } from '@/services/fornecedoresService'
import { getErrorMessage } from '@/services/helpers'
import type { PageParams } from '@/types/api'
import type { Fornecedor, FornecedorInput } from '@/types/fornecedor'

// Escopo do módulo: toda tela que chamar useFornecedores() vê o MESMO estado
const fornecedores = ref<Fornecedor[]>([])
const totalFornecedores = ref(0)
const loading = ref(false)
const error = ref<string | null>(null)

// Leitura: erro vai para `error` (a page mostra banner + "tentar novamente")
async function fetchFornecedores(params: PageParams) {
  loading.value = true
  error.value = null
  try {
    const pagina = await fornecedoresService.listPage(params)
    fornecedores.value = pagina.results
    totalFornecedores.value = pagina.total
  } catch (err) {
    error.value = getErrorMessage(err)
  } finally {
    loading.value = false
  }
}

// Escrita: o erro SOBE para a page decidir (Notify, manter dialog aberto)
function add(input: FornecedorInput) {
  return fornecedoresService.create(input)
}

async function update(id: string, patch: Partial<FornecedorInput>) {
  const atualizado = await fornecedoresService.update(id, patch)
  const index = fornecedores.value.findIndex(f => f.id === id)
  if (index !== -1) fornecedores.value[index] = atualizado
}

function toggleActive(fornecedor: Fornecedor) {
  return update(fornecedor.id, { is_active: !fornecedor.is_active })
}

export function useFornecedores() {
  return {
    fornecedores,
    totalFornecedores,
    loading,
    error,
    fetchFornecedores,
    add,
    update,
    toggleActive
  }
}
```

## 4. Page — trechos obrigatórios

```vue
<q-banner v-if="error" class="bg-red-1 text-negative q-mb-md" rounded>
  {{ error }}
  <template #action>
    <q-btn flat label="Tentar novamente" @click="reload" />
  </template>
</q-banner>

<q-table
  v-model:pagination="pagination"
  :rows="fornecedores"
  :columns="columns"
  :loading="loading"
  :rows-per-page-options="[10, 20, 50]"
  row-key="id"
  flat
  @request="onRequest"
/>
```

```ts
const pagination = ref({ page: 1, rowsPerPage: 20, rowsNumber: 0 })

async function loadPage(page: number, rowsPerPage: number) {
  await fetchFornecedores({ page, size: rowsPerPage })
  pagination.value = { page, rowsPerPage, rowsNumber: totalFornecedores.value }
}

function onRequest(props: {
  pagination: { page: number; rowsPerPage: number }
}) {
  return loadPage(props.pagination.page, props.pagination.rowsPerPage)
}

function reload() {
  return loadPage(pagination.value.page, pagination.value.rowsPerPage)
}

onMounted(reload)

async function onSave(data: FornecedorInput) {
  saving.value = true
  try {
    if (editing.value) await update(editing.value.id, data)
    else {
      await add(data)
      await reload() // total e página mudam
    }
    dialogOpen.value = false
    $q.notify({ type: 'positive', message: 'Fornecedor salvo' })
  } catch (err) {
    // dialog continua aberto para o usuário corrigir
    $q.notify({ type: 'negative', message: getErrorMessage(err) })
  } finally {
    saving.value = false
  }
}
```

## Checklist final

- [ ] Tipo espelha o JSON real (conferido no Swagger ou numa resposta real)
- [ ] `params` tipado como `PageParams`
- [ ] `ref`s fora da função no composable
- [ ] Leitura → `error`; escrita → lança para a page
- [ ] Todo `catch` da page usa `getErrorMessage`
- [ ] `pnpm typecheck` e `pnpm lint:check` sem erros
- [ ] Testado no navegador: listar, paginar, criar, editar, ativar/desativar,
      erro de validação (ex.: nome repetido) mantém o dialog aberto
