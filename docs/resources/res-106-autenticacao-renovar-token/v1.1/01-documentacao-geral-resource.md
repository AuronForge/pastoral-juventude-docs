# RES-106 — Renovar Access Token

**Versão:** 1.1  
**Status:** Em revisão  
**Baseline lógica:** MVP Domain Baseline v1.6  
**Base física:** Especificação Física do Banco v1.5  
**Catálogo:** Catálogo de Resources/APIs v1.3  
**Endpoint:** `POST /api/v1/autenticacao/renovar-token`

---

## 1. Identificação do Resource

| Campo | Valor |
|---|---|
| Domínio / Bounded Context | Autenticação e Autorização |
| Capacidade | Renovar Access Token |
| Nome do Resource | RES-106 — Renovar Access Token |
| Ação | Renovar token |
| Versão da especificação | 1.1 |
| Status | Em revisão |
| Última atualização | 2026-09-17 |

### 1.1 Objetivo

A capacidade permite que uma sessão normal ainda válida obtenha um novo Access Token sem exigir novo login, utilizando exclusivamente o Refresh Token mantido no cookie `refresh_token`.

O Resource DEVE validar o Refresh Token, a sessão, o usuário, a duração absoluta da sessão e o limite de renovações. Em caso de sucesso, DEVE rotacionar o Refresh Token, invalidar o valor anterior, incrementar o contador de renovações e emitir novo Access Token de 15 minutos, sem estender a duração absoluta da sessão.

### 1.2 Escopo

Inclui:

- leitura do cookie `refresh_token`;
- validação da existência do cookie;
- validação do hash do Refresh Token;
- validação da sessão ativa;
- validação da expiração absoluta da sessão;
- validação do limite máximo de 3 renovações;
- rotação obrigatória do Refresh Token;
- emissão de novo Access Token;
- atualização da sessão;
- validação de origem para proteção CSRF;
- auditoria de sucesso e falha;
- rejeição de sessão substituída, inválida, expirada, de usuário bloqueado ou inativo.

Não inclui:

- login (`RES-001`);
- alteração de senha (`RES-002`);
- recuperação de senha (`RES-003`);
- logout (`RES-107`);
- criação de nova sessão independente.

---

## 2. Regras de Negócio

