# RES-003 — Recuperar senha

**Versão:** 1.1  
**Status:** Em revisão  
**Baseline lógica:** MVP Domain Baseline v1.6  
**Base física:** Especificação Física do Banco v1.5  
**Catálogo:** Catálogo de Resources/APIs v1.3  
**Endpoint:** `POST /api/v1/autenticacao/recuperar-senha`

---

## 1. Identificação do Resource

| Campo | Valor |
|---|---|
| Domínio / Bounded Context | Autenticação e Autorização |
| Capacidade | Recuperar senha |
| Nome do Resource | RES-003 — Recuperar senha |
| Ação | Recuperar senha |
| Versão da especificação | 1.1 |
| Status | Em revisão |
| Última atualização | 2026-09-17 |

### 1.1 Objetivo

A capacidade permite que uma pessoa recupere o acesso quando não conhece mais sua senha, confirmando um conjunto de dados pessoais e institucionais previamente cadastrados.

Em caso de correspondência, o Resource DEVE gerar uma senha temporária aleatória, persistir somente seu hash, exigir troca obrigatória, invalidar a sessão normal existente e devolver a senha temporária uma única vez na response. O Resource NÃO DEVE revelar qual dado de identificação divergiu quando não houver correspondência.

### 1.2 Escopo

Inclui:

- validação de nome;
- validação de e-mail;
- validação de data de nascimento;
- validação do nome da paróquia;
- normalização dos campos aplicáveis;
- rate limit da recuperação;
- geração de senha temporária;
- persistência somente do hash da senha temporária;
- validade de 15 minutos;
- impedimento de múltiplas recuperações válidas simultâneas;
- invalidação de sessão/Refresh Token existentes;
- auditoria da recuperação e da invalidação de sessão;
- suporte a usuário bloqueado sem desbloqueá-lo;
- rejeição de usuário inativo.

Não inclui:

- login com senha temporária (`RES-001`);
- alteração da senha após autenticação temporária (`RES-002`);
- envio de senha por e-mail, SMS ou outro canal;
- reativação de usuário inativo;
- desbloqueio de usuário bloqueado.

---

## 2. Regras de Negócio

