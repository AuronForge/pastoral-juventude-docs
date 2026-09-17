# RES-107 — Casos de Teste — Logout

**Versão:** 1.1  
**Objetivo:** base Gherkin reutilizável para futura geração de `.feature`.  
**Observação:** este artefato NÃO gera arquivos `.feature`.

---

## CT-AUT-055 — Logout com sucesso

```gherkin
@CT-AUT-055
@UC-AUT-028
@CA-001
@e2e
@sessao
Cenário: Encerrar uma sessão ativa
  Dado que o usuário possui uma sessão ativa
  Quando enviar POST para "/api/v1/autenticacao/logout"
  Então a resposta deve possuir status 204
  E a sessão deve se tornar inativa
  E o motivo de invalidação deve ser "LOGOUT"
  E o cookie "refresh_token" deve ser expirado
  E o Access Token da sessão anterior não deve mais ser aceito
```

---

## CT-AUT-056 — Auditoria do logout

```gherkin
@CT-AUT-056
@CA-012
@CA-013
@auditoria
Cenário: Auditar encerramento efetivo da sessão
  Dado que a sessão está ativa
  Quando o logout for concluído
  Então deve existir auditoria com ação "LOGOUT"
  E deve existir auditoria com ação "INVALIDAR_SESSAO"
  E o motivo seguro deve ser "LOGOUT"
```

---

## CT-AUT-057 — Logout repetido

```gherkin
@CT-AUT-057
@UC-AUT-029
@CA-006
@e2e
@idempotencia
Cenário: Repetir logout sem erro funcional
  Dado que a sessão já foi encerrada
  Quando o cliente solicitar logout novamente
  Então a resposta deve possuir status 204
  E nenhuma nova sessão deve ser criada
```

---

## CT-AUT-058 — Preservar motivo histórico

```gherkin
@CT-AUT-058
@UC-AUT-030
@CA-007
@e2e
Cenário: Não sobrescrever motivo anterior de invalidação
  Dado que a sessão foi invalidada com motivo "NOVO_LOGIN"
  Quando o cliente antigo solicitar logout
  Então a resposta deve possuir status 204
  E o motivo persistido deve continuar "NOVO_LOGIN"
```

---

## CT-AUT-059 — Sessão expirada

```gherkin
@CT-AUT-059
@UC-AUT-031
@CA-011
@e2e
Cenário: Tratar logout de sessão expirada como concluído
  Dado que a sessão já expirou
  Quando o cliente solicitar logout
  Então a resposta deve possuir status 204
  E o cookie "refresh_token" deve ser expirado
```

---

## CT-AUT-060 — Usuário bloqueado

```gherkin
@CT-AUT-060
@CA-008
@e2e
Cenário: Permitir logout de usuário bloqueado
  Dado que o usuário está "BLOQUEADO"
  E ainda existe sessão ativa identificável
  Quando solicitar logout
  Então a resposta deve possuir status 204
  E a sessão deve ser invalidada
```

---

## CT-AUT-061 — Usuário inativo

```gherkin
@CT-AUT-061
@CA-009
@e2e
Cenário: Permitir logout de usuário inativo
  Dado que o usuário está "INATIVO"
  E ainda existe sessão identificável
  Quando solicitar logout
  Então a resposta deve possuir status 204
```

---

## CT-AUT-062 — Não vazar tokens

```gherkin
@CT-AUT-062
@CA-015
@seguranca
Cenário: Não registrar tokens durante logout
  Quando ocorrer logout
  Então logs e auditoria não devem conter o Access Token
  E não devem conter o Refresh Token
  E não devem conter o hash do Refresh Token
```

---

## CT-AUT-063 — Logout concorrente

```gherkin
@CT-AUT-063
@UC-AUT-032
@CA-016
@concorrencia
@e2e
Cenário: Convergir duas chamadas simultâneas para sessão inativa
  Dado que duas requisições de logout são iniciadas para a mesma sessão ativa
  Quando forem processadas
  Então ambas podem retornar status 204
  E a sessão deve terminar inativa
  E somente uma transição ativa para inativa deve ser persistida
  E a auditoria de invalidação não deve ser duplicada para a mesma transição
```

---

## CT-AUT-064 — Origin não autorizada

```gherkin
@CT-AUT-064
@CA-018
@seguranca
@e2e
Cenário: Rejeitar logout de origem não autorizada antes de alterar a sessão
  Dado que a origem da request não pertence à allow-list
  Quando o cliente solicitar logout
  Então a resposta deve possuir status 403
  E a sessão não deve ser alterada por essa request
```

---

## CT-AUT-065 — Correlation ID

```gherkin
@CT-AUT-065
@CA-017
@observabilidade
@e2e
Cenário: Retornar correlation id no logout
  Quando o cliente solicitar logout
  Então a resposta deve conter "X-Correlation-Id"
  E o valor deve ser UUID válido
```
