# Docelera Cofre — plano de produto e segurança

Status: **aprovado** (2026-11-17), exceto o preço; próxima etapa: fase 0

Gerenciador de senhas da Docelera, vendido como pacote adicional. Ele guarda logins, passkeys, links, notas seguras, códigos TOTP, cartões e anexos, e se conecta ao navegador por uma extensão.

## Decisões tomadas

| Tema | Decisão |
|---|---|
| Base | Desenvolvimento próprio, do zero |
| Nível de segurança | O mais exigente possível (meta: OWASP ASVS nível 3) |
| Senha mestra esquecida | Recuperação pelo administrador da empresa |
| Cofres | Cofre da empresa + cofre pessoal |
| MVP | App web + extensão Chrome/Edge; Firefox e celular depois |
| Armazenamento | Somente nuvem Docelera, atrás de uma interface de armazenamento (híbrido/self-hosted possível no futuro) |
| Auditoria externa | Sim, com orçamento separado: revisão de criptografia + pentest antes de vender |
| Preço | Valor fixo por mês |

## Propostas aprovadas (2026-11-17)

1. **Repositório separado** (`DOCELERA-COFRE`) e serviço separado (`cofre.docelera.com`, banco Postgres próprio). Isso reduz a superfície de ataque, deixa a auditoria só sobre o código do cofre e evita que uma falha no QualiSheet alcance o cofre.
2. **O cofre pessoal não pode ser recuperado pelo admin.** A recuperação pelo admin cobre só o cofre da empresa. O cofre pessoal tem um *código de recuperação pessoal*, entregue num "kit de emergência" em PDF no cadastro. Isso protege a privacidade do funcionário (LGPD).
3. **Recuperação exige 2 administradores** (quórum): um pede, outro aprova, o usuário é avisado por e-mail e tudo fica na auditoria.
## Em aberto

- **Preço:** o valor fixo é por empresa ou por usuário? Qual o valor? (não bloqueia as fases 0 a 4)

## Arquitetura de criptografia

O servidor **nunca** recebe a senha mestra, nenhuma chave em claro nem conteúdo em claro.

```
Senha mestra ──Argon2id (≥64 MiB, t=3)──► chave mestra
                                          │
                     ┌────────────────────┴───────────────────┐
              OPAQUE (login sem enviar senha)     HKDF ► chave de embrulho
                                                              │
                                     chave da conta (256 bits aleatória)
                                       │                 │
                         par X25519/Ed25519 do usuário   chave do cofre pessoal
                                       │                 (NÃO custodiada pelo admin)
                 chaves das coleções da empresa (embrulhadas p/ cada membro)
                                       │
                         itens (XChaCha20-Poly1305 / AES-256-GCM)
```

- **Biblioteca:** libsodium (via WASM). Nenhuma primitiva criptográfica é escrita por nós.
- **Login:** protocolo OPAQUE (aPAKE). Um vazamento do banco do servidor não permite ataque de dicionário offline contra a senha mestra.
- **MFA obrigatório:** passkey (preferencial) ou TOTP. Desbloqueio por passkey com a extensão WebAuthn PRF.
- **Novo dispositivo:** precisa ser aprovado num dispositivo já confiável, ou com e-mail + MFA.
- **Compartilhamento:** cada coleção ("Financeiro", "TI"…) tem chave própria, embrulhada com a chave pública de cada membro. Quando alguém sai da equipe, a chave da coleção é rotacionada.
- **Recuperação pelo admin:** no cadastro, a chave da conta *da parte empresa* é embrulhada com a chave pública de recuperação da organização. Recuperar exige 2 admins; o usuário define nova senha mestra; o evento fica registrado e o usuário é notificado.
- **Integridade:** itens com AEAD e metadados autenticados (id, tipo, coleção), para impedir troca de itens entre registros. Chaves públicas verificáveis (fingerprint) contra substituição pelo servidor.
- **Memória:** bloqueio automático, área de transferência limpa após 30 s, chaves fora do `localStorage`.

## Segurança da aplicação

- App web: CSP estrita sem `unsafe-inline`, Trusted Types, SRI, COOP/COEP, nenhum script de terceiros e nenhum analytics.
- Servidor: rate limiting e bloqueio progressivo, tokens de sessão curtos ligados ao dispositivo, isolamento por organização em toda consulta, banco cifrado em repouso, backups cifrados.
- Log de auditoria à prova de adulteração (cadeia de hash): login, visualização, cópia, compartilhamento, exportação, recuperação.
- Cadeia de suprimentos: dependências fixadas e mínimas, SBOM, builds reproduzíveis da extensão, releases assinadas, revisão obrigatória de PR, varredura de segredos.
- Extensão MV3 com permissões mínimas:
  - preenche só com gesto do usuário;
  - casamento exato de domínio (eTLD+1, com opção de host exato);
  - nunca preenche em HTTP nem em iframe de outra origem;
  - proteção contra clickjacking.
- Conformidade: LGPD (DPA com clientes, registro de tratamento); depois, caminho para ISO 27001/SOC 2.
- Antes da venda: revisão independente da criptografia + pentest de app web e extensão. Depois: programa de bug bounty.

## Integração com a Docelera

- Login de identidade via conta.docelera.com. A senha mestra é separada e nunca passa pela conta Docelera.
- Usuários, equipes e administradores sincronizados do tenant.
- Licença e cobrança mensal pelo fluxo atual (FOCAGIL OMIE / Asaas).
- Menu "Cofre" no Docelera abre `cofre.docelera.com`.

## Fases

| Fase | Entrega |
|---|---|
| 0. Especificação | Modelo de ameaças, documento de criptografia, **revisão externa do desenho** antes de codar |
| 1. Servidor | Contas (OPAQUE + MFA), chaves, itens cifrados, organizações/coleções, auditoria, API |
| 2. App web | Desbloqueio, tipos de item, gerador de senhas, TOTP, compartilhamento, console do admin, recuperação, kit de emergência, importação (Chrome, Bitwarden, 1Password, CSV) |
| 3. Extensão Chrome/Edge | Preenchimento automático, salvar login, gerador, TOTP, desbloqueio |
| 4. Passkeys | Extensão como provedor de passkeys (criar/usar em sites), desbloqueio por passkey |
| 5. Lançamento | Endurecimento, pentest, correções, cobrança, publicação na Chrome Web Store / Edge Add-ons, artigos em docelera.com/ajuda |

Depois: Firefox, app de celular, alertas de senhas vazadas/fracas, envio seguro de segredo por link.