- **RN-001 — Dados de identificação:** A request DEVE conter `nome`, `email`, `dataNascimento` e `nomeParoquia`.
- **RN-002 — E-mail:** O e-mail DEVE ser normalizado com `trim + lowercase`.
- **RN-003 — Nome da paróquia:** `nomeParoquia` DEVE ser normalizado com `trim`, remoção de espaços excedentes e comparação case-insensitive.
- **RN-004 — Nome da pessoa:** O nome informado DEVE ser comparado de acordo com a normalização de texto da aplicação sem alterar o valor persistido.
- **RN-005 — Correspondência completa:** A recuperação somente PODE prosseguir quando todos os dados informados correspondem ao mesmo usuário/pessoa.
- **RN-006 — Falha genérica:** Quando os dados não corresponderem, o sistema NÃO DEVE indicar qual campo divergiu.
- **RN-007 — Código de não localização:** Falha de correspondência DEVE retornar `404 DADOS_RECUPERACAO_NAO_LOCALIZADOS`.
- **RN-008 — Usuário bloqueado:** Usuário `BLOQUEADO` PODE iniciar e concluir a recuperação de senha.
- **RN-009 — Preservação do bloqueio:** A recuperação NÃO DEVE alterar `status = BLOQUEADO`.
- **RN-010 — Usuário inativo:** Usuário `INATIVO` NÃO DEVE iniciar recuperação.
- **RN-011 — Recuperação ativa única:** Um usuário NÃO DEVE possuir duas recuperações válidas simultâneas.
- **RN-012 — Recuperação em andamento:** Se já existir senha temporária válida e não utilizada, uma nova solicitação DEVE retornar `409 RECUPERACAO_SENHA_EM_ANDAMENTO`.
- **RN-013 — Não reexibir senha temporária:** No cenário da RN-012, o sistema NÃO DEVE retornar novamente a senha temporária existente.
- **RN-014 — Geração:** Em recuperação válida, o sistema DEVE gerar uma nova senha temporária aleatória.
- **RN-015 — Persistência segura:** A senha temporária em texto puro NÃO DEVE ser persistida; somente seu hash DEVE ser armazenado.
- **RN-016 — Hash:** O hash da senha temporária DEVE utilizar o mecanismo seguro definido para autenticação do projeto.
- **RN-017 — Validade:** A senha temporária DEVE expirar 15 minutos após sua geração.
- **RN-018 — Troca obrigatória:** Após gerar senha temporária, `troca_senha_obrigatoria` DEVE ser `true`.
- **RN-019 — Tentativas temporárias:** `tentativas_senha_temporaria` DEVE ser zerado quando uma nova senha temporária for gerada.
- **RN-020 — Uso anterior:** `senha_temporaria_utilizada_em` DEVE ser limpa ao iniciar uma nova recuperação válida.
- **RN-021 — Invalidação de sessão:** Recuperação bem-sucedida DEVE invalidar qualquer sessão ativa e Refresh Token do usuário com motivo `RECUPERACAO_SENHA`.
- **RN-022 — Access Token anterior:** Após invalidação da sessão, Access Tokens associados ao `sessionId` anterior NÃO DEVEM mais ser aceitos.
- **RN-023 — Entrega única:** A senha temporária em texto puro DEVE ser retornada somente na response da recuperação que a gerou.
- **RN-024 — Sem sessão automática:** Recuperação bem-sucedida NÃO DEVE criar sessão normal nem emitir Access Token normal.
- **RN-025 — Próximo passo:** O usuário DEVE utilizar a senha temporária no login (`RES-001`), que resultará em token restrito `TROCA_SENHA`.
- **RN-026 — Rate limit:** O sistema DEVE permitir no máximo 3 tentativas de recuperação em uma janela de 30 minutos por usuário/login.
- **RN-027 — Bloqueio temporário:** Ao atingir o limite, novas tentativas DEVEM ser bloqueadas por 30 minutos.
- **RN-028 — Proteção complementar por IP:** O sistema DEVE aplicar proteção complementar por IP/origem sem substituir o controle principal.
- **RN-029 — Segredos:** Senha temporária em texto puro, senha definitiva, hashes, JWT e Refresh Token NÃO DEVEM entrar em logs ou auditoria.
- **RN-030 — Auditoria:** Recuperação bem-sucedida DEVE registrar `RECUPERAR_SENHA`.
- **RN-031 — Auditoria de invalidação:** Se houver sessão ativa, sua invalidação DEVE registrar `INVALIDAR_SESSAO`.
- **RN-032 — Correlação:** A operação DEVE utilizar `X-Correlation-Id`.
- **RN-033 — Campos desconhecidos:** Campos JSON desconhecidos DEVEM causar `400 Bad Request`.
- **RN-034 — Contexto de uma paróquia no MVP:** O MVP utiliza o nome da paróquia informado na request; não é exigido `paroquiaId`.
- **RN-035 — Homônimos futuros:** Tratamento de paróquias homônimas com cidade/diocese é evolução futura e NÃO altera o contrato do MVP atual.

---

## 3. Validações e Invariantes

- **RV-001:** `nome` é obrigatório.
- **RV-002:** `email` é obrigatório.
- **RV-003:** `dataNascimento` é obrigatória.
- **RV-004:** `nomeParoquia` é obrigatório.
- **RV-005:** `email` deve ser string em formato válido.
- **RV-006:** `dataNascimento` deve usar `YYYY-MM-DD`.
- **RV-007:** `nome`, `email` e `nomeParoquia` normalizados não podem ser vazios.
- **RV-008:** request body máximo de 1 MB.
- **RV-009:** campos textuais devem respeitar limite global aplicável.
- **RV-010:** `X-Correlation-Id`, quando enviado, deve ser UUID válido.
- **RV-011:** os quatro dados devem pertencer ao mesmo cadastro.
- **RV-012:** usuário inativo deve ser rejeitado.
- **RV-013:** recuperação válida ainda não expirada impede uma nova recuperação.
- **RV-014:** rate limit deve ser verificado antes de gerar nova senha temporária.

---

## 4. Pré-condições e Pós-condições

### 4.1 Pré-condições

- os dados informados devem ser sintaticamente válidos;
- quando houver correspondência, deve existir `PESSOA` associada a `USUARIO`;
- usuário não pode estar `INATIVO`;
- não pode existir recuperação válida em andamento;
- rate limit não pode estar excedido.

### 4.2 Pós-condições — sucesso

- nova senha temporária aleatória foi gerada;
- somente o hash foi persistido;
- `senha_temporaria_expira_em = geração + 15 minutos`;
- `senha_temporaria_utilizada_em = null`;
- `tentativas_senha_temporaria = 0`;
- `troca_senha_obrigatoria = true`;
- eventual sessão ativa foi invalidada com `RECUPERACAO_SENHA`;
- nenhuma sessão normal foi criada;
- a senha temporária foi devolvida uma única vez;
- auditoria de recuperação foi registrada.

