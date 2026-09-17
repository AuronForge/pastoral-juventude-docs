# RES-002 — Alterar senha

**Versão:** 1.1  
**Status:** Em revisão  
**Baseline lógica:** MVP Domain Baseline v1.6  
**Base física:** Especificação Física do Banco v1.5  
**Catálogo:** Catálogo de Resources/APIs v1.3  
**Endpoint:** `POST /api/v1/autenticacao/alterar-senha`

---

## 1. Identificação do Resource

| Campo | Valor |
|---|---|
| Domínio / Bounded Context | Autenticação e Autorização |
| Capacidade | Alterar senha |
| Nome do Resource | RES-002 — Alterar senha |
| Ação | Alterar senha |
| Versão da especificação | 1.1 |
| Status | Em revisão |
| Última atualização | 2026-09-17 |

### 1.1 Objetivo

A capacidade permite que um usuário defina uma senha definitiva no primeiro acesso/recuperação ou altere voluntariamente a senha atual.

O Resource DEVE distinguir dois fluxos de autenticação: token restrito `TROCA_SENHA` e sessão normal. Em ambos os casos, a nova senha DEVE ser validada, persistida com Argon2id e causar invalidação das credenciais de sessão aplicáveis.

### 1.2 Escopo

Inclui:

- troca obrigatória de senha após primeiro acesso ou recuperação;
- troca voluntária de senha com sessão normal;
- validação de senha atual no fluxo voluntário;
- validação de nova senha e confirmação;
- proibição de reutilização da senha atual;
- persistência do novo hash;
- limpeza da senha temporária;
- invalidação de sessão/Refresh Token;
- auditoria de troca de senha e invalidação.

Não inclui:

- autenticação (`RES-001`);
- recuperação de senha (`RES-003`);
- renovação de token (`RES-106`);
- logout (`RES-107`);
- histórico de senhas além da senha atual.

---

## 2. Regras de Negócio

- **RN-001 — Modos de uso:** O endpoint DEVE aceitar autenticação por sessão normal ou token restrito `TROCA_SENHA`.
- **RN-002 — Troca obrigatória:** Com token `TROCA_SENHA`, o usuário DEVE informar `novaSenha` e `confirmacaoNovaSenha`; `senhaAtual` NÃO é exigida.
- **RN-003 — Troca voluntária:** Com sessão normal, o usuário DEVE informar `senhaAtual`, `novaSenha` e `confirmacaoNovaSenha`.
- **RN-004 — Senhas iguais:** `novaSenha` e `confirmacaoNovaSenha` DEVEM ser idênticas.
- **RN-005 — Comprimento:** A nova senha DEVE possuir de 8 a 128 caracteres.
- **RN-006 — Espaços:** A nova senha NÃO DEVE conter espaços.
- **RN-007 — Conteúdo permitido:** Letras, números e símbolos PODEM ser utilizados.
- **RN-008 — Senhas comuns:** O sistema PODE aceitar senhas comuns; não há blacklist obrigatória no MVP.
- **RN-009 — Reutilização:** A nova senha NÃO DEVE ser igual à senha definitiva atual, quando existir.
- **RN-010 — Hash:** A senha definitiva DEVE ser persistida usando Argon2id com salt individual e parâmetros configuráveis.
- **RN-011 — Segredos:** Senha atual, nova senha, confirmação e hashes NÃO DEVEM aparecer em logs, auditoria ou response.
- **RN-012 — Token restrito:** O token `TROCA_SENHA` somente PODE ser aceito neste endpoint.
- **RN-013 — Expiração do token restrito:** O token `TROCA_SENHA` DEVE expirar em 15 minutos.
- **RN-014 — Validação de estado para token restrito:** O backend DEVE confirmar que `troca_senha_obrigatoria = true` no momento da execução; caso contrário, o token restrito não pode ser reutilizado após uma troca já concluída.
- **RN-015 — Limpeza da senha temporária:** Após troca obrigatória bem-sucedida, `senha_temporaria_hash` e `senha_temporaria_expira_em` DEVEM ser limpos, `senha_temporaria_utilizada_em` DEVE ser registrada e `tentativas_senha_temporaria` DEVE ser zerado.
- **RN-016 — Fim da troca obrigatória:** Após sucesso, `troca_senha_obrigatoria = false`.
- **RN-017 — Sessão após troca obrigatória:** A troca obrigatória NÃO DEVE criar sessão normal; o usuário DEVE realizar novo login.
- **RN-018 — Sessão após troca voluntária:** A troca voluntária DEVE invalidar a sessão normal atual e o Refresh Token com motivo `ALTERACAO_SENHA`.
- **RN-019 — Novo login:** Após qualquer troca bem-sucedida, o usuário DEVE realizar novo login para obter sessão normal.
- **RN-020 — Usuário bloqueado em recuperação:** Usuário `BLOQUEADO` PODE concluir troca obrigatória originada por recuperação, mas DEVE permanecer bloqueado após a troca.
- **RN-021 — Usuário inativo:** Usuário `INATIVO` NÃO DEVE concluir alteração de senha.
- **RN-022 — Auditoria:** Toda alteração bem-sucedida DEVE registrar `TROCAR_SENHA`.
- **RN-023 — Invalidação auditada:** Quando houver sessão ativa invalidada, DEVE ser registrado `INVALIDAR_SESSAO`.
- **RN-024 — Correlação:** O endpoint DEVE utilizar `X-Correlation-Id`.
- **RN-025 — Campos desconhecidos:** Campos JSON desconhecidos DEVEM retornar `400 Bad Request`.
- **RN-026 — E-mail/roles/permissões:** A alteração de senha NÃO DEVE modificar e-mail, roles, permissões, função pastoral ou status do usuário.

