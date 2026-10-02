# Integração da jornada de Login

A Jornada de Login conecta RES-001 e a troca obrigatória de RES-002 ao frontend. Este documento complementa os contratos v1.1 e registra as decisões aprovadas para integrar backend, frontend e testes E2E. A aprovação de cada PR não autoriza seu merge nem a promoção para produção.

## Decisões de acesso

O Login envia somente email e senha para POST /api/v1/autenticacao/login. O frontend normaliza o e-mail com trim e lowercase. Senhas e tokens não são gravados em localStorage ou sessionStorage. O Access Token e o token TROCA_SENHA permanecem em memória; o cookie de Refresh Token é gerenciado pelo navegador com credentials include.

Continuar conectado permanece desabilitado. O contrato conserva Access Token de 900 segundos e sessão absoluta de uma hora. Recuperação de senha, logout, renovação explícita e troca voluntária não integram esta entrega. Ao recarregar a aplicação, o frontend retorna ao Login, pois não há reconstrução de sessão neste escopo.

A composição visual usa os frames aprovados do Figma e BrandLockup oficial. Login é acessível em /login. A rota / é protegida e conserva a página inicial existente; esta entrega não implementa um novo dashboard. O token TROCA_SENHA dá acesso somente a /alterar-senha. A expiração local usa expiresIn da API, sem ampliar a validade ao navegar entre rotas. O backend continua responsável por validar tokens, sessão e permissões em cada recurso.

## Contrato do bloqueio por tentativas

Na resposta HTTP 429 de RES-001, o backend envia Retry-After em segundos inteiros positivos. O valor é o teto do maior TTL restante dos bloqueios de identificador e origem no Redis. Os limites existentes permanecem cinco falhas e janela de quinze minutos. O prazo é calculado ao responder; não deve ser substituído por quinze minutos fixos no cliente. Um bloqueio sem expiração é tratado como indisponibilidade de infraestrutura, sem prazo inventado.

O envelope JSON permanece inalterado. Em acesso entre origens, CORS expõe Retry-After e X-Correlation-Id somente para as origens permitidas. Para este contrato, Retry-After descreve o tempo restante observado; não é uma promessa de sucesso após o prazo, pois outras tentativas podem modificar o bloqueio.

O frontend interpreta segundos ou HTTP-date válidos e impede novos envios enquanto o prazo recebido estiver ativo. Os valores preenchidos são preservados. Cabeçalho ausente ou inválido não produz contador nem duração artificial: a mensagem orienta nova tentativa mais tarde, e a API permanece responsável por rejeitar tentativas bloqueadas.

## Respostas e comportamento da interface

No sucesso Bearer, o frontend guarda o Access Token em memória e navega para a rota inicial protegida. No sucesso TROCA_SENHA, guarda apenas o token restrito e navega para criar a senha definitiva, sem tratar esse token como sessão normal.

Em 401 CREDENCIAIS_INVALIDAS, o feedback é genérico, somente a senha é apagada e o foco retorna a esse campo. Em bloqueio, inatividade ou indisponibilidade de senha temporária, a interface orienta contato com o coordenador, sem implementar recuperação. Falhas de rede e 5xx preservam os campos e permitem repetição. Envios concorrentes são impedidos.

A nova senha deve ter de oito a 128 caracteres, sem espaços, e confirmação idêntica. A requisição de troca obrigatória envia novaSenha e confirmacaoNovaSenha, com Authorization Bearer do token restrito; senhaAtual não é enviada. Erros de validação aparecem junto dos campos. Falhas de rede ou serviço preservam a entrada. Um 401 descarta as senhas e oferece reiniciar o acesso.

Após HTTP 204, o token restrito é descartado e o frontend retorna ao Login com a mensagem Senha alterada e instrução para entrar com a nova senha. Não há autenticação automática após a troca. A expiração oferece retorno ao Login com alerta usando componentes já aprovados; o diálogo do Figma será integrado quando seu componente estiver aprovado.

## Validação e integração dos repositórios

Backend valida TTL, arredondamento, expiração, falha Redis, cabeçalho HTTP, CORS, autenticação e troca. OpenAPI e coleções Postman são regenerados a partir do código. Frontend regenera tipos desse OpenAPI e valida navegação, memória de sessão, formulários, erros, foco e prazo recebido. A cobertura mínima é 85 por cento nas quatro métricas para frontend e backend.

O repositório E2E separa cenários de navegador com respostas controladas de testes com API real. Os primeiros verificam loading, 401, 429, rede e 5xx sem depender de credenciais. Os testes reais exigem contas exclusivas em domínio regressao.invalid, ambiente local, desenvolvimento ou homologação e ativação explícita da suíte. Senhas vêm de variáveis protegidas e nunca de arquivos versionados. Relatórios e traces de autenticação não devem expor credenciais; os testes reais desabilitam captura de trace e vídeo.

A massa de primeiro acesso é mutável e precisa ser reinicializada pelo procedimento regressivo do backend antes de cada execução. O teste não redefine senhas nem reseta Redis de contas reais. A suíte mutável usa um único worker, sem retries automáticos, e uma origem exclusiva para evitar interferência nos limites de tentativas.

Os PRs são separados por repositório. Ordem proposta de integração após autorização: documentação e contrato do backend; frontend; E2E. A infraestrutura atual mantém proxy de mesma origem para /api/v1 e rejeita versões de API não configuradas. URLs, imagens por SHA e liberação do executor devem ser conferidas no deploy de desenvolvimento. Só promover develop para release depois dos testes reais em ambiente com massa exclusiva; promover para main depende de nova autorização e validação em homologação.

## Referências

Frontend: https://github.com/AuronForge/pastoral-juventude-frontend

Backend: https://github.com/AuronForge/pastoral-juventude-backend

E2E: https://github.com/AuronForge/pastoral-juventude-e2e

Figma Login: https://www.figma.com/design/Q3XWkh0oPSxbEM5o3NnAXR?node-id=86-433

Figma troca obrigatória: https://www.figma.com/design/Q3XWkh0oPSxbEM5o3NnAXR?node-id=87-636