### 4.3 Pós-condições — falha de identificação

- nenhuma senha temporária é gerada;
- nenhum estado de autenticação é alterado;
- a response não revela qual campo divergiu.

---

## 5. Fluxos

### FLX-001 — Recuperação com sucesso

1. Receber request.
2. Resolver `X-Correlation-Id`.
3. Validar estrutura e campos.
4. Aplicar rate limit.
5. Normalizar e-mail e nome da paróquia.
6. Localizar cadastro correspondente.
7. Confirmar que todos os dados pertencem ao mesmo usuário.
8. Confirmar que o usuário não está inativo.
9. Verificar se já existe recuperação válida em andamento.
10. Gerar nova senha temporária aleatória.
11. Gerar e persistir somente seu hash.
12. Definir expiração de 15 minutos.
13. Definir `troca_senha_obrigatoria = true`.
14. Zerar tentativas da senha temporária.
15. Invalidar sessão ativa existente, se houver.
16. Registrar `RECUPERAR_SENHA`.
17. Registrar `INVALIDAR_SESSAO`, quando aplicável.
18. Retornar `200 OK` com a senha temporária em texto puro uma única vez.

### FLX-002 — Dados não localizados

1. Dados não correspondem integralmente.
2. Não revelar qual campo divergiu.
3. Contabilizar tentativa de recuperação.
4. Retornar `404 DADOS_RECUPERACAO_NAO_LOCALIZADOS`.

### FLX-003 — Recuperação já em andamento

1. Usuário é identificado.
2. Existe senha temporária válida e não utilizada.
3. Sistema não gera outra senha.
4. Sistema não retorna a senha temporária existente.
5. Retorna `409 RECUPERACAO_SENHA_EM_ANDAMENTO`.

### FLX-004 — Usuário bloqueado

1. Usuário é identificado.
2. `status = BLOQUEADO`.
3. Recuperação prossegue normalmente.
4. Nova senha temporária é gerada.
5. Status continua `BLOQUEADO`.
6. Retorna sucesso.

### FLX-005 — Usuário inativo

1. Usuário é identificado.
2. `status = INATIVO`.
3. Não gerar senha temporária.
4. Retornar `403 USUARIO_INATIVO`.

### FLX-006 — Rate limit excedido

1. Limite de 3 tentativas em 30 minutos foi atingido.
2. Não executar nova recuperação durante 30 minutos.
3. Retornar `429 LIMITE_TENTATIVAS_EXCEDIDO`.

---

## 6. Dados do Endpoint

| Campo | Valor |
|---|---|
| Método | POST |
| Path | `/api/v1/autenticacao/recuperar-senha` |
| Autenticação | Pública |
| Content-Type | `application/json` |
| Timeout | 30 segundos |
| Request body máximo | 1 MB |
| Idempotency-Key | Não aplicável |
| X-Correlation-Id | Opcional na request; obrigatório na response |

---

## 7. Request

```json
{
  "nome": "Nome Completo",
  "email": "usuario@exemplo.com",
  "dataNascimento": "2000-01-15",
  "nomeParoquia": "Paróquia Exemplo"
}
```

| Campo | Tipo | Obrigatório | Regra |
|---|---|---:|---|
| `nome` | string | Sim | texto não vazio após normalização |
| `email` | string | Sim | trim + lowercase |
| `dataNascimento` | string/date | Sim | `YYYY-MM-DD` |
| `nomeParoquia` | string | Sim | trim, remover espaços excedentes e comparar case-insensitive |

---

## 8. Response

### 8.1 Recuperação criada — 200

```json
{
  "senhaTemporaria": "<valor-gerado>",
  "expiraEm": "2026-09-16T11:15:00Z",
  "trocaSenhaObrigatoria": true
}
```

Regras:

- `senhaTemporaria` somente aparece nesta response;
- `expiraEm` deve incluir timezone;
- não há Access Token;
- não há Refresh Token;
- não há criação de sessão.

Headers:

```text
X-Correlation-Id: <uuid>
```

---

## 9. Erros

