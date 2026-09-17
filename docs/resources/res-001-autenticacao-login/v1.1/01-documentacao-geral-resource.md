# RES-001 — Autenticar usuário

**Versão:** 1.1  
**Status:** Em revisão  
**Baseline lógica:** MVP Domain Baseline v1.6  
**Base física:** Especificação Física do Banco v1.5  
**Catálogo:** Catálogo de Resources/APIs v1.3  
**Endpoint:** `POST /api/v1/autenticacao/login`

---

## 1. Identificação do Resource

| Campo | Valor |
|---|---|
| Domínio / Bounded Context | Autenticação e Autorização |
| Capacidade | Autenticar usuário |
| Nome do Resource | RES-001 — Autenticar usuário |
| Ação | Autenticar |
| Versão da especificação | 1.1 |
| Status | Em revisão |
| Última atualização | 2026-09-17 |

### 1.1 Objetivo

A capacidade de autenticação permite que uma pessoa com `USUARIO` válido informe e-mail e senha para obter acesso ao sistema.

O Resource DEVE autenticar credenciais, respeitar estado do usuário, rate limit, sessão única, primeiro acesso, senha temporária e auditoria de segurança. Em autenticação normal, o sistema DEVE criar uma nova `SESSAO_USUARIO`, emitir Access Token JWT e enviar Refresh Token em cookie seguro. Em primeiro acesso ou recuperação, quando a senha temporária for válida e `troca_senha_obrigatoria = true`, o sistema DEVE emitir somente token restrito de troca de senha.

### 1.2 Escopo

Inclui:

- autenticação por e-mail + senha;
- normalização de e-mail;
- validação de usuário ativo/bloqueado/inativo;
- rate limit de login;
- criação e invalidação de sessão;
- emissão de Access Token normal;
- geração e rotação inicial de Refresh Token;
- emissão de token `TROCA_SENHA`;
- contabilização de tentativas inválidas de senha temporária;
- auditoria de sucesso e falha.

Não inclui:

- renovação de Access Token (`RES-106`);
- logout (`RES-107`);
- recuperação de senha (`RES-003`);
- alteração da senha (`RES-002`);
- criação de usuário.

---

## 2. Regras de Negócio

