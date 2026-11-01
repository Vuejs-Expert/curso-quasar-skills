# Produtos, Estoque, Cardápio e indicadores do Dashboard

A integração mais "vida real" do curso. Pré-requisitos: Categorias integrada
(`fetchTodasCategorias` para o select) e `src/utils/slugify.ts`.

Integra, nesta ordem: Produtos (admin) → Estoque (admin) → indicadores do
Dashboard → Cardápio do app. Todas as telas usam **o mesmo composable**
(`useProdutos`).

## API

Na API, um produto vendável são 3 entidades:

```
Product (vitrine)       → nome, categoria (uuid), descrição, is_active
  └─ ProductItem (SKU)   → sku, nome, preço ("32.90" em texto)
       └─ Inventory      → quantity (>= 0)
```

```
GET   /products/             produtos JÁ com category_name e
                             items: [{ id, sku, name, price, inventory_id, stock }]
POST  /products/             produto + items (com stock inicial) — 1 chamada, atômica
PATCH /products/{id}/        nome, categoria, descrição, is_active (NÃO mexe em items)
PATCH /product-items/{id}/   preço/nome do item
PATCH /inventory/{id}/       { quantity } — o id vem de items[].inventory_id
GET   /inventory/            todos os estoques (só para os indicadores)
(sem DELETE em nenhum)
```

- `items` é obrigatório e não pode ser vazio; `sku` é único; `price`
  inválido ou `quantity` negativa → `400`.
- **Atômico**: se um item falhar (ex.: SKU repetido), nada é criado.
- `price` é **texto** (`"32.90"`): converta para número ao ler e envie com
  `toFixed(2)`.
- Produtos inativos somem do GET público (cliente).
- `/inventory/` esconde `quantity` de quem não é admin (só manda `in_stock`).

## Decisão de MVP: 1 produto = 1 item

A API permite variações (P/M/G, cada uma com preço e estoque), mas o curso
usa **um item por produto** (`items[0]`). O formulário fica simples (nome,
categoria, descrição, preço) e o SKU é gerado do nome:
`slugify('Smash Duplo').toUpperCase()` → `"SMASH-DUPLO"`.

## Exceção à regra "sem mapper": a linha da tela

Aqui o composable **achata** o produto aninhado numa linha (`ProdutoCompleto`)
para as telas. É a única exceção do curso, e o motivo é concreto: quando a
API mudou o formato de produtos (de 4 GETs para 1), só o composable e os
services mudaram — **nenhuma page**. Os tipos da API continuam espelhando o
JSON exatamente; só a linha da tela é nossa.

## Tipos — `src/types/produto.ts`

```ts
/**
 * Na API, um produto vendável são 3 entidades:
 *   Produto (vitrine) → ProdutoItem (SKU + preço) → Estoque (quantidade)
 * mas o GET /products/ já devolve tudo junto (`items` com preço e estoque),
 * e o POST /products/ cria as 3 de uma vez.
 * Neste MVP usamos 1 produto = 1 item.
 */

/** Item resumido que vem DENTRO do produto (GET /products/) */
export interface ProdutoItemResumo {
  id: string
  sku: string
  name: string
  price: string // decimal em texto: "32.90"
  inventory_id: string
  stock: number
}

/** Formato exato de `Product` na API (GET /products/). */
export interface Produto {
  id: string
  category: string // uuid da categoria
  category_name: string // nome, para exibir sem buscar as categorias
  name: string
  description: string
  items: ProdutoItemResumo[]
  is_active: boolean
  created_at: string
  updated_at: string
}

/** Item enviado junto na criação do produto */
export interface ProdutoItemCreate {
  sku: string
  name: string
  price: string
  stock?: number // estoque inicial (a API cria o registro de estoque)
}

/** Corpo de POST /products/: produto + itens + estoque numa chamada só. */
export interface ProdutoCreate {
  category: string
  name: string
  description?: string
  is_active?: boolean
  items: ProdutoItemCreate[]
}

/** Campos de PATCH /products/{id}/ (itens NÃO são editados por aqui) */
export interface ProdutoUpdate {
  category?: string
  name?: string
  description?: string
  is_active?: boolean
}

/** Campos de PATCH /product-items/{id}/ (ex.: mudar o preço) */
export interface ProdutoItemUpdate {
  name?: string
  price?: string
  is_active?: boolean
}

/** O que o formulário de produto do backoffice coleta */
export interface ProdutoForm {
  name: string
  category: string
  description: string
  price: number
}

/**
 * Linha das telas (Produtos, Estoque, cardápio): o produto "achatado" com
 * seu (único) item + nome da categoria. Montada no composable — é o que
 * deixa as pages iguais mesmo quando o formato da API muda.
 */
export interface ProdutoCompleto {
  id: string // id do Produto
  name: string
  description: string
  category: string
  categoryName: string
  is_active: boolean
  itemId: string | null
  price: number | null
  estoqueId: string | null
  quantity: number | null // null quando não há item/estoque
}
```

