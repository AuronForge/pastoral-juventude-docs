# RES-003 — Casos de Teste — Recuperar senha

**Versão:** 1.1  
**Objetivo:** base Gherkin reutilizável para futura geração de `.feature`.  
**Observação:** este artefato NÃO gera arquivos `.feature`.

---

## CT-AUT-027 — Recuperação com sucesso

```gherkin
@CT-AUT-027
@UC-AUT-015
@CA-001
@e2e
@recuperacao
Cenário: Gerar senha temporária com dados de recuperação válidos
  Dado que existe um usuário ativo com os dados informados
  E não existe recuperação válida em andamento
  Quando o cliente enviar POST para "/api/v1/autenticacao/recuperar-senha"
  Então a resposta deve possuir status 200
  E deve retornar "senhaTemporaria"
  E deve retornar "expiraEm"
  E "trocaSenhaObrigatoria" deve ser verdadeiro
  E somente o hash da senha temporária deve ser persistido
  E a senha temporária deve expirar em quinze minutos
  E nenhuma sessão normal deve ser criada
```

---

## CT-AUT-028 — Invalidar sessão durante recuperação

```gherkin
@CT-AUT-028
@CA-005
@e2e
@sessao
Cenário: Invalidar sessão existente ao recuperar senha
  Dado que o usuário possui uma sessão ativa
  Quando concluir recuperação de senha
  Então a sessão deve ser invalidada com motivo "RECUPERACAO_SENHA"
  E o Access Token da sessão anterior não deve mais ser aceito
  E deve existir auditoria "INVALIDAR_SESSAO"
```

---

## CT-AUT-029 — Não localizar dados

```gherkin
@CT-AUT-029
@UC-AUT-016
@CA-009
@e2e
@seguranca
Cenário: Responder genericamente quando os dados não correspondem
  Dado que pelo menos um dado de recuperação não corresponde ao cadastro
  Quando o cliente solicitar recuperação
  Então a resposta deve possuir status 404
  E o código deve ser "DADOS_RECUPERACAO_NAO_LOCALIZADOS"
  E a resposta não deve indicar qual campo divergiu
  E nenhuma senha temporária deve ser gerada
```

---

## CT-AUT-030 — Recuperação em andamento

```gherkin
@CT-AUT-030
@UC-AUT-017
@CA-007
@CA-008
@e2e
Cenário: Não gerar nova senha durante recuperação válida
  Dado que existe uma senha temporária válida e não utilizada
  Quando o cliente solicitar nova recuperação
  Então a resposta deve possuir status 409
  E o código deve ser "RECUPERACAO_SENHA_EM_ANDAMENTO"
  E nenhuma nova senha temporária deve ser gerada
  E a senha temporária existente não deve ser retornada
```

---

## CT-AUT-031 — Usuário bloqueado pode recuperar

```gherkin
@CT-AUT-031
@UC-AUT-018
@CA-010
@e2e
Cenário: Permitir recuperação para usuário bloqueado sem desbloqueá-lo
  Dado que os dados correspondem a um usuário com status "BLOQUEADO"
  Quando o cliente solicitar recuperação
  Então a resposta deve possuir status 200
  E deve ser gerada uma senha temporária
  E o status do usuário deve permanecer "BLOQUEADO"
```

---

## CT-AUT-032 — Usuário inativo

```gherkin
@CT-AUT-032
@UC-AUT-019
@CA-011
@e2e
Cenário: Rejeitar recuperação para usuário inativo
  Dado que os dados correspondem a um usuário com status "INATIVO"
  Quando o cliente solicitar recuperação
  Então a resposta deve possuir status 403
  E o código deve ser "USUARIO_INATIVO"
  E nenhuma senha temporária deve ser criada
```

---

## CT-AUT-033 — Rate limit

```gherkin
@CT-AUT-033
@UC-AUT-020
@CA-012
@e2e
@seguranca
Cenário: Aplicar limite de tentativas de recuperação
  Dado que foram realizadas três tentativas de recuperação dentro de trinta minutos
  Quando ocorrer nova tentativa durante o bloqueio temporário
  Então a resposta deve possuir status 429
  E o código deve ser "LIMITE_TENTATIVAS_EXCEDIDO"
```

---

## CT-AUT-034 — Senha temporária retornada uma única vez

```gherkin
@CT-AUT-034
@CA-006
@CA-008
@seguranca
@e2e
Cenário: Não reexibir senha temporária de recuperação
  Dado que uma recuperação foi criada com sucesso
  E a senha temporária foi retornada na response original
  Quando o usuário solicitar nova recuperação enquanto ela estiver válida
  Então a resposta deve possuir status 409
  E não deve conter "senhaTemporaria"
```

---

## CT-AUT-035 — Não persistir senha em texto puro

```gherkin
@CT-AUT-035
@CA-002
@CA-013
@seguranca
Cenário: Persistir somente o hash da senha temporária
  Quando uma recuperação for criada
  Então a senha temporária em texto puro não deve existir no banco
  E não deve existir em logs
  E não deve existir em auditoria
  E somente o hash deve ser persistido
```

---

## CT-AUT-036 — Auditoria da recuperação

```gherkin
@CT-AUT-036
@CA-014
@e2e
@auditoria
Cenário: Registrar recuperação de senha bem-sucedida
  Quando uma recuperação for concluída com sucesso
  Então deve existir auditoria com ação "RECUPERAR_SENHA"
  E a auditoria não deve conter senha temporária
  E a auditoria não deve conter hash da senha temporária
```

---

## CT-AUT-037 — Correlation ID

```gherkin
@CT-AUT-037
@CA-018
@e2e
@observabilidade
Cenário: Retornar correlation id na recuperação
  Quando o cliente solicitar recuperação
  Então a resposta deve conter o header "X-Correlation-Id"
  E o valor deve ser um UUID válido
```

---

## CT-AUT-038 — Concorrência

```gherkin
@CT-AUT-038
@UC-AUT-021
@CA-016
@concorrencia
@e2e
Cenário: Manter somente uma recuperação válida em solicitações concorrentes
  Dado que duas solicitações válidas para o mesmo usuário são iniciadas concorrentemente
  Quando forem processadas
  Então somente uma senha temporária deve permanecer válida
  E não devem existir duas recuperações válidas simultâneas
```

---

## CT-AUT-039 — Não criar sessão

```gherkin
@CT-AUT-039
@CA-017
@e2e
Cenário: Não autenticar automaticamente após recuperação
  Quando a recuperação for concluída com sucesso
  Então não deve ser criado registro ativo de "SESSAO_USUARIO"
  E não deve ser retornado "accessToken"
  E não deve ser criado cookie "refresh_token"
```

---

## CT-AUT-040 — Payload acima do limite

```gherkin
@CT-AUT-040
@e2e
@contrato
Cenário: Rejeitar payload de recuperação acima de 1 MB
  Dado que a request de recuperação possui body maior que 1 MB
  Quando a request for enviada
  Então a resposta deve possuir status 413
  E o código deve ser "PAYLOAD_MUITO_GRANDE"
  E nenhuma senha temporária deve ser gerada
```
