# RES-003 — Integração Frontend — Recuperar senha

**Versão:** 1.1  
**Endpoint:** `POST /api/v1/autenticacao/recuperar-senha`

---

## 1. Objetivo

Este documento descreve como o frontend coleta os dados de recuperação, trata a senha temporária devolvida uma única vez e direciona o usuário para o fluxo de login/troca obrigatória.

---

## 2. Formulário

Campos:

- nome;
- e-mail;
- data de nascimento;
- nome da paróquia.

Request:

```json
{
  "nome": "Nome Completo",
  "email": "usuario@exemplo.com",
  "dataNascimento": "2000-01-15",
  "nomeParoquia": "Paróquia Exemplo"
}
```

O frontend DEVE:

- aplicar `trim` aos campos textuais;
- normalizar visualmente espaços excedentes no nome da paróquia;
- enviar data como `YYYY-MM-DD`;
- não enviar UUID de paróquia;
- não enviar `Idempotency-Key`.

---

## 3. Sucesso

HTTP `200`.

```json
{
  "senhaTemporaria": "<valor-gerado>",
  "expiraEm": "2026-09-16T11:15:00Z",
  "trocaSenhaObrigatoria": true
}
```

### Comportamento obrigatório do frontend

- exibir a senha temporária ao usuário;
- informar que ela expira em 15 minutos;
- informar que ela só serve para iniciar o fluxo de troca obrigatória;
- NÃO armazenar a senha temporária em localStorage/sessionStorage;
- NÃO registrar a senha temporária em telemetry/logs;
- permitir copiar o valor explicitamente pelo usuário;
- orientar o usuário a seguir para login;
- após sair da tela, não depender da API para recuperar novamente a senha temporária.

---

## 4. Recuperação em andamento

HTTP `409`.

Código:

```text
RECUPERACAO_SENHA_EM_ANDAMENTO
```

Comportamento:

- informar que já existe uma recuperação válida;
- não afirmar que uma nova senha foi gerada;
- não esperar que o backend retorne a senha temporária anterior;
- orientar o usuário a usar a senha temporária já recebida ou aguardar expiração para iniciar nova recuperação.

---

## 5. Dados não localizados

HTTP `404`.

Código:

```text
DADOS_RECUPERACAO_NAO_LOCALIZADOS
```

O frontend DEVE apresentar mensagem genérica.

NÃO DEVE indicar:

- nome incorreto;
- e-mail incorreto;
- data de nascimento incorreta;
- paróquia incorreta;
- existência ou inexistência do usuário.

---

## 6. Usuário bloqueado

Um usuário bloqueado pode receber `200` na recuperação.

O frontend NÃO DEVE interpretar isso como desbloqueio.

Após trocar a senha, login normal continuará sujeito ao status `BLOQUEADO`.

---

## 7. Usuário inativo

HTTP `403 USUARIO_INATIVO`.

O frontend deve informar que a conta está inativa e que recuperação de senha não reativa o cadastro.

---

## 8. Rate limit

HTTP `429 LIMITE_TENTATIVAS_EXCEDIDO`.

Comportamento recomendado:

- desabilitar novas submissões;
- informar que novas tentativas estão temporariamente bloqueadas;
- não executar retry automático agressivo.

Payloads com body maior que 1 MB recebem `413 PAYLOAD_MUITO_GRANDE`; o frontend não deve reenviar automaticamente a mesma request.

---

## 9. Segurança frontend

O frontend NÃO DEVE:

- persistir a senha temporária;
- enviar a senha temporária para analytics;
- capturar a senha temporária em logs de erro;
- armazenar qualquer senha em browser storage;
- tentar renovar sessão após recuperação, pois a sessão anterior foi invalidada.

---

## 10. Próximo passo

Fluxo esperado:

```text
RES-003 Recuperar senha
        ↓
senha temporária
        ↓
RES-001 Login
        ↓
token TROCA_SENHA
        ↓
RES-002 Alterar senha
        ↓
novo login normal
```
