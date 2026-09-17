# RES-106 — Integração Frontend — Renovar Access Token

**Versão:** 1.1  
**Endpoint:** `POST /api/v1/autenticacao/renovar-token`

---

## 1. Objetivo

O frontend utiliza este Resource para obter novo Access Token enquanto a sessão absoluta ainda estiver válida.

O Refresh Token fica exclusivamente no cookie `refresh_token` HttpOnly e não é acessível por JavaScript.

---

## 2. Request

```http
POST /api/v1/autenticacao/renovar-token
Origin: https://frontend.exemplo
X-Correlation-Id: <uuid opcional>
Cookie: refresh_token=<gerenciado pelo navegador>
```

Não enviar body.

O cliente HTTP DEVE incluir credenciais:

```typescript
credentials: 'include'
```

---

## 3. Sucesso

HTTP `200`.

```json
{
  "accessToken": "<jwt>",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

`expiresIn` pode ser menor que 900 quando a sessão absoluta estiver próxima de expirar.

O backend envia um novo cookie `refresh_token`. O frontend não lê esse valor.

### Comportamento

- substituir o Access Token em memória;
- descartar imediatamente o Access Token anterior;
- continuar usando a sessão normalmente;
- não persistir Access Token em localStorage/sessionStorage.

---

## 4. Quando renovar

O frontend PODE iniciar renovação quando:

- o Access Token estiver próximo da expiração;
- uma estratégia centralizada de autenticação determinar a necessidade.

O frontend NÃO DEVE fazer loop agressivo de renovação.

Cada sucesso consome uma das 3 renovações disponíveis da sessão.

---

## 5. Erros

| HTTP | Código | Comportamento esperado |
|---:|---|---|
| 400 | `TOKEN_REFRESH_AUSENTE` | tratar como sessão inexistente e ir para login |
| 400 | `CORRELATION_ID_INVALIDO` | gerar novo correlation id |
| 401 | `REFRESH_TOKEN_INVALIDO` | limpar Access Token e ir para login |
| 401 | `REFRESH_TOKEN_EXPIRADO` | limpar sessão local e ir para login |
| 401 | `SESSAO_INVALIDA` | limpar sessão local e ir para login |
| 401 | `SESSAO_EXPIRADA` | limpar sessão local e ir para login |
| 403 | `USUARIO_BLOQUEADO` | encerrar estado autenticado e informar bloqueio |
| 403 | `USUARIO_INATIVO` | encerrar estado autenticado e informar inativação |
| 409 | `SESSAO_SUBSTITUIDA` | informar que houve novo login e encerrar estado local |
| 500 | `ERRO_INTERNO` | erro genérico |
| 503 | `SERVICO_INDISPONIVEL` | retry controlado conforme política da aplicação |

---

## 6. Concorrência no frontend

O frontend DEVE evitar múltiplas renovações simultâneas.

Recomendação de integração:

- centralizar renovação em um único serviço;
- compartilhar a Promise/Observable da renovação em andamento;
- enfileirar requests que dependam do novo Access Token.

Isso evita que duas chamadas concorrentes consumam o mesmo Refresh Token e causem falha por rotação.

---

## 7. Sessão absoluta

Mesmo com renovações, a sessão dura no máximo 1 hora desde o login.

Fluxo conceitual:

```text
Login
  ↓
Access 15 min
  ↓
Refresh 1
  ↓
Access até +30 min
  ↓
Refresh 2
  ↓
Access até +45 min
  ↓
Refresh 3
  ↓
Access até o limite absoluto de +60 min
  ↓
Novo login obrigatório
```

---

## 8. Segurança

O frontend NÃO DEVE:

- ler o cookie HttpOnly;
- armazenar Refresh Token;
- armazenar Access Token em storage persistente;
- enviar Refresh Token em body/header/query;
- registrar Access Token;
- realizar renovação para origens não previstas.

---

## 9. Relação com outros Resources

```text
RES-001 Login
   ↓
sessão + refresh cookie
   ↓
RES-106 Renovar token (até 3 vezes)
   ↓
RES-107 Logout
```

Alteração ou recuperação de senha invalida a sessão e impede nova renovação.