## Tipo — `src/types/estoque.ts`

```ts
/**
 * Formato de `Inventory` na API (GET /inventory/). Um registro por
 * ProdutoItem. A API esconde `quantity` de quem não é admin (só manda
 * `in_stock`), por isso ele é opcional aqui.
 */
export interface Estoque {
  id: string
  product_item: string // uuid do ProdutoItem
  quantity?: number
  in_stock: boolean
  created_at: string
  updated_at: string
}
```

## Services — um por recurso da API

```ts
// src/services/produtosService.ts
import { api } from './api'
import type { Paginated, PageParams } from '@/types/api'
import type { Produto, ProdutoCreate, ProdutoUpdate } from '@/types/produto'

// Sem DELETE: "excluir" um produto é desativá-lo (`is_active: false`)
export const produtosService = {
  /** Já vem com `items` (preço, estoque e id do estoque) dentro */
  async listPage(params: PageParams) {
    const { data } = await api.get<Paginated<Produto>>('/products/', {
      params
    })
    return data
  },

  /** Produto + item + estoque numa chamada só (atômica) */
  async create(input: ProdutoCreate) {
    const { data } = await api.post<Produto>('/products/', input)
    return data
  },

  async update(id: string, patch: ProdutoUpdate) {
    const { data } = await api.patch<Produto>(`/products/${id}/`, patch)
    return data
  }
}
```

```ts
// src/services/produtoItensService.ts
import { api } from './api'
import type { ProdutoItemUpdate } from '@/types/produto'

export const produtoItensService = {
  /** Preço e nome do item são editados no próprio item */
  async update(id: string, patch: ProdutoItemUpdate) {
    await api.patch(`/product-items/${id}/`, patch)
  }
}
```

```ts
// src/services/estoqueService.ts
import { api } from './api'
import type { Paginated, PageParams } from '@/types/api'
import type { Estoque } from '@/types/estoque'

export const estoqueService = {
  /** Todos os estoques — só para os indicadores (total, baixo, esgotados) */
  async listAll() {
    const params: PageParams = { page: 1, size: 100 }
    const { data } = await api.get<Paginated<Estoque>>('/inventory/', {
      params
    })
    return data.results
  },

  /** O id vem do próprio produto: `items[0].inventory_id` */
  async update(id: string, quantity: number) {
    const { data } = await api.patch<Estoque>(`/inventory/${id}/`, {
      quantity
    })
    return data
  }
}
```

## Composable — `src/composables/useProdutos.ts`