- **RN-001 — Cookie obrigatório:** O endpoint DEVE receber o Refresh Token exclusivamente no cookie `refresh_token`.
- **RN-002 — Ausência do cookie:** Se o cookie não estiver presente, o sistema DEVE retornar `400 TOKEN_REFRESH_AUSENTE`.
- **RN-003 — Persistência segura:** O Refresh Token em texto puro NÃO DEVE ser persistido; somente seu hash deve existir em `SESSAO_USUARIO`.
- **RN-004 — Sessão única:** O Refresh Token somente é válido quando associado à sessão ativa atual do usuário.
- **RN-005 — Sessão substituída:** Se a sessão foi invalidada por `NOVO_LOGIN`, o sistema DEVE retornar `409 SESSAO_SUBSTITUIDA`.
- **RN-006 — Sessão inválida:** Sessão invalidada por outro motivo NÃO DEVE ser renovada.
- **RN-007 — Sessão absoluta:** A duração absoluta máxima da sessão é de 1 hora a partir de `data_criacao`.
- **RN-008 — Sem extensão absoluta:** A renovação NÃO DEVE estender `data_expiracao`.
- **RN-009 — Limite de renovações:** A sessão DEVE permitir no máximo 3 renovações.
- **RN-010 — Contador:** Cada renovação bem-sucedida DEVE incrementar `quantidade_refresh` em 1.
- **RN-011 — Limite atingido:** Ao tentar renovar após o limite permitido, a sessão DEVE ser invalidada com motivo `REFRESH_LIMITE_ATINGIDO` e exigir novo login.
- **RN-012 — Rotação:** Toda renovação bem-sucedida DEVE gerar novo Refresh Token.
- **RN-013 — Invalidação do anterior:** O Refresh Token anterior DEVE deixar de ser aceito imediatamente após a renovação.
- **RN-014 — Novo hash:** O novo Refresh Token DEVE substituir o hash anterior persistido na sessão.
- **RN-015 — Access Token:** O novo Access Token DEVE possuir validade nominal de 15 minutos.
- **RN-016 — Limite pela sessão:** O Access Token emitido NÃO DEVE ter `exp` posterior à duração absoluta restante da sessão.
- **RN-017 — JWT:** O novo Access Token DEVE ser assinado com RS256.
- **RN-018 — Claims:** O novo Access Token DEVE conter `sub`, `usuarioId`, `email`, `sessionId`, `roles`, `permissoes`, `iat`, `exp`, `iss`, `aud` e `jti`.
- **RN-019 — Autorização atual:** Roles e permissões do novo Access Token DEVEM refletir o estado de autorização vigente.
- **RN-020 — Usuário bloqueado:** Usuário `BLOQUEADO` NÃO DEVE renovar token.
- **RN-021 — Usuário inativo:** Usuário `INATIVO` NÃO DEVE renovar token.
- **RN-022 — Alteração de autorização:** Quando uma alteração de autorização já tiver invalidado a sessão com `ALTERACAO_AUTORIZACAO`, o Refresh Token NÃO DEVE renovar a sessão.
- **RN-023 — Alteração de senha:** Sessão invalidada por `ALTERACAO_SENHA` ou `RECUPERACAO_SENHA` NÃO DEVE ser renovada.
- **RN-024 — Cookie:** O novo Refresh Token DEVE ser enviado em cookie `refresh_token`, `HttpOnly`, `SameSite=Lax`, `Path=/api/v1/autenticacao`.
- **RN-025 — Secure:** O cookie DEVE usar `Secure=true` em produção; em desenvolvimento local via HTTP, PODE usar `Secure=false`.
- **RN-026 — Expiração do cookie:** `Max-Age`/`Expires` do cookie NÃO DEVE ultrapassar a duração absoluta restante da sessão.
- **RN-027 — CSRF/origem:** O endpoint DEVE validar a origem da requisição, pois depende de cookie.
- **RN-028 — CORS:** Origem deve pertencer à allow-list do ambiente e `allowCredentials=true`.
- **RN-029 — Sem wildcard:** CORS NÃO DEVE usar `*` quando credenciais estiverem habilitadas.
- **RN-030 — Segredos:** Refresh Token, hash de Refresh Token e Access Token NÃO DEVEM aparecer em logs ou auditoria.
- **RN-031 — Auditoria de sucesso:** Renovação bem-sucedida DEVE registrar `RENOVAR_TOKEN`.
- **RN-032 — Auditoria de falha:** Falha de renovação DEVE registrar `RENOVAR_TOKEN_FALHA`.
- **RN-033 — Invalidação por limite:** Quando o limite for atingido, DEVE ser registrado `INVALIDAR_SESSAO`.
- **RN-034 — Correlação:** O endpoint DEVE utilizar `X-Correlation-Id`.
- **RN-035 — Body:** O endpoint NÃO DEVE exigir body.
- **RN-036 — Idempotency-Key:** `Idempotency-Key` NÃO se aplica.
- **RN-037 — Concorrência:** Duas renovações concorrentes usando o mesmo Refresh Token anterior NÃO DEVEM produzir duas rotações válidas.
- **RN-038 — Reuso de token antigo:** Um Refresh Token já rotacionado DEVE ser tratado como inválido.

---

## 3. Validações e Invariantes