---

## 3. Validações e Invariantes

- **RV-001:** `novaSenha` é obrigatória.
- **RV-002:** `confirmacaoNovaSenha` é obrigatória.
- **RV-003:** No fluxo voluntário, `senhaAtual` é obrigatória.
- **RV-004:** No fluxo obrigatório, `senhaAtual` é ignorada se ausente e NÃO DEVE ser necessária.
- **RV-005:** `novaSenha` e `confirmacaoNovaSenha` devem coincidir.
- **RV-006:** `novaSenha` deve conter 8–128 caracteres.
- **RV-007:** `novaSenha` não pode conter espaços.
- **RV-008:** `novaSenha` não pode ser igual à senha definitiva atual.
- **RV-009:** `Authorization` deve conter token válido para um dos dois fluxos.
- **RV-010:** token restrito só é válido quando `tipoToken = TROCA_SENHA`.
- **RV-011:** token restrito reutilizado após `troca_senha_obrigatoria=false` deve ser rejeitado.
- **RV-012:** sessão normal deve estar ativa no fluxo voluntário.
- **RV-013:** request body máximo de 1 MB.
- **RV-014:** `X-Correlation-Id`, quando enviado, deve ser UUID válido.
- **RV-015:** usuário inativo deve ser rejeitado.
- **RV-016:** no fluxo voluntário, `senhaAtual` deve corresponder ao hash definitivo atual.

---

## 4. Pré-condições e Pós-condições

### 4.1 Troca obrigatória — pré-condições

- token `TROCA_SENHA` válido e não expirado;
- `USUARIO.troca_senha_obrigatoria = true`;
- usuário não inativo;
- nova senha válida.

### 4.2 Troca obrigatória — pós-condições

- `senha_hash` preenchida com novo hash Argon2id;
- `troca_senha_obrigatoria = false`;
- `senha_temporaria_hash = null`;
- `senha_temporaria_expira_em = null`;
- `senha_temporaria_utilizada_em` preenchida;
- `tentativas_senha_temporaria = 0`;
- nenhuma sessão normal criada;
- token restrito deixa de ser funcional pelo estado persistido;
- auditoria `TROCAR_SENHA` registrada.

### 4.3 Troca voluntária — pré-condições

- sessão normal válida;
- usuário ativo;
- senha atual correta;
- nova senha válida.

### 4.4 Troca voluntária — pós-condições

- novo hash definitivo persistido;
- sessão atual invalidada com `ALTERACAO_SENHA`;
- Refresh Token correspondente invalidado;
- cookie `refresh_token` deve ser expirado/removido;
- Access Token atual deixa de ser aceito pela validação de `sessionId`;
- auditorias `TROCAR_SENHA` e `INVALIDAR_SESSAO` registradas;
- usuário precisa fazer novo login.

