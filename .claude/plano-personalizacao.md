# Plano de personalização MTCI sobre o Open edX

Decidido em 31/10/2026. Toda personalização segue esta estrutura. Mudanças no plano
são registradas aqui antes de serem implementadas.

## Princípio

Dois repositórios, com papéis que não se misturam:

| Repo | Papel |
|---|---|
| `mtci-openedx` (este fork, branch `ulmo`) | Upstream intocado. Só recebe merges de `upstream/release/ulmo`, principalmente correções de segurança. |
| `mtci-custom` (a criar, em `~/Documents/code/mtci-custom`) | Tudo que é nosso: sites, design, plugins, XBlocks, MFE slots, tema, infra Tutor. |

O core do `edx-platform` não é editado. Se não houver filter, event ou plugin slot
para o caso, a saída é abrir PR no upstream, não alterar o fork.

## Estrutura do `mtci-custom`

```
mtci-custom/
├── CLAUDE.md
├── .claude/
│   ├── agents/                   plugin-dev, design-system, tutor-ops, test-runner
│   └── skills/                   novo-site, nova-marca, sync-upstream
├── sites/                        config por site como código (fonte da verdade)
│   ├── _padrao.yml               valores herdados por todos os sites
│   └── <site>.yml
├── design-system/
│   ├── tokens/base/              padrão visual: cores semânticas, tipografia, espaçamento
│   ├── tokens/sites/<site>/      só o que muda por site: primária, fonte
│   ├── assets/<site>/            logo, favicon
│   └── build/                    CSS de tokens por site (gerado, formato Paragon)
├── tutor-plugins/mtci/           substitui o multisite.py; lê sites/*.yml
│   ├── plugin.py                 Caddy, ALLOWED_HOSTS, CORS, CSRF, OAuth, MFE
│   └── patches/
├── backend/mtci_core/            Django plugin (entry points lms.djangoapp / cms.djangoapp)
│   ├── filters/                  pipeline steps de Open edX Filters
│   ├── events/                   handlers de Open edX Events
│   ├── api/                      endpoints e models próprios
│   └── management/commands/sync_sites.py   sites/*.yml → Site + SiteConfiguration
├── xblocks/                      um pacote por XBlock
├── frontend/slots/               componentes React para plugin slots dos MFEs
└── theme/mtci/                   tema legado do LMS, filho do indigo, lê variáveis CSS
```

## Formato de um site (`sites/<site>.yml`)

```yaml
dominio: educacao.local.openedx.io
nome: "Educação"
orgs: [EDUC]                    # course_org_filter
marca: educacao                 # → design-system/tokens/sites/educacao
tema_legado: mtci               # → theme/mtci
mfe:                            # vai para MFE_CONFIG na SiteConfiguration
  LOGO_URL: /static/mtci/educacao/logo.svg
  SITE_NAME: "Educação"
features:                       # flags consultadas pelo mtci_core por site
  gamificacao: true
```

Um `.yml` alimenta três camadas:

```
sites/<site>.yml
   ├─→ tutor-plugins/mtci      domínio no Caddy, hosts, CORS, CSRF (http e https), OAuth   (tutor config save)
   ├─→ sync_sites              Site + SiteConfiguration + MFE_CONFIG                        (./manage.py lms sync_sites)
   └─→ design-system build     CSS de tokens do site                                        (npm run build)
```

`SiteConfiguration` não é editada à mão no admin: é sempre gerada a partir dos `.yml`.

## Design: padrão e marca por site

- `tokens/base` define o padrão de todos os sites; `tokens/sites/<site>` só sobrescreve marca.
- Nenhuma cor, fonte ou espaçamento fixo no código: só tokens / variáveis CSS.
- MFEs: o tema chega por `MFE_CONFIG` da `SiteConfiguration` (lido em
  `lms/djangoapps/mfe_config_api/views.py` via `configuration_helpers.get_value`),
  apontando para o CSS de tokens do site (`PARAGON_THEME_URLS`), mais `LOGO_URL` e `FAVICON_URL`.
  **A verificar:** se os MFEs do Ulmo leem os temas em tempo de execução.
