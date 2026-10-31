# MTCI Open edX

Este repo é o fork `mtci-openedx` do `edx-platform` (branch `main`, hoje na release Ulmo), rodando com Tutor 21.

## Regras

- **Não editar o código deste repo.** A `main` só recebe merges de `upstream/release/ulmo`
  (na troca de release, da release seguinte).
  Toda personalização vai para o repo `mtci-custom`, conforme o plano abaixo.
- Antes de propor qualquer personalização, consultar a tabela "Onde cada coisa vai" do plano.
- Aqui, o uso é de leitura: encontrar pontos de extensão (Open edX Filters, Events,
  plugin slots, waffle flags) para implementar em `mtci-custom`.
- Nunca ler `~/Library/Application Support/tutor/config.yml` nem `env/**` (segredos).
  Usar `tutor config printvalue CHAVE`.
- Testes rodam dentro do container: `tutor dev run lms pytest <caminho>`.
- Commits e PRs saem sem atribuição ao Claude: sem `Co-Authored-By`, sem
  "Generated with Claude Code" e sem citar o Claude na mensagem. O texto é escrito
  como da equipe. O `.claude/settings.json` já desliga a atribuição automática.

## Plano de personalização

@plano-personalizacao.md
