# Integração da recuperação de senha

Este guia registra a implementação de RES-003 v1.1 e Journey002 na Pastoral da Juventude. A recuperação confirma quatro dados do cadastro e entrega uma senha temporária na própria tela. Depois, o usuário deve entrar com ela, trocar a senha e realizar um novo login. Não há envio por e-mail nem autenticação automática.

## Contrato e dados

O frontend chama `POST /api/v1/autenticacao/recuperar-senha` com nome, e-mail, nascimento e paróquia. A interface recebe a data em dd/mm/aaaa e envia YYYY-MM-DD. O backend normaliza o e-mail e compara os nomes com NFC, espaços normalizados e sem diferença de caixa, preservando os acentos. Todos os dados devem corresponder à mesma pessoa. Ausência, ambiguidade e divergência retornam o mesmo 404, sem apontar o campo divergente.

200 entrega senhaTemporaria, expiraEm e trocaSenhaObrigatoria true, com Cache-Control no-store. A credencial vale por 15 minutos. O banco guarda apenas Argon2id, mantém o hash definitivo anterior e invalida todas as sessões em uma transação auditada. Uma senha temporária ainda válida e com menos de cinco falhas produz 409 sem ser reexibida.

Contas inativas recebem 403. Contas bloqueadas podem recuperar e concluir a troca com token restrito, mas continuam bloqueadas para login normal. Redis permite três tentativas por e-mail em 30 minutos; a terceira inicia um bloqueio de 30 minutos para os próximos pedidos. O limite complementar é de 30 tentativas por IP no mesmo intervalo. 429 informa Retry-After. Redis ou persistência indisponíveis retornam 503.

## Interface e credenciais

A referência visual é a página 150:1274 do Figma Q3XWkh0oPSxbEM5o3NnAXR, nos estados desktop, tablet e celular. A composição reutiliza o Design System de acesso, Figtree e Fraunces e a marca PastorApp já integrada, substituindo o placeholder PJ. Controles móveis mantêm o alvo de toque de 44 px.

A senha temporária fica apenas no estado local do componente. Não é enviada ao Redux, storage, URL ou telemetria. Copiar exige ação explícita. Sair ou recarregar descarta o valor; expirar oculta a credencial e libera novo pedido. Não compartilhar respostas da API nem capturas com senhas. A recuperação limpa a sessão em memória do frontend e não cria uma nova.

A migration V022 adiciona o UUID opcional da credencial temporária. Cada recuperação gera um UUID novo. O JWT restrito vincula UUID e expiração, revalidados sob lock na conclusão da troca. Tokens de uma recuperação substituída não podem consumir a nova, mesmo com expirações iguais. A criação de sessão normal revalida o hash e o estado sob o mesmo lock, impedindo login concorrente com senha anterior.

## Ordem de integração

1. Integrar o PR do backend em develop e aguardar CI, migration V022 e deploy aprovados.
2. Integrar o PR E2E em develop e aguardar sua CI estática. O merge E2E não publica o frontend.
3. Integrar o PR frontend em develop e aguardar a publicação Vercel, o deploy e a execução E2E do ambiente.
4. Integrar este guia na main do repositório de documentação.

Os merges pertencem ao operador. Não redefinir a massa automaticamente. A recuperação exige nascimento e paróquia completos no cadastro; fixtures sem esses dados retornam 404. O reset existente continua explícito, altera as duas contas e invalida suas sessões. Não executar durante testes concorrentes.

## Evidências e validação pública

Os testes locais incluem 168 testes de backend, 183 de frontend e 22 cenários Playwright de Login, recuperação e restauração com respostas HTTP controladas. A cobertura global do backend é 95,88% de instruções e 89,59% de branches; a do frontend é 99,63% e 98,51%, respectivamente. Builds e Storybook são verificados pelos gates. Os números descrevem esta implementação e devem ser atualizados se os testes mudarem.

A CI do backend acrescenta um ciclo HTTP com PostgreSQL e Redis descartáveis, Argon2id e RS256 reais: recuperação concorrente 200/409, invalidação de refresh, recusa de senha anterior, credencial substituída, troca 204, login definitivo, status preservados e reserva Redis concorrente. O script exige CI true e não deve ser executado no Ubuntu de desenvolvimento. Os testes de navegador controlados não comprovam a integração pública real.

Depois dos deploys, o operador deve usar uma conta autorizada com os quatro dados completos. Abrir Esqueci minha senha, preencher os dados e confirmar HTTP 200. Copiar a senha temporária sem compartilhá-la, voltar ao Login, concluir a troca com HTTP 204 e entrar com a definitiva. Recarregar deve restaurar a sessão pelo RES-106. Registrar somente resultado, data e SHAs implantados, sem credenciais. A validação pública desta funcionalidade permanece pendente até essa execução.

## Referências

Contrato funcional: docs/resources/res-003-autenticacao-recuperar-senha/v1.1 no repositório pastoral-juventude-docs. Detalhes técnicos: backend/docs/RECUPERACAO-SENHA.md e frontend/docs/RECUPERACAO-SENHA.md. Cenários: e2e/tests/password-recovery.spec.ts.
