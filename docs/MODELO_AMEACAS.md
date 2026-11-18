# Modelo de ameaças — Docelera Cofre

Versão 0.1 · 2026-11-17 · rascunho para revisão externa

## 1. Objetivo de segurança

> Mesmo com o servidor da Docelera totalmente comprometido, ninguém lê ou altera o conteúdo de um cofre sem a senha mestra (ou a passkey de desbloqueio) do usuário. Para itens da empresa, a exceção é o fluxo de recuperação por **dois** administradores, que é sempre auditado e notificado.

## 2. Ativos

| Ativo | Onde existe em claro | Impacto se vazar |
|---|---|---|
| Senha mestra | Só na memória do dispositivo, durante o desbloqueio | Crítico |
| Chave da conta, chave pessoal, chaves de coleção | Só na memória de dispositivo desbloqueado | Crítico |
| Conteúdo dos itens (senhas, chaves privadas de passkey, notas, TOTP) | Só em dispositivo desbloqueado | Crítico |
| Metadados (quantidade de itens, tamanhos, horários, membros das coleções) | Servidor | Médio |
| Chave privada de recuperação da organização | Cifrada para os admins; em claro só no dispositivo do admin durante a recuperação | Crítico para os cofres da empresa |
| Chaves de assinatura da extensão e do app | Sistema de build/CI | Crítico (permite distribuir código malicioso) |
| Log de auditoria | Servidor | Alto (integridade) |

**Importante:** URLs e nomes de itens também são cifrados. O servidor não sabe em quais sites o usuário tem conta.

## 3. Atacantes

| # | Atacante | Capacidades assumidas |
|---|---|---|
| A1 | Atacante externo na internet | Tráfego de rede, credential stuffing, phishing |
| A2 | Invasor do servidor/banco | Lê e altera o banco e os backups; pode responder errado à API |
| A3 | Funcionário malicioso da Docelera | Igual ao A2, mais acesso ao deploy |
| A4 | Admin malicioso do tenant cliente | Gerencia usuários, equipes e coleções da própria empresa |
| A5 | Colega de trabalho malicioso | Membro legítimo de algumas coleções |
| A6 | Site malicioso ou de phishing | Executa JS na página onde a extensão atua |
| A7 | Malware no dispositivo do usuário | Fora do escopo enquanto o cofre está desbloqueado (ver §6) |
| A8 | Ataque à cadeia de suprimentos | Dependência npm comprometida, CI comprometido |
| A9 | Ex-funcionário | Teve acesso legítimo no passado |

## 4. Ameaças e mitigações

### 4.1 Servidor comprometido (A2, A3)

| Ameaça | Mitigação |
|---|---|
| Ler o conteúdo dos cofres | Todo conteúdo é cifrado no cliente (AEAD); o servidor não tem chaves |
| Ataque de dicionário offline contra a senha mestra a partir do banco | Login OPAQUE: o banco não guarda hash verificável da senha. O blob de chaves cifradas usa Argon2id com parâmetros altos |
| Trocar a chave pública de um membro para receber chaves de coleção (MITM) | Chaves públicas assinadas e com fingerprint verificável; a cadeia de identidade é assinada pela chave da organização (ESPEC_CRIPTO §6) |
| Reverter um item para versão antiga (rollback) ou trocar itens entre si | Os dados associados do AEAD incluem id do item, id da coleção, tipo e versão; o cliente recusa versão menor que a já vista |
| Servir JS malicioso do app web para capturar a senha mestra | **Risco residual principal do app web.** Mitigações: CSP estrita, SRI, build reproduzível com hash publicado, desbloqueio preferencial pela extensão (código assinado pela loja). Admins podem exigir uso só pela extensão |
| Forjar um pedido de recuperação | A recuperação exige assinatura de 2 admins no cliente; o servidor sozinho não tem a chave de recuperação |
| Apagar ou alterar o log de auditoria | Log em cadeia de hash, com âncora periódica assinada e exportada para o cliente |

### 4.2 Externo (A1)

