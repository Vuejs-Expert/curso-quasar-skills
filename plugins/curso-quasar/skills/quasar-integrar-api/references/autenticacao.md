# Autenticação (JWT) — feita uma vez, antes de qualquer CRUD

Pré-requisito: a base de [infraestrutura.md](infraestrutura.md) pronta.
Adicione `'LocalStorage'` aos `plugins` do `quasar.config.ts` e
`boot: ['axios']`.

## Como a API funciona

| Endpoint                  | Faz o quê                                                     |
| ------------------------- | ------------------------------------------------------------- |
| `POST /auth/login/`       | `{ email, password }` → `{ access, refresh }`                 |
| `POST /auth/refresh/`     | `{ refresh }` → `{ access }` (access dura 5 min, refresh 24h) |
| `GET /auth/me/`           | o usuário dono do token, com o `role` (`admin`/`customer`)    |
| `POST /users/`            | cadastro público: sempre cria `customer`                      |
| `GET/POST /customers/me/` | perfil do cliente logado (404 = ainda não tem perfil)         |

Admin e cliente usam o **mesmo** login; o que muda é o `role`. Por isso há
**uma store de sessão só** (substitui as stores de auth falsas de cada área)
e o guard do router confere o `role` de cada área.

## Ordem de implementação (cada passo testável no navegador)

1. Tipos: `src/types/auth.ts`, `Usuario`/`Role` em `src/types/usuario.ts`,
   `Cliente`/`ClienteInput` em `src/types/cliente.ts`.
2. `src/services/authService.ts` e `getMe`/`createMe` em
   `src/services/clientesService.ts`.
3. `src/stores/auth.ts` (sessão única) + tela de login do admin. Neste ponto
   o login funciona, mas as outras chamadas ainda dão 401.
4. `src/services/tokenStorage.ts` + **só o interceptor de request** em
   `src/boot/axios.ts` → as chamadas passam a levar o token.
5. Guard do router com `restoreSession` → rotas protegidas e sessão mantida
   depois do F5.
6. Botão "Sair" no layout (`logout()` + `router.push` para o login).
7. Login e "Criar conta" do app do cliente.
8. Interceptor de response (refresh automático em 401).
9. Remover as stores de auth antigas e corrigir o que o typecheck apontar.
   Dados que só existiam na store falsa (ex.: endereços) e que a API não
   tem viram um composable local.

## Tipos

```ts
// src/types/auth.ts
import type { ClienteInput } from './cliente'

export interface Credenciais {
  email: string
  password: string
}

/** Resposta de POST /auth/login/ */
export interface TokenPair {
  access: string
  refresh: string
}

/** Tudo que a tela "Criar conta" do app coleta: login + perfil do cliente */
export type Cadastro = Credenciais & ClienteInput
```

```ts
// src/types/usuario.ts
export type Role = 'admin' | 'customer'

/** Formato exato de `User` na API (GET /users/, GET /auth/me/). */
export interface Usuario {
  id: number
  email: string
  role: Role
  is_active: boolean
  created_at: string
  updated_at: string
}
```

```ts
// src/types/cliente.ts
/** Formato exato de `Customer` na API. A API não devolve e-mail aqui. */
export interface Cliente {
  id: string
  user: number
  name: string
  phone: string
  document: string
  created_at: string
  updated_at: string
}

/** Campos de POST /customers/me/ e PATCH /customers/{id}/ */
export interface ClienteInput {
  name: string
  phone: string
  document: string
}
```

## Services

```ts
// src/services/authService.ts
import { api } from './api'
import type { Credenciais, TokenPair } from '@/types/auth'
import type { Usuario } from '@/types/usuario'

export const authService = {
  async login(credenciais: Credenciais) {
    const { data } = await api.post<TokenPair>('/auth/login/', credenciais)
    return data
  },

  /** Troca o refresh token por um access token novo (o access dura 5 min). */
  async refresh(refresh: string) {
    const { data } = await api.post<{ access: string }>('/auth/refresh/', {
      refresh
    })
    return data.access
  },

  /** Quem é o dono do token atual — é aqui que descobrimos o `role`. */
  async me() {
    const { data } = await api.get<Usuario>('/auth/me/')
    return data
  },

  /** Cadastro público: sempre cria um usuário com role `customer`. */
  async register(credenciais: Credenciais) {
    const { data } = await api.post<Usuario>('/users/', credenciais)
    return data
  }
}
```

