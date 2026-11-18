# Revisão interna do desenho — rodada 1

Data: 2026-11-17 · Escopo: [ESPEC_CRIPTO.md](ESPEC_CRIPTO.md) v0.1, [MODELO_AMEACAS.md](MODELO_AMEACAS.md) v0.1, [PLANO.md](PLANO.md)

## Como foi feita

Três revisores independentes leram os documentos sem ver o raciocínio de quem os escreveu. Cada um atacou uma área:

1. Núcleo criptográfico.
2. Organização, recuperação e compartilhamento.
3. Extensão, passkeys e app web.

Os achados foram conferidos contra o texto da especificação e consolidados. Achados repetidos entre revisores foram unidos. As referências a incidentes públicos são as citadas pelos revisores e devem ser confirmadas antes de entrarem em material externo.

## Conclusão

**A v0.1 não cumpre a promessa principal** ("mesmo com o servidor comprometido, ninguém lê o cofre"). Nenhuma criptografia é quebrada: o problema é que o servidor entrega chaves públicas, listas de admins e listas de membros que **ninguém verifica**. O cofre pessoal está bem protegido. Os cofres da empresa, não.

Todos os achados têm correção conhecida. Eles exigem uma **v0.2 da especificação** antes de qualquer código, e **3 decisões de produto** (§4).

## 1. Críticos

| # | Achado | Ataque | Correção |
|---|---|---|---|
| C1 | **ORK_pub e OSK sem âncora de confiança** (ESPEC §6, §7) | O servidor entrega sua própria chave de recuperação no cadastro e lê a AK de todo usuário novo. Num dispositivo novo, entrega uma OSK falsa (o TOFU acontece justamente no primeiro contato) | ORK_pub assinada pela OSK. Fingerprint da OSK entregue fora do servidor (link de convite com segredo no fragmento `#`, ou QR impresso). OSK, ORK_pub e versão guardados dentro do estado cifrado do usuário, para que dispositivos novos herdem após o desbloqueio |
| C2 | **Lista de admins não assinada** (ESPEC §6.1, §7) | O servidor, ou um admin sozinho, cria contas "admin" e recebe a OSK e as partes Shamir → reconstrói a ORK → lê todas as AKs | Manifesto de admins versionado e assinado. Incluir admin exige quórum, carência de 72 h e alerta em todos os dispositivos de admin. Partes e OSK só vão para admins do manifesto |
| C3 | **A recuperação reconstrói a ORK inteira no dispositivo de um admin** (ESPEC §7) | Depois da primeira recuperação, o admin B (ou quem invadir o dispositivo dele) abre **todas** as AKs para sempre. "Rotacionar a ORK" não resolve, porque o servidor guarda as custódias antigas | **Decifração limiar** (ristretto255): cada admin devolve `s_i·R` com prova DLEQ, ligada ao pedido. Só o dispositivo do usuário combina os resultados. A ORK nunca existe inteira em lugar nenhum |
| C4 | **Chave efêmera E_pub do pedido não autenticada** (ESPEC §7) | O servidor troca E_pub pela sua chave; dois admins honestos aprovam e a AK vai para o servidor | O código de segurança de E_pub aparece no dispositivo do usuário e o admin confere ao vivo (ligação ou presencial). As aprovações assinam `(pedido, usuário, E_pub, validade)` |
| C5 | **O app web servido pelo servidor pode capturar a senha mestra** (MODELO §4.1, §6) | O servidor controla o HTML, o CSP e o SRI. Basta servir JS alterado a um usuário específico. A criptografia de admin (assinar atestados, recuperação) roda no mesmo JS | **Decisão D1:** a extensão (código assinado pela loja) passa a ser o único cliente que manipula chaves e faz operações de admin |

## 2. Altos