- **RV-001:** cookie `refresh_token` deve existir.
- **RV-002:** o valor do cookie deve corresponder ao hash persistido da sessão ativa.
- **RV-003:** a sessão deve estar `ativa = true`.
- **RV-004:** `data_invalidacao` deve ser nula para sessão ativa.
- **RV-005:** `data_expiracao` deve estar no futuro.
- **RV-006:** `quantidade_refresh` deve ser menor que 3 antes da renovação.
- **RV-007:** usuário deve estar `ATIVO`.
- **RV-008:** origem da request deve ser aceita pelas regras de CORS/CSRF.
- **RV-009:** `X-Correlation-Id`, quando enviado, deve ser UUID válido.
- **RV-010:** não deve haver body obrigatório.
- **RV-011:** a sessão deve pertencer ao usuário associado ao Refresh Token.
- **RV-012:** o novo `exp` do Access Token não pode exceder `data_expiracao` da sessão.
- **RV-013:** a atualização do hash e do contador deve ser atômica.

---

## 4. Pré-condições e Pós-condições

### 4.1 Pré-condições

- cookie `refresh_token` presente;
- Refresh Token corresponde à sessão ativa;
- sessão ainda dentro da duração absoluta de 1 hora;
- `quantidade_refresh < 3`;
- usuário ativo;
- origem aceita;
- banco e mecanismo JWT disponíveis.

### 4.2 Pós-condições — sucesso

- `quantidade_refresh` incrementada;
- `data_ultima_renovacao` atualizada;
- novo `refresh_token_hash` persistido;
- Refresh Token anterior invalidado por substituição do hash;
- novo Access Token emitido;
- novo cookie `refresh_token` enviado;
- `data_expiracao` da sessão permanece inalterada;
- auditoria `RENOVAR_TOKEN` registrada.

### 4.3 Pós-condições — limite atingido

- sessão marcada inativa;
- `data_invalidacao` preenchida;
- `motivo_invalidacao = REFRESH_LIMITE_ATINGIDO`;
- cookie deve ser expirado/removido;
- novo login exigido;
- auditorias `RENOVAR_TOKEN_FALHA` e `INVALIDAR_SESSAO` registradas.

---

## 5. Fluxos

### FLX-001 — Renovação com sucesso

1. Receber request.
2. Resolver `X-Correlation-Id`.
3. Validar Origin/CORS.
4. Ler cookie `refresh_token`.
5. Localizar sessão pelo hash correspondente.
6. Confirmar sessão ativa.
7. Confirmar usuário ativo.
8. Confirmar `data_expiracao` ainda válida.
9. Confirmar `quantidade_refresh < 3`.
10. Gerar novo Refresh Token.
11. Persistir o novo hash substituindo o anterior.
12. Incrementar `quantidade_refresh`.
13. Atualizar `data_ultima_renovacao`.
14. Recarregar roles e permissões vigentes.
15. Emitir novo Access Token.
16. Enviar novo cookie `refresh_token`.
17. Registrar `RENOVAR_TOKEN`.
18. Retornar `200 OK`.

### FLX-002 — Cookie ausente

1. Cookie `refresh_token` não está presente.
2. Nenhuma sessão é alterada.
3. Registrar `RENOVAR_TOKEN_FALHA`.
4. Retornar `400 TOKEN_REFRESH_AUSENTE`.

### FLX-003 — Refresh Token inválido ou já rotacionado

1. Hash não corresponde à sessão válida.
2. Não emitir novo token.
3. Registrar `RENOVAR_TOKEN_FALHA`.
4. Retornar `401 REFRESH_TOKEN_INVALIDO`.

### FLX-004 — Sessão expirada

1. Sessão atingiu `data_expiracao`.
2. Invalidar sessão, se necessário, com `EXPIRACAO_SESSAO`.
3. Expirar cookie.
4. Registrar falha.
5. Retornar `401 SESSAO_EXPIRADA`.

### FLX-005 — Limite de refresh atingido

1. `quantidade_refresh >= 3`.
2. Invalidar sessão com `REFRESH_LIMITE_ATINGIDO`.
3. Expirar cookie.
4. Registrar `RENOVAR_TOKEN_FALHA`.
5. Registrar `INVALIDAR_SESSAO`.
6. Retornar `401 SESSAO_EXPIRADA`.
7. Exigir novo login.

