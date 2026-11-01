# Infraestrutura base

Crie só o que estiver faltando. Tudo aqui é feito **uma vez** por projeto.

## 1. Instalar o axios

```bash
pnpm add axios
```

## 2. Proxy do dev server — `quasar.config.ts`

A API não envia headers de CORS: em dev, o próprio `quasar dev` repassa as
chamadas, e para o navegador tudo é "same-origin". Dentro de `devServer`:

```ts
proxy: {
  '/api': {
    target: 'https://api-curso-03sk.onrender.com',
    changeOrigin: true,
    // A API autentica por JWT (header), não por cookie. Sem isto, cookies
    // grandes de localhost vão junto e a API responde 431.
    headers: { cookie: '' }
  }
}
```

E em `framework`:

```ts
lang: 'pt-BR', // textos dos componentes (ex.: rodapé do q-table)
plugins: ['Notify']
```

## 3. Instância do axios — `src/services/api.ts`

```ts
import axios from 'axios'

/**
 * Instância única do axios usada por todos os services.
 *
 * - `baseURL`: `QCLI_API_URL` (fixada no BUILD) ou `/api/v1`, que em dev
 *   passa pelo proxy do quasar.config.ts.
 * - `timeout` generoso: no plano free o Render "dorme" e a primeira
 *   requisição pode levar ~50s.
 */
export const api = axios.create({
  baseURL: import.meta.env.QCLI_API_URL || '/api/v1',
  timeout: 60_000
})
```

Tipar a variável em `env.d.ts` (raiz do projeto):

```ts
interface ImportMetaEnv {
  /** URL base da API. Opcional: sem ela, usa `/api/v1` (proxy em dev). */
  readonly QCLI_API_URL?: string
}
```

Só variáveis com prefixo `QCLI_` chegam ao navegador — e ficam visíveis no
JavaScript do site. Nunca coloque segredos nelas.

## 4. Paginação — `src/types/api.ts`

```ts
/** Envelope que toda listagem da API devolve. */
export interface Paginated<T> {
  page: number
  size: number
  total: number
  totalPages: number
  results: T[]
}

/** `?page=2&size=20` — size padrão 20, máximo 100. Página inexistente = 404. */
export interface PageParams {
  page: number
  size: number
}
```

## 5. Mensagens de erro — `src/services/helpers.ts`

```ts
import { isAxiosError } from 'axios'

/**
 * Procura a primeira mensagem de erro, mesmo quando ela vem aninhada:
 * - `{ "email": ["Enter a valid email."] }`              → "email: Enter a valid email."
 * - `{ "items": { "0": { "sku": ["...already exists."] } } }` → "sku: ...already exists."
 */
function primeiraMensagem(data: object, campo = ''): string | null {
  for (const [chave, valor] of Object.entries(data)) {
    // índices de lista ("0") e "non_field_errors" não ajudam como nome de campo
    const nome =
      /^\d+$/.test(chave) || chave === 'non_field_errors' ? campo : chave
    if (typeof valor === 'string') return nome ? `${nome}: ${valor}` : valor
    if (valor && typeof valor === 'object') {
      const mensagem = primeiraMensagem(valor, nome)
      if (mensagem) return mensagem
    }
  }
  return null
}

/**
 * Converte qualquer erro (de rede, do axios ou da API) numa mensagem
 * legível. A API (Django REST Framework) responde erros como
 * `{ "detail": "..." }`, `{ "campo": ["..."] }` ou aninhados.
 */
export function getErrorMessage(error: unknown): string {
  if (isAxiosError(error)) {
    if (!error.response) {
      return error.code === 'ECONNABORTED'
        ? 'O servidor demorou para responder (ele pode estar acordando). Tente novamente.'
        : 'Não foi possível conectar ao servidor.'
    }

    const data: unknown = error.response.data
    if (data && typeof data === 'object') {
      if ('detail' in data && typeof data.detail === 'string') {
        return data.detail
      }
      const mensagem = primeiraMensagem(data)
      if (mensagem) return mensagem
    }
    return `Erro ${error.response.status} na requisição.`
  }

  return error instanceof Error ? error.message : 'Erro inesperado.'
}
```

## Conferência

Reinicie o `quasar dev` (mudanças no `quasar.config.ts` exigem restart) e
confira no DevTools → Network que uma chamada a `/api/v1/...` responde pelo
proxy (e não com erro de CORS).
