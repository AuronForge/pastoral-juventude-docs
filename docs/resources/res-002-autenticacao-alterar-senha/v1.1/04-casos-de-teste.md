# RES-002 — Casos de Teste — Alterar senha

**Versão:** 1.1  
**Objetivo:** base Gherkin reutilizável para futura geração de `.feature`.  
**Observação:** este artefato NÃO gera arquivos `.feature`.

---

## CT-AUT-015 — Troca obrigatória com sucesso

```gherkin
@CT-AUT-015
@UC-AUT-009
@CA-001
@e2e
@seguranca
Cenário: Definir senha definitiva com token de troca válido
  Dado que o usuário possui token "TROCA_SENHA" válido
  E "trocaSenhaObrigatoria" é verdadeiro
  Quando enviar nova senha válida e confirmação correspondente
  Então a resposta deve possuir status 204
  E "trocaSenhaObrigatoria" deve se tornar falso
  E "senhaTemporariaHash" deve ser removida
  E "senhaTemporariaExpiraEm" deve ser removida
  E "senhaTemporariaUtilizadaEm" deve ser preenchida
  E nenhuma sessão normal deve ser criada
  E deve existir auditoria com ação "TROCAR_SENHA"
```

---

## CT-AUT-016 — Token restrito não pode ser reutilizado

```gherkin
@CT-AUT-016
@UC-AUT-011
@CA-004
@e2e
@seguranca
Cenário: Rejeitar token de troca após conclusão da troca obrigatória
  Dado que o usuário já concluiu a troca obrigatória
  E o token "TROCA_SENHA" ainda não expirou criptograficamente
  Quando tentar alterar a senha novamente com o mesmo token
  Então a resposta deve possuir status 401
  E o código deve ser "TOKEN_INVALIDO"
  E a senha não deve ser alterada
```

---

## CT-AUT-017 — Troca voluntária com sucesso

```gherkin
@CT-AUT-017
@UC-AUT-010
@CA-005
@CA-006
@e2e
@sessao
Cenário: Alterar senha com sessão normal
  Dado que o usuário possui sessão ativa
  E informa a senha atual correta
  E informa uma nova senha válida
  Quando enviar POST para "/api/v1/autenticacao/alterar-senha"
  Então a resposta deve possuir status 204
  E a nova senha deve ser persistida
  E a sessão atual deve ser invalidada com motivo "ALTERACAO_SENHA"
  E o Refresh Token deve ser invalidado
  E o Access Token anterior não deve mais ser aceito
  E deve existir auditoria "TROCAR_SENHA"
  E deve existir auditoria "INVALIDAR_SESSAO"
```

---

## CT-AUT-018 — Senha atual incorreta

```gherkin
@CT-AUT-018
@UC-AUT-013
@CA-007
@e2e
Cenário: Rejeitar senha atual incorreta
  Dado que o usuário possui sessão válida
  Quando informar uma senha atual incorreta
  Então a resposta deve possuir status 401
  E o código deve ser "CREDENCIAIS_INVALIDAS"
  E a senha definitiva não deve ser modificada
  E a sessão deve permanecer ativa
```

---

## CT-AUT-019 — Confirmação divergente

```gherkin
@CT-AUT-019
@CA-008
@e2e
@contrato
Cenário: Rejeitar nova senha e confirmação divergentes
  Quando "novaSenha" e "confirmacaoNovaSenha" forem diferentes
  Então a resposta deve possuir status 400
  E o código deve ser "DADOS_INCONSISTENTES"
  E o erro deve apontar o campo "confirmacaoNovaSenha"
```

---

## CT-AUT-020 — Política de comprimento

```gherkin
@CT-AUT-020
@CA-009
@e2e
@contrato
Esquema do Cenário: Rejeitar senha fora do comprimento permitido
  Quando a nova senha possuir <quantidade> caracteres
  Então a resposta deve possuir status 400
  E o código deve ser "DADOS_INCONSISTENTES"

  Exemplos:
    | quantidade |
    | 7          |
    | 129        |
```

---

## CT-AUT-021 — Espaço não permitido

```gherkin
@CT-AUT-021
@CA-009
@e2e
Cenário: Rejeitar nova senha contendo espaço
  Quando a nova senha contiver espaço
  Então a resposta deve possuir status 400
  E o código deve ser "DADOS_INCONSISTENTES"
```

---

## CT-AUT-022 — Reutilização da senha atual

```gherkin
@CT-AUT-022
@CA-010
@e2e
@seguranca
Cenário: Rejeitar reutilização da senha definitiva atual
  Dado que o usuário possui uma senha definitiva
  Quando informar a mesma senha como nova senha
  Então a resposta deve possuir status 400
  E o código deve ser "DADOS_INCONSISTENTES"
  E o erro deve apontar o campo "novaSenha"
```

---

## CT-AUT-023 — Usuário bloqueado em troca obrigatória

```gherkin
@CT-AUT-023
@UC-AUT-012
@CA-011
@e2e
Cenário: Permitir troca obrigatória sem desbloquear usuário
  Dado que o usuário possui status "BLOQUEADO"
  E possui token "TROCA_SENHA" válido
  Quando alterar a senha com sucesso
  Então a resposta deve possuir status 204
  E o status do usuário deve continuar "BLOQUEADO"
  E nenhuma sessão normal deve ser criada
```

---

## CT-AUT-024 — Usuário inativo

```gherkin
@CT-AUT-024
@CA-012
@e2e
Cenário: Impedir alteração de senha de usuário inativo
  Dado que o usuário possui status "INATIVO"
  Quando tentar alterar a senha
  Então a resposta deve possuir status 403
  E o código deve ser "USUARIO_INATIVO"
  E a senha não deve ser modificada
```

---

## CT-AUT-025 — Não vazar segredos

```gherkin
@CT-AUT-025
@CA-013
@seguranca
Cenário: Não registrar credenciais durante alteração de senha
  Quando ocorrer uma alteração de senha
  Então logs e auditoria não devem conter "senhaAtual"
  E não devem conter "novaSenha"
  E não devem conter "confirmacaoNovaSenha"
  E não devem conter hashes de senha
```

---

## CT-AUT-026 — Concorrência com token restrito

```gherkin
@CT-AUT-026
@concorrencia
@e2e
Cenário: Processar somente uma troca com o mesmo token restrito em chamadas concorrentes
  Dado que duas requisições usam o mesmo token "TROCA_SENHA"
  E ambas são iniciadas concorrentemente
  Quando forem processadas
  Então somente uma alteração deve concluir com sucesso
  E a outra deve ser rejeitada por estado inválido do token ou da troca obrigatória
```

---

## CT-AUT-027 — Payload acima do limite

```gherkin
@CT-AUT-027
@e2e
@contrato
Cenário: Rejeitar payload de alteração de senha acima de 1 MB
  Dado que a request de alteração de senha possui body maior que 1 MB
  Quando a request for enviada
  Então a resposta deve possuir status 413
  E o código deve ser "PAYLOAD_MUITO_GRANDE"
  E a senha não deve ser modificada
```
