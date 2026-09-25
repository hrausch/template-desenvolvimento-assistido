# TITULO DO PROJETO

Repositório de documentação e protótipo do **Sistema de Gestão de XXXX**: ....

| Pasta | Conteúdo |
|---|---|
| [`docs/`](docs/) | Requisitos, casos de uso, design system, arquitetura, plano de testes e CI/CD. |
| [`prototipo/`](prototipo/) | Protótipo funcional das telas (React + TypeScript, API mockada com `json-server`) — ver [`prototipo/README.md`](prototipo/README.md) para rodar. |

Este é o repositório de documentação do padrão descrito em [`docs/reference-architecture.md`](docs/reference-architecture.md): quando os repositórios de backend (`bioterio-backend`) e frontend (`bioterio-frontend`) forem criados, este repositório deve ser submodulado como `docs/` dentro dos dois — ver [`docs/cicd/CICD.md`](docs/cicd/CICD.md). Até lá, o único código executável aqui é o protótipo de frontend, que roda isolado (`prototipo/`) com API mockada, sem depender de nenhum backend real.
