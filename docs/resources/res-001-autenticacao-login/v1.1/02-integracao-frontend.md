# RES-001 — Integração Frontend — Autenticar usuário

**Versão:** 1.1  
**Endpoint:** `POST /api/v1/autenticacao/login`

---

## 1. Objetivo

Este documento descreve como o frontend deve consumir o login, interpretar sucesso normal, troca obrigatória de senha, erros e cookie de Refresh Token.

---

## 2. Request

```http
POST /api/v1/autenticacao/login
Content-Type: application/json
X-Correlation-Id: <uuid opcional>
```

```json
{
  "email": "usuario@exemplo.com",
  "senha": "Senha123!"
}
```

O frontend DEVE:

- aplicar `trim` visualmente ao e-mail antes de enviar;
- não persistir a senha;
- não enviar campos extras;
- não enviar `Idempotency-Key`.

---

## 3. Sucesso — sessão normal

HTTP `200`.

```json
{
  "accessToken": "<jwt>",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

O backend também envia:

```text
Set-Cookie: refresh_token=...; HttpOnly; SameSite=Lax; Path=/api/v1/autenticacao
X-Correlation-Id: <uuid>
```

### Comportamento do frontend

- manter `accessToken` somente em memória;
- usar `Authorization: Bearer <accessToken>` nos endpoints autenticados;
- não tentar ler `refresh_token`, pois ele é `HttpOnly`;
- usar `RES-106` para renovar token quando necessário;
- ao recarregar totalmente a aplicação, reconstruir a sessão via fluxo de renovação, quando aplicável.

---

## 4. Sucesso — troca obrigatória

HTTP `200`.

```json
{
  "tokenTrocaSenha": "<jwt-restrito>",
  "tokenType": "TROCA_SENHA",
  "expiresIn": 900,
  "trocaSenhaObrigatoria": true
}
```

### Comportamento do frontend

- não tratar esse retorno como sessão normal;
- não tentar acessar áreas autenticadas;
- navegar diretamente para a tela de alteração de senha;
- usar `tokenTrocaSenha` apenas em `POST /api/v1/autenticacao/alterar-senha`;
- não haverá cookie `refresh_token` nesse cenário;
- após troca bem-sucedida, descartar o token e direcionar para novo login.

---

## 5. Tratamento de erros

| HTTP | Código | Ação sugerida no frontend |
|---:|---|---|
| 400 | `DADOS_INCONSISTENTES` | destacar campos inválidos |
| 400 | `CORRELATION_ID_INVALIDO` | erro técnico; gerar novo correlation id |
| 401 | `CREDENCIAIS_INVALIDAS` | mensagem genérica de e-mail/senha inválidos |
| 401 | `SENHA_TEMPORARIA_EXPIRADA` | direcionar para nova recuperação |
| 401 | `SENHA_TEMPORARIA_INVALIDADA` | direcionar para nova recuperação |
| 403 | `USUARIO_BLOQUEADO` | informar bloqueio |
| 403 | `USUARIO_INATIVO` | informar inativação |
| 413 | `PAYLOAD_MUITO_GRANDE` | não reenviar o payload e informar erro de requisição |
| 429 | `LIMITE_TENTATIVAS_EXCEDIDO` | impedir novas tentativas até liberação |
| 500 | `ERRO_INTERNO` | mensagem genérica e correlationId |
| 503 | `SERVICO_INDISPONIVEL` | mensagem de indisponibilidade temporária |

O frontend NÃO DEVE inferir existência de usuário a partir de `CREDENCIAIS_INVALIDAS`. Os códigos de usuário bloqueado ou inativo só podem ser retornados pelo backend depois da validação de uma credencial correspondente.

---

## 6. Correlation ID

O frontend PODE gerar um UUID por operação e enviar:

```text
X-Correlation-Id: <uuid>
```

Se enviado, o backend deve devolver o mesmo valor.

Se não enviado, o frontend deve capturar o `X-Correlation-Id` retornado pelo backend para suporte/troubleshooting.

---

## 7. CORS e cookies

Como o backend usa `refresh_token` em cookie, chamadas que dependem dele DEVEM usar credenciais no cliente HTTP.

Exemplo conceitual:

```typescript
credentials: 'include'
```

A URL do frontend deve estar na allow-list CORS do ambiente.

---

## 8. Estados de UI recomendados

- `idle`
- `submitting`
- `authenticated`
- `passwordChangeRequired`
- `invalidCredentials`
- `blocked`
- `inactive`
- `rateLimited`
- `unavailable`

Esses estados são de integração; não alteram o contrato da API.

---

## 9. Não fazer

O frontend NÃO DEVE:

- armazenar Refresh Token em localStorage/sessionStorage;
- tentar ler o cookie HttpOnly;
- persistir senha;
- exibir detalhes internos de erro;
- diferenciar “e-mail inexistente” de “senha incorreta”;
- reutilizar `tokenTrocaSenha` para endpoints normais.
