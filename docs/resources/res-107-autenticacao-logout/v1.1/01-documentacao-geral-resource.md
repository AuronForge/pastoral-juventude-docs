# RES-107 — Logout

**Versão:** 1.1  
**Status:** Em revisão  
**Baseline lógica:** MVP Domain Baseline v1.6  
**Base física:** Especificação Física do Banco v1.5  
**Catálogo:** Catálogo de Resources/APIs v1.3  
**Endpoint:** `POST /api/v1/autenticacao/logout`

---

## 1. Identificação do Resource

| Campo | Valor |
|---|---|
| Domínio / Bounded Context | Autenticação e Autorização |
| Capacidade | Encerrar sessão |
| Nome do Resource | RES-107 — Logout |
| Ação | Logout |
| Versão da especificação | 1.1 |
| Status | Em revisão |
| Última atualização | 2026-09-17 |

### 1.1 Objetivo

A capacidade permite encerrar explicitamente a sessão normal atual do usuário.

O Resource DEVE invalidar a `SESSAO_USUARIO` ativa com motivo `LOGOUT`, tornar o Refresh Token inválido, expirar/remover o cookie `refresh_token` e fazer com que o Access Token da sessão deixe de ser aceito imediatamente pela validação de `sessionId`.

### 1.2 Escopo

Inclui:

- logout explícito da sessão normal;
- validação de sessão quando identificável;
- invalidação de `SESSAO_USUARIO`;
- invalidação do Refresh Token;
- expiração/remoção do cookie `refresh_token`;
- rejeição imediata do Access Token da sessão invalidada;
- comportamento idempotente para chamadas repetidas;
- validação de origem por depender de cookie;
- auditoria `LOGOUT` e `INVALIDAR_SESSAO`;
- `X-Correlation-Id`.

Não inclui:

- logout global de todas as sessões, pois o MVP já limita a uma sessão ativa por usuário;
- login;
- renovação de token;
- alteração de senha;
- recuperação de senha;
- exclusão física da sessão histórica.

---

## 2. Regras de Negócio

- **RN-001 — Endpoint:** O logout DEVE utilizar `POST /api/v1/autenticacao/logout`.
- **RN-002 — Sessão atual:** Quando existir sessão ativa identificável, ela DEVE ser invalidada.
- **RN-003 — Motivo:** A invalidação causada por logout DEVE utilizar `motivo_invalidacao = LOGOUT`.
- **RN-004 — Histórico:** A sessão invalidada NÃO DEVE ser excluída fisicamente.
- **RN-005 — Refresh Token:** O Refresh Token da sessão DEVE deixar de ser aceito após o logout.
- **RN-006 — Cookie:** O cookie `refresh_token` DEVE ser expirado/removido na response.
- **RN-007 — Access Token:** O Access Token atual DEVE deixar de ser aceito imediatamente após a invalidação da sessão pela validação de `sessionId`.
- **RN-008 — Idempotência funcional:** Repetir logout após a sessão já estar invalidada NÃO DEVE gerar erro funcional.
- **RN-009 — Sem reativação:** Logout repetido NÃO DEVE reativar sessão nem produzir novo token.
- **RN-010 — Resultado:** Logout bem-sucedido ou repetido DEVE retornar `204 No Content`.
- **RN-011 — Cookie ausente:** A ausência do cookie `refresh_token` NÃO DEVE impedir o logout idempotente quando a operação puder ser tratada como já concluída.
- **RN-012 — Token ausente:** A ausência de Access Token NÃO DEVE tornar o logout repetido funcionalmente inválido quando não houver sessão ativa a encerrar.
- **RN-013 — Sessão substituída:** Se a sessão anterior já tiver sido invalidada por `NOVO_LOGIN`, o logout deve ser tratado como já concluído e retornar `204`.
- **RN-014 — Sessão expirada:** Sessão já expirada deve ser tratada como já encerrada para fins de logout idempotente.
- **RN-015 — Usuário bloqueado/inativo:** Se a sessão ainda estiver identificável, o logout DEVE ser permitido independentemente do status atual do usuário.
- **RN-016 — Auditoria de logout:** Quando uma sessão ativa for efetivamente encerrada, o sistema DEVE registrar `LOGOUT`.
- **RN-017 — Auditoria de invalidação:** A invalidação efetiva da sessão DEVE registrar `INVALIDAR_SESSAO`.
- **RN-018 — Repetição idempotente:** Logout repetido sem nova invalidação NÃO DEVE criar múltiplos registros de invalidação para a mesma transição de estado.
- **RN-019 — Segredos:** Access Token, Refresh Token e hashes NÃO DEVEM aparecer em logs ou auditoria.
- **RN-020 — Origem:** O endpoint DEVE validar Origin/CORS porque expira cookie autenticacional.
- **RN-021 — Cookie attributes:** A remoção do cookie DEVE usar `Path=/api/v1/autenticacao` e atributos compatíveis com o cookie original.
- **RN-022 — Secure:** Em produção, a resposta de expiração DEVE respeitar `Secure=true`.
- **RN-023 — SameSite:** O cookie deve manter `SameSite=Lax`.
- **RN-024 — Correlação:** O endpoint DEVE utilizar `X-Correlation-Id`.
- **RN-025 — Body:** O endpoint NÃO DEVE exigir body.
- **RN-026 — Idempotency-Key:** `Idempotency-Key` NÃO se aplica.
- **RN-027 — Concorrência:** Duas chamadas concorrentes de logout para a mesma sessão DEVEM convergir para uma sessão invalidada, sem erro funcional.
- **RN-028 — Transação:** A mudança de estado da sessão e o registro de auditoria aplicável DEVEM ser consistentes.

