# Docelera Cofre

Gerenciador de senhas, passkeys, links e notas seguras da Docelera, com criptografia de ponta a ponta. Vendido como pacote adicional da plataforma Docelera.

**Status:** fase 0, especificação. Nenhum código de produção é escrito antes da revisão externa do desenho criptográfico.

## Documentos

| Documento | Conteúdo |
|---|---|
| [docs/PLANO.md](docs/PLANO.md) | Decisões de produto aprovadas e fases |
| [docs/MODELO_AMEACAS.md](docs/MODELO_AMEACAS.md) | Atacantes, ativos, ameaças e mitigações |
| [docs/ESPEC_CRIPTO.md](docs/ESPEC_CRIPTO.md) | Hierarquia de chaves, protocolos e fluxos |
| [docs/REVISAO_INTERNA.md](docs/REVISAO_INTERNA.md) | Revisão interna do desenho (rodada 1): achados e decisões pendentes |
| [docs/REVISAO_EXTERNA.md](docs/REVISAO_EXTERNA.md) | Escopo e candidatos para a auditoria independente |

## Regras do repositório

- Nenhuma primitiva criptográfica própria: só libsodium/WebCrypto.
- Toda mudança entra por PR com revisão. Mudanças em `docs/ESPEC_CRIPTO.md` ou no código de criptografia exigem nova revisão de segurança.
- Dependências fixadas, mínimas e justificadas no PR que as adiciona.
- Segredos nunca entram no repositório, nem os de teste reais.

## Vulnerabilidades

Relate de forma privada para seguranca@docelera.com, nunca em issue pública.