### FLX-006 — Sessão substituída por novo login

1. Refresh Token referencia sessão invalidada com `NOVO_LOGIN`.
2. Não emitir tokens.
3. Expirar cookie antigo.
4. Retornar `409 SESSAO_SUBSTITUIDA`.

### FLX-007 — Usuário bloqueado ou inativo

1. Sessão ainda pode existir fisicamente.
2. Backend detecta status atual do usuário.
3. Não renovar tokens.
4. Invalidar sessão, se ainda ativa, usando motivo correspondente.
5. Retornar `403 USUARIO_BLOQUEADO` ou `403 USUARIO_INATIVO`.

### FLX-008 — Duas renovações concorrentes

1. Duas requests usam o mesmo Refresh Token atual.
2. Somente uma pode substituir o hash persistido.
3. A primeira concluída rotaciona o token.
4. A segunda encontra token antigo inválido.
5. A segunda retorna `401 REFRESH_TOKEN_INVALIDO`.

---

## 6. Dados do Endpoint

| Campo | Valor |
|---|---|
| Método | POST |
| Path | `/api/v1/autenticacao/renovar-token` |
| Autenticação | Cookie `refresh_token` |
| Content-Type | Não requer body |
| Timeout | 30 segundos |
| Idempotency-Key | Não aplicável |
| X-Correlation-Id | Opcional na request; obrigatório na response |
| CSRF/origem | Validação obrigatória |
| Credentials CORS | Obrigatório |

---

## 7. Request

O endpoint não requer body.

Headers/cookie relevantes:

```text
Origin: https://frontend.exemplo
Cookie: refresh_token=<valor>
X-Correlation-Id: <uuid opcional>
```

Campos extras em body não são necessários; se body JSON for enviado com campos desconhecidos, deve seguir a política global de rejeição.

---

## 8. Response

### 8.1 Sucesso — 200