| ID | HTTP | Código | Condição |
|---|---:|---|---|
| ERR-001 | 400 | `DADOS_INCONSISTENTES` | payload inválido ou campo desconhecido |
| ERR-002 | 400 | `CORRELATION_ID_INVALIDO` | correlation id inválido |
| ERR-003 | 404 | `DADOS_RECUPERACAO_NAO_LOCALIZADOS` | conjunto de identificação não corresponde |
| ERR-004 | 403 | `USUARIO_INATIVO` | usuário identificado está inativo |
| ERR-005 | 409 | `RECUPERACAO_SENHA_EM_ANDAMENTO` | já existe senha temporária válida |
| ERR-006 | 429 | `LIMITE_TENTATIVAS_EXCEDIDO` | limite de recuperação excedido |
| ERR-007 | 500 | `ERRO_INTERNO` | falha interna |
| ERR-008 | 503 | `SERVICO_INDISPONIVEL` | dependência essencial indisponível |
| ERR-009 | 413 | `PAYLOAD_MUITO_GRANDE` | request body excede o limite global de 1 MB |

Todos os erros utilizam o envelope global de erro.

---

## 10. Segurança

- Endpoint público sujeito a rate limit.
- Falha de identificação deve ser genérica.
- Senha temporária deve ser gerada com fonte criptograficamente segura.
- Somente o hash pode ser persistido.
- A senha temporária em texto puro só pode existir durante processamento e na response de sucesso.
- O valor em texto puro não deve aparecer em logs, auditoria, traces ou métricas.
- Recuperação invalida sessão normal existente.
- IP e user-agent podem ser usados como contexto seguro de auditoria.
- Headers de proxy só podem ser confiados quando a requisição vier de proxy confiável.
- CORS deve usar allow-list por ambiente.

---

## 11. Dependências

| Dependência | Uso | Falha |
|---|---|---|
| PostgreSQL | identificação, usuário, senha temporária, sessão e auditoria | 503 |
| Redis | contadores e bloqueio temporário do rate limit por login e proteção complementar por IP/origem | 503 |
| Gerador criptográfico seguro | gerar senha temporária | 500 |
| Hash de senha | persistir senha temporária de forma segura | 500 |

---

## 12. Persistência

### `PESSOA`

Leitura para identificação:

- nome;
- data de nascimento;
- e-mail;
- vínculo com paróquia.

### `PAROQUIA`

Leitura:

- nome normalizado para comparação.

### `USUARIO`

Atualizações em sucesso:

- `troca_senha_obrigatoria = true`;
- `senha_temporaria_hash`;
- `senha_temporaria_expira_em`;
- `senha_temporaria_utilizada_em = null`;
- `tentativas_senha_temporaria = 0`;
- `data_atualizacao`.

A senha definitiva existente NÃO precisa ser removida durante a recuperação; o próximo login obrigatório deve utilizar a senha temporária conforme fluxo de autenticação aprovado.

### `SESSAO_USUARIO`

Se existir sessão ativa:

- `ativa = false`;
- `data_invalidacao` preenchida;
- `motivo_invalidacao = RECUPERACAO_SENHA`.

### `AUDIT_LOG`

Ações:

- `RECUPERAR_SENHA`;
- `INVALIDAR_SESSAO`, quando aplicável.

---

## 13. Idempotência e Concorrência

### 13.1 Idempotência

`Idempotency-Key` NÃO se aplica.

A proteção contra duplicação ocorre pela regra de domínio:

- se já existir recuperação válida em andamento, retornar `409 RECUPERACAO_SENHA_EM_ANDAMENTO`;
- não gerar nova senha;
- não retornar a senha temporária anterior novamente.

### 13.2 Concorrência

Duas solicitações concorrentes para o mesmo usuário NÃO DEVEM produzir duas senhas temporárias válidas.

A verificação de recuperação em andamento, a geração/persistência da nova senha temporária, a invalidação da sessão ativa e a auditoria aplicável DEVEM ocorrer na mesma transação. O processamento DEVE serializar por usuário, por bloqueio transacional ou mecanismo equivalente, para que uma única solicitação conclua com sucesso e as demais retornem `409 RECUPERACAO_SENHA_EM_ANDAMENTO`.

---

## 14. Eventos

Não há evento de domínio assíncrono obrigatório.

A operação gera eventos de segurança via auditoria.

---

## 15. Observabilidade e Auditoria

### 15.1 Logs

PODEM conter:

- correlationId;
- resultado;
- duração;
- IP seguro;
- user-agent;
- motivo genérico.

NÃO DEVEM conter:

- senha temporária em texto puro;
- hash da senha temporária;
- senha definitiva;
- hashes;
- JWT;
- Refresh Token.

### 15.2 Auditoria

Sucesso:

```text
acao = RECUPERAR_SENHA
recursoTipo = USUARIO
recursoId = usuarioId
origem = SISTEMA
```

Se houver sessão invalidada:

```text
acao = INVALIDAR_SESSAO
recursoTipo = SESSAO_USUARIO
recursoId = sessaoId
detalhes.motivo = RECUPERACAO_SENHA
```

Falhas de identificação podem ser auditadas como evento de segurança sem `usuarioId` e `recursoId`, desde que nenhum dado sensível seja exposto.

---

## 16. Requisitos Não Funcionais

- **RNF-001:** timeout padrão de 30 segundos.
- **RNF-002:** nenhuma senha ou hash pode vazar.
- **RNF-003:** resposta deve possuir `X-Correlation-Id`.
- **RNF-004:** rate limit deve ser aplicado.
- **RNF-005:** geração da senha temporária deve utilizar aleatoriedade criptograficamente segura.
- **RNF-006:** operação de criação da recuperação deve ser consistente sob concorrência.
- **RNF-007:** invalidação de sessão deve produzir efeito imediato.

---

## 17. Critérios de Aceite

- **CA-001:** Dados válidos e correspondentes geram senha temporária e retornam `200`.
- **CA-002:** Somente o hash da senha temporária é persistido.
- **CA-003:** Senha temporária expira em 15 minutos.
- **CA-004:** Recuperação define `troca_senha_obrigatoria=true`.
- **CA-005:** Recuperação invalida sessão ativa existente.
- **CA-006:** A senha temporária é retornada uma única vez.
- **CA-007:** Nova solicitação durante recuperação válida retorna `409 RECUPERACAO_SENHA_EM_ANDAMENTO`.
- **CA-008:** O `409` não retorna novamente a senha temporária.
- **CA-009:** Dados que não correspondem retornam `404 DADOS_RECUPERACAO_NAO_LOCALIZADOS` sem indicar o campo divergente.
- **CA-010:** Usuário bloqueado pode recuperar senha e permanece bloqueado.
- **CA-011:** Usuário inativo recebe `403 USUARIO_INATIVO`.
- **CA-012:** Após 3 tentativas em 30 minutos, o limite é aplicado por 30 minutos.
- **CA-013:** Nenhum segredo aparece em logs ou auditoria.
- **CA-014:** Sucesso registra `RECUPERAR_SENHA`.
- **CA-015:** Invalidação de sessão registra `INVALIDAR_SESSAO`.
- **CA-016:** Duas requests concorrentes não produzem duas recuperações válidas.
- **CA-017:** O endpoint não cria sessão, Access Token ou Refresh Token.
- **CA-018:** Correlation ID é retornado em todas as respostas.

---

## 18. Diretrizes obrigatórias para implementação por IA

1. Não inventar canal de envio de senha temporária.
2. Não enviar senha temporária por e-mail/SMS neste MVP.
3. Não revelar qual dado de recuperação divergiu.
4. Não gerar nova senha se existir recuperação válida.
5. Não retornar novamente a senha temporária existente.
6. Não desbloquear usuário bloqueado.
7. Não permitir recuperação para usuário inativo.
8. Não registrar senhas, hashes ou tokens.
9. Garantir consistência sob concorrência.
10. Gerar testes cobrindo todos os critérios de aceite.

---

## 19. Checklist de completude da especificação

- [x] Objetivo e escopo definidos.
- [x] Dados de identificação definidos.
- [x] Normalização definida.
- [x] Recuperação em andamento definida.
- [x] Rate limit definido.
- [x] Usuário bloqueado/inativo definidos.
- [x] Request e response definidos.
- [x] Erros definidos.
- [x] Persistência definida.
- [x] Invalidação de sessão definida.
- [x] Idempotência e concorrência definidas.
- [x] Auditoria definida.
- [x] Critérios de aceite definidos.
- [x] Casos de teste documentados em artefato próprio.
- [x] OpenAPI documentado em artefato próprio.
- [x] Markdown e DOCX devem permanecer equivalentes.

---

## 20. Pendências e Decisões em Aberto

Nenhuma pendência funcional conhecida para o contrato do `RES-003`.

O rate limit normativo deste Resource utiliza Redis como armazenamento operacional dos contadores e bloqueios temporários. A indisponibilidade do Redis deve resultar em `503 SERVICO_INDISPONIVEL`, sem prosseguir com a recuperação.

---

## 21. Histórico de alterações

| Versão | Data | Alteração |
|---|---|---|
| 1.1 | 2026-09-17 | Atualização das baselines; inclusão de Redis no rate limit, erro 413 e tratamento transacional determinístico. |
| 1.0 | 2026-09-16 | Criação da especificação do RES-003. |
