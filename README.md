# MVP Pastoral da Juventude — Documentação

Repositório oficial da documentação funcional e técnica do MVP da Pastoral da Juventude.

## Organização

- `docs/discovery/`: visão, escopo, jornadas e requisitos;
- `docs/architecture/`: arquitetura da aplicação e infraestrutura;
- `docs/adrs/`: registros de decisões arquiteturais;
- `docs/resources/`: especificações versionadas dos Resources/APIs;
- `docs/database/`: modelo e especificação física do banco;
- `docs/diagrams/`: diagramas da solução;
- `docs/operations/`: publicação, deploy e procedimentos operacionais integrados.

## Publicação e deploy

Consulte [o guia de publicação e deploy](docs/operations/publicacao-deploy.md) para o fluxo entre os cinco repositórios, checks da CI, imagens GHCR, preparação do Ubuntu, ativação do deploy, evidências e recuperação.

As alterações devem ser realizadas preferencialmente por branch e Pull Request. Quando aplicável, documentos Markdown e DOCX devem permanecer semanticamente equivalentes.

## Jornada de Login

Consulte [a integração da jornada](docs/operations/integracao-login.md) para decisões de persistência, Retry-After, troca obrigatória e validação entre repositórios. A versão [DOCX](docs/operations/integracao-login.docx) contém o mesmo texto.
