# RES-106 — Casos de Teste — Renovar Access Token

**Versão:** 1.1  
**Objetivo:** base Gherkin reutilizável para futura geração de `.feature`.  
**Observação:** este artefato NÃO gera arquivos `.feature`.

---

## CT-AUT-040 — Renovação com sucesso

```gherkin
@CT-AUT-040
@UC-AUT-022
@CA-001
@e2e
@sessao
Cenário: Renovar Access Token com Refresh Token válido
  Dado que existe uma sessão ativa
  E "quantidadeRefresh" é menor que 3
  E o cookie "refresh_token" corresponde ao hash da sessão
  Quando o cliente enviar POST para "/api/v1/autenticacao/renovar-token"
  Então a resposta deve possuir status 200
  E deve retornar um novo "accessToken"
  E deve enviar um novo cookie "refresh_token"
  E o hash persistido deve ser substituído
  E "quantidadeRefresh" deve ser incrementado em 1
  E deve existir auditoria "RENOVAR_TOKEN"
```

---

## CT-AUT-041 — Token rotacionado não pode ser reutilizado

```gherkin
@CT-AUT-041
@UC-AUT-023
@CA-002
@CA-009
@e2e
@seguranca
Cenário: Rejeitar Refresh Token anterior após rotação
  Dado que um Refresh Token já foi usado com sucesso
  E um novo Refresh Token foi emitido
  Quando o cliente tentar reutilizar o token anterior
  Então a resposta deve possuir status 401
  E o código deve ser "REFRESH_TOKEN_INVALIDO"
  E nenhuma nova rotação deve ocorrer
```

---

## CT-AUT-042 — Limite de três renovações

```gherkin
@CT-AUT-042
@UC-AUT-024
@CA-005
@CA-006
@e2e
Cenário: Exigir novo login após três renovações
  Dado que a sessão possui "quantidadeRefresh" igual a 3
  Quando o cliente tentar renovar novamente
  Então a resposta deve possuir status 401
  E o código deve ser "SESSAO_EXPIRADA"
  E a sessão deve ser invalidada com motivo "REFRESH_LIMITE_ATINGIDO"
  E o cookie "refresh_token" deve ser expirado
```

---

## CT-AUT-043 — Sessão absoluta não é estendida

```gherkin
@CT-AUT-043
@CA-004
@e2e
@sessao
Cenário: Preservar expiração absoluta durante renovação
  Dado que a sessão possui uma "dataExpiracao"
  Quando uma renovação for concluída com sucesso
  Então "dataExpiracao" deve permanecer inalterada
```

---

## CT-AUT-044 — Access Token limitado pela sessão

```gherkin
@CT-AUT-044
@CA-007
@e2e
@sessao
Cenário: Não emitir Access Token além da expiração absoluta
  Dado que restam menos de quinze minutos para a expiração da sessão
  Quando o cliente renovar o token
  Então "expiresIn" deve ser menor que 900
  E a expiração do Access Token não deve ultrapassar a expiração da sessão
```

---

## CT-AUT-045 — Cookie ausente

```gherkin
@CT-AUT-045
@CA-008
@e2e
Cenário: Rejeitar renovação sem cookie de Refresh Token
  Dado que a request não possui cookie "refresh_token"
  Quando o cliente solicitar renovação
  Então a resposta deve possuir status 400
  E o código deve ser "TOKEN_REFRESH_AUSENTE"
```

---

## CT-AUT-046 — Sessão substituída por novo login

```gherkin
@CT-AUT-046
@UC-AUT-025
@CA-011
@e2e
@sessao
Cenário: Informar sessão substituída
  Dado que a sessão foi invalidada com motivo "NOVO_LOGIN"
  Quando o cliente antigo tentar renovar o token
  Então a resposta deve possuir status 409
  E o código deve ser "SESSAO_SUBSTITUIDA"
```

---

## CT-AUT-047 — Sessão expirada

```gherkin
@CT-AUT-047
@UC-AUT-026
@CA-010
@e2e
Cenário: Rejeitar renovação após expiração absoluta
  Dado que a sessão atingiu "dataExpiracao"
  Quando o cliente tentar renovar
  Então a resposta deve possuir status 401
  E o código deve ser "SESSAO_EXPIRADA"
  E nenhum token deve ser emitido
```

---

## CT-AUT-048 — Usuário bloqueado

```gherkin
@CT-AUT-048
@CA-012
@e2e
Cenário: Impedir renovação para usuário bloqueado
  Dado que o usuário possui status "BLOQUEADO"
  Quando o cliente tentar renovar
  Então a resposta deve possuir status 403
  E o código deve ser "USUARIO_BLOQUEADO"
  E nenhum token deve ser emitido
```

---

## CT-AUT-049 — Usuário inativo

```gherkin
@CT-AUT-049
@CA-013
@e2e
Cenário: Impedir renovação para usuário inativo
  Dado que o usuário possui status "INATIVO"
  Quando o cliente tentar renovar
  Então a resposta deve possuir status 403
  E o código deve ser "USUARIO_INATIVO"
```

---

## CT-AUT-050 — Origem não autorizada

```gherkin
@CT-AUT-050
@CA-014
@seguranca
@e2e
Cenário: Rejeitar renovação originada de origem não permitida
  Dado que a request possui cookie "refresh_token"
  Mas a origem não pertence à allow-list do ambiente
  Quando o cliente solicitar renovação
  Então a renovação deve ser rejeitada
  E o Refresh Token não deve ser rotacionado
```

---

## CT-AUT-051 — Não vazar tokens

```gherkin
@CT-AUT-051
@CA-015
@seguranca
Cenário: Não registrar tokens durante renovação
  Quando ocorrer uma renovação
  Então logs e auditoria não devem conter o Refresh Token
  E não devem conter o hash do Refresh Token
  E não devem conter o Access Token
```

---

## CT-AUT-052 — Concorrência de refresh

```gherkin
@CT-AUT-052
@UC-AUT-027
@CA-019
@concorrencia
@e2e
Cenário: Permitir somente uma rotação para o mesmo Refresh Token
  Dado que duas requests utilizam o mesmo Refresh Token
  E são iniciadas concorrentemente
  Quando forem processadas
  Então no máximo uma request deve concluir com status 200
  E as demais devem retornar 401 com código "REFRESH_TOKEN_INVALIDO"
  E "quantidadeRefresh" deve ser incrementado apenas uma vez
```

---

## CT-AUT-053 — Auditoria

```gherkin
@CT-AUT-053
@CA-016
@CA-017
@auditoria
Cenário: Registrar sucesso e falha de renovação
  Quando uma renovação for concluída com sucesso
  Então deve existir auditoria "RENOVAR_TOKEN"
  Quando uma renovação for rejeitada
  Então deve existir auditoria "RENOVAR_TOKEN_FALHA"
```

---

## CT-AUT-054 — Correlation ID

```gherkin
@CT-AUT-054
@CA-020
@observabilidade
@e2e
Cenário: Retornar correlation id na renovação
  Quando o cliente solicitar renovação
  Então a resposta deve conter "X-Correlation-Id"
  E o valor deve ser UUID válido
```
