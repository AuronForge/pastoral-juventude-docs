# RES-002 — Integração Frontend — Alterar senha

**Versão:** 1.1  
**Endpoint:** `POST /api/v1/autenticacao/alterar-senha`

---

## 1. Objetivo

O frontend possui dois fluxos para o mesmo endpoint:

1. troca obrigatória usando token `TROCA_SENHA`;
2. troca voluntária usando Access Token normal.

---

## 2. Troca obrigatória

Authorization:

```text
Authorization: Bearer <tokenTrocaSenha>
```

Request:

```json
{
  "novaSenha": "NovaSenha123!",
  "confirmacaoNovaSenha": "NovaSenha123!"
}
```

O frontend NÃO DEVE enviar `senhaAtual`.

### Sucesso

HTTP `204`.

Após sucesso:

- apagar `tokenTrocaSenha` da memória;
- navegar para login;
- não tentar entrar diretamente na área autenticada;
- exibir confirmação de senha alterada.

---

## 3. Troca voluntária

Authorization:

```text
Authorization: Bearer <accessToken>
```

Request:

```json
{
  "senhaAtual": "SenhaAtual123!",
  "novaSenha": "NovaSenha456!",
  "confirmacaoNovaSenha": "NovaSenha456!"
}
```

### Sucesso

HTTP `204`.

O backend invalida a sessão e expira `refresh_token`.

Após sucesso, o frontend DEVE:

- apagar Access Token da memória;
- tratar o usuário como desautenticado;
- navegar para login;
- não tentar renovar a sessão antiga.

---

## 4. Regras de formulário

O frontend DEVE validar antes do envio:

- nova senha obrigatória;
- confirmação obrigatória;
- 8–128 caracteres;
- sem espaços;
- nova senha e confirmação iguais.

No fluxo voluntário:

- senha atual obrigatória.

O frontend NÃO DEVE persistir qualquer senha.

---

## 5. Erros

| HTTP | Código | Comportamento sugerido |
|---:|---|---|
| 400 | `DADOS_INCONSISTENTES` | exibir erros de campo |
| 400 | `CORRELATION_ID_INVALIDO` | regenerar correlation id |
| 401 | `TOKEN_INVALIDO` | redirecionar para autenticação adequada |
| 401 | `TOKEN_EXPIRADO` | primeiro acesso: refazer login/recuperação; voluntário: novo login |
| 401 | `CREDENCIAIS_INVALIDAS` | informar que a senha atual não confere |
| 401 | `SESSAO_INVALIDA` | limpar sessão local e ir para login |
| 401 | `SESSAO_EXPIRADA` | limpar sessão local e ir para login |
| 403 | `USUARIO_INATIVO` | informar que a conta está inativa |
| 413 | `PAYLOAD_MUITO_GRANDE` | não reenviar o payload e informar erro de requisição |
| 500 | `ERRO_INTERNO` | mensagem genérica com correlationId |
| 503 | `SERVICO_INDISPONIVEL` | solicitar nova tentativa posteriormente |

---

## 6. Usuário bloqueado

No fluxo obrigatório decorrente de recuperação, o usuário bloqueado pode trocar a senha.

Após sucesso:

- NÃO considerar usuário autenticado;
- NÃO indicar que o bloqueio foi removido;
- direcionar para login, onde o status bloqueado continuará impedindo sessão normal.

---

## 7. Segurança frontend

NÃO DEVE:

- persistir senha em storage;
- persistir `tokenTrocaSenha` além da memória da aplicação;
- armazenar Access Token em localStorage/sessionStorage;
- logar payload de alteração de senha;
- exibir tokens em mensagens de erro;
- tentar reaproveitar token após sucesso.