```ts
import { computed, ref } from 'vue'
import { Notify } from 'quasar'
import { produtosService } from '@/services/produtosService'
import { produtoItensService } from '@/services/produtoItensService'
import { estoqueService } from '@/services/estoqueService'
import { getErrorMessage } from '@/services/helpers'
import { slugify } from '@/utils/slugify'
import type { PageParams } from '@/types/api'
import type { Estoque } from '@/types/estoque'
import type { Produto, ProdutoCompleto, ProdutoForm } from '@/types/produto'

// Página atual (paginação no servidor) + total de produtos na API
const produtos = ref<ProdutoCompleto[]>([])
const totalProdutos = ref(0)
// Todos os estoques — só para os indicadores (fetchIndicadores)
const estoques = ref<Estoque[]>([])
const loading = ref(false)
const error = ref<string | null>(null)

const ESTOQUE_BAIXO = 15

const totalUnidades = computed(() =>
  estoques.value.reduce((soma, e) => soma + (e.quantity ?? 0), 0)
)
const lowStockCount = computed(
  () =>
    estoques.value.filter(
      e =>
        e.quantity !== undefined && e.quantity > 0 && e.quantity < ESTOQUE_BAIXO
    ).length
)
const outOfStockCount = computed(
  () => estoques.value.filter(e => e.quantity === 0).length
)
const activeCount = computed(
  () => produtos.value.filter(p => p.is_active).length
)

/**
 * "Achata" o produto da API numa linha de tabela: o item (preço, estoque e
 * id do estoque) e o nome da categoria já vêm dentro do produto.
 * 1 produto = 1 item: usamos o primeiro.
 */
function montarLinha(produto: Produto): ProdutoCompleto {
  const item = produto.items[0]
  return {
    id: produto.id,
    name: produto.name,
    description: produto.description,
    category: produto.category,
    categoryName: produto.category_name,
    is_active: produto.is_active,
    itemId: item?.id ?? null,
    price: item ? Number(item.price) : null, // "32.90" → 32.9
    estoqueId: item?.inventory_id ?? null,
    quantity: item?.stock ?? null
  }
}

/** Uma página de produtos — 1 requisição (tudo já vem dentro do produto) */
async function fetchProdutos(params: PageParams) {
  loading.value = true
  error.value = null
  try {
    const pagina = await produtosService.listPage(params)
    produtos.value = pagina.results.map(montarLinha)
    totalProdutos.value = pagina.total
  } catch (err) {
    error.value = getErrorMessage(err)
  } finally {
    loading.value = false
  }
}

/**
 * Indicadores de estoque (total, baixo, esgotados) olham TODOS os estoques,
 * não só a página da tabela — por isso uma busca separada.
 */
async function fetchIndicadores() {
  try {
    estoques.value = await estoqueService.listAll()
  } catch (err) {
    error.value = getErrorMessage(err)
  }
}

/**
 * Criar = 1 POST: produto + item (preço) + estoque inicial juntos.
 * A API é atômica: se o item falhar (ex.: SKU repetido), nada é criado.
 * Quem chamou recarrega a página da tabela.
 */
async function add(form: ProdutoForm) {
  await produtosService.create({
    category: form.category,
    name: form.name,
    description: form.description,
    items: [
      {
        sku: slugify(form.name).toUpperCase(), // "Smash Duplo" → "SMASH-DUPLO"
        name: form.name,
        price: form.price.toFixed(2), // a API espera texto: "32.90"
        stock: 0 // o estoque é controlado na tela de Estoque
      }
    ]
  })
}

/**
 * Editar = PATCH no produto (nome, categoria, descrição) e PATCH no item
 * (preço) — o PATCH do produto não altera os itens.
 */
async function update(linha: ProdutoCompleto, form: ProdutoForm) {
  await produtosService.update(linha.id, {
    category: form.category,
    name: form.name,
    description: form.description
  })
  if (linha.itemId) {
    await produtoItensService.update(linha.itemId, {
      name: form.name,
      price: form.price.toFixed(2)
    })
  }
}

// A API não tem DELETE: "excluir" um produto é desativá-lo
async function toggleActive(linha: ProdutoCompleto) {
  const atualizado = await produtosService.update(linha.id, {
    is_active: !linha.is_active
  })
  linha.is_active = atualizado.is_active
}

// ---- Estoque ----

/**
 * Debounce por estoque: cliques seguidos no +/− mudam o número na tela na
 * hora, mas só o ÚLTIMO valor vai para a API (500ms depois do último
 * clique). Um timer por id, para dois produtos não se atropelarem.
 */
const timers = new Map<string, ReturnType<typeof setTimeout>>()

function salvarQuantidade(estoqueId: string, quantity: number) {
  clearTimeout(timers.get(estoqueId))
  timers.set(
    estoqueId,
    setTimeout(() => {
      estoqueService.update(estoqueId, quantity).catch(err => {
        Notify.create({ type: 'negative', message: getErrorMessage(err) })
      })
    }, 500)
  )
}

function setQuantity(linha: ProdutoCompleto, quantity: number) {
  if (!linha.estoqueId || quantity < 0) return
  linha.quantity = quantity
  // mantém os indicadores (que usam `estoques`) em dia
  const estoque = estoques.value.find(e => e.id === linha.estoqueId)
  if (estoque) estoque.quantity = quantity
  salvarQuantidade(linha.estoqueId, quantity)
}

function incStock(linha: ProdutoCompleto) {
  setQuantity(linha, (linha.quantity ?? 0) + 1)
}
function decStock(linha: ProdutoCompleto) {
  setQuantity(linha, (linha.quantity ?? 0) - 1)
}
function restock(linha: ProdutoCompleto) {
  setQuantity(linha, (linha.quantity ?? 0) + 20)
}

export function useProdutos() {
  return {
    produtos,
    totalProdutos,
    loading,
    error,
    totalUnidades,
    lowStockCount,
    outOfStockCount,
    activeCount,
    ESTOQUE_BAIXO,
    fetchProdutos,
    fetchIndicadores,
    add,
    update,
    toggleActive,
    incStock,
    decStock,
    restock
  }
}
```