- **RN-001 — Identificador de login:** O sistema DEVE autenticar utilizando `email + senha`.
- **RN-002 — Normalização do e-mail:** O e-mail DEVE ser normalizado com `trim` e `lowercase` antes da busca.
- **RN-003 — Resposta genérica de credencial inválida:** O sistema NÃO DEVE informar se a falha ocorreu por e-mail inexistente ou senha incorreta.
- **RN-004 — Sessão única:** Um usuário DEVE possuir no máximo uma sessão ativa.
- **RN-005 — Novo login:** Um login normal bem-sucedido DEVE invalidar qualquer sessão ativa anterior com motivo `NOVO_LOGIN`.
- **RN-006 — Access Token:** O Access Token normal DEVE expirar em 15 minutos.
- **RN-007 — Sessão absoluta:** A sessão normal DEVE possuir duração absoluta máxima de 1 hora.
- **RN-008 — Refresh:** A sessão normal DEVE permitir no máximo 3 renovações posteriores, tratadas por `RES-106`.
- **RN-009 — Refresh Token:** O Refresh Token DEVE ser armazenado apenas como hash em `SESSAO_USUARIO`.
- **RN-010 — Cookie de Refresh Token:** O Refresh Token DEVE ser enviado em cookie `refresh_token`, `HttpOnly`, `SameSite=Lax`, `Path=/api/v1/autenticacao` e `Secure=true` em produção.
- **RN-011 — JWT:** O Access Token normal DEVE ser assinado com RS256.
- **RN-012 — Claims obrigatórias:** O JWT normal DEVE conter `sub`, `usuarioId`, `email`, `sessionId`, `roles`, `permissoes`, `iat`, `exp`, `iss`, `aud` e `jti`.
- **RN-013 — Primeiro acesso / recuperação:** Se `troca_senha_obrigatoria = true` e a senha temporária informada for válida, o sistema NÃO DEVE criar sessão normal.
- **RN-014 — Token restrito:** No cenário da RN-013, o sistema DEVE emitir token `TROCA_SENHA` válido por 15 minutos.
- **RN-015 — Escopo do token restrito:** O token `TROCA_SENHA` NÃO DEVE conter `roles`, `permissoes` ou `sessionId` de sessão normal e somente PODE ser usado em `POST /api/v1/autenticacao/alterar-senha`.
- **RN-016 — Usuário bloqueado:** Após validar a credencial informada, usuário `BLOQUEADO` NÃO DEVE receber sessão normal e DEVE receber `403 USUARIO_BLOQUEADO`. Credencial inválida continua retornando somente `401 CREDENCIAIS_INVALIDAS`.
- **RN-017 — Usuário inativo:** Após validar a credencial informada, usuário `INATIVO` NÃO DEVE receber sessão normal e DEVE receber `403 USUARIO_INATIVO`. Credencial inválida continua retornando somente `401 CREDENCIAIS_INVALIDAS`.
- **RN-018 — Rate limit:** O sistema DEVE permitir no máximo 5 tentativas inválidas de login em uma janela de 15 minutos por usuário/login; ao atingir o limite, DEVE aplicar bloqueio temporário de 15 minutos.
- **RN-019 — Proteção complementar:** O sistema DEVE aplicar proteção complementar por IP/origem sem substituir o controle por usuário/login.
- **RN-020 — Contador de login:** Login normal válido DEVE zerar o contador de tentativas inválidas de login.
- **RN-021 — Senha temporária:** Senha temporária DEVE possuir validade de 15 minutos e ser de uso único.
- **RN-022 — Tentativas de senha temporária:** Após 5 tentativas inválidas de senha temporária, ela DEVE ser invalidada e uma nova recuperação passa a ser necessária.
- **RN-023 — Segredos:** Senhas, hashes, JWTs e Refresh Tokens NÃO DEVEM ser registrados em logs ou auditoria.
- **RN-024 — Correlação:** A operação DEVE possuir `X-Correlation-Id`, reutilizando UUID válido recebido ou gerando um novo.
- **RN-025 — Auditoria:** Login bem-sucedido DEVE registrar ação `LOGIN`; falha DEVE registrar `LOGIN_FALHA`.
- **RN-026 — Alteração de último login:** Em login normal bem-sucedido, `USUARIO.ultimo_login` DEVE ser atualizado.
- **RN-027 — CORS:** O endpoint DEVE aceitar apenas origens configuradas por ambiente e NÃO DEVE utilizar wildcard com credenciais.
- **RN-028 — Campos desconhecidos:** Campos JSON desconhecidos DEVEM causar `400 Bad Request`.

---

## 3. Validações e Invariantes

- **RV-001:** `email` é obrigatório.
- **RV-002:** `senha` é obrigatória.
- **RV-003:** `email` deve ser string e respeitar a validação de e-mail da aplicação.
- **RV-004:** `senha` deve ser string.
- **RV-005:** request body máximo de 1 MB.
- **RV-006:** `X-Correlation-Id`, quando enviado, deve ser UUID válido.
- **RV-007:** `email` normalizado não deve ser vazio.
- **RV-008:** autenticação normal só é permitida para `USUARIO.status = ATIVO`.
- **RV-009:** token restrito só pode ser emitido quando `troca_senha_obrigatoria = true` e a senha temporária estiver válida.
- **RV-010:** senha temporária expirada não pode ser autenticada.
- **RV-011:** senha temporária invalidada por excesso de tentativas não pode ser autenticada.

---

## 4. Pré-condições e Pós-condições

### 4.1 Pré-condições

- usuário cadastrado para o e-mail informado, quando existir;
- roles/permissões efetivas disponíveis para geração do JWT normal;
- chave privada RS256 disponível;
- configuração de `iss` e `aud` disponível;
- banco disponível para consulta de usuário e criação/invalidação de sessão.

### 4.2 Pós-condições — Login normal

- eventual sessão anterior invalidada com `NOVO_LOGIN`;
- nova `SESSAO_USUARIO` ativa persistida;
- `quantidade_refresh = 0`;
- Refresh Token armazenado apenas como hash;
- Access Token emitido;
- cookie `refresh_token` enviado;
- `ultimo_login` atualizado;
- auditoria `LOGIN` registrada.

### 4.3 Pós-condições — Troca obrigatória

- nenhuma sessão normal criada;
- nenhum Refresh Token de sessão normal emitido;
- token `TROCA_SENHA` emitido;
- auditoria `LOGIN` registrada sem segredos.

---

## 5. Fluxos

### FLX-001 — Login normal com sucesso