| # | Achado | Correção |
|---|---|---|
| A1 | Admin sozinho atesta uma chave falsa para um colega e passa a receber as CKs dele | Troca de chave assinada pela chave anterior do próprio usuário (continuidade), ou por quórum quando ela foi perdida. Alerta "a chave de Fulano mudou" para quem compartilha com ele |
| A2 | Sem rotação ou revogação da OSK; ex-admin continua podendo forjar atestados | Chaves de assinatura por admin, certificadas por uma raiz de quórum. Revogar ao remover o admin. Especificar a perda total da OSK |
| A3 | Partes Shamir antigas continuam válidas depois da redistribuição | Troca de admins gera ORK nova + nova custódia (com a decifração limiar de C3) |
| A4 | Concessão de CK (`seal` + assinatura) não amarra coleção, versão nem destinatário: o servidor reaproveita concessões entre coleções ou versões | Assinar `("cofre/grant/v1", org, coleção, key_version, destinatário, enc_pub, signatário, H(ct))`. Preferir HPKE (RFC 9180) modo Auth com esse contexto. Conferir que o signatário pode conceder |
| A5 | Lista de membros da coleção não assinada: na rotação o servidor injeta um membro extra | Manifesto assinado por coleção/versão (membros, papéis, versão atual), encadeado à versão anterior |
| A6 | A IK não muda: ex-membro lê a senha **nova** depois de "trocar senha". Mover um item para o cofre pessoal mantém uma IK conhecida pela empresa | IK nova em toda escrita após rotação e em toda movimentação; recifrar o item. Recifração completa da coleção após remoção |
| A7 | Rollback, itens ocultos e itens apagados que reaparecem, sobretudo em dispositivo novo | Manifesto assinado por cofre/coleção com raiz Merkle de `(id, versão, H(blob))` e marcas de exclusão |
| A8 | O log de auditoria pode ser bifurcado ou truncado pelo servidor | Log Merkle estilo Certificate Transparency com provas de consistência. Eventos sensíveis assinados pelo cliente e incluindo a última "cabeça" vista. Unir com transparência de chaves |
| A9 | Preenchimento por eTLD+1 por padrão vaza credenciais entre subdomínios e hospedagem compartilhada | **Decisão D3:** host exato por padrão; domínio-base como opção por item |
| A10 | Iframes, cadeia de ancestrais, frames sandbox e `about:blank` não especificados | O service worker calcula a cadeia inteira de frames. Só preenche se todos os ancestrais combinam; recusa origens opacas |
| A11 | UI injetada na página pode sofrer clickjacking via CSS (opacidade, sobreposição) | Checagem de visibilidade (IntersectionObserver v2), atraso de ~500 ms, e confirmação de cartão, identidade, nota e passkey só em popup próprio da extensão |
| A12 | Mensagens entre página, content script e service worker sem fronteira definida | A origem vem só de `sender`. RPC tipado com lista permitida. Nenhum `externally_connectable`. O content script nunca recebe a lista de itens |
| A13 | Interceptação de `navigator.credentials` no mundo da página é manipulável | O shim só repassa os dados. A origem é derivada pelo service worker. `clientDataJSON` montado no service worker. Toda criação ou uso de passkey confirmada em UI da extensão |
| A14 | Validação de rpId/origem frouxa | rpId = domínio efetivo ou sufixo registrável (nunca sufixo público); só contexto seguro; regras para iframes de outra origem |
| A15 | Flag UV ligada só porque o cofre estava desbloqueado | UV só com verificação recente (≤ 5 min); UP sempre com clique em UI da extensão |
| A16 | "Manter desbloqueado" é texto claro na prática; o MV3 mata o service worker | Só na extensão, documentado como sem proteção em dispositivo roubado. Chaves num offscreen document ou em `storage.session` restrito. Bloqueio por `chrome.idle` e `chrome.alarms` |
| A17 | Tomada da conta de publicação na loja publica uma extensão maliciosa | Upload verificado (chave offline), conta com chaves físicas, dois publicadores, hashes publicados, build reproduzido |
| A18 | XSS armazenado via campos de itens compartilhados; links `javascript:`; anexos renderizados na origem do cofre | Trusted Types sem política de HTML, renderização só como texto, abrir só http/https, anexos sempre como download |

## 3. Médios e baixos (resumo)

**Especificação**
- **Encapsulamentos de chave sem dados associados** (AK↔PK trocáveis; risco de vazar a PK para a custódia): rótulo + org + usuário + id em cada encapsulamento, e conferir as chaves públicas derivadas.
- **Cabeçalho `{v, alg, kid}` fora dos dados associados:** incluí-lo, com lista de algoritmos permitidos e versão mínima.
- **A frase "OPAQUE impede ataque offline" está errada.** OPAQUE impede *pré-computação*; com o banco vazado o ataque offline continua possível, ao custo Argon2id por tentativa. Corrigir o texto, exigir força mínima de senha e fixar os parâmetros no cliente.
- **Contexto e identidades do OPAQUE** não definidos (mesmo e-mail em várias organizações).
- **Senha antiga e passkey removida continuam abrindo a AK/PK** guardadas pelo servidor: oferecer "girar AK/PK". Salt do PRF fixo no cliente.
- **Itens sem assinatura:** membro "somente leitura" consegue alterar itens. Assinar cada versão com a chave do autor.
- **A recuperação expõe as CKs do usuário recuperado:** girar as coleções afetadas.
- **Rótulos de contexto em todas as assinaturas;** separar chaves por uso.
- **CBOR determinístico, HKDF info, formato do código de recuperação e versões concorrentes** subespecificados.
- **Metadados vazam pelo tamanho dos blobs:** aplicar padding; considerar cifrar o `type`.

