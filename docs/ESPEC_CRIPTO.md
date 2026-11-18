# Especificação criptográfica — Docelera Cofre

Versão 0.1 · 2026-11-17 · rascunho para revisão externa. Lê-se junto com [MODELO_AMEACAS.md](MODELO_AMEACAS.md).

## 1. Princípios

- Só primitivas de bibliotecas consagradas: **libsodium** (via WASM) e **WebCrypto**. Nada implementado por nós.
- Toda chave é gerada no cliente com CSPRNG (`randombytes_buf` / `crypto.getRandomValues`).
- Todo blob cifrado tem cabeçalho versionado `{v, alg, kid}`, para permitir trocar algoritmos sem quebrar dados antigos.
- O servidor guarda só blobs cifrados, chaves públicas, assinaturas e metadados mínimos.

## 2. Primitivas

| Uso | Primitiva |
|---|---|
| Login sem expor a senha | OPAQUE-3DH (RFC 9807), grupo ristretto255, KSF = Argon2id |
| Função de custo da senha | Argon2id, m = 128 MiB, t = 3, p = 1 (a confirmar na revisão; mínimo aceitável 64 MiB) |
| Derivação de chaves | HKDF-SHA-256 com rótulos de domínio `cofre/<uso>/v1` |
| Cifra simétrica | XChaCha20-Poly1305 (AEAD), nonce aleatório de 192 bits |
| Cifra para chave pública | `crypto_box_seal` (X25519 + XSalsa20-Poly1305), sempre acompanhada de assinatura do remetente |
| Assinatura | Ed25519 |
| Passkeys guardadas | ECDSA P-256 / ES256 (WebCrypto), conforme o site exigir |
| Divisão de segredo | Shamir 2-de-n sobre GF(256) (biblioteca auditada, a escolher na revisão) |
| Ligação da sessão ao dispositivo | ECDSA P-256 não extraível (WebCrypto), estilo DPoP |

## 3. Hierarquia de chaves do usuário

```
senha mestra ──OPAQUE (KSF Argon2id)──► export_key (só o cliente conhece)
                                              │ HKDF "cofre/kek/v1"
                                              ▼
                                         KEK (chave de embrulho)
                         ┌────────────────────┼─────────────────────────┐
                         ▼                                              ▼
           AK  chave da conta (empresa)                    PK  chave do cofre pessoal
           ├─ custódia: selada p/ chave de                 ├─ também embrulhada pela RK
           │  recuperação da organização (§7)              │  (código de recuperação pessoal)
           ├─ embrulha: chave privada X25519 (enc)         └─ NUNCA custodiada pela organização
           └─ embrulha: chave privada Ed25519 (assinatura)
                         │
                         ▼
        CK  chaves das coleções (seladas para a X25519 de cada membro + assinadas)
                         │
                         ▼
        IK  chave de cada item (embrulhada pela CK ou pela PK)
                         │
                         ▼
        conteúdo do item (XChaCha20-Poly1305)
```

| Chave | Tamanho | Gerada em | Guardada no servidor como |
|---|---|---|---|
| AK | 256 bits | Cadastro | `AEAD(KEK, AK)`, `AEAD(KEK_prf, AK)` por passkey, `seal(ORK_pub, AK)` |
| PK | 256 bits | Cadastro | `AEAD(KEK, PK)`, `AEAD(KEK_prf, PK)`, `AEAD(RK, PK)` |
| RK | 256 bits | Cadastro | Nunca. Entregue ao usuário como código Base32 no kit de emergência |
| Identidade X25519/Ed25519 | — | Cadastro | Públicas em claro + atestado da organização; privadas como `AEAD(AK, priv)` |
| CK | 256 bits | Criação da coleção | `seal(membro_X25519_pub, CK)` + assinatura Ed25519 de quem concedeu |
| IK | 256 bits | Criação do item | `AEAD(CK ou PK, IK)` |

