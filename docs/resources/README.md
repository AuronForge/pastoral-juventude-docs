# Resources de autenticação

Especificações funcionais e técnicas versionadas dos endpoints de autenticação do MVP da Pastoral da Juventude.

| Resource | Endpoint | Versão |
|---|---|---:|
| [RES-001 — Login](./res-001-autenticacao-login/v1.1/) | `POST /api/v1/autenticacao/login` | 1.1 |
| [RES-002 — Alterar senha](./res-002-autenticacao-alterar-senha/v1.1/) | `POST /api/v1/autenticacao/alterar-senha` | 1.1 |
| [RES-003 — Recuperar senha](./res-003-autenticacao-recuperar-senha/v1.1/) | `POST /api/v1/autenticacao/recuperar-senha` | 1.1 |
| [RES-106 — Renovar token](./res-106-autenticacao-renovar-token/v1.1/) | `POST /api/v1/autenticacao/renovar-token` | 1.1 |
| [RES-107 — Logout](./res-107-autenticacao-logout/v1.1/) | `POST /api/v1/autenticacao/logout` | 1.1 |

Cada diretório contém documentação geral, integração frontend, casos de uso e fluxos, casos de teste, contrato OpenAPI e manifesto. Os documentos Markdown e DOCX correspondentes devem permanecer semanticamente equivalentes.

A integração de RES-001 e RES-002 segue também [as decisões da Jornada de Login](../operations/integracao-login.md), incluindo o cabeçalho Retry-After no HTTP 429.