1. Receber request.
2. Resolver `X-Correlation-Id`.
3. Validar payload.
4. Normalizar e-mail.
5. Aplicar rate limit.
6. Localizar usuário pelo e-mail normalizado.
7. Validar senha definitiva.
8. Validar `status = ATIVO`.
9. Invalidar sessão ativa anterior, se houver.
10. Criar `SESSAO_USUARIO`.
11. Gerar Refresh Token e persistir apenas seu hash.
12. Emitir Access Token RS256.
13. Atualizar `ultimo_login`.
14. Registrar `LOGIN`.
15. Retornar `200 OK` e cookie `refresh_token`.

### FLX-002 — Credenciais inválidas

1. Não revelar se e-mail existe.
2. Contabilizar tentativa inválida.
3. Registrar `LOGIN_FALHA`.
4. Retornar `401 CREDENCIAIS_INVALIDAS`.

### FLX-003 — Senha temporária válida

1. Localizar usuário.
2. Confirmar `troca_senha_obrigatoria = true`.
3. Validar hash e expiração da senha temporária.
4. Não criar sessão normal.
5. Emitir token `TROCA_SENHA` por 15 minutos.
6. Registrar `LOGIN`.
7. Retornar `200 OK`.

### FLX-004 — Senha temporária expirada

1. Identificar que a senha temporária expirou.
2. Não criar sessão.
3. Registrar `LOGIN_FALHA`.
4. Retornar `401 SENHA_TEMPORARIA_EXPIRADA`.

### FLX-005 — Usuário bloqueado

1. Validar credencial sem expor segredos.
2. Detectar `status = BLOQUEADO`.
3. Não criar sessão.
4. Registrar `LOGIN_FALHA`.
5. Retornar `403 USUARIO_BLOQUEADO`.

### FLX-006 — Usuário inativo

1. Validar a credencial sem expor segredos.
2. Detectar `status = INATIVO` somente após a credencial ser confirmada.
3. Não criar sessão.
4. Registrar `LOGIN_FALHA`.
5. Retornar `403 USUARIO_INATIVO`.

### FLX-007 — Rate limit excedido

1. Detectar limite atingido.
2. Não validar nova credencial durante a janela de bloqueio.
3. Registrar evento de segurança.
4. Retornar `429 LIMITE_TENTATIVAS_EXCEDIDO`.

---

## 6. Dados do Endpoint

| Campo | Valor |
|---|---|
| Método | POST |
| Path | `/api/v1/autenticacao/login` |
| Autenticação | Pública |
| Content-Type | `application/json` |
| Timeout | 30 segundos |
| Request body máximo | 1 MB |
| Idempotency-Key | Não aplicável |
| X-Correlation-Id | Opcional na request; obrigatório na response |
| Cookie de sucesso normal | `refresh_token` |

---

## 7. Request

```json
{
  "email": "usuario@exemplo.com",
  "senha": "Senha123!"
}
```

| Campo | Tipo | Obrigatório | Regra |
|---|---|---:|---|
| `email` | string | Sim | normalizar com trim + lowercase |
| `senha` | string | Sim | não registrar em log/auditoria |

Campos adicionais não previstos NÃO DEVEM ser aceitos.

---

## 8. Response

### 8.1 Login normal — 200