**Por que existe uma IK por item:** anexos grandes e rotação de coleção ficam baratos, porque basta reembrulhar as IKs e não recifrar todo o conteúdo.

## 4. Formato de um item

```
item = {
  id, vault_id, type, version, key_version,
  wrapped_ik: AEAD(CK|PK, IK, ad = canon(id, vault_id, key_version)),
  blob:       AEAD(IK, json_conteudo, ad = canon(id, vault_id, type, version))
}
```

- `canon` = codificação canônica (CBOR determinístico) dos campos.
- O conteúdo cifrado inclui **nome, URLs, usuário, senha, notas, TOTP e campos personalizados**. Só ficam em claro `type` e os carimbos de data.
- O cliente guarda a maior `version` vista de cada item e recusa versões menores (proteção contra rollback pelo servidor).

Tipos: `login`, `passkey`, `link`, `nota`, `cartao`, `identidade`, `anexo`.

Uma passkey guarda: `rpId`, `credentialId`, `userHandle`, `userName`, chave privada (PKCS#8), algoritmo, `signCount` (sempre 0 em passkeys sincronizadas) e a data de criação.

## 5. Cadastro e login

**Cadastro**
1. O cliente executa o registro OPAQUE e obtém `export_key`. O servidor guarda só o registro OPAQUE.
2. O cliente gera AK, PK, RK, as identidades X25519/Ed25519 e a chave do dispositivo.
3. O cliente envia os blobs embrulhados da §3, as chaves públicas e o pedido de atestado (§6).
4. É gerado o **kit de emergência** (PDF local, nunca enviado): RK e instruções. O usuário confirma que guardou.
5. O MFA é configurado: passkey (preferencial) ou TOTP.

**Login / desbloqueio**
1. Login OPAQUE + MFA → sessão ligada à chave do dispositivo.
2. `export_key` → KEK → decifra AK e PK. As chaves ficam só em memória.
3. **Desbloqueio por passkey:** a extensão WebAuthn PRF gera `prf_out`. HKDF `"cofre/kek-prf/v1"` sobre `prf_out` dá `KEK_prf`, que abre AK e PK. A identidade continua passando por OPAQUE ou pelo token do dispositivo.
4. Bloqueio automático: as chaves são apagadas da memória (inatividade configurável pelo admin, padrão 15 min).

**Novo dispositivo:** depois do login, o servidor exige aprovação. O dispositivo confiável mostra um código de 6 dígitos derivado do transcript, e o usuário confirma nos dois. A alternativa é link por e-mail + MFA, com notificação a todos os dispositivos.

**Troca de senha mestra:** novo registro OPAQUE → nova KEK → reembrulho de AK e PK. AK e PK não mudam.

## 6. Autenticidade das chaves públicas

O servidor poderia trocar a chave pública de alguém para receber CKs. Por isso:

1. A organização tem uma chave de assinatura **OSK** (Ed25519), criada pelo primeiro admin. A OSK privada fica selada para cada admin.
2. Quando um usuário entra na organização, um admin assina `atestado = sign(OSK, canon(org_id, user_id, email, enc_pub, sig_pub, versão))`.
3. Antes de selar uma CK para alguém, o cliente verifica o atestado com a OSK pública. Essa chave é **fixada no primeiro uso** (TOFU) e tem a fingerprint exibida no console do admin.
4. Se uma chave pública mudar sem atestado novo, o cliente bloqueia o compartilhamento e alerta.

Em aberto para a revisão: adicionar um log de transparência de chaves (§MODELO_AMEACAS 7).

## 7. Recuperação pelo administrador (só o cofre da empresa)

**Configuração da organização**
- O primeiro admin gera o par de recuperação **ORK** (X25519).
- A ORK privada é dividida com **Shamir 2-de-n** entre os admins. Cada parte fica selada para a X25519 do admin dono e é assinada.
- A recuperação só fica disponível com **2 ou mais admins**. Com 1 admin, o console avisa que a recuperação está desativada.
- Quando a lista de admins muda, as partes são redistribuídas e as antigas descartadas.

**Custódia:** no cadastro, o cliente envia `seal(ORK_pub, AK)`. A PK nunca entra na custódia.

**Fluxo de recuperação**
1. O usuário (com identidade confirmada por e-mail + verificação do admin) gera num dispositivo novo um par efêmero X25519 `E` e abre o pedido.
2. O admin A aprova e envia a sua parte, cifrada para o admin B e assinada.
3. O admin B aprova. No dispositivo do B: as duas partes reconstroem a ORK privada em memória, abrem `seal(ORK_pub, AK)` e geram `seal(E_pub, AK)`, assinado pelo B. Depois disso a ORK e a AK são apagadas da memória.
4. O usuário abre a AK com `E`, define nova senha mestra (novo OPAQUE e nova KEK) e **gira** AK e identidade: gera novas chaves, recebe novos atestados e reembrulha as CKs a que tinha acesso.
5. Cofre pessoal: o usuário digita a RK (kit de emergência). Sem RK, a PK é perdida e os itens pessoais ficam irrecuperáveis. Isso é dito claramente no cadastro.
6. Eventos no log: pedido, cada aprovação, conclusão. Notificação por e-mail ao usuário e a todos os admins.

**Risco residual documentado:** o dispositivo do admin B vê a ORK por um instante. Por isso a etapa 4 gira a AK do usuário recuperado, e a ORK pode ser rotacionada pelo console depois de cada recuperação (recomendado por padrão).

## 8. Coleções e compartilhamento

- **Criar coleção:** gerar CK; selar para cada membro com atestado válido; assinar com a Ed25519 de quem concede.
- **Adicionar membro:** quem tem a CK sela para o novo membro e assina.
- **Remover membro:** gerar CK', reembrulhar todas as IKs da coleção com CK', selar CK' para os membros restantes e marcar `key_version` +1. Os itens sinalizados como "sensíveis" entram numa lista de "trocar senha" no painel.
- **Mover item entre cofre pessoal e coleção:** reembrulhar a IK e registrar no log.

## 9. Sessão, transporte e armazenamento local

- TLS 1.3 com HSTS preload. Nada de criptografia de transporte própria.
- Token de acesso de 5 min + refresh rotativo ligado à chave do dispositivo (prova de posse em cada requisição).
- Cache local (IndexedDB) só com blobs cifrados. As chaves nunca são persistidas em claro.
- "Manter desbloqueado neste dispositivo" fica **desligado por padrão**. Se o admin permitir, a AK/PK é embrulhada por uma chave AES-GCM não extraível do WebCrypto, presa ao perfil do navegador.
- A área de transferência é limpa após 30 s, quando o navegador permite.

## 10. Log de auditoria

- Cada evento tem `hash_n = SHA-256(hash_{n-1} || canon(evento))`. O evento é assinado pela chave do dispositivo de quem agiu, quando houver.
- Uma âncora diária assinada pelo servidor é exportável pelo admin. Qualquer alteração retroativa quebra a cadeia.
- O log nunca contém conteúdo de item, só ids, tipos de ação, autor, dispositivo e horário.

## 11. Exportação e importação

- A importação (CSV do Chrome, Bitwarden JSON, 1Password 1PUX, CSV genérico) acontece toda no cliente. O arquivo nunca vai ao servidor.
- A exportação exige senha mestra + MFA, segue a política do admin e é registrada no log. O formato padrão é cifrado; o formato em claro exige confirmação dupla.

## 12. Questões para o revisor

1. Parâmetros do Argon2id dentro do OPAQUE no navegador.
2. Shamir 2-de-n com reconstrução num dispositivo, ou um esquema limiar que nunca reconstrua a ORK.
3. TOFU da OSK vs. log de transparência de chaves.
4. Riscos do app web servido pelo próprio servidor (MODELO_AMEACAS §4.1).
5. Interceptação de `navigator.credentials` pela extensão, para atuar como provedor de passkeys.