---

## 5. Fluxos

### FLX-001 — Troca obrigatória com sucesso

1. Receber request com token `TROCA_SENHA`.
2. Resolver `X-Correlation-Id`.
3. Validar token e expiração.
4. Carregar usuário.
5. Confirmar `troca_senha_obrigatoria = true`.
6. Validar status do usuário.
7. Validar nova senha e confirmação.
8. Validar que a nova senha não reutiliza a senha definitiva atual, se houver.
9. Gerar hash Argon2id.
10. Atualizar `USUARIO`.
11. Limpar dados da senha temporária.
12. Registrar `TROCAR_SENHA`.
13. Retornar `204 No Content`.
14. Usuário realiza novo login.

### FLX-002 — Troca voluntária com sucesso

1. Receber request com Access Token normal.
2. Validar sessão por `sessionId`.
3. Carregar usuário ativo.
4. Validar `senhaAtual`.
5. Validar nova senha e confirmação.
6. Validar não reutilização da senha atual.
7. Gerar novo hash Argon2id.
8. Persistir nova senha.
9. Invalidar sessão com `ALTERACAO_SENHA`.
10. Expirar cookie `refresh_token`.
11. Registrar `TROCAR_SENHA` e `INVALIDAR_SESSAO`.
12. Retornar `204 No Content`.
13. Usuário realiza novo login.

### FLX-003 — Token restrito expirado ou inválido

1. Rejeitar autenticação.
2. Não alterar senha.
3. Retornar `401 TOKEN_EXPIRADO` ou `401 TOKEN_INVALIDO`.

### FLX-004 — Token restrito reutilizado

1. Token JWT ainda pode estar criptograficamente válido.
2. Backend consulta `troca_senha_obrigatoria`.
3. Se estiver `false`, não permite nova alteração por esse token.
4. Retorna `401 TOKEN_INVALIDO`.

### FLX-005 — Senha atual inválida

1. Fluxo voluntário possui sessão válida.
2. `senhaAtual` não corresponde ao hash atual.
3. Nenhuma alteração é persistida.
4. Retorna `401 CREDENCIAIS_INVALIDAS`.

### FLX-006 — Usuário bloqueado com token de recuperação

1. Usuário bloqueado apresenta token `TROCA_SENHA` válido.
2. Troca obrigatória pode ser concluída.
3. `status` continua `BLOQUEADO`.
4. Nenhuma sessão normal é criada.

### FLX-007 — Usuário inativo

1. Detectar `INATIVO`.
2. Não alterar senha.
3. Retornar `403 USUARIO_INATIVO`.

---

## 6. Dados do Endpoint

| Campo | Valor |
|---|---|
| Método | POST |
| Path | `/api/v1/autenticacao/alterar-senha` |
| Autenticação | Bearer normal ou Bearer restrito `TROCA_SENHA` |
| Content-Type | `application/json` |
| Timeout | 30 segundos |
| Request body máximo | 1 MB |
| Idempotency-Key | Não aplicável |
| X-Correlation-Id | Opcional na request; obrigatório na response |

---

## 7. Request

### 7.1 Troca obrigatória

```json
{
  "novaSenha": "NovaSenha123!",
  "confirmacaoNovaSenha": "NovaSenha123!"
}
```

### 7.2 Troca voluntária

```json
{
  "senhaAtual": "SenhaAtual123!",
  "novaSenha": "NovaSenha456!",
  "confirmacaoNovaSenha": "NovaSenha456!"
}
```

| Campo | Tipo | Obrigatório | Regra |
|---|---|---:|---|
| `senhaAtual` | string | Condicional | obrigatória apenas no fluxo voluntário |
| `novaSenha` | string | Sim | 8–128, sem espaços |
| `confirmacaoNovaSenha` | string | Sim | deve ser igual a `novaSenha` |

---

## 8. Response

### 8.1 Sucesso

HTTP `204 No Content`.

Headers:

```text
X-Correlation-Id: <uuid>
```

No fluxo voluntário, o backend também DEVE expirar/remover o cookie `refresh_token`.