O estoque é a **única escrita que não lança para a page**: com debounce, a
page já seguiu em frente quando o PATCH falha, então o próprio composable
avisa com `Notify.create` (o plugin `Notify`, fora de componente).

## Util — `src/utils/currency.ts` (se ainda não existir)

```ts
export function formatBRL(value: number) {
  return `R$ ${value.toFixed(2).replace('.', ',')}`
}
```

## Dialog — `ProdutoFormDialog.vue`

Props: `produto: ProdutoCompleto | null`, `categorias: Categoria[]`,
`saving: boolean`; emite `save: [payload: ProdutoForm]`. Campos:

```vue
<q-input
  v-model="form.name"
  label="Nome do produto"
  outlined
  dense
  autofocus
  :rules="[v => !!v.trim() || 'Informe o nome']"
/>
<!-- Mostra o nome, guarda o id (uuid) que a API espera -->
<q-select
  v-model="form.category"
  :options="categorias"
  option-label="name"
  option-value="id"
  emit-value
  map-options
  label="Categoria"
  outlined
  dense
  :rules="[v => !!v || 'Escolha a categoria']"
/>
<q-input
  v-model.number="form.price"
  type="number"
  step="0.01"
  min="0"
  label="Preço (R$)"
  outlined
  dense
  :rules="[v => v > 0 || 'Informe o preço']"
/>
<q-input
  v-model="form.description"
  label="Descrição"
  type="textarea"
  autogrow
  outlined
  dense
/>
<div class="text-caption text-grey-7">
  O estoque é controlado na tela de Estoque (produto novo começa com 0).
</div>
```

```ts
const form = reactive<ProdutoForm>({
  name: '',
  category: '',
  description: '',
  price: 0
})

watch(modelValue, open => {
  if (!open) return
  form.name = props.produto?.name ?? ''
  form.category = props.produto?.category ?? props.categorias[0]?.id ?? ''
  form.description = props.produto?.description ?? ''
  form.price = props.produto?.price ?? 0
})

function save() {
  emit('save', { ...form, name: form.name.trim() })
}
```

O campo de estoque **sai** do formulário (se o mock tinha).

## Page — `ProdutosPage.vue`

- Paginação no servidor igual a Categorias, com `fetchProdutos`.
- No `onMounted`: `reload()` **e** `fetchTodasCategorias()` (de
  `useCategorias`, para o select do dialog).
- Colunas: `name`, `categoryName`, `price` (com `formatBRL`), `quantity`
  (ex.: badge "Esgotado" em 0 e alerta abaixo de `ESTOQUE_BAIXO`),
  `is_active` (toggle → `toggleActive`), ações.
- `onSave`: `editing ? update(editing, form) : add(form)` e **sempre
  `reload()`** depois — criar/editar mexe em 2–3 entidades, recarregar a
  página é o mais simples.