---

## 3. Validações e Invariantes

- **RV-001:** `X-Correlation-Id`, quando enviado, deve ser UUID válido.
- **RV-002:** Origin deve respeitar allow-list do ambiente quando aplicável ao contexto web.
- **RV-003:** nenhuma sessão invalidada pode voltar para `ativa=true`.
- **RV-004:** uma sessão invalidada por logout deve ter `data_invalidacao` preenchida.
- **RV-005:** `motivo_invalidacao` deve ser `LOGOUT` somente quando o logout efetivamente causar a transição ativa → inativa.
- **RV-006:** o endpoint não requer request body.
- **RV-007:** nenhum token deve ser persistido ou registrado em texto puro.

---

## 4. Pré-condições e Pós-condições

### 4.1 Pré-condições

Não há pré-condição funcional obrigatória além de uma request válida.

Quando houver sessão normal válida, ela deve ser identificada pelo contexto de autenticação e/ou cookie correspondente.

### 4.2 Pós-condições — sessão ativa

- `SESSAO_USUARIO.ativa = false`;
- `data_invalidacao` preenchida;
- `motivo_invalidacao = LOGOUT`;
- Refresh Token anterior deixa de ser válido;
- cookie `refresh_token` expirado;
- Access Token associado ao `sessionId` passa a ser rejeitado;
- auditoria `LOGOUT` registrada;
- auditoria `INVALIDAR_SESSAO` registrada;
- response `204`.

### 4.3 Pós-condições — logout repetido

- estado permanece sem sessão ativa;
- cookie é expirado novamente de forma segura;
- nenhuma nova sessão é criada;
- response continua `204`;
- não há erro funcional por sessão já encerrada.

---

## 5. Fluxos

### FLX-001 — Logout com sessão ativa

1. Receber request.
2. Resolver `X-Correlation-Id`.
3. Validar origem.
4. Identificar sessão atual.
5. Confirmar que está ativa.
6. Atualizar sessão para inativa.
7. Definir `data_invalidacao`.
8. Definir `motivo_invalidacao = LOGOUT`.
9. Registrar `LOGOUT`.
10. Registrar `INVALIDAR_SESSAO`.
11. Expirar/remover cookie `refresh_token`.
12. Retornar `204 No Content`.

### FLX-002 — Logout repetido

1. Request chega sem sessão ativa utilizável.
2. Backend não cria erro funcional.
3. Expira cookie `refresh_token`, caso exista no navegador.
4. Não altera sessão histórica já invalidada.
5. Retorna `204 No Content`.

### FLX-003 — Sessão substituída anteriormente

1. Sessão antiga foi invalidada por `NOVO_LOGIN`.
2. Cliente antigo chama logout.
3. Backend trata a sessão como já encerrada.
4. Expira cookie antigo.
5. Retorna `204`.

### FLX-004 — Sessão expirada anteriormente

1. Sessão já expirou.
2. Cliente chama logout.
3. Backend trata logout como idempotente.
4. Expira cookie.
5. Retorna `204`.

### FLX-005 — Duas chamadas concorrentes

1. Duas requests de logout chegam simultaneamente.
2. Uma realiza a transição ativa → inativa.
3. A outra encontra a sessão já invalidada.
4. Ambas retornam `204`.
5. Apenas uma transição de invalidação é persistida.

