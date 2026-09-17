# RES-001 — Casos de Teste — Autenticar usuário

**Versão:** 1.1  
**Objetivo:** servir como base reutilizável para posterior geração dos arquivos `.feature` e automação E2E.  
**Observação:** este artefato NÃO gera arquivos `.feature`.

---

## 1. Diretrizes

- Cenários independentes entre si.
- Dados devem ser preparados por cenário/fixture.
- Não depender da ordem de execução.
- Não usar seletores de UI, Cypress ou Playwright neste documento.
- Tags devem preservar rastreabilidade com caso de uso e critério de aceite.

---

## CT-AUT-001 — Login normal com sucesso

Rastreabilidade: `UC-AUT-001`, `CA-001`.

```gherkin
@CT-AUT-001
@UC-AUT-001
@CA-001
@e2e
@regressao
Cenário: Autenticar usuário ativo com senha definitiva válida
  Dado que existe um usuário ativo com e-mail "usuario@exemplo.com"
  E que "trocaSenhaObrigatoria" é falso
  E que a senha definitiva informada é válida
  Quando o cliente enviar POST para "/api/v1/autenticacao/login"
  Então a resposta deve possuir status 200
  E deve retornar "accessToken"
  E "tokenType" deve ser "Bearer"
  E "expiresIn" deve ser 900
  E a resposta deve possuir o header "X-Correlation-Id"
  E deve ser criado o cookie HttpOnly "refresh_token"
  E deve existir somente uma sessão ativa para o usuário
  E deve existir auditoria com ação "LOGIN"
```

---

## CT-AUT-002 — Credenciais inválidas não revelam existência do e-mail

Rastreabilidade: `UC-AUT-003`, `CA-003`.

```gherkin
@CT-AUT-002
@UC-AUT-003
@CA-003
@e2e
@seguranca
Cenário: Rejeitar credenciais inválidas com mensagem genérica
  Dado que o cliente possui uma combinação inválida de e-mail e senha
  Quando enviar POST para "/api/v1/autenticacao/login"
  Então a resposta deve possuir status 401
  E o código de erro deve ser "CREDENCIAIS_INVALIDAS"
  E a resposta não deve informar se o e-mail existe
  E a resposta não deve informar se a senha está incorreta
  E deve existir auditoria com ação "LOGIN_FALHA"
```

---

## CT-AUT-003 — Novo login invalida sessão anterior

Rastreabilidade: `UC-AUT-001`, `CA-002`.

```gherkin
@CT-AUT-003
@UC-AUT-001
@CA-002
@e2e
@sessao
Cenário: Invalidar sessão anterior ao realizar novo login
  Dado que o usuário possui uma sessão ativa
  Quando realizar um novo login com credenciais válidas
  Então a sessão anterior deve ser invalidada com motivo "NOVO_LOGIN"
  E deve existir somente uma sessão ativa para o usuário
  E o Access Token da sessão anterior não deve mais ser aceito
```

---

## CT-AUT-004 — Primeiro acesso retorna token restrito

Rastreabilidade: `UC-AUT-002`, `CA-006`.

```gherkin
@CT-AUT-004
@UC-AUT-002
@CA-006
@e2e
@primeiro-acesso
Cenário: Autenticar com senha temporária válida
  Dado que existe um usuário ativo com "trocaSenhaObrigatoria" igual a verdadeiro
  E a senha temporária está válida e não utilizada
  Quando o cliente autenticar com a senha temporária
  Então a resposta deve possuir status 200
  E deve retornar "tokenTrocaSenha"
  E "tokenType" deve ser "TROCA_SENHA"
  E "expiresIn" deve ser 900
  E "trocaSenhaObrigatoria" deve ser verdadeiro
  E não deve existir cookie "refresh_token"
  E não deve ser criada sessão normal
```

---

## CT-AUT-005 — Senha temporária expirada

Rastreabilidade: `UC-AUT-007`, `CA-007`.

```gherkin
@CT-AUT-005
@UC-AUT-007
@CA-007
@e2e
@recuperacao
Cenário: Rejeitar senha temporária expirada
  Dado que a senha temporária do usuário está expirada
  Quando o usuário tentar autenticar com essa senha
  Então a resposta deve possuir status 401
  E o código de erro deve ser "SENHA_TEMPORARIA_EXPIRADA"
  E nenhuma sessão deve ser criada
```

---

## CT-AUT-006 — Usuário bloqueado

Rastreabilidade: `UC-AUT-004`, `CA-004`.

```gherkin
@CT-AUT-006
@UC-AUT-004
@CA-004
@e2e
@seguranca
Cenário: Impedir login de usuário bloqueado
  Dado que existe um usuário com status "BLOQUEADO"
  Quando ele tentar autenticar com credencial correspondente
  Então a resposta deve possuir status 403
  E o código de erro deve ser "USUARIO_BLOQUEADO"
  E nenhuma sessão deve ser criada
```