| Ameaça | Mitigação |
|---|---|
| Credential stuffing | OPAQUE + MFA obrigatório + rate limit por conta/IP + bloqueio progressivo |
| Phishing da senha mestra | Desbloqueio por passkey (vinculada à origem); a extensão alerta quando a senha mestra é digitada fora de cofre.docelera.com |
| Roubo de sessão | Tokens curtos ligados ao dispositivo (DPoP / chave do dispositivo), cookies `__Host-`, `SameSite=Strict` |
| Novo dispositivo controlado pelo atacante | Aprovação a partir de um dispositivo confiável, ou e-mail + MFA; notificação a todos os dispositivos |

### 4.3 Admin malicioso do tenant (A4)

| Ameaça | Mitigação |
|---|---|
| Ler o cofre pessoal de um funcionário | Chave pessoal fora da custódia de recuperação; não existe caminho técnico |
| Recuperar sozinho a conta de um funcionário | Quórum de 2 admins; usuário notificado por e-mail; evento no log |
| Adicionar a si mesmo a uma coleção sigilosa | Permitido (o admin manda na empresa), mas registrado no log e visível aos membros da coleção |

### 4.4 Colega e ex-funcionário (A5, A9)

| Ameaça | Mitigação |
|---|---|
| Ver itens de coleções fora da sua equipe | Sem chave de coleção não há como decifrar; permissões por coleção |
| Manter acesso depois de sair da equipe | Rotação da chave da coleção na remoção, com novos itens cifrados com a chave nova. Itens antigos podem estar em cache local: a remoção pede **troca das senhas sensíveis**, sinalizadas no painel |
| Exportar tudo antes de sair | Exportação controlada por política do admin e sempre registrada no log |

### 4.5 Extensão e sites (A6)

| Ameaça | Mitigação |
|---|---|
| Site falso recebe preenchimento | Casamento por eTLD+1 (Public Suffix List) ou host exato; nunca em HTTP; nunca em iframe de outra origem sem confirmação |
| Clickjacking da UI de preenchimento | A UI roda em popup/iframe da extensão, fora do DOM da página; preenchimento só com gesto do usuário |
| Página lê os dados da extensão | Content script em mundo isolado; mensagens validadas por origem; zero segredos no DOM além do campo preenchido |
| Site usa a extensão para criar ou usar passkey de outro RP | Validação do rpId contra a origem real informada pelo navegador, seguindo as regras WebAuthn |
| Página força o autofill de campos invisíveis | Campos ocultos ou com tamanho zero são ignorados |

### 4.6 Cadeia de suprimentos (A8)

| Ameaça | Mitigação |
|---|---|
| Dependência npm maliciosa | Dependências mínimas e fixadas por hash; atualização só por PR revisado; SBOM; nada de scripts `postinstall` |
| CI comprometido publica extensão maliciosa | Publicação manual nas lojas com 2FA; build reproduzível conferido por segunda máquina; chaves de assinatura fora do CI |

## 5. Propriedades garantidas

1. Confidencialidade e integridade do conteúdo contra A2/A3.
2. O cofre pessoal é inacessível ao A4.
3. Toda recuperação, exportação, compartilhamento e mudança de permissão gera evento auditável à prova de adulteração.
4. Comprometer um dispositivo não revela cofres de outros usuários além do que esse usuário já acessava.

## 6. Fora do escopo (riscos aceitos e documentados)

- Malware com controle total de um dispositivo **desbloqueado** (keylogger, leitura de memória).
- Servidor que serve JS malicioso ao **app web**: mitigado, mas não eliminado (ver §4.1). A extensão é o caminho recomendado.
- Análise de metadados de tráfego (tamanho e frequência de sincronização).
- Coerção do usuário.

## 7. Itens para a revisão externa decidir

- Parâmetros do Argon2id compatíveis com celular no futuro.
- Necessidade de transparência de chaves (log público de chaves) além das assinaturas pela organização.
- Proteção do app web contra servidor malicioso (modelo "web app assinado" via extensão).