---

## 6. Dados do Endpoint

| Campo | Valor |
|---|---|
| Método | POST |
| Path | `/api/v1/autenticacao/logout` |
| Autenticação | Sessão normal quando disponível; comportamento idempotente sem sessão ativa |
| Content-Type | Não requer body |
| Timeout | 30 segundos |
| Idempotency-Key | Não aplicável |
| X-Correlation-Id | Opcional na request; obrigatório na response |
| CSRF/origem | Validação obrigatória no contexto web com cookie |
| Response | `204 No Content` |

---

## 7. Request

O endpoint não exige body.

Exemplo:

```http
POST /api/v1/autenticacao/logout
Authorization: Bearer <access-token>
Cookie: refresh_token=<valor>
Origin: https://frontend.exemplo
X-Correlation-Id: <uuid opcional>
```

Em chamadas repetidas, `Authorization` e/ou cookie podem já não representar uma sessão válida; o comportamento continua idempotente.

---

## 8. Response

### 8.1 Sucesso / logout repetido

HTTP `204 No Content`.

Headers:

```text
X-Correlation-Id: <uuid>
Set-Cookie: refresh_token=; Max-Age=0; Path=/api/v1/autenticacao; HttpOnly; SameSite=Lax; Secure
```

`Secure` é obrigatório em produção.

Não há body de sucesso.

---

## 9. Erros

| ID | HTTP | Código | Condição |
|---|---:|---|---|
| ERR-001 | 400 | `CORRELATION_ID_INVALIDO` | correlation id inválido |
| ERR-002 | 400 | `DADOS_INCONSISTENTES` | body/campos não permitidos quando aplicável |
| ERR-003 | 403 | `ACESSO_NEGADO` | origem rejeitada pela política de segurança |
| ERR-004 | 500 | `ERRO_INTERNO` | erro interno inesperado |
| ERR-005 | 503 | `SERVICO_INDISPONIVEL` | persistência indisponível quando necessária à invalidação |

Erros de sessão ausente, inválida, expirada ou substituída NÃO DEVEM ser retornados no logout idempotente quando a operação puder ser considerada já concluída.

---

## 10. Segurança

- O endpoint deve validar origem no contexto web.
- O cookie deve ser expirado com os mesmos atributos relevantes usados na criação.
- Nenhum token deve aparecer em logs ou auditoria.
- A sessão deve permanecer historicamente registrada.
- Access Token deve perder validade prática imediatamente via `sessionId`.
- Não há geração de novos tokens.
- Logout deve ser permitido para usuário bloqueado ou inativo quando a sessão ainda for identificável.

---

## 11. Dependências

| Dependência | Uso | Falha |
|---|---|---|
| PostgreSQL | sessão e auditoria | 503 |
| Configuração CORS/origem | proteção da operação baseada em cookie | 403/erro de configuração |

---

## 12. Persistência

### `SESSAO_USUARIO`

Quando ativa:

- `ativa = false`;
- `data_invalidacao = agora`;
- `motivo_invalidacao = LOGOUT`.

Não excluir o registro.

Quando já inativa:

- não alterar motivo histórico existente;
- não sobrescrever `NOVO_LOGIN`, `ALTERACAO_SENHA`, `RECUPERACAO_SENHA`, `EXPIRACAO_SESSAO` ou outro motivo prévio apenas porque houve uma chamada de logout posterior.

### `AUDIT_LOG`

Quando logout efetivamente encerra sessão ativa:

- `LOGOUT`;
- `INVALIDAR_SESSAO`.

Logout repetido não exige novo evento de invalidação para a mesma transição já concluída.

---

## 13. Idempotência e Concorrência

### 13.1 Idempotência

O logout é funcionalmente idempotente.

Chamadas repetidas:

- retornam `204`;
- não recriam sessão;
- não alteram novamente a sessão histórica;
- podem repetir a expiração do cookie sem efeito adverso.

`Idempotency-Key` não é utilizado.

### 13.2 Concorrência

A transição ativa → inativa deve ser atômica ou condicional.

Duas chamadas concorrentes:

- no máximo uma realiza a transição;
- as demais observam sessão já inativa;
- todas podem retornar `204`;
- auditoria de invalidação não deve ser duplicada para a mesma transição.

---

## 14. Eventos

Não há evento de domínio assíncrono obrigatório.

A operação gera eventos de segurança via auditoria quando houver transição efetiva.

---

## 15. Observabilidade e Auditoria

