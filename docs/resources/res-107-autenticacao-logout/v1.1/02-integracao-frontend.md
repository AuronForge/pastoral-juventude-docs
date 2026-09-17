# RES-107 — Integração Frontend — Logout

**Versão:** 1.1  
**Endpoint:** `POST /api/v1/autenticacao/logout`

---

## 1. Objetivo

O frontend utiliza este Resource para encerrar explicitamente a sessão atual e limpar o estado autenticado local.

O backend é responsável por invalidar a sessão e expirar o cookie `refresh_token`.

---

## 2. Request

Exemplo:

```http
POST /api/v1/autenticacao/logout
Authorization: Bearer <accessToken>
Origin: https://frontend.exemplo
X-Correlation-Id: <uuid opcional>
Cookie: refresh_token=<gerenciado pelo navegador>
```

Não enviar body.

A chamada deve incluir credenciais do navegador:

```typescript
credentials: 'include'
```

---

## 3. Sucesso

HTTP `204 No Content`.

O backend expira o cookie:

```text
Set-Cookie: refresh_token=; Max-Age=0; Path=/api/v1/autenticacao; HttpOnly; SameSite=Lax
```

### Comportamento obrigatório do frontend

Após `204`:

- apagar Access Token da memória;
- limpar estado autenticado;
- limpar cache de dados privados;
- navegar para área pública/login;
- não tentar usar o Access Token anterior;
- não tentar renovar a sessão encerrada.

---

## 4. Logout idempotente

A segunda chamada também retorna `204`.

O frontend pode tratar logout como concluído mesmo que:

- o Access Token já tenha expirado;
- a sessão tenha sido substituída;
- o cookie já tenha sido removido;
- a sessão já tenha sido invalidada.

Isso simplifica o encerramento local sem exigir distinção do estado remoto.

---

## 5. Erros

| HTTP | Código | Comportamento |
|---:|---|---|
| 400 | `CORRELATION_ID_INVALIDO` | gerar novo correlation id, se apropriado |
| 400 | `DADOS_INCONSISTENTES` | corrigir request |
| 403 | `ACESSO_NEGADO` | tratar origem/política de segurança |
| 500 | `ERRO_INTERNO` | limpar estado local com cautela e informar falha |
| 503 | `SERVICO_INDISPONIVEL` | informar indisponibilidade e evitar assumir invalidação remota sem política definida |

Para respostas 204, o frontend sempre deve concluir o logout local.

---

## 6. Usuário bloqueado/inativo

O frontend NÃO DEVE impedir logout porque o usuário foi bloqueado ou inativado.

Se o backend ainda reconhecer a sessão, ele a invalida. Se já estiver invalidada, o resultado continua idempotente.

---

## 7. Concorrência

Se duas partes da aplicação dispararem logout simultaneamente:

- ambas podem receber `204`;
- o frontend deve convergir para estado desautenticado;
- não deve tentar restaurar sessão com base em uma resposta concorrente.

---

## 8. Segurança frontend

O frontend NÃO DEVE:

- tentar apagar cookie HttpOnly diretamente via JavaScript;
- persistir Access Token;
- continuar requests autenticadas após logout;
- chamar renovação após logout;
- registrar tokens em logs.

---

## 9. Fluxo

```text
Usuário solicita logout
        ↓
RES-107 POST /logout
        ↓
204 No Content
        ↓
backend invalida sessão + expira refresh cookie
        ↓
frontend limpa Access Token + estado privado
        ↓
navegação para login/área pública
```