```ts
// src/services/clientesService.ts (parte do app; listPage/update vêm depois)
import { isAxiosError } from 'axios'
import { api } from './api'
import type { Cliente, ClienteInput } from '@/types/cliente'

export const clientesService = {
  /**
   * Perfil do cliente logado. 404 = o usuário existe mas ainda não criou o
   * perfil — não é erro "de verdade", então devolvemos `null`.
   */
  async getMe() {
    try {
      const { data } = await api.get<Cliente>('/customers/me/')
      return data
    } catch (error) {
      if (isAxiosError(error) && error.response?.status === 404) return null
      throw error
    }
  },

  async createMe(perfil: ClienteInput) {
    const { data } = await api.post<Cliente>('/customers/me/', perfil)
    return data
  }
}
```

```ts
// src/services/tokenStorage.ts
import { LocalStorage } from 'quasar'
import type { TokenPair } from '@/types/auth'

/**
 * Tokens entre um F5 e outro. localStorage é o mais simples; o trade-off é
 * que qualquer script da página consegue lê-lo (risco em XSS). O
 * `LocalStorage` do Quasar não quebra quando o storage está bloqueado.
 * Uma sessão por navegador: logar numa área substitui a sessão da outra.
 */
const ACCESS_KEY = 'auth.access'
const REFRESH_KEY = 'auth.refresh'

export const tokenStorage = {
  getAccess: () => LocalStorage.getItem<string>(ACCESS_KEY),
  getRefresh: () => LocalStorage.getItem<string>(REFRESH_KEY),

  save(tokens: TokenPair) {
    LocalStorage.setItem(ACCESS_KEY, tokens.access)
    LocalStorage.setItem(REFRESH_KEY, tokens.refresh)
  },
  saveAccess(access: string) {
    LocalStorage.setItem(ACCESS_KEY, access)
  },
  clear() {
    LocalStorage.removeItem(ACCESS_KEY)
    LocalStorage.removeItem(REFRESH_KEY)
  }
}
```

## Store de sessão — `src/stores/auth.ts`

Pinia aqui é justificado: a sessão é lida pelo guard do router, fora de
qualquer componente.

```ts
import { ref, computed } from 'vue'
import { defineStore, acceptHMRUpdate } from 'pinia'
import { authService } from '@/services/authService'
import { clientesService } from '@/services/clientesService'
import { tokenStorage } from '@/services/tokenStorage'
import type { Cadastro, Credenciais } from '@/types/auth'
import type { Cliente } from '@/types/cliente'
import type { Role, Usuario } from '@/types/usuario'

/** Sessão do usuário logado — uma só para as duas áreas (admin e app). */
export const useAuthStore = defineStore('auth', () => {
  const user = ref<Usuario | null>(null)
  // Só existe para role `customer` (GET /customers/me/)
  const profile = ref<Cliente | null>(null)

  const isAuthenticated = computed(() => user.value !== null)

  async function loadUser() {
    user.value = await authService.me()
    profile.value =
      user.value.role === 'customer' ? await clientesService.getMe() : null
  }

  /** Login em 3 passos: tokens → quem é (`/auth/me/`) → confere o `role`. */
  async function login(credenciais: Credenciais, expectedRole: Role) {
    logout()
    tokenStorage.save(await authService.login(credenciais))
    await loadUser()

    if (user.value?.role !== expectedRole) {
      logout()
      throw new Error(
        expectedRole === 'admin'
          ? 'Esta conta não tem acesso ao painel administrativo.'
          : 'Esta conta não é de cliente.'
      )
    }
  }

  /** "Criar conta" do app: cria o usuário, loga, cria o perfil. */
  async function register(cadastro: Cadastro) {
    const { name, phone, document, ...credenciais } = cadastro
    logout()
    await authService.register(credenciais)
    await login(credenciais, 'customer')
    profile.value = await clientesService.createMe({ name, phone, document })
  }

  /** Depois de um F5: se ainda há token, pergunta à API quem é o usuário. */
  async function restoreSession() {
    if (user.value || !tokenStorage.getAccess()) return
    try {
      await loadUser()
    } catch {
      logout() // token inválido e refresh falhou: sessão perdida
    }
  }

  function logout() {
    tokenStorage.clear()
    user.value = null
    profile.value = null
  }

  return {
    user,
    profile,
    isAuthenticated,
    login,
    register,
    restoreSession,
    logout
  }
})

if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useAuthStore, import.meta.hot))
}
```

## Tela de login (trecho do `<script setup>`)

```ts
const email = ref('')
const password = ref('')
const loading = ref(false)
const errorMessage = ref<string | null>(null)

const router = useRouter()
const authStore = useAuthStore()

async function onSubmit() {
  loading.value = true
  errorMessage.value = null
  try {
    await authStore.login(
      { email: email.value, password: password.value },
      'admin' // no app do cliente: 'customer'
    )
    await router.push({ name: 'admin-dashboard' })
  } catch (err) {
    errorMessage.value = getErrorMessage(err)
  } finally {
    loading.value = false
  }
}
```