**Organização**
- **Organização com 1 admin:** não gerar a ORK nem custódia até existirem 2 admins.
- **Recuperação forçada sem o usuário saber:** **Decisão D2** (carência com veto).
- **Saída de funcionário:** rotação pendente visível ao admin, remoção da custódia e do atestado, e cache local antigo (documentar).
- **Exclusão de tenant:** quórum, carência de 30 dias, exportação e expurgo de backups.
- **LGPD:** eventos do cofre pessoal fora do log visível ao admin; papéis de controlador e operador no DPA; retenção; exportação do cofre pessoal na saída.
- **Reset de MFA e troca de e-mail:** exigem quórum e carência.

**Extensão e app**
- **Campos ocultos e redirecionamento de formulário:** checar visibilidade e alertar quando o envio for para outra origem ou HTTP.
- **Detecção de phishing da senha mestra:** sem verificador rápido guardado; priorizar o desbloqueio por passkey.
- **Passkeys:** fora de exportação em claro; avisar ao compartilhar; declarar que passkeys do cofre da empresa são recuperáveis pelo admin.
- **Convivência com chaves físicas e Windows Hello:** sempre oferecer "usar outro dispositivo"; attestation `none`; AAGUID próprio.
- **Área de transferência via offscreen document:** limpar só se ainda contém o valor copiado; o histórico Win+V guarda cópia (documentar).
- **Memória:** chaves no heap WASM com `sodium_memzero`, `no-store`, bloqueio em `pagehide`.
- **Sessão:** escolher um modelo só (DPoP *ou* cookies). SSO com PKCE, `state` e `nonce`.
- **Aprovação de dispositivo:** digitar o código em vez de comparar; definir a revogação de dispositivo perdido.

**Lacunas OWASP ASVS nível 3:** política de senha mestra; lista e revogação de sessões e dispositivos; autorização no servidor por objeto (IDOR) e isolamento entre tenants; baseline de cabeçalhos; limites de anexos; requisitos OIDC; catálogo de eventos e alertas; anti-automação nos fluxos de recuperação e convite.

## 4. Decisões de produto necessárias

| # | Decisão | Recomendação | Impacto |
|---|---|---|---|
| **D1** | Onde o usuário abre o cofre | **Só pela extensão** (página própria da extensão, assinada pela loja). `cofre.docelera.com` vira página de entrada. Desbloqueio pelo navegador sem extensão fica desligado por padrão, liberável pelo admin com aviso permanente | Elimina o maior risco residual. O usuário precisa instalar a extensão, o que no MVP já era previsto |
| **D2** | Recuperação pelo admin | **Carência de 48 h** em que os dispositivos do usuário recebem alerta e podem vetar. Pedido iniciado pelo próprio usuário dispensa a carência | Impede recuperação "silenciosa" por dois admins ou por admin + servidor |
| **D3** | Preenchimento automático | **Host exato por padrão** (login.empresa.com ≠ blog.empresa.com); domínio-base opcional por item | Mais seguro; o usuário às vezes precisa marcar "vale para todo o domínio" |

## 5. O que o desenho já acerta

- Cofre pessoal fora da custódia, com código de recuperação próprio e a troca deixada explícita.
- OPAQUE + chave de embrulho derivada do `export_key`; só libsodium/WebCrypto; XChaCha20 com nonce aleatório.
- Chave por item, com dados associados ligando id, cofre, tipo e versão.
- Quórum na recuperação e risco residual documentado com honestidade.
- Sessão ligada ao dispositivo; perguntas certas já listadas para o revisor (§12).

## 6. Próximos passos

1. Decisões D1–D3.
2. ESPEC_CRIPTO v0.2 e MODELO_AMEACAS v0.2 incorporando os itens acima, com destaque para:
   - manifesto assinado da organização (admins, membros, ORK_pub);
   - decifração limiar;
   - log Merkle de chaves e auditoria;
   - IK nova nas escritas;
   - HPKE nas concessões.
3. Rodada 2 de revisão interna sobre a v0.2, até não restarem críticos nem altos.
4. Só então começar a fase 1 (servidor).