### 15.1 Logs

PODEM conter:

- correlationId;
- usuarioId/sessionId quando conhecidos;
- duração;
- resultado;
- origem/IP seguro;
- user-agent.

NÃO DEVEM conter:

- Access Token;
- Refresh Token;
- hash do Refresh Token.

### 15.2 Auditoria

Sessão efetivamente encerrada:

```text
acao = LOGOUT
recursoTipo = SESSAO_USUARIO
recursoId = sessaoId
```

Invalidação:

```text
acao = INVALIDAR_SESSAO
recursoTipo = SESSAO_USUARIO
recursoId = sessaoId
detalhes.motivo = LOGOUT
```

Logout idempotente sem nova transição NÃO DEVE sobrescrever ou duplicar o motivo histórico da sessão.

---

## 16. Requisitos Não Funcionais

- **RNF-001:** timeout padrão de 30 segundos.
- **RNF-002:** invalidação de sessão deve produzir efeito imediato.
- **RNF-003:** endpoint deve ser seguro sob chamadas concorrentes.
- **RNF-004:** nenhum token/hash deve vazar.
- **RNF-005:** response deve possuir `X-Correlation-Id`.
- **RNF-006:** expiração do cookie deve ser compatível com os atributos do cookie original.
- **RNF-007:** comportamento idempotente deve ser preservado.

---

## 17. Critérios de Aceite

- **CA-001:** Sessão ativa é invalidada com motivo `LOGOUT`.
- **CA-002:** Refresh Token anterior deixa de ser aceito após logout.
- **CA-003:** Cookie `refresh_token` é expirado/removido.
- **CA-004:** Access Token da sessão invalidada deixa de ser aceito imediatamente.
- **CA-005:** Logout retorna `204 No Content`.
- **CA-006:** Segunda chamada de logout também retorna `204`.
- **CA-007:** Logout repetido não sobrescreve motivo histórico anterior de uma sessão já invalidada.
- **CA-008:** Usuário bloqueado pode encerrar sessão.
- **CA-009:** Usuário inativo pode encerrar sessão quando a sessão ainda for identificável.
- **CA-010:** Sessão substituída anteriormente é tratada como logout já concluído.
- **CA-011:** Sessão expirada anteriormente é tratada como logout já concluído.
- **CA-012:** Sessão ativa encerrada registra `LOGOUT`.
- **CA-013:** Invalidação efetiva registra `INVALIDAR_SESSAO`.
- **CA-014:** Logout repetido não duplica a transição de invalidação.
- **CA-015:** Nenhum token/hash aparece em logs ou auditoria.
- **CA-016:** Duas chamadas concorrentes convergem para uma única sessão inativa e ambas podem retornar `204`.
- **CA-017:** Correlation ID é retornado em todas as respostas.
- **CA-018:** Origem não autorizada é rejeitada antes da alteração de estado.

---

## 18. Diretrizes obrigatórias para implementação por IA

1. Não transformar logout em DELETE de sessão.
2. Não excluir historicamente `SESSAO_USUARIO`.
3. Preservar idempotência funcional.
4. Não retornar erro apenas porque a sessão já está encerrada.
5. Não sobrescrever motivo de invalidação histórico existente em logout repetido.
6. Expirar o cookie em toda resposta 204 de logout.
7. Não registrar tokens/hashes.
8. Garantir concorrência segura.
9. Validar origem antes de alterar a sessão no contexto web.
10. Gerar testes para todos os critérios de aceite.

---

## 19. Checklist de completude da especificação

- [x] Objetivo e escopo definidos.
- [x] Invalidação de sessão definida.
- [x] Cookie definido.
- [x] Access Token pós-logout definido.
- [x] Idempotência definida.
- [x] Concorrência definida.
- [x] Request/response definidos.
- [x] Erros definidos.
- [x] Persistência definida.
- [x] Auditoria definida.
- [x] Critérios de aceite definidos.
- [x] Casos de teste documentados em artefato próprio.
- [x] OpenAPI documentado em artefato próprio.
- [x] Markdown e DOCX devem permanecer equivalentes.

---

## 20. Pendências e Decisões em Aberto

Nenhuma pendência funcional conhecida para o contrato do `RES-107`.

---

## 21. Histórico de alterações

| Versão | Data | Alteração |
|---|---|---|
| 1.1 | 2026-09-17 | Atualização das baselines documentais vigentes. |
| 1.0 | 2026-09-16 | Criação da especificação do RES-107. |