## Interceptors — `src/boot/axios.ts`

Ensine em dois momentos: primeiro só o bloco 1 (request); o bloco 2
(refresh) quando o aluno já viu o 401 depois de 5 minutos.

```ts
import { defineBoot } from '#q-app'
import { isAxiosError, type InternalAxiosRequestConfig } from 'axios'
import { api } from '@/services/api'
import { authService } from '@/services/authService'
import { tokenStorage } from '@/services/tokenStorage'
import { useAuthStore } from '@/stores/auth'

// Rotas que NUNCA levam token: a API responde 401 até no login se receber
// um token inválido/expirado no header.
const ROTAS_SEM_TOKEN = ['/auth/login/', '/auth/refresh/']

// Várias requisições com 401 ao mesmo tempo esperam o MESMO refresh
let refreshEmAndamento: Promise<string> | null = null

type ConfigComRetry = InternalAxiosRequestConfig & { _retry?: boolean }

export default defineBoot(({ router }) => {
  // 1) Antes de cada requisição: coloca o token no header
  api.interceptors.request.use(config => {
    const access = tokenStorage.getAccess()
    if (access && !ROTAS_SEM_TOKEN.includes(config.url ?? '')) {
      config.headers.Authorization = `Bearer ${access}`
    }
    return config
  })

  // 2) Resposta 401 (access expirado): renova UMA vez e refaz a requisição
  api.interceptors.response.use(
    response => response,
    async (error: unknown) => {
      if (!isAxiosError(error) || !error.config) throw error

      const original: ConfigComRetry = error.config
      const refresh = tokenStorage.getRefresh()
      const podeRenovar =
        error.response?.status === 401 &&
        refresh !== null &&
        !original._retry &&
        !ROTAS_SEM_TOKEN.includes(original.url ?? '')

      if (!podeRenovar) throw error

      original._retry = true
      try {
        refreshEmAndamento ??= authService.refresh(refresh).finally(() => {
          refreshEmAndamento = null
        })
        tokenStorage.saveAccess(await refreshEmAndamento)
      } catch {
        // Refresh também expirou: encerra a sessão e vai para o login da área
        useAuthStore().logout()
        const area = router.currentRoute.value.meta.requiresAuth
        if (area) {
          await router.push({
            name: area === 'admin' ? 'admin-login' : 'app-login'
          })
        }
        throw error
      }

      // Refaz FORA do try/catch: se a requisição falhar de novo (ex.: 400),
      // o erro é dela — vai para quem chamou e o usuário NÃO é deslogado.
      return api(original)
    }
  )
})
```

## Guard do router — `src/router/index.ts`

Uma regra por `if`, de propósito (mais fácil de ler do que uma tabela):

```ts
Router.beforeEach(async to => {
  const authStore = useAuthStore()

  // 1. Depois de um F5: se ainda há token salvo, descobre quem é o usuário
  await authStore.restoreSession()
  const role = authStore.user?.role

  // 2. Área protegida sem o role certo → login da área
  if (to.meta.requiresAuth === 'admin' && role !== 'admin') {
    return { name: 'admin-login' }
  }
  if (to.meta.requiresAuth === 'app' && role !== 'customer') {
    return { name: 'app-login' }
  }

  // 3. Já logado tentando abrir o login → entra direto
  if (to.name === 'admin-login' && role === 'admin') {
    return { name: 'admin-dashboard' }
  }
  if (to.name === 'app-login' && role === 'customer') {
    return { name: 'app-catalogo' }
  }
})
```

Ajuste os nomes de rota (`admin-login`, `admin-dashboard`, `app-login`,
`app-catalogo`) aos que existem no `routes.ts` do projeto; se forem
diferentes, use os do projeto e avise.

## Checklist de testes

- [ ] Login admin com conta admin → dashboard
- [ ] Login admin com conta de cliente → mensagem "não tem acesso"
- [ ] Senha errada → mensagem da API na tela
- [ ] F5 numa rota protegida → continua logado
- [ ] Abrir rota protegida deslogado → vai para o login da área
- [ ] Sair → volta ao login e o F5 não restaura a sessão
- [ ] DevTools → Network: chamadas levam `Authorization: Bearer ...`
- [ ] Criar conta no app → já entra logado como cliente
- [ ] Depois de 5 min parado, uma ação ainda funciona (refresh automático)