---

## CT-AUT-007 — Usuário inativo

Rastreabilidade: `UC-AUT-005`, `CA-005`.

```gherkin
@CT-AUT-007
@UC-AUT-005
@CA-005
@e2e
Cenário: Impedir login de usuário inativo
  Dado que existe um usuário com status "INATIVO"
  Quando ele tentar autenticar com credencial correspondente
  Então a resposta deve possuir status 403
  E o código de erro deve ser "USUARIO_INATIVO"
  E nenhuma sessão deve ser criada
```

---

## CT-AUT-008 — Rate limit

Rastreabilidade: `UC-AUT-006`, `CA-009`.

```gherkin
@CT-AUT-008
@UC-AUT-006
@CA-009
@e2e
@seguranca
Cenário: Bloquear temporariamente após cinco tentativas inválidas
  Dado que o usuário realizou cinco tentativas inválidas dentro de quinze minutos
  Quando realizar nova tentativa durante o bloqueio temporário
  Então a resposta deve possuir status 429
  E o código de erro deve ser "LIMITE_TENTATIVAS_EXCEDIDO"
```

---

## CT-AUT-009 — Campos desconhecidos

Rastreabilidade: `CA-012`.

```gherkin
@CT-AUT-009
@CA-012
@e2e
@contrato
Cenário: Rejeitar campo JSON desconhecido
  Quando o cliente enviar o payload de login contendo o campo "campoNaoSuportado"
  Então a resposta deve possuir status 400
  E o código de erro deve ser "DADOS_INCONSISTENTES"
  E a resposta deve indicar que "campoNaoSuportado" não é suportado
```

---

## CT-AUT-010 — Correlation ID informado

Rastreabilidade: `CA-013`.

```gherkin
@CT-AUT-010
@CA-013
@e2e
@observabilidade
Cenário: Reutilizar correlation id informado pelo cliente
  Dado que o cliente envia um "X-Correlation-Id" UUID válido
  Quando realizar o login
  Então a resposta deve conter o mesmo "X-Correlation-Id"
```

---

## CT-AUT-011 — Correlation ID gerado pelo backend

Rastreabilidade: `CA-014`.

```gherkin
@CT-AUT-011
@CA-014
@e2e
@observabilidade
Cenário: Gerar correlation id quando não informado
  Dado que o cliente não envia "X-Correlation-Id"
  Quando realizar o login
  Então a resposta deve conter "X-Correlation-Id"
  E o valor deve ser um UUID válido
```

---

## CT-AUT-012 — Não vazar segredos

Rastreabilidade: `CA-010`.

```gherkin
@CT-AUT-012
@CA-010
@seguranca
@e2e
Cenário: Não registrar segredos durante autenticação
  Quando ocorrer uma tentativa de login
  Então logs e auditoria não devem conter a senha
  E não devem conter hash de senha
  E não devem conter o Access Token
  E não devem conter o Refresh Token
  E não devem conter o hash do Refresh Token
```

---

## CT-AUT-013 — Dois logins concorrentes

Rastreabilidade: `UC-AUT-008`, `CA-002`, `RN-004`.

```gherkin
@CT-AUT-013
@UC-AUT-008
@RN-004
@e2e
@concorrencia
Cenário: Manter somente uma sessão ativa com dois logins concorrentes
  Dado que duas requisições válidas de login do mesmo usuário são iniciadas concorrentemente
  Quando ambas forem processadas
  Então deve existir somente uma sessão ativa para o usuário
  E qualquer sessão substituída deve estar inválida
```

---

## CT-AUT-014 — Limite de tentativas da senha temporária

Rastreabilidade: `CA-008`.

```gherkin
@CT-AUT-014
@CA-008
@e2e
@recuperacao
Cenário: Invalidar senha temporária após cinco tentativas inválidas
  Dado que o usuário possui uma senha temporária válida
  E já realizou quatro tentativas inválidas
  Quando realizar a quinta tentativa inválida
  Então a senha temporária deve ser invalidada
  E uma tentativa posterior deve retornar 401
  E o código de erro deve ser "SENHA_TEMPORARIA_INVALIDADA"
```

---

## CT-AUT-015 — Payload acima do limite

Rastreabilidade: limite global de request body do Catálogo de Resources/APIs v1.3.

```gherkin
@CT-AUT-015
@e2e
@contrato
Cenário: Rejeitar payload de login acima de 1 MB
  Dado que a request de login possui body maior que 1 MB
  Quando a request for enviada
  Então a resposta deve possuir status 413
  E o código deve ser "PAYLOAD_MUITO_GRANDE"
  E nenhuma tentativa de autenticação deve ser processada
```