- Páginas legadas: `theme/mtci` (base indigo) lendo as variáveis CSS. `SiteTheme` só
  se um site precisar de templates diferentes, não apenas de cores.
- E-mails e certificados: valores por site na `SiteConfiguration`.

## Onde cada coisa vai

| Quero... | Vai em |
|---|---|
| Novo domínio ou site | `sites/<site>.yml` |
| Cor, fonte ou logo de um site | `design-system/tokens/sites/<site>` e `assets/<site>` |
| Mudar o padrão visual de todos | `design-system/tokens/base` |
| Mudar uma regra (matrícula, certificado, cadastro...) | `backend/mtci_core/filters` |
| Reagir a algo (nota, matrícula...) | `backend/mtci_core/events` |
| Nova API ou model | `backend/mtci_core/api` |
| Novo tipo de atividade no curso | `xblocks/` |
| Trocar header, footer ou widget de MFE | `frontend/slots` |
| Página antiga do LMS | `theme/mtci` |
| Config de infra | `tutor-plugins/mtci` |
| Editar `edx-platform` | Nunca. PR no upstream. |

## Desenvolvimento local

```bash
tutor plugins enable mtci                         # no lugar de multisite
tutor mounts add ~/Documents/code/mtci-custom/backend/mtci_core
tutor dev launch
tutor dev run lms ./manage.py lms sync_sites
tutor dev run lms pytest <caminho>                # testes sempre dentro do container
```

## Agentes e skills previstos (em `mtci-custom/.claude/`)

| Nome | Tipo | Função |
|---|---|---|
| `openedx-explorer` | agente, só leitura | Achar pontos de extensão (filters, events, slots, waffle flags) no monólito. |
| `plugin-dev` | agente | Implementar em `backend/`, `xblocks/`, `frontend/slots` a partir dos cookiecutters da openedx; nunca no core. |
| `design-system` | agente | Guardar o padrão: tokens, nada de valor fixo, revisar telas contra os tokens. |
| `tutor-ops` | agente | Plugin Tutor, mounts, builds, logs. Sem ler segredos. |
| `test-runner` | agente, Haiku | Rodar pytest/jest no container e resumir falhas. |
| `security-reviewer` | agente, só leitura | Escaping Mako, authz de views/handlers, course keys. |
| `novo-site` | skill | Criar `sites/<site>.yml`, `tutor config save`, `sync_sites`. |
| `nova-marca` | skill | Gerar `tokens/sites/<site>` e assets a partir de cor, logo e fonte. |
| `sync-upstream` | skill | Merge de `upstream/release/ulmo` no fork e testes. |

## Commits e PRs

- Sem atribuição ao Claude: nada de `Co-Authored-By: Claude`, nada de
  "Generated with Claude Code" e nenhuma menção ao Claude na mensagem ou na descrição.
  Atrapalha os filtros por autor e não acrescenta informação.
- Garantido por `attribution` no `.claude/settings.json` de cada repo (este e o `mtci-custom`).
- Mensagens no formato do upstream: `tipo: descrição` (`feat`, `fix`, `docs`, `build`...).

## Segurança

- Nunca ler `~/Library/Application Support/tutor/config.yml` nem `env/**` diretamente
  (contêm chaves e senhas). Para consultar valores: `tutor config printvalue CHAVE`.

## Estado atual (31/10/2026)

- Multisite feito pelo plugin avulso `~/Library/Application Support/tutor-plugins/multisite.py`,
  com um site (`educacao.local.openedx.io`). Só cobre LMS e Caddy; faltam Studio, MFE,
  OAuth e `https` no CSRF. Será migrado para `tutor-plugins/mtci`.
- Plugins Tutor ativos: `indigo`, `mfe`, `multisite`. Tutor 21 (Ulmo).
- `mtci-custom` ainda não existe. Primeiro passo: esqueleto do repo, migração do
  `multisite.py`, comando `sync_sites`, site `educacao` como exemplo.
