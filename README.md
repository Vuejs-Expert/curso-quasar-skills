# curso-quasar-skills

Plugin do Claude Code usado no curso **Vue.js Expert** para integrar o front
Quasar 2 + Vue 3 + TypeScript com a API do curso.

## O que vem no plugin

| Skill                 | Comando                             | Para que serve                                                                                                                                 |
| --------------------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `quasar-integrar-api` | `/curso-quasar:quasar-integrar-api` | Base do axios, login JWT e os recursos do curso (Categorias, Usuários, Clientes, Produtos e Estoque, Dashboard, Cardápio), no padrão das aulas |

## Instalar

No terminal, na **raiz do seu projeto** (fora do Claude Code). O escopo
`project` deixa o plugin valendo só nesse projeto, registrado no
`.claude/settings.json` dele:

```bash
claude plugin marketplace add Vuejs-Expert/curso-quasar-skills --scope project
claude plugin install curso-quasar@curso-quasar-skills --scope project
```

Depois, abra o Claude Code (ou rode `/reload-plugins`) e digite
`/curso-quasar:` para ver a skill.

Prefere que valha em todos os seus projetos? Use `--scope user` nos dois
comandos.

> **Erro de SSH ao adicionar o marketplace?** Se você não tem chave SSH do
> GitHub configurada, rode antes `export CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1`
> (ou adicione essa linha ao seu `~/.zshrc`) e tente de novo.

## Atualizar

Quando a skill mudar no curso:

```
/plugin marketplace update curso-quasar-skills
```

Ou ative a atualização automática em `/plugin` → **Marketplaces** →
`curso-quasar-skills` → **Enable auto-update**.

## MCP do Quasar (recomendado)

A skill consulta o MCP oficial do Quasar para não inventar props de
componentes. Na raiz do seu projeto:

```bash
claude mcp add quasar -- npx -y --fetch-retries=0 @quasar/mcp@latest
```

Confira com `/mcp` dentro do Claude Code.

## Estrutura

```
.claude-plugin/marketplace.json        catálogo (este repositório)
plugins/curso-quasar/
├─ .claude-plugin/plugin.json          manifesto do plugin
└─ skills/quasar-integrar-api/
   ├─ SKILL.md
   └─ references/
      ├─ infraestrutura.md
      ├─ autenticacao.md
      ├─ modelos.md
      └─ recursos/        contrato da API + código de cada recurso do curso
         ├─ categorias.md
         ├─ usuarios.md
         ├─ clientes.md
         └─ produtos-estoque.md
```

O plugin não declara `version`: cada commit no `main` já é uma versão nova
para quem atualizar.