Não há body de sucesso.

---

## 9. Erros

| ID | HTTP | Código | Condição |
|---|---:|---|---|
| ERR-001 | 400 | `DADOS_INCONSISTENTES` | payload inválido, senhas divergentes, tamanho inválido, espaços ou campos desconhecidos |
| ERR-002 | 400 | `CORRELATION_ID_INVALIDO` | `X-Correlation-Id` inválido |
| ERR-003 | 401 | `TOKEN_INVALIDO` | token restrito inválido/reutilizado ou Access Token inválido |
| ERR-004 | 401 | `TOKEN_EXPIRADO` | token de autenticação expirado |
| ERR-005 | 401 | `CREDENCIAIS_INVALIDAS` | senha atual inválida no fluxo voluntário |
| ERR-006 | 401 | `SESSAO_INVALIDA` | sessão normal não está mais ativa |
| ERR-007 | 401 | `SESSAO_EXPIRADA` | sessão normal expirou |
| ERR-008 | 403 | `USUARIO_INATIVO` | usuário inativo |
| ERR-009 | 500 | `ERRO_INTERNO` | erro interno |
| ERR-010 | 503 | `SERVICO_INDISPONIVEL` | persistência/serviço essencial indisponível |
| ERR-011 | 413 | `PAYLOAD_MUITO_GRANDE` | request body excede o limite global de 1 MB |

A reutilização da senha atual DEVE ser reportada como `400 DADOS_INCONSISTENTES`, com erro de campo em `novaSenha`.

---

## 10. Segurança

- Nunca registrar senha atual, nova senha, confirmação ou hashes.
- Argon2id obrigatório para senha definitiva.
- Token `TROCA_SENHA` tem escopo estrito.
- Token restrito não pode ser usado para criar sessão.
- Reutilização do token restrito deve ser bloqueada pela checagem persistida de `troca_senha_obrigatoria`.
- Sessão normal deve ser validada por `sessionId` no fluxo voluntário.
- No sucesso voluntário, Access Token atual deve perder validade prática imediatamente por invalidação da sessão.
- CORS deve respeitar allow-list por ambiente.

---

## 11. Dependências

| Dependência | Uso | Falha |
|---|---|---|
| PostgreSQL | usuário, sessão e auditoria | 503 |
| Argon2id | verificação/geração de hash | 500 se indisponível por erro interno |
| Validador JWT RS256 | token normal/restrito | 401 para token inválido/expirado |

---

## 12. Persistência

### `USUARIO`

Atualizações possíveis:

- `senha_hash`;
- `troca_senha_obrigatoria`;
- `senha_temporaria_hash`;
- `senha_temporaria_expira_em`;
- `senha_temporaria_utilizada_em`;
- `tentativas_senha_temporaria`;
- `data_atualizacao`.

### `SESSAO_USUARIO`

No fluxo voluntário:

- `ativa = false`;
- `data_invalidacao` preenchida;
- `motivo_invalidacao = ALTERACAO_SENHA`.

### `AUDIT_LOG`

Ações:

- `TROCAR_SENHA`;
- `INVALIDAR_SESSAO` quando houver sessão invalidada.

---

## 13. Idempotência e Concorrência

### 13.1 Idempotência

`Idempotency-Key` NÃO se aplica.

Após sucesso:

- o token `TROCA_SENHA` deixa de ser aceito pelo estado `troca_senha_obrigatoria = false`;
- a sessão voluntária é invalidada.

Logo, repetir a mesma request não deve repetir a alteração com a mesma credencial de autenticação.

### 13.2 Concorrência

Duas trocas concorrentes com o mesmo token restrito NÃO DEVEM resultar em duas alterações independentes válidas.

A atualização deve ser transacional e condicional ao estado esperado (`troca_senha_obrigatoria = true`).

No fluxo voluntário, apenas uma request pode operar sobre a sessão ativa antes de sua invalidação.

---

## 14. Eventos

Não há evento de domínio assíncrono obrigatório.

Auditoria cobre os eventos de segurança relevantes.

---

## 15. Observabilidade e Auditoria

### 15.1 Logs

PODEM conter:

- correlationId;
- duração;
- resultado;
- usuarioId quando conhecido;
- IP seguro;
- user-agent.

NÃO PODEM conter:

- senha atual;
- nova senha;
- confirmação;
- hash;
- JWT;
- Refresh Token.

### 15.2 Auditoria

Sucesso:

```text
acao = TROCAR_SENHA
recursoTipo = USUARIO
recursoId = usuarioId
```

Se sessão normal for invalidada:

```text
acao = INVALIDAR_SESSAO
recursoTipo = SESSAO_USUARIO
recursoId = sessaoId
detalhes.motivo = ALTERACAO_SENHA
```

---

## 16. Requisitos Não Funcionais

- **RNF-001:** timeout padrão de 30 segundos.
- **RNF-002:** alteração deve ser transacional.
- **RNF-003:** não deve haver vazamento de segredos.
- **RNF-004:** response deve conter `X-Correlation-Id`.
- **RNF-005:** hash definitivo deve usar Argon2id.
- **RNF-006:** invalidação de sessão deve produzir efeito imediato.

---

## 17. Critérios de Aceite

- **CA-001:** Token `TROCA_SENHA` válido + nova senha válida resulta em `204`.
- **CA-002:** Após troca obrigatória, `troca_senha_obrigatoria=false` e dados temporários são limpos.
- **CA-003:** Após troca obrigatória, nenhum Access Token normal ou Refresh Token é emitido.
- **CA-004:** Token `TROCA_SENHA` reutilizado após sucesso é rejeitado.
- **CA-005:** Sessão normal + senha atual correta + nova senha válida resulta em `204`.
- **CA-006:** Após troca voluntária, sessão e Refresh Token são invalidados.
- **CA-007:** Senha atual incorreta retorna `401 CREDENCIAIS_INVALIDAS`.
- **CA-008:** Nova senha diferente da confirmação retorna `400 DADOS_INCONSISTENTES`.
- **CA-009:** Nova senha com menos de 8, mais de 128 ou com espaço retorna `400`.
- **CA-010:** Nova senha igual à atual é rejeitada.
- **CA-011:** Usuário bloqueado com token restrito válido pode concluir a troca, mas permanece bloqueado.
- **CA-012:** Usuário inativo não consegue alterar senha.
- **CA-013:** Nenhuma senha/hash/token é gravada em logs ou auditoria.
- **CA-014:** Auditoria `TROCAR_SENHA` é registrada no sucesso.
- **CA-015:** No fluxo voluntário, auditoria `INVALIDAR_SESSAO` é registrada.
- **CA-016:** Correlation ID é retornado em todas as respostas.

---

## 18. Diretrizes obrigatórias para implementação por IA

1. Não inventar requisitos além desta especificação.
2. Preservar path, códigos e comportamento.
3. Não criar sessão normal após troca obrigatória.
4. Não desbloquear usuário bloqueado após recuperação.
5. Não permitir alteração para usuário inativo.
6. Não registrar credenciais.
7. Fazer atualização transacional.
8. Rejeitar reutilização de token restrito pelo estado persistido.
9. Gerar testes para todos os critérios de aceite.
10. Manter rastreabilidade entre regras, fluxos, erros e testes.

---

## 19. Checklist de completude da especificação

- [x] Objetivo e escopo definidos.
- [x] Dois fluxos de alteração definidos.
- [x] Política de senha definida.
- [x] Request e response definidos.
- [x] Erros definidos.
- [x] Persistência definida.
- [x] Invalidação de sessão definida.
- [x] Auditoria definida.
- [x] Idempotência e concorrência definidas.
- [x] Critérios de aceite definidos.
- [x] Casos de teste documentados em artefato próprio.
- [x] OpenAPI documentado em artefato próprio.
- [x] Markdown e DOCX devem permanecer equivalentes.

---

## 20. Pendências e Decisões em Aberto

Nenhuma pendência funcional conhecida para o contrato do `RES-002`.

---

## 21. Histórico de alterações

| Versão | Data | Alteração |
|---|---|---|
| 1.1 | 2026-09-17 | Atualização das baselines e inclusão explícita do erro 413 para o limite global de payload. |
| 1.0 | 2026-09-15 | Criação da especificação do RES-002. |