- **Busca e filtro por categoria saem**: a API não filtra, e numa tabela
  paginada eles só buscariam na página atual.

## Page — `EstoquePage.vue`

- Mesma paginação, com `fetchProdutos`.
- No `onMounted`: `reload()` **e** `fetchIndicadores()`.
- Cards de indicadores (KPIs) com `totalUnidades`, `lowStockCount`,
  `outOfStockCount` — calculados sobre **todos** os estoques.
- Coluna "Ajustar": botões − / + chamando `decStock(row)` / `incStock(row)`;
  o − desabilitado quando `row.quantity === 0`.
- Coluna "Reposição": botão "Repor +20" (`restock(row)`) quando
  `row.quantity < ESTOQUE_BAIXO`.

## Dashboard — indicadores de produto

```ts
const { activeCount, lowStockCount, fetchProdutos, fetchIndicadores } =
  useProdutos()

// Até 100 produtos para os indicadores
onMounted(() => {
  fetchProdutos({ page: 1, size: 100 })
  fetchIndicadores()
})
```

Os indicadores de pedidos e faturamento continuam no mock até a aula de
Pedidos.

## Cardápio do app (cliente) — `CatalogoPage.vue`

Sem `q-table` e sem paginação: cards e chips.

```ts
const { produtos, fetchProdutos } = useProdutos()
const { todasCategorias, fetchTodasCategorias } = useCategorias()

const categoriaSelecionada = ref('Todos')

onMounted(() => {
  fetchTodasCategorias()
  // Cardápio sem paginação: até 100 produtos numa requisição. O GET público
  // já esconde produtos e categorias inativos.
  fetchProdutos({ page: 1, size: 100 })
})

const categoriasAtivas = computed(() =>
  todasCategorias.value.filter(c => c.is_active)
)

const produtosFiltrados = computed(() =>
  produtos.value.filter(
    p =>
      p.is_active &&
      p.price !== null && // sem item/preço ainda não dá para vender
      (categoriaSelecionada.value === 'Todos' ||
        p.categoryName === categoriaSelecionada.value)
  )
)
```

O carrinho continua local (store do carrinho) até a aula de carrinho e
checkout: guarde no carrinho o `id` do produto (uuid).

## Testar

- [ ] Listagem de produtos = **1 GET** `/products/`, com nome da categoria,
      preço e estoque
- [ ] Criar "Suco Natural" (Bebidas, R$ 9,90) → **1 POST** 201 + reload;
      aparece como esgotado (estoque 0)
- [ ] Criar de novo com o mesmo nome → `400` (SKU repetido) → Notify e o
      dialog continua aberto; o total não muda (atômico)
- [ ] Editar o preço → PATCH no produto **e** PATCH em `/product-items/{id}/`
- [ ] Desativar → some do cardápio do app
- [ ] Estoque: 3 cliques rápidos no + → **1** PATCH em
      `/inventory/{inventory_id}/`; F5 mantém o valor; indicadores atualizam
- [ ] "Repor +20" aparece abaixo de 15 e soma 20
- [ ] Dashboard mostra produtos ativos e estoque baixo da API
- [ ] Cardápio do app mostra os produtos com preço e filtra pelos chips

## Pontos de aula

- **A API evoluiu e as pages não mudaram.** Na primeira versão da API,
  listar eram 4 GETs cruzados no front e criar eram 3 POSTs (com risco de
  produto órfão). Na atual, 1 GET e 1 POST atômico. Ao trocar os tipos, o
  typecheck apontou erros só no composable e nos services: as pages usam
  `ProdutoCompleto`, que não mudou.
- **Mudança de comportamento que o typecheck não pega:** quando os
  indicadores saíram do `fetchProdutos` para o `fetchIndicadores`, Estoque e
  Dashboard precisaram passar a chamá-lo no `onMounted`. Sem isso, os KPIs
  ficam zerados e nada acusa erro.
- **Debounce por id (`Map`)**: um `debounce` único perderia o clique do
  primeiro produto quando o usuário clica em dois produtos seguidos.
