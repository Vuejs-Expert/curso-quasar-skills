# Categorias — a tela-modelo do CRUD

Primeiro recurso a integrar: listar paginado, criar, editar e
ativar/desativar. Todos os outros copiam este padrão.

## API

```
GET   /categories/?page=1&size=20   lista paginada
POST  /categories/                  { name, slug, is_active? }
PATCH /categories/{id}/             { name?, slug?, is_active? }
(sem DELETE: "excluir" = is_active false)
```

- `slug` é **obrigatório e único**: geramos a partir do nome (`slugify`).
  Slug repetido → `400` → Notify, e o dialog continua aberto.
- O GET público (cliente) já esconde categorias inativas.

## Tipo — `src/types/categoria.ts`

```ts
/** Formato exato de `Category` na API (GET /categories/). */
export interface Categoria {
  id: string
  name: string
  slug: string
  is_active: boolean
  created_at: string
  updated_at: string
}

/** Campos que a API aceita em POST/PATCH /categories/ */
export interface CategoriaInput {
  name: string
  slug: string
  is_active?: boolean
}
```

Trocar o tipo do mock (ex.: `active`) por este e rodar o typecheck: cada erro
é um lugar a ajustar.

## Util — `src/utils/slugify.ts`

```ts
/**
 * "Hambúrgueres Artesanais" → "hamburgueres-artesanais"
 * A API exige um `slug` único por categoria; geramos a partir do nome.
 */
export function slugify(texto: string) {
  return texto
    .normalize('NFD') // separa letra e acento: "ú" → "u" + "´"
    .replace(/[̀-ͯ]/g, '') // remove os acentos
    .toLowerCase()
    .trim()
    .replace(/[^a-z0-9]+/g, '-') // tudo que não é letra/número vira "-"
    .replace(/^-+|-+$/g, '') // tira "-" das pontas
}
```

Também é usado em Produtos (SKU).

## Service — `src/services/categoriasService.ts`

```ts
import { api } from './api'
import type { Paginated, PageParams } from '@/types/api'
import type { Categoria, CategoriaInput } from '@/types/categoria'

/**
 * Service = só HTTP. Não guarda estado, não sabe de tela.
 * Não existe `remove`: a API não tem DELETE para categorias.
 */
export const categoriasService = {
  /** Uma página só — para tabelas com paginação no servidor (q-table) */
  async listPage(params: PageParams) {
    const { data } = await api.get<Paginated<Categoria>>('/categories/', {
      params
    })
    return data
  },

  /**
   * Lista completa numa requisição só — para listas pequenas de apoio
   * (select de categoria, chips do cardápio). 100 é o máximo da API.
   */
  async listAll() {
    const params: PageParams = { page: 1, size: 100 }
    const { data } = await api.get<Paginated<Categoria>>('/categories/', {
      params
    })
    return data.results
  },

  async create(input: CategoriaInput) {
    const { data } = await api.post<Categoria>('/categories/', input)
    return data
  },

  async update(id: string, patch: Partial<CategoriaInput>) {
    const { data } = await api.patch<Categoria>(`/categories/${id}/`, patch)
    return data
  }
}
```

## Composable — `src/composables/useCategorias.ts`

Igual ao modelo genérico (`modelos.md`), com **duas listas**:

- `categorias` + `totalCategorias`: a página atual da tabela do admin
  (`fetchCategorias(params)`).
- `todasCategorias`: lista completa para selects e chips
  (`fetchTodasCategorias()` → `categoriasService.listAll()`). Usada depois
  pelo formulário de Produtos e pelo cardápio do app.

Exponha: `categorias, totalCategorias, todasCategorias, loading, error,
fetchCategorias, fetchTodasCategorias, add, update, toggleActive`.

Se o composable do mock começava com dados, ele passa a começar **vazio**: a
page chama o fetch no `onMounted`.

## Dialog — `src/components/backoffice/CategoriaFormDialog.vue`

```vue
<template>
  <q-dialog v-model="modelValue" :persistent="saving">
    <q-card style="width: 380px">
      <q-card-section class="text-subtitle1">
        {{ categoria ? 'Editar Categoria' : 'Nova Categoria' }}
      </q-card-section>
      <q-separator />
      <q-form @submit="save">
        <q-card-section>
          <q-input
            v-model="name"
            label="Nome da categoria"
            outlined
            dense
            autofocus
            :hint="`Slug: ${slug || '—'}`"
            :rules="[v => !!v.trim() || 'Informe o nome']"
          />
        </q-card-section>
        <q-separator />
        <q-card-actions align="right">
          <q-btn
            v-close-popup
            flat
            color="primary"
            label="Cancelar"
            :disable="saving"
          />
          <!-- Sem v-close-popup: quem fecha é a page, só se a API aceitar -->
          <q-btn
            type="submit"
            unelevated
            color="primary"
            label="Salvar"
            :loading="saving"
          />
        </q-card-actions>
      </q-form>
    </q-card>
  </q-dialog>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { slugify } from '@/utils/slugify'
import type { Categoria, CategoriaInput } from '@/types/categoria'

const modelValue = defineModel<boolean>({ required: true })
const props = defineProps<{
  categoria: Categoria | null
  saving: boolean
}>()
const emit = defineEmits<{
  save: [payload: CategoriaInput]
}>()

const name = ref('')
const slug = computed(() => slugify(name.value))

watch(modelValue, open => {
  if (open) name.value = props.categoria?.name ?? ''
})

function save() {
  emit('save', { name: name.value.trim(), slug: slug.value })
}
</script>
```

## Page — `src/pages/backoffice/CategoriasPage.vue`

Exatamente o modelo de page de `modelos.md`: `q-table` paginado no servidor,
banner de erro, `q-toggle` de ativo na coluna `is_active` (chama
`toggleActive`), botão de editar. Depois de **criar**, `reload()` (o total e a
página mudam); depois de **editar**, não precisa (o composable troca o item).

Remova colunas que dependiam do mock e que a API não fornece (ex.: "Produtos
vinculados" — a API não manda contagem).

## Testar

- [ ] Lista vem da API e pagina (`?page=2&size=20` no Network)
- [ ] Trocar "linhas por página" recarrega com o novo `size`
- [ ] Criar → aparece na tabela; o hint mostra o slug gerado
- [ ] Criar com nome repetido → Notify vermelho com a mensagem da API e o
      dialog continua aberto
- [ ] Editar o nome → PATCH 200 e a linha muda sem recarregar
- [ ] Toggle ativo → PATCH com `is_active`

## Pontos de aula

- Os slots do `q-table` tipam `props.row` como `any`: `props.row.active`
  (nome antigo) passa no typecheck e quebra a coluna. Só o navegador mostra.
- Uma coluna que comparava `produto.category === categoria.name` continua
  compilando quando `category` vira uuid (os dois são `string`) — e dá
  sempre 0. Tipos iguais não garantem significado igual.