```json
{
  "accessToken": "<jwt>",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

`expiresIn` representa a validade efetiva em segundos e PODE ser menor que 900 quando o tempo restante da sessão absoluta for inferior a 15 minutos.

Headers:

```text
X-Correlation-Id: <uuid>
Set-Cookie: refresh_token=<novo-valor>; HttpOnly; SameSite=Lax; Path=/api/v1/autenticacao; Secure
```

O cookie anterior deixa de ser válido imediatamente.

---

## 9. Erros

| ID | HTTP | Código | Condição |
|---|---:|---|---|
| ERR-001 | 400 | `TOKEN_REFRESH_AUSENTE` | cookie não enviado |
| ERR-002 | 400 | `CORRELATION_ID_INVALIDO` | correlation id inválido |
| ERR-003 | 400 | `DADOS_INCONSISTENTES` | body/campos não permitidos |
| ERR-004 | 401 | `REFRESH_TOKEN_INVALIDO` | token não corresponde ao hash atual ou já foi rotacionado |
| ERR-005 | 401 | `REFRESH_TOKEN_EXPIRADO` | Refresh Token expirado quando distinguível da sessão |
| ERR-006 | 401 | `SESSAO_INVALIDA` | sessão invalidada por motivo diferente de novo login |
| ERR-007 | 401 | `SESSAO_EXPIRADA` | sessão absoluta expirada ou limite de refresh atingido |
| ERR-008 | 403 | `USUARIO_BLOQUEADO` | usuário bloqueado |
| ERR-009 | 403 | `USUARIO_INATIVO` | usuário inativo |
| ERR-010 | 409 | `SESSAO_SUBSTITUIDA` | sessão foi substituída por novo login |
| ERR-011 | 500 | `ERRO_INTERNO` | erro interno |
| ERR-012 | 503 | `SERVICO_INDISPONIVEL` | persistência/serviço essencial indisponível |

Todos os erros DEVEM usar o envelope global.

---

## 10. Segurança

- Refresh Token deve existir somente no cookie HttpOnly no cliente.
- Frontend não deve ler nem persistir Refresh Token.
- O backend deve comparar o token apresentado com o hash persistido de modo seguro.
- Rotação é obrigatória em toda renovação.
- Reutilização de token anterior deve falhar.
- Validar Origin porque o endpoint depende de cookie.
- `SameSite=Lax` é obrigatório no MVP.
- Se futuramente `SameSite=None` for utilizado, proteção CSRF explícita adicional será obrigatória.
- Access Token permanece apenas no body da response e deve ser mantido em memória pelo frontend.
- Nenhum token ou hash deve entrar em logs/auditoria.

---

## 11. Dependências

| Dependência | Uso | Falha |
|---|---|---|
| PostgreSQL | sessão, usuário, roles, permissões e auditoria | 503 |
| Gerador criptográfico seguro | novo Refresh Token | 500 |
| JWT RS256 | novo Access Token | 500/503 |
| Configuração CORS/origem | validação CSRF/origem | 403/erro de configuração |

---

## 12. Persistência

### `SESSAO_USUARIO`

Leitura:

- `id`;
- `usuario_id`;
- `refresh_token_hash`;
- `quantidade_refresh`;
- `data_criacao`;
- `data_expiracao`;
- `data_ultima_renovacao`;
- `data_invalidacao`;
- `motivo_invalidacao`;
- `ativa`.

Atualização em sucesso:

- `refresh_token_hash = <novo hash>`;
- `quantidade_refresh = quantidade_refresh + 1`;
- `data_ultima_renovacao = agora`.

Não alterar:

- `data_criacao`;
- `data_expiracao`.

### `USUARIO`

Leitura:

- `status`;
- identificação necessária às claims.

### Autorização

Roles e permissões vigentes devem ser lidas para compor o novo Access Token.

### `AUDIT_LOG`

Ações:

- `RENOVAR_TOKEN`;
- `RENOVAR_TOKEN_FALHA`;
- `INVALIDAR_SESSAO`, quando aplicável.

---

## 13. Idempotência e Concorrência

### 13.1 Idempotência

`Idempotency-Key` NÃO se aplica.

Uma renovação bem-sucedida consome logicamente o Refresh Token atual e gera outro. Repetir a mesma request com o token anterior DEVE falhar.

### 13.2 Concorrência

A rotação deve ser atômica.

A implementação DEVE garantir compare-and-swap lógico do hash atual, ou mecanismo transacional equivalente, para que somente uma request possa consumir um determinado Refresh Token.

Duas renovações concorrentes com o mesmo token:

- uma pode ter sucesso;
- as demais devem falhar com `REFRESH_TOKEN_INVALIDO`;
- `quantidade_refresh` deve incrementar apenas uma vez.

---

## 14. Eventos

Não há evento de domínio assíncrono obrigatório.

Eventos de segurança são registrados via auditoria.

---

## 15. Observabilidade e Auditoria

### 15.1 Logs

PODEM conter:

- correlationId;
- sessionId;
- usuarioId;
- duração;
- resultado;
- motivo genérico seguro;
- IP seguro;
- user-agent.

NÃO DEVEM conter:

- Refresh Token;
- hash do Refresh Token;
- Access Token;
- claims completas se contiverem informação desnecessária.

### 15.2 Auditoria

Sucesso:

```text
acao = RENOVAR_TOKEN
recursoTipo = SESSAO_USUARIO
recursoId = sessaoId
```

Falha:

```text
acao = RENOVAR_TOKEN_FALHA
recursoTipo = SESSAO_USUARIO
recursoId = sessaoId quando identificável
```

Invalidação:

```text
acao = INVALIDAR_SESSAO
recursoTipo = SESSAO_USUARIO
recursoId = sessaoId
detalhes.motivo = <motivo seguro>
```

---

## 16. Requisitos Não Funcionais

- **RNF-001:** timeout padrão de 30 segundos.
- **RNF-002:** rotação deve ser transacional/atômica.
- **RNF-003:** nenhum segredo pode vazar em logs, erros ou auditoria.
- **RNF-004:** response deve possuir `X-Correlation-Id`.
- **RNF-005:** novo Access Token deve usar RS256.
- **RNF-006:** novo cookie deve respeitar a duração restante da sessão.
- **RNF-007:** o endpoint deve ser seguro sob chamadas concorrentes.
- **RNF-008:** validação de origem deve ocorrer antes da rotação.

---

## 17. Critérios de Aceite

- **CA-001:** Refresh Token válido em sessão ativa gera novo Access Token e novo Refresh Token.
- **CA-002:** Refresh Token anterior deixa de ser aceito após sucesso.
- **CA-003:** `quantidade_refresh` incrementa exatamente uma vez por renovação bem-sucedida.
- **CA-004:** `data_expiracao` da sessão não é estendida.
- **CA-005:** Sessão permite no máximo 3 renovações.
- **CA-006:** Após atingir o limite, novo login é obrigatório.
- **CA-007:** Access Token emitido não ultrapassa a expiração absoluta da sessão.
- **CA-008:** Cookie ausente retorna `400 TOKEN_REFRESH_AUSENTE`.
- **CA-009:** Token inválido/rotacionado retorna `401 REFRESH_TOKEN_INVALIDO`.
- **CA-010:** Sessão expirada retorna `401 SESSAO_EXPIRADA`.
- **CA-011:** Sessão substituída por novo login retorna `409 SESSAO_SUBSTITUIDA`.
- **CA-012:** Usuário bloqueado não renova token.
- **CA-013:** Usuário inativo não renova token.
- **CA-014:** Origem não permitida não consegue renovar.
- **CA-015:** Nenhum token/hash aparece em logs ou auditoria.
- **CA-016:** Sucesso registra `RENOVAR_TOKEN`.
- **CA-017:** Falha registra `RENOVAR_TOKEN_FALHA`.
- **CA-018:** Limite de refresh registra invalidação com `REFRESH_LIMITE_ATINGIDO`.
- **CA-019:** Duas renovações concorrentes com o mesmo token resultam em no máximo um sucesso.
- **CA-020:** Correlation ID é retornado em todas as respostas.

---

## 18. Diretrizes obrigatórias para implementação por IA

1. Não aceitar Refresh Token em body, query string ou header customizado.
2. Não estender a duração absoluta da sessão.
3. Rotacionar Refresh Token em todo sucesso.
4. Não aceitar token anterior após rotação.
5. Não exceder 3 renovações.
6. Garantir operação atômica sob concorrência.
7. Não registrar tokens ou hashes.
8. Validar Origin/CORS antes de renovar.
9. Gerar novo Access Token com autorização vigente.
10. Gerar testes para todos os critérios de aceite.

---

## 19. Checklist de completude da especificação

- [x] Objetivo e escopo definidos.
- [x] Cookie e rotação definidos.
- [x] Sessão absoluta definida.
- [x] Limite de 3 renovações definido.
- [x] Claims do novo Access Token definidas.
- [x] CORS/CSRF definidos.
- [x] Request e response definidos.
- [x] Erros definidos.
- [x] Persistência definida.
- [x] Idempotência e concorrência definidas.
- [x] Auditoria definida.
- [x] Critérios de aceite definidos.
- [x] Casos de teste documentados em artefato próprio.
- [x] OpenAPI documentado em artefato próprio.
- [x] Markdown e DOCX devem permanecer equivalentes.

---

## 20. Pendências e Decisões em Aberto

Nenhuma pendência funcional conhecida para o contrato do `RES-106`.

A tecnologia de coordenação/locking usada para a rotação atômica é decisão de implementação, desde que preserve as regras definidas.

---

## 21. Histórico de alterações

| Versão | Data | Alteração |
|---|---|---|
| 1.1 | 2026-09-17 | Atualização das baselines documentais vigentes. |
| 1.0 | 2026-09-16 | Criação da especificação do RES-106. |
