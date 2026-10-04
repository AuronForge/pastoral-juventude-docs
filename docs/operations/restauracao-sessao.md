# Restauração da sessão de autenticação

A aplicação restaura a sessão normal ao recarregar a página usando o RES-106. O navegador envia o cookie HttpOnly e recebe um novo Access Token, mantido somente em memória. Esta alteração precisa de merge e deploy do backend e do frontend para funcionar em desenvolvimento.

## Contrato e limites

A fonte funcional é o RES-106 v1.1, complementado pelo discovery v1.7 e catálogo v1.4. A rota é POST /api/v1/autenticacao/renovar-token, sem body obrigatório. O Refresh Token é aceito exclusivamente no cookie refresh_token. A origem deve estar na configuração CORS_ORIGINS do backend; no ambiente publicado, incluir https://pastoral-juventude-frontend.vercel.app. O proxy Vercel deve preservar o header Origin.

Cada sucesso rotaciona o cookie, incrementa o contador uma vez e invalida o valor anterior. São permitidas três renovações por sessão, compartilhadas entre abas. Cada reload bem-sucedido consome uma renovação. A duração absoluta é de uma hora desde o login. O JWT vale até 900 segundos, limitado ao tempo restante da sessão; Max-Age do cookie também respeita esse prazo.

O cookie mantém HttpOnly, SameSite=Lax, Path=/api/v1/autenticacao e Secure conforme o ambiente. O banco armazena somente o hash do Refresh Token. Nenhum token, hash ou senha entra em auditoria. A rotação e as auditorias usam uma transação e o mesmo advisory lock por usuário utilizado no login e nas trocas de senha. Roles e permissões ativas são lidas novamente na renovação.

## Comportamento na interface

Ao iniciar a aplicação, as rotas aguardam uma única tentativa de restauração. A tela mostra o LoadingIndicator aprovado e não redireciona antecipadamente para o login. O estado Redux evita requisições duplicadas pelos efeitos do React StrictMode. Não há gravação de credenciais ou tokens em localStorage ou sessionStorage.

Sucesso devolve o acesso à rota solicitada. Cookie ausente retorna 400 TOKEN_REFRESH_AUSENTE e apresenta o login sem aviso de expiração. Token inválido, sessão inválida ou vencida encerram o estado autenticado. Sessão substituída por novo login retorna 409 SESSAO_SUBSTITUIDA e apresenta aviso. Usuário bloqueado ou inativo retorna 403 e apresenta orientação para contatar o coordenador.

Falhas de rede, origem não permitida e indisponibilidade mostram um alerta com ação Tentar novamente. Não há loop de retry. O backend preserva o cookie em falhas de serviço ou rejeição de origem. A tentativa de renovar depois da terceira renovação invalida a sessão com REFRESH_LIMITE_ATINGIDO e exige novo login.

O token TROCA_SENHA continua restrito à memória e não pode restaurar sessão. Recarregar a página durante a troca obrigatória exige reiniciar o login temporário. Não há renovação periódica, logout na interface nesta entrega. O login não oferece a opção de continuar conectado. A expiração do Access Token durante uma página aberta conserva o comportamento existente de retorno ao login.

## Validação e integração

Os testes do backend cobrem contrato HTTP, origem, cookie protegido, auditoria, expiração e limite. A CI executa verify-refresh-session.ts depois da massa no PostgreSQL descartável, usando HTTP Fastify, assinatura e verificação RS256 reais, rotação concorrente e limite absoluto. Não executa reset no Ubuntu nem utiliza as senhas do operador.

Os testes do frontend cobrem espera inicial, StrictMode, respostas tardias, sucesso, erros de sessão e retry. O repositório E2E contém cenários controlados de Chromium e um cenário opt-in com API real que verifica a permanência do login após reload. Testes controlados não comprovam a integração com o ambiente publicado.

Integrar primeiro o backend, confirmar CI e deploy, depois frontend e E2E. O repositório docs utiliza main como base documental; os repositórios de aplicação e E2E utilizam develop. O operador realiza os merges. Para o E2E interno da infraestrutura usar a nova revisão do repositório E2E em DEV_E2E_REF depois do merge; a referência fixa anterior não inclui os novos testes.

Após os deploys, fazer login normal no frontend publicado e recarregar a página. Confirmar HTTP 200 na renovação e permanência na página inicial. Validar as três renovações e o retorno ao login na quarta tentativa. Não enviar capturas que mostrem Authorization, cookies, senhas ou tokens. Registrar o resultado após confirmação do operador.

## Evidência anterior à restauração

O login normal foi confirmado pelo operador em 02/10/2026. A troca obrigatória e o login definitivo foram confirmados em 03/10/2026 após o reset explícito da massa. O deploy 37124243179 concluiu E2E interno e publicação das evidências; a CI 37123894247 passou no commit 5be21940bf7c5bde7a41e0dc0b52807c0d7c7e79. O operador informou “Funcionou perfeitamente!”. O HTTP 204 era esperado no roteiro e não foi capturado independentemente. Esta evidência não valida a nova restauração de sessão.