```json
{
  "accessToken": "<jwt>",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

Headers:

```text
X-Correlation-Id: <uuid>
Set-Cookie: refresh_token=<valor>; HttpOnly; SameSite=Lax; Path=/api/v1/autenticacao; Secure
```

`Secure` é obrigatório em produção e pode ser desabilitado apenas em desenvolvimento local via HTTP.

### 8.2 Troca obrigatória — 200

```json
{
  "tokenTrocaSenha": "<jwt-restrito>",
  "tokenType": "TROCA_SENHA",
  "expiresIn": 900,
  "trocaSenhaObrigatoria": true
}
```

Neste cenário NÃO DEVE haver cookie `refresh_token`.

---

## 9. Erros

| ID | HTTP | Código | Condição |
|---|---:|---|---|
| ERR-001 | 400 | `DADOS_INCONSISTENTES` | payload inválido ou campos desconhecidos |
| ERR-002 | 400 | `CORRELATION_ID_INVALIDO` | `X-Correlation-Id` não é UUID válido |
| ERR-003 | 401 | `CREDENCIAIS_INVALIDAS` | e-mail inexistente ou senha definitiva incorreta |
| ERR-004 | 401 | `SENHA_TEMPORARIA_EXPIRADA` | senha temporária expirou |
| ERR-005 | 401 | `SENHA_TEMPORARIA_INVALIDADA` | senha temporária foi invalidada após limite de tentativas |
| ERR-006 | 403 | `USUARIO_BLOQUEADO` | usuário está bloqueado |
| ERR-007 | 403 | `USUARIO_INATIVO` | usuário está inativo |
| ERR-008 | 429 | `LIMITE_TENTATIVAS_EXCEDIDO` | limite de tentativas de login excedido |
| ERR-009 | 500 | `ERRO_INTERNO` | erro interno não tratado |
| ERR-010 | 503 | `SERVICO_INDISPONIVEL` | dependência necessária indisponível |
| ERR-011 | 413 | `PAYLOAD_MUITO_GRANDE` | request body excede o limite global de 1 MB |

Todos os erros DEVEM utilizar o envelope global de erro.

---

## 10. Segurança

- Endpoint público, mas protegido por rate limit.
- Senha nunca deve ser registrada.
- JWT assinado com RS256.
- Chave privada não deve estar no repositório.
- `iss` e `aud` devem ser configuráveis por ambiente.
- Refresh Token só existe em cookie `HttpOnly`.
- Hash do Refresh Token persiste em `SESSAO_USUARIO`.
- Frontend não deve acessar o Refresh Token por JavaScript.
- CORS deve usar allow-list de origens.
- IP e user-agent podem ser registrados em `AUDIT_LOG.detalhes` para eventos de segurança.
- Headers de proxy só são confiáveis quando a requisição vier de proxy confiável.

---

## 11. Dependências

| Dependência | Uso | Falha |
|---|---|---|
| PostgreSQL | usuário, roles, permissões, sessão e auditoria | 503 |
| Redis | contadores e bloqueio temporário do rate limit por login e proteção complementar por IP/origem | 503 |
| Provedor/serviço JWT interno | assinatura RS256 | 500/503 conforme natureza |
| Configuração de segurança | `iss`, `aud`, chave privada | falha de inicialização/configuração |

---

## 12. Persistência

### `USUARIO`

Leitura:

- `pessoa_id`;
- `senha_hash`;
- `status`;
- `troca_senha_obrigatoria`;
- `senha_temporaria_hash`;
- `senha_temporaria_expira_em`;
- `tentativas_senha_temporaria`.

Atualização em sucesso normal:

- `ultimo_login`;
- dados de controle de tentativas, quando aplicável.

### `SESSAO_USUARIO`

Criação em login normal:

- `usuario_id`;
- `refresh_token_hash`;
- `quantidade_refresh = 0`;
- `data_criacao`;
- `data_expiracao`;
- `ativa = true`.

Sessão ativa anterior deve ser invalidada com `NOVO_LOGIN`.

### `AUDIT_LOG`

Ações:

- `LOGIN`;
- `LOGIN_FALHA`;
- `INVALIDAR_SESSAO` quando aplicável.

---

## 13. Idempotência e Concorrência

### 13.1 Idempotência

`Idempotency-Key` NÃO se aplica ao login.

Repetir uma autenticação válida é uma nova intenção de login e, pela regra de sessão única, o novo login invalida a sessão anterior.

### 13.2 Concorrência

A invalidação da sessão anterior, a criação da nova sessão e a auditoria correspondente DEVEM ocorrer na mesma transação de banco.

A criação de sessão DEVE respeitar a garantia física de uma única sessão ativa por usuário. O processamento DEVE serializar por usuário, por bloqueio transacional ou mecanismo equivalente, e tratar de forma determinística eventual violação do índice único parcial.

Duas autenticações concorrentes do mesmo usuário não podem resultar em duas sessões ativas. Ao final, apenas a sessão criada pela transação vencedora permanece ativa; qualquer sessão substituída deve estar invalidada com motivo `NOVO_LOGIN`.

---

## 14. Eventos

Não há evento de domínio assíncrono obrigatório definido para este Resource.

Eventos de segurança são representados via auditoria.

---

## 15. Observabilidade e Auditoria

### 15.1 Logs

PODEM registrar:

- `correlationId`;
- status final;
- duração;
- motivo técnico genérico;
- IP seguro/origem;
- user-agent.

NÃO DEVEM registrar:

- senha;
- hash;
- JWT;
- Refresh Token;
- hash de Refresh Token.

### 15.2 Auditoria

Sucesso:

```text
acao = LOGIN
recursoTipo = USUARIO
recursoId = usuarioId
origem = USUARIO
```

Falha sem usuário identificado:

```text
acao = LOGIN_FALHA
usuarioId = null
recursoId = null
origem = SISTEMA
```

`detalhes` pode conter IP, user-agent e motivo genérico seguro.

---

## 16. Requisitos Não Funcionais

- **RNF-001:** timeout máximo padrão de 30 segundos.
- **RNF-002:** nenhuma credencial deve vazar em log, erro ou auditoria.
- **RNF-003:** response deve sempre possuir `X-Correlation-Id`.
- **RNF-004:** endpoint deve respeitar rate limit.
- **RNF-005:** Access Token deve ser RS256.
- **RNF-006:** cookie deve respeitar atributos definidos para o ambiente.
- **RNF-007:** implementação deve manter apenas uma sessão ativa por usuário.

---

## 17. Critérios de Aceite

- **CA-001:** Dado usuário ativo com credencial definitiva correta, quando autenticar, então recebe Access Token de 15 minutos e cookie `refresh_token`.
- **CA-002:** Dado usuário com sessão ativa, quando realizar novo login válido, então a sessão anterior é invalidada.
- **CA-003:** Dado e-mail inexistente ou senha incorreta, então a resposta é `401 CREDENCIAIS_INVALIDAS` sem revelar qual dado falhou.
- **CA-004:** Dado usuário bloqueado, então não recebe sessão e recebe `403 USUARIO_BLOQUEADO`.
- **CA-005:** Dado usuário inativo, então não recebe sessão e recebe `403 USUARIO_INATIVO`.
- **CA-006:** Dado usuário com senha temporária válida, então recebe somente token `TROCA_SENHA`, sem sessão e sem cookie de Refresh Token.
- **CA-007:** Dada senha temporária expirada, então recebe `401 SENHA_TEMPORARIA_EXPIRADA`.
- **CA-008:** Após 5 tentativas inválidas de senha temporária, ela é invalidada.
- **CA-009:** Após atingir o rate limit de login, o sistema retorna `429 LIMITE_TENTATIVAS_EXCEDIDO`.
- **CA-010:** Nenhuma senha/token/hash aparece em logs ou auditoria.
- **CA-011:** Sucesso e falha geram os registros de auditoria aplicáveis.
- **CA-012:** Campo JSON desconhecido retorna 400.
- **CA-013:** `X-Correlation-Id` válido recebido é devolvido sem alteração.
- **CA-014:** Ausência de `X-Correlation-Id` faz o backend gerar um UUID.

---

## 18. Diretrizes obrigatórias para implementação por IA

1. Não inventar requisitos adicionais.
2. Preservar exatamente path, método, códigos de erro e contratos públicos definidos.
3. Não registrar senha, hash, Access Token ou Refresh Token.
4. Respeitar sessão única.
5. Não transformar token `TROCA_SENHA` em sessão normal.
6. Não permitir acesso normal para usuário bloqueado ou inativo.
7. Gerar testes para todos os critérios de aceite.
8. Manter rastreabilidade entre código/testes e IDs `RN-*`, `RV-*`, `FLX-*`, `ERR-*` e `CA-*`.
9. Respeitar a arquitetura vigente do repositório.
10. Antes de concluir, executar o checklist da seção 19.

---

## 19. Checklist de completude da especificação

- [x] Objetivo e escopo claros.
- [x] Regras de negócio identificadas.
- [x] Validações e erros definidos.
- [x] Fluxo principal e exceções descritos.
- [x] Path e método definidos.
- [x] Segurança definida.
- [x] Request e response em JSON textual válido.
- [x] Dependências descritas.
- [x] Persistência descrita.
- [x] Idempotência e concorrência definidas.
- [x] Observabilidade e auditoria definidas.
- [x] Critérios de aceite objetivos.
- [x] Casos de teste documentados em artefato próprio.
- [x] OpenAPI documentado em artefato próprio.
- [x] Markdown e DOCX devem permanecer equivalentes.

---

## 20. Pendências e Decisões em Aberto

Nenhuma pendência funcional conhecida para o contrato do `RES-001`.

A execução real das migrations/seeds/bootstrap em PostgreSQL continua sendo pendência técnica global do projeto, não deste Resource.

---

## 21. Histórico de alterações

| Versão | Data | Alteração |
|---|---|---|
| 1.1 | 2026-09-17 | Atualização das baselines; inclusão de Redis no rate limit, erro 413, ordem segura de validação de status e tratamento transacional determinístico. |
| 1.0 | 2026-09-15 | Criação da especificação do RES-001 com base na baseline v1.3. |
